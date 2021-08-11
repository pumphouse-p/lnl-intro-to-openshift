# ConfigMaps and Secrets

## Overview

Applications are developed, and later used, for specific purposes. The way an
application is used by one user, can differ substantially from another. A
software developer considers the variables that can change from one user to
another, or maybe one environment to another, and externalizes the 
configuration of the application from the source code. By externalizing the
application's business logic from the configuration, the application becomes
more flexible to accommodating these different needs. Traditionally, these
configuration variables are provided through either configuration files, or
environment variables.

OpenShift supports external configuration through two different types of 
objects: `ConfigMap`s and `Secret`s. The main distinction between the two is
that `ConfigMap` objects are used to store non-confidential data as they do
not provide any secrecy or encryption. `Secret` objects are stored encrypted
and access can be controlled through Role Based Access Control (RBAC) rules.

## Setup

Create a couple of namespaces/projects that we'll use to see how `ConfigMap`s and `Secret`s work.

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/ns.yaml
```

## ConfigMaps

Review the manifest containing the `ConfigMap` objects we will use to provide configuration
data to our workloads:

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/configmaps.yaml; echo
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: httpd-index
    namespace: httpd-dev
data: 
    index.html: |
        <h1>Dev landing page</h1>
---
apiVersion: v1
kind: ConfigMap
metadata:
    name: httpd-index
    namespace: httpd-prod
data:
    index.html: |
        <h1>Prod landing page</h1>
```

The manifest above contains the definition for two `ConfigMap` objects. The first
is named `httpd-index` and is assigned to the `httpd-dev` namespace. The `data` field
is where you supply the underlying configuration values as key/value pairs that you want 
to present into your Pod(s). The second manifest is almost identical to the first, with
the main differences being the namespace and configuration value.

Create the `ConfigMap` objects:

```bash
$ oc create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/configmaps.yaml
```

Review the details of both `ConfigMap`s:

```bash
$ oc -n httpd-dev describe configmap httpd-index
```
```
Name:         httpd-index
Namespace:    httpd-dev
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
<h1>Dev landing page</h1>

Events:  <none>
```

```bash
$ oc -n httpd-prod describe configmap httpd-index
```
```
Name:         httpd-index
Namespace:    httpd-prod
Labels:       <none>
Annotations:  <none>

Data
====
index.html:
----
<h1>Prod landing page</h1>

Events:  <none>
```

Next, we want to launch some workloads that consume those `ConfigMap` objects. Review the
sample Pod manifest we will use:

```bash
$ curl https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/httpd-pod.yaml; echo
```
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: httpd
spec:
    containers:
    - name: httpd
      image: registry.access.redhat.com/ubi8/httpd-24
      ports:
      - name: http
        containerPort: 8080
      volumeMounts:
      - name: docroot
        mountPath: /var/www/html
        readOnly: true
    volumes:
    - name: docroot
      configMap:
        name: httpd-index
```

The interesting bits are under `.spec.volumes` and `.spec.containers.volumeMounts`. The
`.spec.volumes` list initializes a data volume with the contents of the `ConfigMap` in the
same namespace/project named `httpd-index`. The data volume will have files with the names
that match the keys in the `ConfigMap`, and the file contents will be the associated value
found in the `ConfigMap`. In the `.spec.containers.volumeMounts` list, we are mounting that same data volume, named `docroot` into the container at `/var/www/html`, which is the
default document root for Apache (httpd).

Let's launch two Pods, one in the `httpd-dev` namespace and the other in `httpd-prod`.

```bash
$ oc -n httpd-dev create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/httpd-pod.yaml

$ oc -n httpd-prod create -f https://raw.githubusercontent.com/pumphouse-p/lnl-intro-to-openshift/main/manifests/configmaps-secrets/httpd-pod.yaml
```

Let's check everything works as expected. To do this, let's just hop into each Pod and
submit an HTTP request. We should get different responses from each Pod, due to the data
in the `ConfigMap` being customized for each Pod.

First, check the Pod in the httpd-dev namespace.

```bash
$ oc -n httpd-dev rsh httpd
# Prompt change will signify you have entered the container.
sh-4.4$ curl http://localhost:8080 
```
```html
<h1>Dev landing page</h1>
```

Great, exit out of the container.

```bash
sh-4.4$ exit
# Prompt change signifies we're out of the container.
$ 
```

Next, check the Pod in the httpd-prod namespace.

```bash
$ oc -n httpd-prod rsh httpd
# Prompt change will signify you have entered the container.
sh-4.4$ curl http://localhost:8080 
```
```html
<h1>Prod landing page</h1>
```

## Secrets