# (Not Recommended) Peer Pod

In this mode, a Kata Container is a full VM (using a custom Pod VM image) running next to the OpenShift cluster in the same VPC.

Every Peer Pod is allocated its own VM. The Peer Pod that you see in the cluster has a shim to communicate with a kata-agent on the VM.

It is generally recommended to use Bare Metal mode for efficiency instead of Peer Pod.

You will need access to the cluster's AWS environment with the `awscli` to configure this mode.

> NOTE: The SNO cluster provisioned in the demo system does NOT use STS for installation. If you used STS for installation OR you deployed ROSA, please follow these [steps](https://docs.redhat.com/en/documentation/openshift_sandboxed_containers/1.13/html/deploying_openshift_sandboxed_containers_on_aws/install-osc-overview_aws-osc#sts-authentication_aws-osc).

## Enable Ports

Retrieve instance ID:

```sh
INSTANCE_ID=$(oc get nodes -l 'node-role.kubernetes.io/worker' \
  -o jsonpath='{.items[0].spec.providerID}' | sed 's#[^ ]*/##g')
```

Retrieve AWS region:

```sh
AWS_REGION=$(oc get infrastructure/cluster -o jsonpath='{.status.platformStatus.aws.region}')
```

Retrieve Security Groups:

```sh
AWS_SG_IDS=($(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[*].Instances[*].SecurityGroups[*].GroupId' \
  --output text --region $AWS_REGION))
```

Authorize peer pod shim to communicate with kata-agent:

> NOTE: If you are running ROSA, create clusters with additional security groups and modify those instead of the default security groups.

```sh
for AWS_SG_ID in "${AWS_SG_IDS[@]}"; do \
  aws ec2 authorize-security-group-ingress --group-id $AWS_SG_ID --protocol tcp --port 15150 --source-group $AWS_SG_ID --region $AWS_REGION; \
  aws ec2 authorize-security-group-ingress --group-id $AWS_SG_ID --protocol tcp --port 9000 --source-group $AWS_SG_ID --region $AWS_REGION; \
done
```

## Peer Pod Config Map

The Peer Pod Config Map requires you to define the AWS region, VPC, subnet, and security group IDs. 

Retrieve the VPC:

```sh
AWS_VPC_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[*].Instances[*].VpcId' --region ${AWS_REGION} \
    --output text) && echo "AWS_VPC_ID: \"$AWS_VPC_ID\""
```

Retrieve the subnet:

```sh
AWS_SUBNET_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[*].Instances[*].SubnetId' --region ${AWS_REGION} \
    --output text) && echo "AWS_SUBNET_ID: \"$AWS_SUBNET_ID\""
```

Create the Peer Pod Config Map:

```sh
cat << EOF | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: peer-pods-cm
  namespace: openshift-sandboxed-containers-operator
data:
  CLOUD_PROVIDER: "aws"
  VXLAN_PORT: "9000"
  PROXY_TIMEOUT: "8m"
  PODVM_INSTANCE_TYPE: "t3.medium"
  PODVM_INSTANCE_TYPES: "t2.small,t2.medium,t3.large"
  PODVM_AMI_ID: ""
  AWS_REGION: ${AWS_REGION}
  AWS_SUBNET_ID: ${AWS_SUBNET_ID}
  AWS_VPC_ID: ${AWS_VPC_ID}
  AWS_SG_IDS: ${AWS_SG_IDS}
  TAGS: ""
  PEERPODS_LIMIT_PER_NODE: "10"
  ROOT_VOLUME_SIZE: "6"
  DISABLECVM: "true"
EOF
```

## Install Kata Runtime Class

Create custom resource:

```sh
oc create -f resources/osc-config/peerpod-kataconfig.yaml
```

> Note: Compute nodes will reboot

Wait until condition `InProgress` is `False`:

```sh
watch "oc describe kataconfig | sed -n /^Status:/,/^Events/p"
```

Verify runtime class

```sh
oc get runtimeclass
```

You should see `kata-remote`.

Verify pod VM image:

```sh
PODVM_AMI_ID=$(oc get configmap peer-pods-cm -n openshift-sandboxed-containers-operator -o jsonpath='{.data.PODVM_AMI_ID}{"\n"}')
echo $PODVM_AMI_ID
```

Verify the AMI in AWS:

```sh
aws ec2 describe-images --image-ids $PODVM_AMI_ID
```

## Test

Deploy a Kata Container workload:

```sh
oc create -f resources/osc-workload/example-ns.yaml
oc create -f resources/osc-workload/example-peer-pod.yaml
```

Verify Peer Pod is running:

```sh
oc get pods -n test
```

Verify the VM associated with the Peer Pod:

```sh
aws ec2 describe-instances \
  --filters "Name=image-id,Values=$PODVM_AMI_ID"
```

