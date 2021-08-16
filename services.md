# Services

## Overview

A Service object proxies network traffic destined to a name and port to a set of Pods that
should actually handle the requests. Services will also load-balance client connections
across your Pods so that you can distribute the load and scale your apps dynamically. 

Create a Deployment. We'll configure a Service in just a bit that will be used to access the
Pods of this Deployment.

Before you begin, apply the manifest with the command below to set up some prerequisites we
need for this exercise.

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/services/prereqs.yaml
```

Switch to the `omc` project:

```bash
$ oc project omc
```

Review the manifest for the workload we will deploy:

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/services/omc-deploy.yaml; echo
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: omc
  namespace: omc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: omc
  template:
    metadata:
      name: omc-pod
      labels:
        app: omc
    spec:
      serviceAccountName: omc-sa
      containers:
      - name: omc-app-container
        image: quay.io/deparris/openshift-mini-console:latest
        ports:
        - name: http
          containerPort: 5000
```

Create the Deployment:

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/services/omc-deploy.yaml
```

The main area to focus in on is the block for `ports` at the bottom of the output. The 
application listens on port 5000 in the container, and we are going to reference this
when we define our Service, as we will want the Service to route traffic into the containers
on this port.

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/services/omc-svc.yaml; echo
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: openshift-mini-console
  namespace: omc
spec:
  selector:
    app: omc
  ports:
  - name: http
    targetPort: http
    port: 80
```

Create the Service:

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/services/omc-svc.yaml
```

Review the Pods, making note of the IP addresses assigned to each:

```bash
$ oc get pods -o wide
```
```
NAME                   READY   STATUS    RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
omc-548ff44cc6-7zrlm   1/1     Running   0          4m28s   10.217.0.96   crc-pkjt4-master-0   <none>           <none>
omc-548ff44cc6-ndpsg   1/1     Running   0          4m28s   10.217.0.95   crc-pkjt4-master-0   <none>           <none>
omc-548ff44cc6-qmklr   1/1     Running   0          4m28s   10.217.0.94   crc-pkjt4-master-0   <none>           <none>
```

> ℹ️ The IP addresses will likely be different in your environment.

You should see the Pod IP addresses listed as `Endpoints` when reviewing the details of
the Service:

```bash
$ oc describe service openshift-mini-console | grep '^Endpoints:'
```
```
Endpoints:         10.217.0.94:5000,10.217.0.95:5000,10.217.0.96:5000
```

Let's test this out. The Service will be reachable from inside the cluster, so first
launch a Pod that we can test connectivity from:

```bash
$ oc run -it --rm --image=registry.access.redhat.com/ubi8-minimal -- bash
```

There will be a DNS entry that matches the name of the Service that we can use to get to
the backend Pods. The name we assigned to the Service was `openshift-mini-console`, so we
can use that:

```bash
[root@client /] while true; do curl http://openshift-mini-console/pod/ipaddress; echo; sleep 2; done
```

The `curl` command executing in the `while` loop above is sending an HTTP request through
the Service we created to the `/pod/ipaddress` endpoint. This is an endpoint exposed by 
the application running in our Pods that will return the IP address observed within the
container.

Because we are hitting each Pod through the Service, the Service is distributing the
requests across all three Pods we have deployed. This means that you should see three
different IP addresses in the responses you receive in your terminal, showing that the
Service is load balancing traffic across all the backend Pods.

If you scale the number of Pods, the Service will automatically pick up the new Pods
and add them to the list of Endpoints to route traffic to. Try this out by opening a 
new Terminal and running the following command:

```bash
$ oc scale deploy/omc --replicas=5
```

Back in the Terminal tab where you're running the `while` loop, you should begin to see
two new IP addresses show up in the output.

Once you're satisfied, stop the `while` loop by hitting `Ctrl c` on your keyboard, and then
type `exit` to return back to your client system.