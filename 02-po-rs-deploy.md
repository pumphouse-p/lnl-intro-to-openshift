# Pods, ReplicaSets, and Deployments

In this document, we will explore some of the core resources used to define the 
applications we want to run and how we want them to run.

## Pods

Any workload that you run on OpenShift is realized as an object known as a pod.
A pod is a collection of containers that can run on a host in the environment. Pods are
requested by clients, and are then scheduled onto host(s).

A pod *may* encapsulate multiple containers, but there's nothing wrong with only having a
single container in a pod, in fact that is very common. The use-case for multi-container 
pods is typically when you have multiple processes that are tightly coupled, in that they
were written with the expectation to run on the same machine. Maybe they share data via 
local filesystem, or communicate with one another over localhost.

While many resources in OpenShift can be created imperatively through either the `oc` 
command or from the OpenShift Console, it's considered a best practice to define your
resources in YAML manifests so that you can store them in a version control system 
(e.g. `git`) and track changes there. So let's go ahead and write a manifest that defines
a pod.

```yaml
$ tee -a kuard-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
    name: kuard-pod
spec:
    containers:
    - name: kuard-app-container
      image: gcr.io/kuar-demo/kuard-amd64:blue
EOF
```

The meat of the YAML manifest is made up of four top level fields.

1. `apiVersion` defines the versioned schema of this representation of an object.
1. `kind` defines the type of object being defined in this block fo the manifest.
1. `metadata` defines, well, metadata of the object, like the name, labels, and namespace.
1. `spec` is used to set properties of the object being defined. The contents of `spec` will be dependent on the `kind` of object.

> ℹ️ You can use the `oc explain <resource_kind>` command to get more info on how to configure the various resources in OpenShift.

Let's now pass the manifest to the cluster so that it can work on standing up our pod.

```bash
$ oc create -f kuard-pod.yaml
```

Watch the pod get provisioned onto the cluster.

```bash
$ oc get pods --watch
```
```
NAME        READY   STATUS              RESTARTS   AGE
kuard-pod   0/1     ContainerCreating   0          4s
kuard-pod   1/1     Running             0          8s
```

> ℹ️ Hit **Ctrl c** to return to your prompt.

Cool. We have ourselves a pod, in a `Running` state (`STATUS` column), with `1/1` containers in good state (`READY` column).

Launch a shell in the container.

```bash
$ oc rsh kuard-pod
~ $ # Note the prompt change - you are now in the container
```

Take a look at what is going on in the container.

```bash
~ $ ps aux
```
```
PID   USER     TIME  COMMAND
    1 nobody    0:00 /kuard
   29 nobody    0:00 /bin/sh
   36 nobody    0:00 ps aux
```

```bash
~ $ ip addr
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if98: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1400 qdisc noqueue state UP 
    link/ether 0a:58:0a:d9:00:56 brd ff:ff:ff:ff:ff:ff
    inet 10.217.0.86/23 brd 10.217.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::e810:53ff:fe0c:94c6/64 scope link 
       valid_lft forever preferred_lft forever
```

```bash
~ $ hostname
```
```
kuard-pod
```

```bash
~ $ netstat -tulpn
```
```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 :::8080                 :::*                    LISTEN      1/kuard
```

The application we deployed exposes an HTTP API. Send a request to verify the application is
responsive.

```bash
~ $ wget -qO- http://localhost:8080/healthy; echo
```
```
ok
```

That's good for now, go ahead and exit out of the container.

```bash
~ $ exit
$
```