# Prerequisites

### Terminal

Clone this repository:

```sh
git clone --depth 1 \
  https://github.com/redhat-na-ssa/sandboxed-containers-aws.git
```

Change into the `sandboxed-containers-aws` directory:

```sh
cd sandboxed-containers-aws
```

Create a scratch directory:

```sh 
mkdir scratch
```

### Cluster
    
Request an environment from the Demo Catalog system.

- [AWS with OpenShift Open Environment](https://catalog.demo.redhat.com/catalog?item=babylon-catalog-prod/sandboxes-gpte.sandbox-ocp.prod&utm_source=webapp&utm_medium=share-link)
    - Activity: `Practice / Enablement`
    - Purpose: `Trying out a technical solution`
    - Region: `us-east-2`
    - OpenShift Version: `4.20`
    - Control Plane Count: `1`
    - Control Plane Instance Type: `m6a.4xlarge`

Login to the cluster with your credentials.

### Compute

This is a single node OpenShift cluster. We do NOT want to run Kata on the control node.

Label and scale the compute nodes:

```sh
MACHINESET=$(oc get machineset -n openshift-machine-api -o jsonpath='{.items[0].metadata.name}')
oc patch machineset $MACHINESET -n openshift-machine-api --type=merge -p '{
  "spec": {
    "replicas": 1,
    "template": {
      "spec": {
        "metadata": {
          "labels": {
            "kata-enabled": true
          }
        }
      }
    }
  }
}'
```

### OpenShift Sandboxed Container Operator

Install the operator:

```sh
oc create -f resources/osc-operator/osc-ns.yaml
oc create -f resources/osc-operator/osc-operatorgroup.yaml
oc create -f resources/osc-operator/osc-sub.yaml
```


