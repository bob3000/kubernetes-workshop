# Kubernetes workshop part II

This is part two of the introduction to Kubernetes. Part one can be found
[here](README.md).

## Table of contents

- [Kubernetes workshop part II](#kubernetes-workshop-part-ii)
  - [Table of contents](#table-of-contents)
  - [What is Helm?](#what-is-helm)
  - [Why Helm?](#why-helm)
    - [Helm high level goals](#helm-high-level-goals)
    - [Helm functionality](#helm-functionality)
    - [Installing Helm](#installing-helm)
  - [Upgrading our Ghost configuration to a Helm Chart](#upgrading-our-ghost-configuration-to-a-helm-chart)
    - [Setting the right context](#setting-the-right-context)
    - [Installing MySQL from a Helm repository](#installing-mysql-from-a-helm-repository)
    - [Your own _Ghost_ chart from scratch](#your-own-ghost-chart-from-scratch)
      - [Scaffolding a Helm Chart](#scaffolding-a-helm-chart)
      - [The `Chart.yaml` files](#the-chartyaml-files)
      - [The `.helmignore` files](#the-helmignore-files)
      - [The `templates/NOTES.txt` file](#the-templatesnotestxt-file)
      - [The `values.yaml` files](#the-valuesyaml-files)
      - [Configuring the Chart](#configuring-the-chart)
      - [The first test run](#the-first-test-run)
      - [Making the environment more flexible](#making-the-environment-more-flexible)
      - [Including MySQL as a requirement](#including-mysql-as-a-requirement)
      - [Exposing the _Service_ using an _Ingress_](#exposing-the-service-using-an-ingress)
        - [Deploying the _Ingress Controller_](#deploying-the-ingress-controller)
        - [Deploying the _Ingress_](#deploying-the-ingress)
        - [Sending the correct `Host` header](#sending-the-correct-host-header)
      - [Cleaning up](#cleaning-up)
  - [Further reading](#further-reading)

## What is Helm?

Helm is a package manger for Kubernetes - Helm Packages are called
_Charts_ - Helm _Charts_ help you define, install, and upgrade even the
most complex Kubernetes application.

![Helm banner](https://cdn1.vogel.de/unsafe/fit-in/1000x0/images.vogel.de/vogelonline/bdb/1567000/1567093/original.jpg)

## Why Helm?

### Helm high level goals

- **Manage Complexity**: Charts describe even the most complex apps, provide repeatable application installation, and serve as a single point of authority.
- **Easy Updates**: Take the pain out of updates with in-place upgrades and custom hooks.
- **Simple Sharing**: Charts are easy to version, share, and host on public or private servers.
- **Rollbacks**: Use helm rollback to roll back to an older version of a release with ease.

### Helm functionality

- Create new charts from scratch
- Package charts into chart archive (tgz) files
- Interact with chart repositories where charts are stored
- Install and uninstall charts into an existing Kubernetes cluster
- Manage the release cycle of charts that have been installed with Helm

### Installing Helm

Refer to installation instructions on the Helm website

[https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

## Upgrading our Ghost configuration to a Helm Chart

### Setting the right context

Create a new namespace and make sure to set the namespace as the default
in your context.

```sh
kubectl create namespace ghost
kubectl config set-context --current --namespace ghost
```

### Installing MySQL from a Helm repository

Just like `kubectl` Helm works with several sub commands. The first one of
our interest is `helm repo add` which will allow us to add a repository
where we can get our MySQL chart from.

```sh
helm repo list
```

If Helm was just installed no repository should show up. Let's add one and
get the repository's index.

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
```

This is the chart we want to install
[https://github.com/helm/charts/tree/master/stable/mysql](https://github.com/helm/charts/tree/master/stable/mysql). Have a brief look!

Go ahead and install

```sh
helm install \
  --set mysqlRootPassword=secretpassword,mysqlUser=ghost,mysqlPassword=secret,mysqlDatabase=ghost \
    ghost-mysql stable/mysql
```

then look what happens

```sh
kubectl get pods
```

Wonderful! A running MySQL instance without writing a single line of
configuration. Noting special about this MySQL container - just a lot less
effort then last time when we had to write the Kube configuration
ourselves.

For a list of installed Helm _Charts_ do

```sh
helm list
```

Note that the concept of namespaces applies to the `helm` command just
like it applies to the `kubectl` command. We are already in the desired
namespace `ghost` but if you want to see installed charts from a different
namespace you have to specify it with the `-n` switch (e.g.
`helm -n kube-system list`) or use the `--all-namespaces` option to see
all installed _Charts_.

Just like `kubectl` the `helm` command also has a build in help system.
If you want to get a certain command explained or if you want to see the
list of possible command line options try

```sh
helm help <sub-command>
```

### Your own _Ghost_ chart from scratch

Let's reiterate the configuration we wrote last time. We will start from
scratch the Helm way but we can always look at our config from part one as
a reference.

#### Scaffolding a Helm Chart

Helm can help you to scaffold the basic chart structure. But first lets
create a chart directory and change to it.

```sh
mkdir charts
cd charts
```

Next we want to use the `create` sub-command. The built in help can show
you what the `create` can do for us.

```sh
helm help create
```

Scan through the displayed help and then go on with

```sh
helm create ghost
```

Let's have a look what we got

```sh
> find ghost
ghost
ghost/templates
ghost/templates/serviceaccount.yaml
ghost/templates/NOTES.txt
ghost/templates/deployment.yaml
ghost/templates/_helpers.tpl
ghost/templates/ingress.yaml
ghost/templates/tests
ghost/templates/tests/test-connection.yaml
ghost/templates/service.yaml
ghost/templates/hpa.yaml
ghost/charts
ghost/.helmignore
ghost/Chart.yaml
ghost/values.yaml
```

Lot's of stuff! Helm is taking a lot of assumptions about what we might
need in our Chart. The good news is that we can ignore quite some of the
files for the moment.

So what is all that files? Open the file
[charts/ghost/templates/service.yaml](charts/ghost/templates/service.yaml)
and compare it's contents with the file
[resources/service.yaml](resources/ghost-service.yaml) from our last
session.

Both files look pretty similar but the one file has curly braces and in
between there are some variables. You probably starting to understand what
we're dealing here with.

> Helm is a template engine with with some network capabilities.
>
> -- _Peter Scholl-Latour_

Remember when we just installed MySQL? When we issued the command
`helm install [...] stable/mysql` basically three things happened.

1. Helm downloaded a `.tgz` archive from the repository with contents
   similar to those we just created
2. It replaced the variables in the templates with the actual values
3. Then it sent the templated configurations to the Kubernetes API

From here we will look at some files in more detail.

#### The `Chart.yaml` files

This file is as simple to explain as it is important. It contains
mandatory meta-data such as the Chart's name and it's version. It can
also contain a list of dependent _Charts_.

#### The `.helmignore` files

Similar to the `.gitignore` file it's just a list of glob patterns of
files which are not supposed to be included in the `.tgz` package whenever
our chart is packaged. No need for change here either.

#### The `templates/NOTES.txt` file

These are some notes which will be displayed as information to the user
after the belonging Chart was installed. Templating is possible inside
here so you can provide concrete instructions to the user about further
steps to be undertaken with for instance correctly templated URLs
tailored to the very system where the Chart was just deployed.

#### The `values.yaml` files

This file contains the values with which the template engine will replace
the variables in the template files. Let's have a closer look at
[charts/ghost/values.yaml](charts/ghost/values.yaml).

#### Configuring the Chart

**In `values.yaml`:**

1. Let's configure the desired Docker image. In the `image:` paragraph
   Change the line `repository: nginx` to `repository: ghost` and the line
   `tag: ""` to `tag: "3.22"` (**do not omit the quotes** since all values
   have to be strings).
2. Configure the correct service port for Ghost which is `2368`. Look for
   the `service:` section and replace `port: 80` with port `port: 2368`.

**In `deployment.yaml`:**

1. We still need to put some environment variables into the container so
   our Ghost will find the database. Open the files
   [resources/ghost-deployment.yaml](resources/ghost-deployment.yaml)
   and [charts/ghost/templates/deployment.yaml](charts/ghost/templates/deployment.yaml)
   side by side and take some time to compare these files. You will
   notice that Helm added lots of things we didn't particularly ask for.
   The idea is that you get a proper skeleton of a _Deployment_
   configuration which can later on simply be configured by adding some
   lines to the `values.yaml` files. We'll see in a bit what that exactly
   means.  
   Let's move on and look at the `env` section in the list of `containers`
   in the configuration we wrote in the former part of the tutorial. Copy
   the whole paragraph and put it at exactly the same place in the Helm
   template and double check if the indention level is correct. Finally
   adjust the database name, the database user and the password to the
   values we used above when we created the mysql database.

   For convenience and clarity I copied the paragraph here for you with
   two lines of context prepended so it's clearer where to put it

```yaml
containers:
  - name: {{ .Chart.Name }}
    env:
      - name: database__client
        value: mysql
      - name: database__connection__host
        value: ghost-mysql
      - name: database__connection__user
        value: ghost
      - name: database__connection__password
        value: secret
      - name: database__connection__database
        value: ghost
      - name: url
        value: http://localhost:8080
```

2. Another thing we must do is provide the correct port to the container
   definition. Look for the line `containerPort: 80` and replace the port
   number with `2368`.

#### The first test run

We are ready for a first test. A dry run will show up a preview of the
templated configuration Helm is going to send to the Kubernetes API.

```sh
helm upgrade --install ghost ./ghost --dry-run
```

Scan though the generated yaml configuration and then repeat the command
above without the `--dry-run` argument.

```sh
helm upgrade --install ghost ./ghost
```

Is everything running as expected? Do you remember your toolkit from
part one? Learn these commands by heart.

```sh
kubectl get pods
kubectl describe deployment ghost
```

We just described the _Deployment_ which shows different information
as the description of a _Pod_. Look at the `Labels` section. Looks a bit
different what Helm put there compared to what we ourselves the last time
but everything works exactly the same way. Remember you can describe a
_Pod_ by it's labels?

```sh
kubectl describe pod -l=app.kubernetes.io/name=ghost
```

The same works for logs

```sh
kubectl logs -l=app.kubernetes.io/name=ghost
```

Using the labels saves you from copy and pasting the cryptic _Pod_ 's
name which changes all the time and is bad karma for you shell history.

If everything works we could risk a quick look through the browser
window. Do the port forward

```sh
kubectl port-forward deploy/ghost 8080:2368
```

and visit [http://localhost:8080](http://localhost:8080).

#### Making the environment more flexible

Let's think back for a moment to the MySQL instance we deployed. We were
able to adjust the database name, the user name and the password by just
defining it with the `--set` option. Wouldn't it be nice to built in the
same kind of flexibility into our Ghost Chart?

What we did was just copying the whole section from the configuration of
part one where we hard coded all the values and we didn't change much
about it so far. So how can we make this more flexible.

Let's have a brief look into the MySQL's
[values.yaml](https://github.com/helm/charts/blob/master/stable/mysql/values.yaml) file.
Use your browser's search function to find the string `mysqlUser`. There
seems to be a commented value but where does it go from there. The story
continues in the MySQL's
[deployment.yaml](https://github.com/helm/charts/blob/master/stable/mysql/templates/deployment.yaml)
there you can see how it's put into an environment variable called
`MYSQL_USER` in the container definition. The browser search will help
you to find it.

**Exercise:**

It's you turn now. Implement our Ghost Chart in exactly the same way by
replacing the hard coded environment variables in
[charts/ghost/templates/deployment.yaml](charts/ghost/templates/deployment.yaml)
with variables and put the belonging values into the
[charts/ghost/values.yaml](charts/ghost/values.yaml).

Don't get confused by the length of the MySQL chart's `deployment.yaml`.
Just focus on the `.env` section. Variables are written between curly
braces and their values have to be defined in `values.yaml`. If you have
ever written an HTML template in your life you should feel kind of at
home.

In `deployment.ymal` replace the hard coded value `ghost` for
`database__connection__user` with a variable between curly braces like
`{{ .Values.database__connection__user }} and then give it a value in
`values.yaml` like `database__connection__user: ghost`. That's all it
takes.

If you want to use default values in the _Deployment_ definition is up
to you. Personally I think it's cleaner to set the values right in the
`values.yaml` file and treat them as the defaults.

When you're done delete and redeploy the chart

```sh
helm delete ghost
helm upgrade --install ghost ./ghost
```

And verify it everything works just like before.

> „Der Sport ist nicht Selbstzweck, sondern Mittel zum Zweck.“
>
> ― _Erich Honecker_


#### Including MySQL as a requirement

Our setup consists of two components: The Ghost blog and it's MySQL
database. First we deployed the database and then we issued another
command to deploy Ghost. Our next objective is to include the mysql
chart as a requirement in the Ghost Chart.

Dependencies to other charts can be configured in two places. Either
in a separate file called `requirements.yaml` or in the file `Chart.yaml`.
Here we will go for the second option.

Open the file [charts/ghost/Chart.yaml](charts/ghost/Chart.yaml) and add
the following paragraph at the end of the file

```yaml
dependencies:
  - name: mysql
    version: ^1.6.6
    repository: https://kubernetes-charts.storage.googleapis.com/
    condition: mysql.enabled
```

That's basically it. Now we need to tear down everything first so we
can check if the dependency is properly added.

```sh
helm delete ghost ghost-mysql
```

Then we can fetch the dependency and start over again.

```sh
cd ghost
helm dependency update
```

We have a new folder inside our project containing the MySQL dependency.

```sh
> ls charts
mysql-1.6.6.tgz
> tar -tf charts/mysql-1.6.6.tgz
mysql/Chart.yaml
mysql/values.yaml
mysql/templates/NOTES.txt
mysql/templates/_helpers.tpl
mysql/templates/configurationFiles-configmap.yaml
mysql/templates/deployment.yaml
mysql/templates/initializationFiles-configmap.yaml
mysql/templates/pvc.yaml
mysql/templates/secrets.yaml
mysql/templates/serviceaccount.yaml
mysql/templates/servicemonitor.yaml
mysql/templates/svc.yaml
mysql/templates/tests/test-configmap.yaml
mysql/templates/tests/test.yaml
mysql/.helmignore
mysql/README.md
```

It's just another helm chart compressed into a `tgz` file included in our
Ghost chart as it's dependency.

So let's see if it works

```sh
cd ..
helm upgrade --install ghost ./ghost
kubectl get pods
```

We see both pods running even though we issued just one deployment
command. That's great but the Ghost _Pod_ is crashing. What's wrong here?
Remember wen we started MySQL as a separate deployment we specified the
`--set` option to configures a user name and a password etc. ? We hadn't
had the chance to tell our dependency about our preferences yet.

Open the file [charts/ghost/values.yaml](charts/ghost/values.yaml) and
create a new top level section named after our dependency (`mysql`)
and specify our options like so

```yaml
mysql:
  enabled: true
  mysqlRootPassword: secretpassword
  mysqlUser: ghost
  mysqlPassword: secret
  mysqlDatabase: ghost
```

This section basically does the same thing like the `--set` option - it
overwrites the values defined in the
[MySQL Chart's `values.yaml file](https://github.com/helm/charts/blob/master/stable/mysql/values.yaml).
You got all the freedom to adjust a _Chart_ you haven't written yourself.
Hopefully by now you got a grasp of the power of Helm and how it can help
you to avoid to reinvent the wheel over and over again.

Let's make our setup work again

```sh
helm delete ghost
helm upgrade --install ghost ./ghost
kubectl get pods
```

You might see a few restarts or even a _ChashLoopBackOff_ status. Which
is due to the Ghost _Pod_ being up faster than the database it requires.
Those things can be handled more gracefully but that's out of the scope
of this tutorial.

**Optional dependency**

Just in case we or somebody else who might use our chart in the future
does not want to use the mysql dependency which comes with the Chart
we made it optional by adding a condition in the `Chart.yaml`. You can
can bring your own database from somewhere and just set `enabled: true`
to `enabled: false` and that's it.

#### Exposing the _Service_ using an _Ingress_

What about accessing our blog though the browser. We were able to access
it via port forwarding but this is of course no suitable way to publish
a website.

Kubernetes has the concept of an _Ingress_ to make Applications available
from a regular URL.

> An Ingress is an API object that defines rules which allow external
> access to services in a cluster. An Ingress controller fulfills the
> rules set in the Ingress.
>
> -- _Laotse_

In plain english that means in order to expose our App we do not only
need an _Ingress_ object but _Ingress Controller_ which knows what to do
with the _Ingress_.

##### Deploying the _Ingress Controller_

An Ingress Controller is an abstract concept and several different
implementations of that concept are available. We will go for one of the
most popular implementations here which is the Nginx Ingress Controller.

That sounds much more complicated as it actually is. Basically we will
have an Nginx running and another component which listens for _Ingress_
objects and turns them into an Nginx config as soon as they appear and
funnels the configuration into the running Nginx instance.

Luckily somebody already prepared an Nginx Ingress Controller Helm
chart for us to deploy.

```sh
helm install --set controller.service.type=NodePort,controller.service.nodePorts.http=30080 nginx-ingress stable/nginx-ingress
kubectl get pods
```

Most commonly the ingress controller is deployed behind a load balancer.
This option is not available here on our local environment and that's
why we have to apply a little workaround here and make the _Ingress
Controller_ listen on port `30080`. In a production environment that would
of course not be the case.

At this point you might ask yourself if the ingress controller would be
another candidate to be added to our list of dependencies. The answer is
simple and it's **no**. One _Ingress Controller_ will handle all _Ingress_
objects on your Kubernetes. There are valid reasons to have more than one
_Ingress Controller_ but definitely not one _Controller_ per application.

##### Deploying the _Ingress_

When this is done the only thing we're missing is the actual _Ingress_.
Helm has thought of everything for us and already created and _Ingress_
for us. We just have to enable and configure it.

Open the file
[charts/ghost/templates/ingress.yaml](charts/ghost/templates/ingress.yaml)
read through it and see if you can relate variables to the values from
[charts/ghost/values.yaml](charts/ghost/values.yaml) at the section
`ingress`.

In the `values.yaml` under the section `ingress` we have to replace the
line `enabled: false` with `enabled: true`. The other thing of interest
is the subparagraph `hosts`. Replace the whole paragraph with

```yaml
hosts:
  - host: ghost.local
    paths:
      - /
```

Since we now adjusted the URL from which Ghost is to be reached we also
have to tell the application itself about it by editing the `values.yaml`
and go to the section where you defined the Ghost environment variables.
replace the line `url: http://localhost:8080` with

```yaml
url: http://ghost.local:30080
```

Then you can upgrade the Ghost chart with

```sh
helm upgrade --install ghost ./ghost
```

Now we got an _Ingress_. If you want to know which _Ingress_ are available
you can apply the same grammar as you did when you listed all _Pods_. Just
change the object

```sh
kubectl get ingress
```

and you guessed it, it can also be described

```sh
kubectl describe ingress ghost
```

##### Sending the correct `Host` header

The last thing necessary to access our blog is to send a request
containing the correct `Host` header which we just set to `ghost.local`.

This has nothing to do with Kubernetes it's more a necessity of our
local environment because of the way how web servers deal with multiple
domains on a single server. Basically we pointed the host `ghost.local`
to our blog. If the `Host` header in a request is set to something else
such as `localhost` the _Ingress Controller_ just wouldn't know which
Application is responsible to handle the request.

Long story short: Edit the file `/etc/hosts` and add this line
at the bottom

```
127.0.0.1 ghost.local
```

if you use Docker Desktop. If you're using minikube replace the
`127.0.0.1` with what ever IP gives you the `minikube ip` command.

When this is done visit
[http://ghost.local:30080/](http://ghost.local:30080/) and enjoy your
Ghost blog running on Kubernetes deployed with Helm.

#### Cleaning up

If you're done playing around with the stack the last thing to learn
for today is how to remove a deployed helm chart.

```sh
helm list
helm delete ghost nginx-ingress
helm list
```

Stay tuned for more happy days on Kubernetes.

## Further reading

- [https://v2.helm.sh/docs/developing_charts/](https://v2.helm.sh/docs/developing_charts/)
- [https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)
