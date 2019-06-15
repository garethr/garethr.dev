---
title: "Ephemeral Kubernetes Clusters With Kind and Make"
date: 2019-05-27T19:15:24+01:00
keywords: kind
---

There are a number of options for persistent local Kubernetes clusters, but when you're developing tools against the Kubernetes APIs it's often best to be throwing things away fairly regularly. Enter [Kind](https://kind.sigs.k8s.io/). Originally designed as a tool for testing Kubernetes itself, Kind runs a working Kubernetes cluster on top of Docker. 

At its simplest we can create a new Kubernetes cluster like so:

```shell
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.14.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Creating kubeadm config 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl cluster-info
```

The instructions from running the command show how to connect to the new cluster. Running the Docker commands will show the running container.

```shell
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED              STATUS              PORTS                                  NAMES
0e39e49feaed        kindest/node:v1.14.2   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   61227/tcp, 127.0.0.1:61227->6443/tcp   kind-control-plan
```

## Automating Kind

You can use the Kind CLI tool directly, but it's also a handy utility for simple automation tasks. The [Roadmap](https://kind.sigs.k8s.io/docs/contributing/1.0-roadmap/) promises better support for Kind as a Go library in the future, but for the moment I've started using `make`.

```make
WAIT := 200s
APPLY = kubectl apply --kubeconfig=$$(kind get kubeconfig-path --name $@) --validate=false --filename
NAME = $$(echo $@ | cut -d "-" -f 2- | sed "s/%*$$//")

tekton tekton%: create-tekton%
        @$(APPLY) https://storage.googleapis.com/tekton-releases/latest/release.yaml

gatekeeper gatekeeper%: create-gatekeeper%
        @$(APPLY) https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper-constraint.yaml

create-%:
        -@kind create cluster --name $(NAME) --wait $(WAIT)

delete-%:
        @kind delete cluster --name $(NAME)

env-%:
        @kind get kubeconfig-path --name $(NAME)

clean:
        @kind get clusters | xargs -L1 -I% kind delete cluster --name %

list:
        @kind get clusters


.PHONY: tekton gatekeeper tekton% gatekeeper% create-% delete-% env-% clean list
```

The above `Makefile` provides a few handy shortcuts for Kind commands. Not strictly necessary but useful for consistency. So I can now run commands like:

```shell
$ make create-hello
# Create a new cluster called hello
$ make delete-hello
# Delete the hello cluster
$ make list
# Provide a list of all cluster
$ make clean
# Delete all of my clusters
```

More useful are the higher-level commands, in the example above `make tekton` and `make gatekeeper`. [Tekton](https://tekton.dev/) is a project aiming to provide Kubernetes-style resources for declaring CI/CD pipelines. [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) provides a policy controller for Kubernetes, using Open Policy Agent. Both projects add new custom resources to Kubernetes. With the `Makefile` above I can run one command and spin up an ephemeral
Kubernetes cluster pre-provisioned with Tekton, or Gatekeeper. Because I got carried away the Makefile also supports grabbing multiple Tekton or Gatekeeper clusters with different names, so the following works too:

```shell
$ make tekton1
$ make tekton2
$ make tekton-loves-make
```

## Make?

I can imagine Kind, or a sub-project, growing a `Kindfile` in the future. And an improved CLI which moves beyond the initial cluster testing scenarios it was originally designed for. But for now, and for me at least, Make makes a great prototyping tool. I can grab arbitrary ephemeral Kubernetss clusters until I run out of memory, and adding new projects I'm interested in is trivial. Just as importantly thowing them away is easy too.

Make is a great tool to have in your toolbox in my experience, and learning just enough to be dangerous
goes a long way in terms of expressiveness and power. But as [mentioned before]({{< relref "automating-the-cue-workflow-with-tilt.md" >}}#you-like-dsls-then), I have a bit of a thing for DSLs.
