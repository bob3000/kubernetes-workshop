# Kubernetes Workshop

## Table of contents

- [Kubernetes Workshop](#kubernetes-workshop)
  - [Table of contents](#table-of-contents)
  - [What is Kubernetes](#what-is-kubernetes)
  - [Benefits of Kubernetes](#benefits-of-kubernetes)
  - [Kubernetes components](#kubernetes-components)
    - [Control Pane components](#control-pane-components)
    - [Node components](#node-components)
    - [Addons](#addons)
  - [Kubernetes objects](#kubernetes-objects)
  - [Hands on: Set up an almost production ready Ghost blogging system](#hands-on-set-up-an-almost-production-ready-ghost-blogging-system)
    - [Kubectl contexts](#kubectl-contexts)
    - [Creating a _Namespace_](#creating-a-namespace)
    - [A _Deployment_ creates your pods](#a-deployment-creates-your-pods)
    - [Make your _Pods_ available via _Services_](#make-your-pods-available-via-services)
    - [What's wrong with Ghost?](#whats-wrong-with-ghost)
    - [Tailing the log](#tailing-the-log)
    - [Running Nginx](#running-nginx)
    - [Accessing a _Pod_ via port forwarding](#accessing-a-pod-via-port-forwarding)
    - [How to inject a file](#how-to-inject-a-file)
    - [Scaling up](#scaling-up)
    - [Getting persistent](#getting-persistent)
    - [Updating a container image](#updating-a-container-image)
    - [Cleaning up](#cleaning-up)
  - [Further reading](#further-reading)

## What is Kubernetes

Kubernetes is an open source orchestrator for deploying and managing
containerized applications and workloads.

- the resources of many servers nodes are federated into a resource pool
- workloads are dynamically scheduled depending on available resources
  and defined constraints
- Kubernetes provides several other vital mechanisms needed to make
  applications running on different nodes play together transparently

## Benefits of Kubernetes

- ease of use, simple zero downtime deployments
- immutability, applications are replaced not mutated
- declarative configuration in yaml files
- health checks / self healing systems
- simple horizontal scaling

## Kubernetes components

![kubernetes components](https://d33wubrfki0l68.cloudfront.net/7016517375d10c702489167e704dcb99e570df85/7bb53/images/docs/components-of-kubernetes.png "kubernetes components")

### Control Pane components

- _kube-api-server_ receives commands from client apps (e.g. `kubectl`)
- _etcd_ keeps Control Pane Nodes in sync
- _cloud-controller-manager_ communicates with Cloud APIs (e.g. AWS)
- _kube-controller-manager_ coordinates nodes and application replication
- _kube-scheduler_ distributes pods evenly on worker nodes

### Node components

- _kubelet_ turns object specifications into Kubernetes objects
- _kube-proxy_ makes pods reachable via network

### Addons

- _coredns_ provides service discovery mechanisms via DNS

## Kubernetes objects

Kubernetes objects are persistent entities in the Kubernetes system.
Kubernetes uses these entities to represent the state of your cluster.
Objects are specified in yaml configuration files which can be turned
for example into running applications, cloud resources, cluster behavior
and much more. There are numerous Kubernetes objects. Some important
examples are:

- _Pods_: A Pod (as in a pod of whales or pea pod) is a group of one or
  more containers (such as Docker containers), with shared storage/network,and a specification for how to run the containers
- _ReplicaSet_ maintain a stable set of replica Pods running at any given
  time
- _Deployments_ provide declarative updates for Pods and ReplicaSets such
  as rolling deployments or rollbacks
- _Services_ are an abstract way to expose an application running on a
  set of Pods as a network service

## Hands on: Set up an almost production ready Ghost blogging system

In this section we're going to set up a Ghost blogging system consisting
of multiple Kubernetes objects such as _Deployments_, _Pods_ and
_Services_.

The object specification in yaml are already prepared for you and are
stripped down to the essentials so it will hopefully look less confusing.

The setup will consist of three components: the ghost blogging system
itself, a MySQL database and an Nginx reverse proxy.

### Kubectl contexts

Before we start doing anything we need to make sure we are operating on
the correct desired environment.

Kubectl contexts refer to a bunch of configurations and credentials which
belong to a certain Kubernetes installation. Therefore switching to a
certain context means you're switching the Kubernetes installation you
are working on (presuming you have several installation configured).

What context are we operating? To look up the current context and the
available context use these commands.

```sh
kubectl config current-context
kubectl config get-contexts
```

To set the desired context do

```sh
kubectl config use-context <mycontext>
```

### Creating a _Namespace_

Namespaces allow to logically separate Kubernetes objects into different
concerns.

We create a namespace like this

```sh
kubectl create namespace ghost
```

Namespaces either can be be set temporarily with the `-n` switch like
`kubectl -n ghost ...` or permanently like so

```sh
kubectl config use-context <mycontext> --namespace ghost
```

To see the currently selected namespace do

```sh
kubectl config view --minify | grep namespace:
```

### A _Deployment_ creates your pods

Kubernetes' units of execution are called _Pods_. A _Pods_ is an object type
that runs one or more containers (usually it should be just one).

_Pods_ on their own are mostly used for debugging purpose or quick manual
interventions. For reliably running applications you should use _Deployments_
which will create the Pods for you from a template and will take care of
keeping the pods running.

Have a look at the file [resources/mysql-deployment.yaml](resources/mysql-deployment.yaml)
and read through it. Most of the statements will actually be even self-explanatory
if you recall what we're up to. Note for instance the `env` section which provides
the pod the necessary configuration to make it run. Be aware that passwords and
other secrets are not supposed to be written in plain text. There is a Kube object
type called _Secret_ which is not in scope of this tutorial though.

As the first step we want to run a mysql database. Do `cd resources` and follow
along the commands. In order to send a yaml configuration to the Kubernetes API
we use the command `kubectl apply -f <myconfig>.yaml`.

Apply the configuration for the mysql deployment

```sh
kubectl apply -f mysql-deployment.yaml
```

and see what it's doing

```sh
kubectl get pods --watch
```

Detailed information about the deployment can be obtained with the
`kubectl describe ..` command. Try

```sh
kubectl describe deployment mysql
```

No worries if you do not understand every single detail yet. What you might find
helpful for the moment is looking at the last few lines in the _Events_ section
which will display different kinds of errors if anything goes wrong. Right now
you should see something like

```
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  7m38s  deployment-controller  Scaled up replica set mysql-58b5575c56 to 1
```

Looks good? Let's connect our database to the network.

### Make your _Pods_ available via _Services_

In Kubernetes _Pods_ are not supposed to be contacted directly. The whole
topic of network connectivity is encapsulated into an object type called
_Service_.

To be absolutely clear about it, every _Pod_ which is supposed to be
available via net work - may it privately / internally or publicly /
externally - must have a _Service_ object associated thus we need a service
for all our _Deployments_.

Let's start off with MySQL.

```sh
kubectl apply -f mysql-service.yaml
```

and then check out it's internal

```sh
kubectl describe service mysql
```

Looks almost good except this `Endpoints:         <none>`. Apparently
it has no _Pods_ listening to the service but how is it even supposed
to know to which _Pods_ it should bind to? Here is the answer.

Every _Service_ uses a label selector to find all the pods which belong
to a certain type of service. Have a look at the file
[resources/mysql-service](resources/mysql-service.yaml) and
compare the `selector` statement with the labels set in the file
[resources/mysql-service](resources/mysql-deployment.yaml).

After you corrected the error apply the configuration again.

```sh
kubectl apply -f mysql-service.yaml
kubectl describe service mysql
```

Now it should look similar to `Endpoints:         172.17.0.4:3306`.

### What's wrong with Ghost?

Let's move on to the ghost blog. Read through
[resources/ghost-deployment.yaml](resources/ghost-deployment.yaml) and apply it with

```sh
kubectl apply -f ghost-deployment.yaml
```

and see what it's doing

```sh
kubectl get pods --watch
```

Not great! As we see the _Pod_ quits with an error until it stays in a
_CrashLoopBackOff_ state. We already learned a trick on obtaining more information.
We describe the _Pos_ selected by it's label.

```sh
kubectl describe pod -l=app=ghost
```

That doesn't look very helpful though. What we see here is telling us that Kubernetes
basically has no Problems with our configuration which points us to a problem with
the application configuration.

First thing we can do in such a case is to have a look inside the application logs.

### Tailing the log

```sh
kubectl logs -f deploy/ghost
```

It says something about a databse error. That is suspicious! Have a look at the two
deployment configurations 
[resources/ghost-deployment.yaml](resources/ghost-deployment.yaml) and
[resources/mysql-deployment.yaml](resources/mysql-deployment.yaml) and see if you
can spot the error **without immediately telling everybody** so everyone gets his
opportunity the get familiar with the syntax of the files.

Fix the error and apply the configuration again.

```sh
kubectl apply -f ghost-deployment.yaml
```

and see how we're doing this time.

```sh
kubectl get pods --watch
```

### Running Nginx

Next thing to do is to run Nginx.

```sh
kubectl apply -f nginx-deployment.yaml
kubectl get pods
```

That was almost to too easy. We have all our component running. Let's
wire everything together.

Now you can set up the _Services_ for Ghost and Nginx.

```sh
kubectl apply -f ghost-service.yaml
kubectl apply -f nginx-service.yaml
```

Maybe you should check if the services are configured correctly as we
discussed above.

### Accessing a _Pod_ via port forwarding

All _Pods_ are now available via network but how do we actually connect
to our blog? The way you would do it in a production grade setup is out
of scope of this very tutorial but for now there is a neat trick which is
also very handy for debugging purpose.

The command `kubectl port-forward` let's you access all _Pods_ via a
tunnel to your computer. This way are able to connect to our Nginx.

Try that

```sh
kubectl port-forward deploy/nginx 8080:80
```

and navigate to [http://localhost:8080](http://localhost:8080).

Warm greetings from Nginx! Nice, but not exactly what we wanted. What
we see here is just a plain vanilla Nginx without any configuration.
To get it configured you could either build a custom Nginx Docker image
with your configuration included or you can just let Kubernetes inject
the configuration file for you.

### How to inject a file

In Kubernetes files can be mounted like volumes after the file contents
have been placed inside another Kubernetes object type called _ConfigMap_.

Read through the file
[resources/nginx-configmap.yaml](resources/nginx-configmap.yaml) and
apply it

```sh
kubectl apply -f nginx-configmap.yaml
```

Bonus questions: Do you maybe have an idea on how to list _ConfigMaps_
deployed in our _Namespace_?

Next we need to adjust the Nginx _Deployment_ so the configuration is
actually being mounted. Add the following lines to the end of 
[resources/nginx-deployment.yaml](resources/nginx-deployment.yaml).

```yaml
          volumeMounts:
            - name: nginx
              subPath: default.conf
              mountPath: /etc/nginx/conf.d/default.conf
      volumes:
        - name: nginx
          configMap:
            name: nginx
```

and update the deployment

```sh
kubectl apply -f nginx-deployment.yaml
```

Finally have another look at [http://localhost:8080](http://localhost:8080).
If it's not working you might need to restart you `kubectl port-forward`
command. Also you might have to delete your browser cache in order to see
the updated content.

Let's set up the blog at [http://localhost:8080/ghost](http://localhost:8080/ghost).
Fill in the form and enjoy the view of the admin panel for a moment and then prepare
for the next 1000 concurrent users. The set up of the blog is necessary for the
next steps to come.

### Scaling up

If our blog gets overwhelmed by the high numbers
of users we can user horizontal scaling to leverage our computation powers.

```sh
kubectl scale --replicas 4 deploy/ghost
kubectl get pods
```

Looks good? Scale down the _Pods_ to `0` for our next experiment

```sh
kubectl scale --replicas 0 deploy/ghost
kubectl get pods
```

### Getting persistent

Let's make some trouble and delete the database _Pod_.

```sh
kubectl delete $(kubectl get pod -l=app=mysql -o name)
kubectl get pods --watch
```

Why is the _Pod_ still there if we just deleted it? It's still there because the
_Deployment_ object which spawned the _Pod_ for us is doing a good job - it replaces
a _Pod_ whenever it fails.

Now you learned to difference between scaling a _Deployment_ and deleting a _Pod_.
If you just fire up a pod yourself without a _Deployment_ it will not be respawned
automatically.

Let's bring back our Ghost _Pod_ and see how our Blog is doing.

```sh
kubectl scale --replicas 1 deploy/ghost
kubectl get pods
```

Look again at 
[http://localhost:8080/ghost](http://localhost:8080/ghost)

All is lost! When we killed the Database it sent it's data into oblivion. Containers
are spawned from immutable images and all data saved inside a container is completely
ephemeral. In order to make data persistent we have to mount a volume!

Add the following lines to
[resources/mysql-deployment.yaml](resources/mysql-deployment.yaml)

```yaml
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql
      volumes:
        - name: mysql
          emptyDir: {}
```

That should look familiar to you from the Nginx configuration. The main difference is
the very last line which is the volume type. Before we declared a volume from a
_ConfigMap_ and now it says `emptyDir: {}` which means that we're just mounting a
folder from the host machine into the container. This is something you won't see very
often in production but it's good enough for our example.

Kubernetes supports all kinds of volumes from different persistency mechanisms
and cloud providers. We can't go into detail here but we can get familiar with the
basic syntax of mounting a volume into a _Pod_. A complete list of supported volume
types can be found at
[https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/).

Finally we have to apply our changes

```sh
kubectl apply -f mysql-deployment.yaml
kubectl get pods --watch
```

### Updating a container image

Guess what, a new version of Ghost was just released as we speak! Time to update.
Edit [resources/ghost-deployment.yaml](resources/ghost-deployment.yaml) and change
the line `- image: ghost` to `- image: ghost:3.22` and redeploy. You should already
know what to do by now. if not review the section where we initially deployed Ghost.

Before we tear everything down feel free to do further experiments and check if
this time the data is still there after you deleted the MySQL pod.

### Cleaning up

When you're done you can get rid of the whole setup with a single command.

```sh
kubectl delete -f .
```

And since our configuration is working now you can also easily bring everything
back with a single command.

```sh
kubectl apply -f .
```

Stay tuned for more happy days on Kubernetes!

## Further reading

- [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [Benefits of Kubernetes](https://medium.com/platformer-blog/benefits-of-kubernetes-e6d5de39bc48)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Using kubectl to generate Kubernetes YAML](https://www.liammoat.com/blog/2019/using-kubectl-to-generate-kubernetes-yaml)
