---
title: "Introducing Conftest"
date: 2019-06-16T11:14:07+01:00
keywords: conftest
---

For the past few months I've been hacking on [Conftest](https://github.com/instrumenta/conftest), a tool
for writing tests for configuration, using [Open Policy Agent](https://www.openpolicyagent.org/). I spoke
about Conftest at [KubeCon](https://speakerdeck.com/garethr/unit-testing-your-kubernetes-configuration-with-open-policy-agent) but 
am only now getting round to writing up a quick introducion.


## The problem

We're all busy writing lots of configuration. There are already more than 1 million Kubernetes config files public on GitHub, and about
70 million YAML files too. We're using a range of tools, from CUE to Kustomize to templates to DSLs to general purpose programming languages
to writing YAML by hand. But we're rarely automatically enforcing policy against any of that configuration. That leaves two approaches
to spread best practices; manually via code review, or not at all. Conftest is aiming to be a handy tool to help with that problem. 


## Open Policy Agent

Conftest builds on Open Policy Agent.

> Open Policy Agent (OPA) is a general-purpose policy engine with uses ranging from authorization and admission control to data filtering. OPA provides greater flexibility and expressiveness than hard-coded service logic or ad-hoc domain-specific languages. And it comes with powerful tooling to help you get started.

OPA introduces the [Rego data assertion language](https://www.openpolicyagent.org/docs/latest/how-do-i-write-policies/)

> Rego was inspired by [Datalog](https://en.wikipedia.org/wiki/Datalog), which is a well understood, decades old query language. Rego extends Datalog to support structured document models such as JSON.

Most of the usecases OPA has been applied to so far have been on the server side. Authentication systems, proxies, Kubernetes admission controllers, storage system policies.
That says more about the flexibility of OPA more than anything else though. OPA is a very general purpose tool with lots of potential uses. Conftest
simply runs with that in the direction of a nice local user interface, and usage in continuous integration pipelines.


## Conftest for Kubernetes

Conftest is available from Homebrew for macOS and from Scoop for Windows, Docker images, as well as executables being
[you can download directly](https://github.com/instrumenta/conftest/releases). You can find [installation instructions](https://github.com/instrumenta/conftest#installation)
in the README. With Conftest installed, let's write a simple test for a Kubernetes configuration file.

Save the following as `policy/base.rego`. `policy` is just the default directory where Conftest will look for 
`.rego` policy files, you can override that if desired.

```
package main


deny[msg] {
  input.kind = "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot = true
  msg = "Containers must not run as root"
}

deny[msg] {
  input.kind = "Deployment"
  not input.spec.selector.matchLabels.app
  msg = "Containers must provide app label for pod selectors"
}
```

The tests here are:

1. Checking for Deployments with containers running as root
2. Checking for Deployments with containers without a specific label

Both realistic, if simple, examples of the type of thing you might want to enforce in your cluster. Assuming
we have our Deployment described in a file called `deployment.yaml` we would run Conftest like so:


```
$ conftest test deployment.yaml
deployment.yaml
   Containers must not run as root
   Deployments are not allowed
```

Here we see our Deployment failed both tests and Conftest is returning the message we defined in the policy above.
Conftest will return a non-zero status code when tests fail, and can also take input via stdin rather than reading files.

```
$ cat deployment.yaml | conftest test -
   Containers must not run as root
   Deployments are not allowed
```

The above example tests a single configuration via against a single policy file. However you can split your policies over
multiple files in the `policy` directory, and also point Conftest at multiple files at the same time as well as multi-document
YAML files.


## Conftest for other structured data

Conftest isn't just for Kubernetes configs. You can use it with any structured data, starting with JSON and YAML. More input
formats are likely to be supported in the future.

The repository has [examples](https://github.com/instrumenta/conftest/tree/master/examples) using Conftest to test
CUE, Typescript, Docker Compose, Serverless configs and Terraform state. Let's take a look at a Terraform example.

```
package main


blacklist = [
  "google_iam",
  "google_container"
]

deny[msg] {
  check_resources(input.resource_changes, blacklist)
  banned := concat(", ", blacklist)
  msg = sprintf("Terraform plan will change prohibited resources in the following namespaces: %v", [banned])
}

# Checks whether the plan will cause resources with certain prefixes to change
check_resources(resources, disallowed_prefixes) {
  startswith(resources[_].type, disallowed_prefixes[_])
}
```

Here the policy is a little more complicated than the examples above, showing some of the power of Rego. Here we
check the list of resource changes for any resources on a blacklist.


## Conclusion and next steps

This post is just a quick introduction to Conftest. The tool already has a few features I'll try and cover in future posts, including:

* Store OPA bundles in OCI registries, making reusing policies easier
* Debugging and tracing the Rego assertions
* Usage in a CI pipeline
* A `kubectl` plug for making assertions against a running cluster
* Workflows for using Conftest locally and then loading the policies into [Gatekeeper](https://github.com/open-policy-agent/gatekeeper)

It also needs a bit of tidying up and better testing now I have the basic UI down.

Conftest is looking for other contributors as well. If you have ideas for features, or fancy hacking on a small but useful Go tool, a 
few of the current issues are marked as [good for test time contributors](https://github.com/instrumenta/conftest/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22).


