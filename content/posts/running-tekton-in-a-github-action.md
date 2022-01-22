---
title: "Running Tekton in a Github Action"
date: 2019-08-30T16:54:42+01:00
---

I really like [Tekton](https://tekton.dev/). The project is at a fairly early stage, but is building the primitives
(around Tasks and Pipelines) that other higher-level (and user facing) tools can be built on. It's plumbing, with
lots of potential for sharing and abstraction. So my kind of project.

Now Tekton's abstractions are similar to GitHub Actions. Tasks broadly map to Actions and Pipelines to Workflows.
I'd love to see this broken out of the separate implementations and a general standard emerge. Portable abstractions
would allow for more innovation on top and less make work for integrators like me. For instance see the overlap in the
[Kubeval Action](https://github.com/instrumenta/kubeval-action) and the [Kubeval Task](https://github.com/tektoncd/catalog/tree/master/kubeval).
But that's a separate tangent to the one in this post. 

In this post I'm doing something questionable. I'm going to use GitHub Actions to:

1. Spin up an ephemeral Kubernetes cluster for each change
2. Install Tekton and some tasks on the cluster
3. Run a task
4. Grab the results

Now while you could use this to run Tekton tasks on GitHub instead of Actions, you would be paying quite a large performance
cost and wasting lots of compute cycles to do it. This is however useful in a few ways. It's handy if you're building
and testing Tekton tasks or pipelines. It's also useful as an example of the terrible things you can do with a platform
as flexible as GitHub Actions.

## Show me how already

Save the following at `.github/workflows/push.yml`:

```yaml
name: "Demonstrate using Tekton Pipelines in GitHub Actions"
on: [pull_request, push]

jobs:
  tekton:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
    - uses: engineerd/setup-kind@v0.1.0
    - name: Install jq
      run: |
        sudo apt-get install jq
    - name: Install Tekton
      run: |
        export KUBECONFIG="$(kind get kubeconfig-path)"
        kubectl apply -f https://storage.googleapis.com/tekton-releases/latest/release.yaml
    - name: Install Kubeval Tekton Task
      run: |
        export KUBECONFIG="$(kind get kubeconfig-path)"
        kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/kubeval/kubeval.yaml
    - name: Run Kubeval Task
      run: |
        export KUBECONFIG="$(kind get kubeconfig-path)"
        kubectl apply -f taskrun.yaml
        STATUS=$(kubectl get taskrun kubeval-example -o json | jq -rc .status.conditions[0].status)
        LIMIT=$((SECONDS+180))
        while [ "${STATUS}" != "Unknown" ]; do
          if [ $SECONDS -gt $LIMIT ]
          then
            echo "Timeout waiting for taskrun to complete"
            exit 2
          fi
          sleep 10
          echo "Waiting for taskrun to complete"
          STATUS=$(kubectl get taskrun kubeval-example -o json | jq -rc .status.conditions[0].status)
        done
    - name: Install Tekton CLI
      run: |
        curl -LO https://github.com/tektoncd/cli/releases/download/v0.2.2/tkn_0.2.2_Linux_x86_64.tar.gz
        sudo tar xvzf tkn_0.2.2_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
    - name: Get TaskRun Logs
      run: |
        export KUBECONFIG="$(kind get kubeconfig-path)"
        tkn taskrun logs kubeval-example -a -f
    - name: Result
      run: |
        export KUBECONFIG="$(kind get kubeconfig-path)"
        REASON=$(kubectl get taskrun kubeval-example -o json | jq -rc .status.conditions[0].reason)
        echo "The job ${REASON}"
        test ${REASON} != "Failed"
```

You can see this running in [garethr/tekton-in-github-actions](https://github.com/garethr/tekton-in-github-actions/commit/eab21df93836348c97a34f3d263a0873a879a27e/checks).


## This is terrible isn't it?

It is. Mixing bash and YAML is however how the internet works now. The above is also really just a proof-of-concept.
There are a number of things we can do to make the above nicer for general usage. To begin with we can abstract it
away behind an action! I think you could probably get the above down to something like the following:

```yaml
- uses: actions/checkout@master
- uses: tekton/setup-tekton@v0.1.0
- uses: tekton/run-tekton-task@v0.1.0
  with:
    task: taskrun.yaml
```

If you don't like the idea of hiding all of that YAML and bash behind some more YAML and a Docker image then please
don't look at how most software works :)

The above example also has at least one more flaw. If you take a look at the `TaskRun` spec, it's actually testing code from a different
repository rather than this one. Now Actions happily provides the relevant details as environment variables, namely
`GITHUB_REPOSITORY` and `GITHUB_SHA`. With a bit of `envsubst` or `sed` you could swap in the relevant local values into
the `TaskRun` template.

Happily, you can also now implement Actions in [TypeScript](https://github.com/actions/javascript-template), so you could write an action in a real programming language using
the Kubernetes client libraries. That would be the best approach for any sense of maintainability if someone wanted
to make this a real thing.


## In summary

Thanks to Radu for the [Kind Action](https://github.com/marketplace/actions/kind-kubernetes-in-docker-action), he also has a [nice post up on writing it](https://radu-matei.com/blog/building-github-actions/).
This post mainly highlights how far you can stretch GitHub Actions without it breaking, which is always a useful property
in my book. It does mean folks are going to do terrible things. All those things people did with Jenkins? They are now in the cloud
and powered by more events.

More usefully, this also demonstrates how useful GitHub Actions could be in testing anything that integrates with Kubernetes.
Kind was build specifically for this usecase, and being able to run it close to the code and suitably abstracted away is
powerful for anyone building clients or tools which use the API.




