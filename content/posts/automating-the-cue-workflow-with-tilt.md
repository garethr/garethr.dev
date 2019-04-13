---
title: "Automating the CUE workflow with Tilt"
date: 2019-04-11T12:00:00Z
keywords: cue
---

We now have [a nice configuration language in which to author our configs]({{< relref "configuring-kubernetes-with-cue.md" >}}) (CUE), and
a way of [validating]({{< relref "validating-cue-kubernetes-configuration-with-kubeval.md" >}}) and [testing]({{< relref "testing-cue-configuration-with-open-policy-agent.md" >}}) that configuration using [Kubeval](https://github.com/garethr/Kubeval)
and [Conftest](https://github.com/igarethr/Kubeval).

Next we want to wrap that in a little automation. When writing our configuration we
probably want to be running it against a Kubernetes cluster as we make changes. For
that I'm going to use [Tilt](https://tilt.dev/).

Before we jump into the Tilt configuration I'll add another useful CUE commannd. Tilt doesn't
yet support CUE directly (not unexpected given the age of both projects) but we can
integrate them manually. To do so we need a command which will dump out the multi-file YAML
document to stdout for our CUE configuration. Save the following as `dump_tool.cue`:

{{< highlight "json" >}}
package kubernetes

import "encoding/yaml"

command dump: {
  task print: {
    kind: "print"
    text: yaml.MarshalStream(objects)
  }
}
{{< / highlight >}}


With that in place let's create a `Tiltfile`. `Tiltfile` uses another DSL, written in Skylark
which is a dialect of Python. I'm showing a very simple example here but you can do a lot more
if you check out the [API docs](https://docs.tilt.dev/api.html). Save the following as `Tiltfile`:

{{< highlight "python" >}}
read_file("deployment.cue")
read_file("policy/*.rego")
local("cue dump | kubeval")
local("cue dump | conftest -")
config = local("cue dump")
k8s_yaml(config)
{{< / highlight >}}

This should be reasonably simple to understand, but for clarify:

1. We use `read_file` to make sure Tilt knows to re-run whenever it sees a change to `deployment.cue`
2. We run both `cue validate` and `cue test`, though note that without [this issue](https://github.com/cuelang/cue/issues/30) being resolved this won't actual fail
3. We then run `cue dump` to get the YAML representation and store it in a variable called `config`
4. Finally we pass that configuration to the Kubernetes API to create the relevant objects

With that in place we can now run `tilt up` to run everything.

{{< highlight "python" >}}
$ tilt up
{{< / highlight >}}

Tilt provides a handy CLI user interface for interrogating logs and generally seeing what's
happening. This makes debugging your configuration easy and fast. Now whenever you change the 
`deployment.cue` file, or the OPA tests, it will re-run the validation, tests and redeploy to
your Kubernetes cluster.

## You like DSLs then?

I mentioned in the [first post of this series]({{< relref "configuring-kubernetes-with-cue.md" >}}) a certain love of DSLs. If you've followed along
with each post you might have spotted:

* CUE - for defining the configuration itself
* Rego - for writing the assertions used for testing the configuration
* Skylark - for writing the `Tiltfile`

I appreciate not everyone likes a good DSL, and that three of them combined together in this
way is going to cause some folks to get angry. I'm not saying this is the ideal workflow for
all (or any) teams to adopt. I don't think that ideal exists howeer, people and teams are
different and have different constrains and sensibilities. It's also why we won't all just
end up writing Lisp.

But if someone wants to say things about "not proper programming languages" or complain about
configuration vs code or pretend configuration can't be programmatically tested or validated
then feel free to point them to these posts.


