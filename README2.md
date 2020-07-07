# Kubernetes workshop part II

## Table of contents

- [Kubernetes workshop part II](#kubernetes-workshop-part-ii)
  - [Table of contents](#table-of-contents)
  - [What is Helm?](#what-is-helm)
  - [Why Helm?](#why-helm)
    - [Helm high level goals](#helm-high-level-goals)
    - [Helm functionality](#helm-functionality)
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

## What is Helm?

Helm is a package manger for Kubernetes - Helm Packages are called
_Charts_ - Helm _Charts_ help you define, install, and upgrade even the
most complex Kubernetes application.

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

## Upgrading our Ghost configuration to a Helm Chart

### Setting the right context

The namespace `ghost` should already exist from part one of the tutorial.
Make sure to set the namespace as the default in your context.

```sh
kubectl create namespace ghost2
kubectl config set-context --current --namespace ghost2
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
    ghost stable/mysql
```

then look what happens

```sh
kubectl get pods
```

Wonderful! A running MySQL instance without writing a single line of
configuration. Noting special about this MySQL container - just a lot less
effort then last time when we had to write the Kube configuration
ourselves.

### Your own _Ghost_ chart from scratch

Let's reiterate the configuration we wrote last time. We will start from
scratch the Helm way but we can always look at our config from part one as
a reference.

#### Scaffolding a Helm Chart

Helm can help you to scaffold the basic chart structure.

```sh
cd charts
helm create ghost
find ghost
```

The last command should give you the following list

```
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
need in our Chart. The good news is that we can get rid of quite some
contents in the listed directory.

```sh
rm -rf ghost/templates/{serviceaccount.yaml,tests,hpa.yaml}
```

So what is all that files? Open the file
[charts/ghost/templates/service.yaml](charts/ghost/templates/service.yaml)
and compare it's contents with the file
[resources/service.yaml](resources/ghost-service.yaml).

Both files look pretty similar but the one file has curly braces and in
between there are some variables. You probably starting to understand what
we're dealing with here.

> Helm is a template engine with with some network capabilities.
>
> -- _Peter Scholl-Latour_

Remember when we just installed MySQL? When we issued the command
`helm install [...] stable/mysql` basically three things happened.

1. Helm downloaded a `.tgz` archive from the repository with contents
   similar to those we just created
2. It replaced the variables in the templates with the actual values
3. Then it send the templated configurations to the Kubernetes API

From here we will look at some files in more detail.

#### The `Chart.yaml` files

This file is as simple to explain as it is important. It contains
mandatory meta-data such as the Chart's name and it's version. Helm
already filled in the basic stuff for us so there is no need to change
for now.

#### The `.helmignore` files

Similar to the `.gitignore` file it's just a list of glob patterns of
files which are not supposed to be included in the `.tgz` package whenever
our chart is packaged. No need for change here either.

#### The `templates/NOTES.txt` file

These are some notes which will be displayed as information to the user
after the belonging Chart was installed. Templating is possible inside
here so you can provide concrete instructions to the user about further
steps to be undertaken with for instance correctly generated URLs
tailored to the very system where the Chart was just deployed.

#### The `values.yaml` files

This file contains the values with which the template engine will replace
the variables in the template files. Let's have a closer look at
[charts/ghost/values.yaml](charts/ghost/values.yaml).

#### Configuring the Chart

**In `values.yaml`:**

Let's configure the desired Docker image. In the `image:` paragraph
Change the line `repository: nginx` to `repository: ghost` and the line
`tag: ""` to `tag: "3.22"` (**do not omit the quotes** since all values
have to be strings)

**In `deployment.yaml`:**

We still need to put some environment variables into the container so
our Ghost will find the database. Open the files
[resources/ghost-deployment.yaml](resources/ghost-deployment.yaml)
and [resources/ghost-deployment.yaml](charts/ghost/templates/deployment.yaml) side by side and take some time to compare these files. You will
notice that Helm added lots of things we didn't particularly ask for.
Normally you should remove everything which is not needed but for the
sake of simplicity of this tutorial we will just focus on adding and
changing what is actually relevant.

Look at the `env` section in the list of `containers` in the configuration
we wrote in the former part of the tutorial. Copy the whole paragraph and
put it at exactly the same place in the Helm template and double check if
the indention level is correct. Finally adjust the database name, the
database user and the password to the values we used above when we created
the mysql database.

the desired values are

```yaml
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
