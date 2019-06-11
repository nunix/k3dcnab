# Purpose
This repo contains the files needed to create and run a `porter` bundle that will launche a `k3d` cluster on `docker`.
The bundle has been created from WSL (clearlinux custom distro).

# Pre-requisites
The 2 main (read mandatory) requirements are:
1. [Porter v0.7.0](https://porter.sh/install/)
2. [Docker v19.03](https://docs.docker.com/docker-for-windows/install/)

And 2 optional (read strongly recommended) requirement:
1. [k3d v1.2.2](https://github.com/rancher/k3d)
2. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

# TLDR: Just Run It!
No need for explanations? Great! here is how to run this bundle after you downloaded or cloned [this repo](https://github.com/nunix/k3dcnab/k3dcnab.git):
```bash
cd /path/to/<k3dcnab cloned directory>
porter build
porter install
export KUBECONFIG="$(k3d get-kubeconfig --name='k3dcnabcluster')"
```

And done. You have now a K3d cluster (1 master node only) running.

Once you finished "playing" with it, you have 2 choices:
1. Stop the cluster
```bash
porter upgrade --param k3dcommand=stop
```
- And alternatively, you can restart the cluster using the same command and replacing `stop` by `start`
```bash
porter upgrade --param k3dcommand=start
```

2. Cleanup all the resources by deleting the cluster
```bash
porter uninstall
```

# Too fast and now you're curious? Cool, here are the explanations...
The bundle relies on 2 files: `porter.yaml` and `Dockerfilek3d`.
These files have each their importance and, as you may have noticed, the normal `Dockerfile` has a different name and this is due to the fact that `porter` will create the final `Dockerfile` with the needed [CNAB spec](https://cnab.io/) section:
```Dockerfile
# exec mixin has no buildtime dependencies

COPY cnab/ /cnab/
COPY porter.yaml /cnab/app/porter.yaml
CMD ["/cnab/app/run"]
```

## `Dockerfilek3d`: the magic
Let's start explaning first the `Dockerfilek3d`.
For the ones knowing how to read it, you will see that `docker in docker` container image is the one used here:
```Dockerfile
FROM docker:19.03-rc-dind
```

This image will actually allow the container to communicate with the host `docker daemon`. 
Really useful and in the same time really **dangerous** so please, be warned when using such image. Even if my intend, or anyone else for that matter, is pure curiousity and no evil at all, it does not mean that there won't be any impact. So **always** review the Dockerfiles that use this type of image.

With that said, the rest of the `Dockerfilek3d` will install `k3d` binary and set the container entrypoint to `/bin/bash`:
```Dockerfile
# required installs
RUN apk add bash curl

# k3d install
RUN curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash

ENTRYPOINT ["/bin/bash"]
```

## `porter.yaml`: the magician
Now that we have the base `Dockerfile`, let's see how we can leverage it with `porter`.

`porter` has several functions that will help create `CNAB bundles` in a very friendly and rapid way without the need to know the `CNAB spec` in details.
The main configuration file is written in `yaml`, therefore the name `porter.yaml`.

While I strongly suggest you to have a look on the [porter.sh] website to understand the whole concept and options, I will explain here the 3 main parts of the `porter.yaml` file:

### 1. The header
In this section, we will:
- define which [mixin](https://porter.sh/mixins/) will be used
- describe the bundle with a name, version, description 
- provide an `invocationImage` name

It will also contains a reference to which `Dockerfile` should be used:
```yaml
mixins:
  - exec

name: k3dcnab
version: 0.7.0
description: "A portable K3s cluster and K3d management tool"
invocationImage: k3dcnab:1.0.0
dockerfile: Dockerfilek3d
```

In this bundle, only the `exec` mixin will be used and will allow us to run shell commands.

### 2. The parameters
This section is, at least for me, what makes `porter` really powerful to use. We will define a serie of parameters (or variables) that will be referenced later in the `actions` section.

For this bundle, we have 4 parameters that will define: 
- the name of the cluster
- the number of workers for the `k3d` cluster
- the command to run for the `upgrade` action (see `actions` section below)
- the ip of the `host docker daemon`, which will be added as a *subject alternative name* (san) to the `k3d kubeconfig` TLS certificate

One last and (very) cool aspect, is that these parameters can be set to different values when calling one of the `actions` (as seen above for stopping and starting the `k3d` cluster):
```yaml
parameters:
  - name: k3dname
    type: string
    default: "k3dcnabcluster"

  - name: k3dworkers
    type: string
    default: 0

  - name: k3dcommand
    type: string
    default: "list"

  - name: k3dip
    type: string
    default: "10.0.75.1"
```

### 3. The actions
The final section is the `actions` that will be used when running the `porter` command.
By default, the standard `CNAB actions` are: `install`, `upgrade` and `uninstall`. However, there's nothing blocking you from creating new ones (but this goes way beyond my knowledge and the purpose of this bundle).

The different `actions` have the same *skeleton* which contains:
- The name of the `action` always comes first and must be unique in the `porter.yaml` file
- The name of the `mixin` which can be used several times in each `mixin` (if needed)
- The `description` that will show in the client output
- The `command` that needs to be run -> this is exclusive to the `exec` mixin
- The `arguments` that will be attached to the `command`

```yaml
install:
  - exec:
      description: "Create a new K3s cluster"
      command: bash
      arguments:
        - -c
        - "k3d create --name {{ bundle.parameters.k3dname }} --workers {{ bundle.parameters.k3dworkers }} -x --tls-san={{ bundle.parameters.k3dip }} --api-port {{ bundle.parameters.k3dip }}:6443"

upgrade:
  - exec:
      description: "check the K3s cluster status"
      command: bash
      arguments:
        - -c
        - "k3d {{ bundle.parameters.k3dcommand }}"

uninstall:
  - exec:
      description: "Delete K3s cluster"
      command: bash
      arguments:
        - -c
        - "k3d delete --name {{ bundle.parameters.k3dname }}"
```

As you can see in the example above, the *mustache* notation is used and the syntax helper can be found [here](https://porter.sh/wiring/)

## Let's run it again in a fun way
Now that the different files and sections of the bundle have been explained, we can try to run it again with different parameters:
```bash
# Create a new k3d cluster with a different name and some workers (because why not)
porter create --param k3dname=k3dftw --param k3dworkers=3

# At the time of this writting, the following step is still mandatory and needs to be run on the host
export KUBECONFIG="$(k3d get-kubeconfig --name='k3dftw')"

# Now let's list and then stop and start our k3d cluster
porter upgrade
porter upgrade --param k3dcommand=stop
porter upgrade --param k3dcommand=start

# Finally, let's clean the k3d and porter resources
porter uninstall
```

# Conclusion
This bundle, while working perfectly, should be considered as a *fun way* to introduce you to `porter`.
Of course, both `k3d` and `docker` are big stars here and I strongly encourage you to have a look. Still, `porter` is really the tool that glue all the components together, making a bundle that can be used and, even more importantly, reused for a fast deployement of your application or, in my case, a full-blown environment.

I truly hope that your curiosity has been triggered while reading this blog post and do not hesitate to share your questions or your own inventions with me and the `porter team` on [twitter](https://twitter.com/nunixtech) and/or on the `porter` channel of the [CNAB slack](https://cloud-native.slack.com).

>>> Nunix out
