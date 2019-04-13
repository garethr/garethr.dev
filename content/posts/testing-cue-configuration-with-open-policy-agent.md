---
title: "Testing Cue Configuration with Open Policy Agent"
date: 2019-04-11T11:00:00Z
keywords: cue
---

In the first two posts we have written [some configuration using CUE]({{< relref "configuring-kubernetes-with-cue.md" >}})
and [validated it against the Kubernetes schemas using Kubeal]({{< relref "validating-cue-kubernetes-configuration-with-kubeval.md" >}}).
In this post we're going to expand testing to include custom assertions.

As mentioned before, I'm using Kubernetes configuration as an example here. The
same approach is just as valid for other structured data configs like CloudFormation,
Azure Resource Manager templates, Circle CI configs, etc.

Syntactically valid configuration doesn't make it correct. It might for instance
breach some internal policy or other, for example:

* Disallow containers running as root
* Mandate certain labels are set of management or auditing purposes
* Ban images using the `latest` tag or without a tag at all
* Restrict which resources can be used

Let's demonstrate this by starting with a slightly modified version of our deployment
written in CUE.

{{< highlight "json" >}}
package kubernetes

deployment <Name>: {
  apiVersion: string
  kind:       "Deployment"
  metadata name: Name
  spec: {
    replicas: 1 | int
    template: {
      metadata labels app: Name
      spec containers: [{name: Name}]
    }
  }
}

deployment "hello-kubernetes": {
  apiVersion: "apps/v1"
  spec: {
    replicas: 3
    template spec containers: [{
      image: "paulbouwer/hello-kubernetes:1.5"
      ports: [{
        containerPort: 8080
      }]
    }]
  }
}
{{< / highlight >}}


## Introducing `conftest`

[Conftest](https://github.com/instrumenta/conftest) is a new project I've been working on. It's
intended for writing tests against structured data, using the Rego language from [Open Policy Agent](https://www.openpolicyagent.org/).
With `conftest` installed we can wire it up to CUE as we did with Kubeval.

{{< highlight "json" >}}
package kubernetes

import "encoding/yaml"

command test: {
  task conftest: {
    kind:   "exec"
    cmd:    "conftest -"
    stdin:  yaml.MarshalStream(objects)
    stdout: string
  }
  task display: {
    kind: "print"
    text: task.conftest.stdout
  }
}
{{< / highlight >}}


## Open Policy Agent

[Open Policy Agent](https://www.openpolicyagent.org/), or OPA for short, is a super interesting project
which has a wide range of usecases. It's described as a general purpose policy engine and already
has several more specific subprojects, for instance [gatekeeper](https://github.com/open-policy-agent/gatekeeper)
for Kubernetes and an Istio plugin. The documentation has examples of enforcing policy around
AWS IAM and Terraform as well.

Open Policy Agent uses a language called Rego to define policies. You can find more information
on Rego and how to write policies in the [official documentation](https://www.openpolicyagent.org/docs/v0.10.7/how-do-i-write-policies/).
Conftest simply provides a nice user interface to using Rego in a local testing context.

Let's write some tests for our deployment config. Save the following as `policy/base.rego`:

{{< highlight "rego" >}}
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
{{< / highlight >}}


We've written two tests here:

1. The first checks that all deployments are not set to run with root permissions
2. The second test ensures that deployments have an `app` label seletor specified


## Running the tests

If you check our deployment you'll notice both of these policies are breached by our
configuration. Let's run `conftest` (via our CUE command):

{{< highlight "shell" >}}
$ cue test
  Containers must not run as root
  Containers must provide app label for pod selectors
{{< / highlight >}}

Here we see our expected failures.

As mentioned previously, `cue` [currently eats the exit code](https://github.com/cuelang/cue/issues/30)
so although this should exit with a non-zero status it doesn't do so currently.

Let's go about fixing our deployment configuration:

{{< highlight "json" >}}
package kubernetes

deployment <Name>: {
  apiVersion: string
  kind:       "Deployment"
  metadata name: Name
  spec: {
    replicas: 1 | int
    selector matchLabels app: Name
    template: {
      metadata labels app: Name
      spec containers: [{name: Name}]
      spec securityContext runAsNonRoot: true
    }
  }
}

deployment "hello-kubernetes": {
  apiVersion: "apps/v1"
  spec: {
    replicas: 3
    template spec containers: [{
      image: "paulbouwer/hello-kubernetes:1.5"
      ports: [{
        containerPort: 8080
      }]
    }]
  }
}
{{< / highlight >}}


With those changes we should expect the tests to pass:

{{< highlight "shell" >}}
$ cue test
$ echo $status
0
{{< / highlight >}}


## Conclusions

This is a simple example of the power of Open Policy Agent applied to static testing.
With Open Policy Agent it's possible to define quite powerful tests, incorporating risk
scoring and more. Open Policy Agent is also intended to be used to protect and define policy
within a cluster, which opens up some powerful workflows for reusing policies for local
development, testing in CI and enforcement in production. The advantage of doing so is
speeding up changes by making those policy violations part of the development process, rather
than just part of the deployment process.

It's the ability to reuse OPA policies in multiple contexts that I find interesting, and
think that makes introducing Rego into the mix worthwhile, even if simply policies can actually
be encoded in CUE itself.
