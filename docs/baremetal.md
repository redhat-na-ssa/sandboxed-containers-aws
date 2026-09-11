# Bare Metal

In this mode, you will create a Kata Container on a Bare Metal machine in AWS.

## Create Bare Metal Machines

> NOTE: Please be mindful of the cost of Bare Metal machines and shut these down as soon as you are done testing

View the MachineSets in your cluster:

```sh
oc get machinesets -n openshift-machine-api
```

```text
NAME                                    DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-xxxxx-xxxxx-worker-us-xxxx-xx   1         1         1       1                
```

Make a copy of the existing MachineSet configuration:

```sh
MACHINESET=$(oc get machineset -n openshift-machine-api -o jsonpath='{.items[0].metadata.name}')
oc get machineset $MACHINESET -n openshift-machine-api -o yaml > scratch/baremetal-machineset.yaml
```

Edit `scratch/baremetal-machineset.yaml`

  - [ ] Delete `creationTimestamp`, `generation`, `resourceVersion`, `uid`
  - [ ] Set `.metadata.name` to `baremetal-machineset`
  - [ ] Set `.spec.replicas` to `1`
  - [ ] Set `.spec.selector.matchLabels["machine.openshift.io/cluster-api-machineset"]` to `baremetal-machineset`
  - [ ] Set `.spec.template.metadata.labels["machine.openshift.io/cluster-api-machineset"]` to `baremetal-machineset`
  - [ ] Set `.spec.template.spec.providerSpec.value.instanceType` to `m5.metal`
  - [ ] Add label `.spec.template.metadata.labels["kata-enabled"] to `true`

Optionally, run this a Spot Machine to keep costs down. It should look like this:

```text
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
spec:
  [...]
  template:
    [...]
    spec:
      [...]
      providerSpec:
        value:
          [...]
          instanceType: m5.metal
          spotMarketOptions: {}
```

Create Bare Metal node:

```bash
oc create -f scratch/baremetal-machineset.yaml
```      

Verify:

```bash
oc get machinesets -n openshift-machine-api
```

```text
NAME                                    DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-xxxxx-xxxxx-worker-us-xxxx-xx   1         1         1       1                
baremetal-machineset                    1         1                                
```

## Install Kata Runtime Class

Create custom resource:

```sh
oc create -f resources/osc-config/baremetal-kataconfig.yaml
```

> Note: Compute nodes will reboot

Wait until condition `InProgress` is `False`:

```sh
watch "oc describe kataconfig | sed -n /^Status:/,/^Events/p"
```

Verify runtime class:

```sh
oc get runtimeclass
```

You should see `kata`.

## Test

Deploy a Kata Container workload:

```sh
oc create -f resources/osc-workload/example-ns.yaml
oc create -f resources/osc-workload/example-baremetal-pod.yaml
```

