---
title: "Validating Cue Kubernetes Configuration with Kubeval"
date: 2019-04-11T10:00:00Z
keywords: cue
---

In the [previous post]({{< relref "configuring-kubernetes-with-cue.md" >}}) I introduced using [CUE](https://github.com/cuelang/cue) for
managing Kubernetes configuration. In this post we'll start building a simple workflow.

One of the features of CUE is a declarative scripting language. This can be used to
add your own commands to the `cue` CLI. Script files are named `*_tool.cue` and are
automatically loaded by the CUE tooling.

First lets lay some of the ground work. This is taken from the [Kubernetes tutorial](https://github.com/cuelang/cue/tree/master/doc/tutorial/kubernetes)
and converts our map of Kubernetes objects to a list.

Save the following as `kubernetes_tool.cue`:

{{< highlight "json" >}}
package kubernetes

objects: [ x for x in deployment  ]
{{< / highlight >}}

We've saved the above in a separate file so it can be reused by other tools more easily.
Next we define our `validate` command. For this we're using [Kubeval](https://github.com/garethr/kubeval).
Save the following as `validate_tool.cue`:

{{< highlight "json" >}}
package kubernetes

import "encoding/yaml"

command validate: {
  task kubeval: {
    kind:   "exec"
    cmd:    "kubeval --filename \"cue\""
    stdin:  yaml.MarshalStream(objects)
    stdout: string
  }
  task display: {
    kind: "print"
    text: task.kubeval.stdout
  }
}
{{< / highlight >}}

Commands are quite powerful, allowing for defining flags and arguments, as well as providing
inline help and usage examples. The only place this is documented is in the [source](https://github.com/cuelang/cue/blob/2b0e7cd9f63a190e762d7c802b98528ff80dcb7c/cmd/cue/cmd/cmd.go#L31)
and I've not been able to get it all working quite yet, but this should be familiar if you've
build tools using a CLI framework like Cobra in Go before.

With our command defined, what can we now do? We can run it:

{{< highlight "script" >}}
$ cue validate
The document "cue" contains a valid Deployment
{{< / highlight >}}

A few things happened here:

1. Our `deployment` defined in CUE was evaluated
2. The map data structure was flatted and converted to a list
3. The list of objects was conerted into a multi-file YAML document
4. That document was piped into `kubeval`

Their is one caveat with the above, the exit code isn't passed through. So if
Kubeval finds an error it will return a non-zero exit code. But CUE doesn't yet support
passing that along, although I have now [opened an issue](https://github.com/cuelang/cue/issues/30).

The nice thing about defining aspects of the workflow in the authoring tool is consistency
and discoverability. Early adopters might like remembering a bewildering number of discreet
tools but it's nice to build up a considered user interface. CUE doesn't support everything
needed to make this happen as yet, but I did mention that it's very new.


## Future ideas

Kubeval is really a thin wrapper around validation using the [Kubernetes JSON Schema](https://github.com/instrumenta/kubernetes-json-schema).
It should be possible to convert JSON Schema to valid CUE templates. That would remove the
need for this step completely as evaluating the CUE definitions would catch any issues.
I think this should have generic utility for formats where you already have a JSON Schema handy
as well as being specifically useful for Kubernetes. I think CUE has some code in the repository
looking at generating CUE templates from Go types. That sounds useful, but I think CUE has
potential outside just Go (and ask me to rant about the Kubernetes Go client versus the
OpenAPI definitions anytime.)
