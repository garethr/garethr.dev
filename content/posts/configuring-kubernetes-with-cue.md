---
title: "Configuring Kubernetes with CUE"
date: 2019-04-11T09:00:00Z
keywords: cue
---


My interest in Kubernetes has always been around the API. It's the potential of a unified API
and set of objects that keeps me coming back to hacking on Kubernetes and building tools
around it. If I think about it, I find that appealing because I've also spend time automating
the management of operating systems which lack anything like a good API for doing so.

I'm also a _little_ obsessed with domain specific languages and general configuration
management topics. I'll happily talk theory, but I'm also an observer of real world
practice. So while somewhere in my head I'm wondering what a configuration language built on
prolog would look like, I also can't help but appreciate people would probably still
prefer to write YAML by hand.

One (very) recent new tool that piqued my interest was the new language [CUE](https://cue.googlesource.com/cue).

> CUE is an open source data constraint language which aims to simplify tasks involving
> defining and using data. It is a superset of JSON, allowing users familiar with JSON
> to get started quickly.

The official documentation suggests a few usecases for CUE:

>  * define a detailed validation schema for your data (manually or automatically from data)
>  *  reduce boilerplate in your data (manually or automatically from schema)
>  *  extract a schema from code
>  *  generate type definitions and validation code
>  *  merge JSON in a principled way
>  *  define and run declarative scripts

Before we get started, a few caveats. CUE is very new, with the first public commits in the
Git repository coming in November last year. I can find zero CUE code (with a `.cue` extension)
anywhere public on GitHub. Installation requires knowledge of Go, there are no official releases
or packages. I've also not contributed to the project, so the following is based on a bit of
experimenting rather than any intimate knowledge. CUE appears to mainly be the work of
[Marcel van Lohiuzen](https://twitter.com/mpvl) from Google and the Go team.


## CUE for Kubernetes

With that out of the way, how about an example?

The documentation already has a [Kubernetes tutorial](https://cue.googlesource.com/cue/+/HEAD/doc/tutorial/kubernetes/README.md)
which covers some of the same ground as this post, but I wanted to explain a few things
differently, and to then build on the example with a few new tricks. If you find this post
interesting then definitely read the official docs too.

Note as well CUE isn't specific to Kubernetes. It can be used for authoring any structured
configuration. CloudBuild or Circle CI configs, Cloud Init, CloudFormation, 
Azure Resource Manager templates, etc. CUE is a tool for authoring configuration, not another
serialisation wire format.

Let's start with a simple `deployment` configuration in YAML.

{{< highlight "yaml" >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.5
        ports:
        - containerPort: 8080
{{< / highlight >}}

CUE comes with tools to help you get started if you already have existing configs. Let's run one here.


{{< highlight "shell" >}}
$ cue import deployment.yaml
$ ls
deployment.cue    deployment.yaml
{{< / highlight >}}

`cue import` converted our configuration into CUE. Let's have a look at what that looks like:

{{< highlight "yaml" >}}
apiVersion: "apps/v1"
kind:       "Deployment"
metadata name: "hello-kubernetes"
spec: {
  replicas: 3
  selector matchLabels app: "hello-kubernetes"
  template: {
    metadata labels app: "hello-kubernetes"
    spec containers: [{
      name:  "hello-kubernetes"
      image: "paulbouwer/hello-kubernetes:1.5"
      ports: [{
        containerPort: 8080
      }]
    }]
  }
}
{{< / highlight >}}

As mentioned above, CUE is a superset of JSON. But note a few differences focused on usability:

* No outer braces
* No trailing commas
* Single line format for nested statements, eg. `selector matchLabels app: "hello-kubernetes"`

## CUE templates

YAML is *just* a serialisation format. To cut down on repetition when managing multiple configuration files
people often apply a templating tool on top. CUE makes data templating a first class part of
the language. I'll only scratch the surface of what's possible here but let's see a simple example
by building on the above automatically converted configuration.


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

Here we have created a `deployment` template which we can reuse, and then used it to define
a concrete deployment called `hello-kubernetes`. A few things to note:

* `apiersion` is specified as a `string` in the template. If it is omitted, or isn't a string, evaluation of the CUE configuration will fail
* `replicas` is specified as defaulting to `1` or taking an `int` value
* The name of the deployment is placed in a variable called `Name` which is then used to populate the metadata and labels

In this simple case, where we have only a single deployment, this may seen a little over the top.
But remember we can reuse the template for lots of deployments. That makes it easy to inject
attributes into all types, or make some attributes required or limited and more. When authoring
YAML the language just sees arbitrary data, with CUE you can introduce semantics, which means
you can reason about your configuration in the language you're writing.

We can evaluation our slightly more abstract configuration, and check our template is working: 

{{< highlight "shell" >}}
$ cue eval
{
    deployment "hello-kubernetes": {
        apiVersion: "apps/v1"
        kind:       "Deployment"
        metadata name: "hello-kubernetes"
        spec: {
            replicas: 3
            selector matchLabels app: "hello-kubernetes"
            template: {
                metadata labels app: "hello-kubernetes"
                spec containers: [{
                    name:  "hello-kubernetes"
                    image: "paulbouwer/hello-kubernetes:1.5"
                    ports: [{
                        containerPort: 8080
                    }]
                }]
            }
        }
    }
}
{{< / highlight >}}

## Exporting to JSON

As noted, CUE is an authoring tool. It's not intended to replace JSON or YAML or other
serialisation formats directly. It supports the concept of exporting to those formats for
use in other tools. Currently CUE only supports exporting to JSON, but we'll look at some
ways around that in a following post.

{{< highlight "shell" >}}
$ cue export
{
  "deployment": {
    "hello-kubernetes": {
      "apiVersion": "apps/v1",
      "kind": "Deployment",
      "metadata": {
        "name": "hello-kubernetes"
       },
      "spec": {
        "replicas": 3,
          "selector": {
          "matchLabels": {
            "app": "hello-kubernetes"
          }
       },
       "template": {
         "metadata": {
           "labels": {
             "app": "hello-kubernetes"
           }
        },
        "spec": {
          "containers": [
            {
              "name": "hello-kubernetes",
              "image": "paulbouwer/hello-kubernetes:1.5",
              "ports": [
                {
                  "containerPort": 8080
                }
              ]
            }
          }
        }
      }
    }
  }
}
{{< / highlight >}}


## Conclusions

I've shown the basics of CUE here using a simple Kubernetes configuration file. The
full [CUE tutorial](https://github.com/cuelang/cue/tree/master/doc/tutorial/basics) covers
more of the language features too.

For me CUE appears a powerful mix of data-centric authoring with built-in validation
and templating. It's pleasant enough even for single examples but it's features are
particularly powerful when managing large amounts of configuration, where introducing
powerful abstractions can drastically cut down the amount of configuration needing to be
managed.

Next I'll look at integrating other tools with CUE, to build a workflow supporting further
testing and validation.

