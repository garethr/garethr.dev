---
title: "Emphemeral Clusters for Helm Charts and Operators"
date: 2019-06-15T16:27:54+01:00
keywords: kind
---

Building on the [previous post]({{< relref "ephemeral-kubernetes-clusters-with-kind-and-make.md" >}}), I found myself
wanting to grab quick clusters for experimenting with Helm Charts and with Kubernetes Operators.


## Helm

```make
helm-%: cluster-%
        kubectl -n kube-system create sa tiller
        kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        helm init --service-account tiller --kubeconfig=$$(kind get kubeconfig-path --name $(NAME))
        helm repo --kubeconfig=$$(kind get kubeconfig-path --name $(NAME)) add incubator https://kubernetes-charts-incubator.storage.googleapis.com
```

The above target for our Makefile makes it easy to grab a new [Kind](https://kind.sigs.k8s.io) cluster and instantiate Helm. Here we're setting
up a new service account for the Tiller component and ensuring Helm has the right permissions to launch things on the cluster. We're
not attempting to secure the cluster in any way here, this is intended purely for throwaway clusters for testing after all. The following
will launch a new cluster named `clustername` with Helm already installed:

```console
make helm-clustername
```

The cluster won't have any Helm Charts installed yet, but you should be able to run `helm install` once you point your `KUBECONFIG` environment
variable at the new cluster as described when you run the command.

```fish
export KUBECONFIG=(kind get kubeconfig-path --name="clustername")
```


## Operators

With the new [Operator Hub](https://operatorhub.io/) serving as a repository for finding and installing Kubernetes Operators, it
was simple enough to add support for bootstrapping and installing operators into a new cluster.


```make
operator-%: cluster-%
        @$(APPLY) https://github.com/operator-framework/operator-lifecycle-manager/releases/download/$(OLM_VERSION)/crds.yaml
        @$(APPLY) https://github.com/operator-framework/operator-lifecycle-manager/releases/download/$(OLM_VERSION)/olm.yaml
        @$(APPLY) https://operatorhub.io/install/$(NAME).yaml
```

The above snippet added to our Makefile allows us to run commands like the following:

```console
make operator-etcdoperator
```

This will create a brand new [Kind](https://kind.sigs.k8s.io) cluster, install the Operator Lifecycle Manager, and the install the operator 
specified in the target name, in this case [etcdoperator](https://operatorhub.io/operator/etcd)
