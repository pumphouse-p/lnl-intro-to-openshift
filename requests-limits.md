# Resource Requests and Limits

## Overview

When you have a Pod to run on OpenShift, you have the ability to specify the amount
of resources such as CPU and RAM the container(s) in the Pod require(s) to
function. When the Pod is being scheduled to run on a node, these resources are
taken into account so that the Pod is assigned to a node that can accommodate the
resources specified.

Resources are specified as requests and limits. A resource request specifies the
bare minimum amount of resources that the container needs to function. The 
scheduler will use the resource request given and select a node that has at least
that amount of the resource available, or unallocated to other workloads. As an
example, if a resource request of 512MiB of RAM is given, the scheduler will place
that Pod on a node that has at least 512MiB of RAM that has not already been 
allocated to other workloads. When that Pod lands on the scheduled node, those
resources are now reserved for that Pod and will come off the total amount of
available RAM that can be used by other Pods.

A resource limit gives a ceiling, or maximum amount of the given resource that the
container can consume on a node. This is useful as it acts as a safety net, so that
a single container can not run wild and starve other containers on the node from 
the resources they need to run. As an example, if a resource limit of 512MiB of RAM
is given, the container can run happily as long as it has less than 512MiB of RAM
allocated. Should a container try to allocate **more** than the 512MiB limit set,
the container will be killed with an Out Of Memory (OOM) error.

## Launch a `BestEffort` Pod

Launch a Pod with no resource requests or limits.

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/ns.yaml
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/best-effort.yaml
```

Modify your command line context to use the project created above.

```bash
$ oc project requests-limits
```

Review the Quality of Service (QoS) class assigned to the Pod that was just created.

```bash
$ oc get pod best-effort -o jsonpath='{.status.qosClass}{"\n"}'
```
```
BestEffort
```

Pods are assigned a QoS class based off of resource requests and limits. Another way to
phrase that, is that you do not directly assign a QoS to a Pod. The QoS class will be 
determined based on the presence (or lack thereof) of requests and limits.

When a Pod is assigned the `BestEffort` QoS class, this indicates that no resource requests
or limits where given to the Pod.

## Launch a `Burstable` Pod

Review the manifest for a Pod with resource requests set:

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/burstable.yaml; echo
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable
  namespace: requests-limits
spec:
  containers:
  - name: ubi
    image: registry.access.redhat.com/ubi8/ubi-minimal
    command: 
    - "sleep"
    args:
    - "infinity"
    resources:
      requests:
        cpu: "0.1"
        memory: "64Mi"
```

Launch the Pod:

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/burstable.yaml
```

Review the QoS class assigned to the Pod.

```bash
$ oc get pod burstable -o jsonpath='{.status.qosClass}{"\n"}'
```
```
Burstable
```

Pods will be assigned a Burstable QoS class when they meet the following characteristics:

1. At least one container in a Pod has a memory request and limit set, where the limit is larger than the request (or not specified).
1. At least one container in a Pod has a CPU request and limit set, where the limit is larger than the request (or not specified).

Review the resource requests and limits set on the Pod.

```bash
$ oc get pod burstable -o jsonpath='{.spec.containers[*].resources}{"\n"}'
```
```json
{"requests":{"cpu":"100m","memory":"64Mi"}}
```

The lack of `limits` in the output indicates no resource limits where set on the container
in this Pod. What we can take away from this is that the Pod was scheduled to a node that 
had at least `100m` of CPU and `64Mi` unallocated to other workloads. Because no limit was
set, the container can burst up and use as much CPU and memory it needs when required.

## Launch a `Guaranteed` Pod

Review the manifest for a Pod with resource requests and limits set:

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/guaranteed.yaml; echo
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
  namespace: requests-limits
spec:
  containers:
  - name: ubi
    image: registry.access.redhat.com/ubi8/ubi-minimal
    command: 
    - "sleep"
    args:
    - "infinity"
    resources:
      requests:
        cpu: "0.1"
        memory: "64Mi"
      limits:
        cpu: "0.1"
        memory: "64Mi"
```

Launch the Pod:

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/requests-limits/guaranteed.yaml
```

Review the QoS class assigned to the Pod.

```bash
$ oc get pod guaranteed -o jsonpath='{.status.qosClass}{"\n"}'
```
```
Guaranteed
```

Here we see the Pod we just launched was assigned the QoS class of Guaranteed. Pods are 
assigned a QoS class of Guaranteed when they meet each of the following characteristics:

1. Every container in the Pod must have memory limits and requests set to the same value.
1. Every container in the Pod must have CPU limits and requests set to the same value.

> ℹ️ When memory and CPU limits are set, but requests are not, OpenShift will automatically set requests equal to the limits.

Review the resource requests and limits set on the Pod.

```bash
$ oc get pod guaranteed -o jsonpath='{.spec.containers[*].resources}{"\n"}'
```
```json
{"limits":{"cpu":"100m","memory":"64Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}
```

As we can see in the output above, the CPU and memory limits are equal to the requests
which explains why the Pod was assigned a class of Guaranteed.

## Wrap Up

Resource requests and limits are important for a number of different reasons. Efficient use
of your cluster resources is one worth noting. If you were to remove OpenShift from the 
equation entirely and run your application on bare-metal or in a virtual machine, you know
there are costs associated with hosting that application. If you use a public cloud, you are
being billed hourly for the usage incurred on that public cloud. You would typically want
to be mindful of that and not grossly oversize your resources if you don't need them. It's
no different in OpenShift. While you get more efficient bin packing in OpenShift, it means
nothing when you don't have resource requests and limits set on your Pods.

Another point to mention is the performance of your application. Resource requests are a
great way to ensure your application has the minimum amount of resources to function
correctly. Resource limits are useful as they give you a way to allow your application to
burst up to a specified threshold and still be a good neighbor to your fellow cluster users.

As we saw in this exercise, setting resource requests and limits or not, your Pods will 
automatically be assigned a QoS class. These classes are important, as they are used behind
the scenes when Pod scheduling and eviction takes place. 

From a scheduling perspective, `BestEffort` Pods will not be assigned to a node if all of 
your nodes are under disk and memory pressure - meaning they have lower amounts of
available disk and RAM. `Burstable` and `Guaranteed` Pods will not be assigned to a node
when all of your nodes are under disk pressure.

When you have a node that is running low on resources, Pods will start being evicted off of
that node and placed somewhere else in the cluster. A few different factors come into play
when it comes to which Pods are chosen for eviction, and one of those factors is the QoS
class assigned to Pods. `BestEffort` and `Burstable` Pods that are using resources which
exceeds their requests will be evicted first. `Burstable` and `Guaranteed` Pods that are
using less than their requests are the last to be evicted, should it come to that.