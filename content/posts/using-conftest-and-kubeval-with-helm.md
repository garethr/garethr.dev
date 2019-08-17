---
title: "Using Conftest and Kubeval With Helm"
date: 2019-08-17T10:26:26+01:00
keywords: conftest kubeval
---

I maintain a few open source projects that help with testing configuration, namely [Kubeval](https://github.com/instrumenta/kubeval)
and [Conftest](https://github.com/instrumenta/conftest). Recently I've been hacking on various integrations for these tools,
the first of which are plugins for [Helm](https://helm.sh/).


## Validate Helm Charts with Kubeval

Kubeval validates Kubernetes manifests against the upstream Kubernetes schemas. It's useful for catching invalid configs, especially
in CI environments or when testing against multiple different versions of Kubernetes. Lots of folks have been using Kubeval with Helm
for a while, mainly using `helm template` and piping to `kubeval` on stdin. The [Helm Kubeval plugin](https://github.com/instrumenta/helm-conftest)
makes that easier to do just from Helm.

You can install the Helm plugin using the Helm plugin manager:

```
helm plugin install https://github.com/instrumenta/helm-kubeval
```

With that installed, let's grab a sample chart and run `helm kubeval <path>`:


```console
$ git clone git@github.com:helm/charts.git
$ helm kubeval charts/stable/nginx-ingress
The file nginx-ingress/templates/serviceaccount.yaml contains a valid ServiceAccount
The file nginx-ingress/templates/clusterrole.yaml contains a valid ClusterRole
The file nginx-ingress/templates/clusterrolebinding.yaml contains a valid ClusterRoleBinding
The file nginx-ingress/templates/role.yaml contains a valid Role
The file nginx-ingress/templates/rolebinding.yaml contains a valid RoleBinding
The file nginx-ingress/templates/controller-service.yaml contains a valid Service
The file nginx-ingress/templates/default-backend-service.yaml contains a valid Service
The file nginx-ingress/templates/controller-deployment.yaml contains a valid Deployment
The file nginx-ingress/templates/default-backend-deployment.yaml contains a valid Deployment
The file nginx-ingress/templates/controller-configmap.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-daemonset.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-hpa.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-metrics-service.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-poddisruptionbudget.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-servicemonitor.yaml contains an empty YAML document
The file nginx-ingress/templates/controller-stats-service.yaml contains an empty YAML document
The file nginx-ingress/templates/default-backend-poddisruptionbudget.yaml contains an empty YAML document
The file nginx-ingress/templates/headers-configmap.yaml contains an empty YAML document
The file nginx-ingress/templates/podsecuritypolicy.yaml contains an empty YAML document
The file nginx-ingress/templates/tcp-configmap.yaml contains an empty YAML document
The file nginx-ingress/templates/udp-configmap.yaml contains an empty YAML document
```

Here we can see the various parts of the template are picked up and, in this case, contain valid Kubernetes resources.

The plugin also supports the various flags on Kubeval, so you can for example test charts against other versions of Kubernetes.

```
helm kubeval . -v 1.9.0
```

You can also test variations on the chart, for instance by setting particular values before validating.

```
helm kubeval charts/stable/nginx-ingress --set controller.image.tag=latest
```


## Check Charts for security issues with Conftest

[Conftest](https://github.com/instrumenta/conftest) is a little more general purpose than Kubeval. Where as Kubeval is specific to
Kubernetes, Conftest is used for testing all kinds of configuration. That means you need to bring your own tests or policy, which is
written using Rego and [Open Policy Agent](https://www.openpolicyagent.org/).

The [Helm Conftest plugin](https://github.com/instrumenta/helm-conftest) makes it easy to use Conftest with Helm, in a similar way to
the above Kubeval plugin. Installation is easy with the Helm plugin manager.


```
helm plugin install https://github.com/instrumenta/helm-conftest
```

Conftest needs you to write or otherwise aquire some policies. For the purposes of this example we'll use an existing policy, written to
test various security properties of Kubernetes configurations.

Create a file called `conftest.toml` in the same directory as the chart and set it to download our sample policy.

```toml
[[policies]]
repository = "instrumenta.azurecr.io/kubernetes"
tag = "latest"
```

With that in place we can run `helm conftest`. The `--update` flag asks the plugin to download any policies from the config file before
running the tests.

```console
$ helm conftest . --update
FAIL - release-name-test in the Pod release-name-mysql-test does not have a memory limit set
FAIL - test-framework in the Pod release-name-mysql-test does not have a memory limit set
FAIL - release-name-test in the Pod release-name-mysql-test does not have a CPU limit set
FAIL - test-framework in the Pod release-name-mysql-test does not have a CPU limit set
FAIL - release-name-test in the Pod release-name-mysql-test doesn't drop all capabilities
FAIL - test-framework in the Pod release-name-mysql-test doesn't drop all capabilities
FAIL - release-name-test in the Pod release-name-mysql-test is not using a read only root filesystem
FAIL - test-framework in the Pod release-name-mysql-test is not using a read only root filesystem
FAIL - release-name-test in the Pod release-name-mysql-test is running as root
FAIL - test-framework in the Pod release-name-mysql-test is running as root
FAIL - The Pod release-name-mysql-test is mounting the Docker socket
FAIL - release-name-mysql in the Deployment release-name-mysql does not have a memory limit set
FAIL - remove-lost-found in the Deployment release-name-mysql does not have a memory limit set
FAIL - release-name-mysql in the Deployment release-name-mysql does not have a CPU limit set
FAIL - remove-lost-found in the Deployment release-name-mysql does not have a CPU limit set
FAIL - release-name-mysql in the Deployment release-name-mysql doesn't drop all capabilities
FAIL - remove-lost-found in the Deployment release-name-mysql doesn't drop all capabilities
FAIL - release-name-mysql in the Deployment release-name-mysql is not using a read only root filesystem
FAIL - remove-lost-found in the Deployment release-name-mysql is not using a read only root filesystem
FAIL - release-name-mysql in the Deployment release-name-mysql is running as root
FAIL - remove-lost-found in the Deployment release-name-mysql is running as root
FAIL - The Deployment release-name-mysql is mounting the Docker socket
FAIL - release-name-mysql must include Kubernetes recommended labels: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/#label
```

Here we can see various failures, mainly related to not setting CPU and memory limits, running as root and not providing the
expected labels.

You can write your own polices to cover any aspect of the configuration, usin the powerful Rego language from [Open Policy Agent](https://www.openpolicyagent.org/).
You can find more [example in the Conftest repository](https://github.com/instrumenta/conftest/tree/master/examples).
