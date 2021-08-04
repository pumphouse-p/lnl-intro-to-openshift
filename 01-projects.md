# Projects

A Project is a unit of collaboration and isolation in OpenShift. From a collaboration
standpoint, a project has one or more members. Members can have different roles within a
Project, where some members may be granted full read/write access to the Project, and
others granted read-only access.

From an isolation standpoint, many different objects in OpenShift exist within a Project.
Take a Pod, for example. A Pod resides in a Project at creation time. That Pod can access
other objects in that Project, a Secret, for example. 

Projects are extensions of Namespace objects from Kubernetes, which is worth knowing as
you will see references to Namespaces throughout OpenShift. In this context, you can
consider the two kinds of objects interchangeable. 

Use the `oc` command to determine your current OpenShift user.

```bash
$ oc whoami
```

You should get output similar to the following:

```
admin
```

There is another user in our environment that we have access to. Let's switch over to them.

```bash
$ oc login -u developer -p developer
```

You should get output similar to the following:

```
Login successful.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>
```

Perfect! As the output indicates, our `developer` user does not have access to any projects, so let's go ahead and create one.

```bash
$ oc new-project myproject
```

```
Now using project "myproject" on server "https://openshift:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname
```

So after creating a new project, we are automatically working within that project. Because
we just created the project, it should be relatively empty. Let's explore what we have.

List out pods:

```bash
$ oc get pods
```
```
No resources found in myproject namespace.
```

List out services:

```bash
$ oc get services
```
```
No resources found in myproject namespace.
```

List out configmaps:

```bash
$ oc get configmaps
```
```
NAME               DATA   AGE
kube-root-ca.crt   1      15m
```

Ah, it looks like we have a configmap, where did that come from? Well, there are some default objects that will be created in each project. The configmap we see above contains the certificate authority bundle to verify connections with the Kubernetes API.

Let's create a configmap of our own just to see how that looks.

```bash
$ oc create configmap myconfigmap --from-literal=msg=myproject-myconfigmap
```

Now we should see an additional configmap when we list them out.

```bash
$ oc get configmaps
```
```
NAME               DATA   AGE
kube-root-ca.crt   1      21m
myconfigmap        1      3s
```

We can see the content of the configmap using the `oc describe` command.

```bash
$ oc describe configmap myconfigmap
```
```
Name:         myconfigmap
Namespace:    myproject
Labels:       <none>
Annotations:  <none>

Data
====
msg:
----
myproject-configmap
```

Next, lets create a second project and a configmap to see how that works.

```bash
$ oc new-project myotherproject
$ oc create configmap myotherconfigmap --from-literal=msg=myotherproject-myotherconfigmap
```

List out your configmaps:

```bash
$ oc get configmaps
```
```
NAME               DATA   AGE
kube-root-ca.crt   1      2m23s
myotherconfigmap   1      2m6s
```

Just like in the `myproject` project, we see the `kube-root-ca.crt` configmap, along with the configmap we just created named `myotherconfigmap`. Notice that we don't see the configmap we created earlier, since it resides in a different namespace.

View details of our newly created configmap:

```bash
$ oc describe configmap myotherconfigmap
```
```
Name:         myotherconfigmap
Namespace:    myotherproject
Labels:       <none>
Annotations:  <none>

Data
====
msg:
----
myotherproject-myotherconfigmap
```

To change the project you are actively working in, you can use the `oc project` command.

```bash
$ oc project myproject
```

If you list out configmaps again, you should see the configmap we initially created named `myconfigmap`.

```bash
$ oc get configmaps
```
```
NAME               DATA   AGE
kube-root-ca.crt   1      33m
myconfigmap        1      11m
```

Go ahead and delete the `myotherproject` project.

```bash
$ oc delete project myotherproject
```

> :warning: **Deleting a project deletes all resources within, be careful!**