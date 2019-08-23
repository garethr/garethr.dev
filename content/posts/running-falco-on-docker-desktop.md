---
title: "Running Falco on Docker Desktop"
date: 2019-08-23T15:03:59+01:00
---

[Falco](https://falco.org/) is a handy open source project for intrusion and abnormality detection. It nicely integrates
with Kubernetes as well as other linux platforms. But trying it out with Docker Desktop (for Mac or Windows), either using
Docker, the built-in Kubernetes cluster, or with [Kind](https://kind.sigs.k8s.io) requires a new kernel module.

This is slighly complicated by the fact the VM used by Docker Desktop is really not intended for direct management by the end user. It's an implementation detail of how Docker Desktop works. The VM is also based on [LinuxKit](https://github.com/linuxkit/linuxkit) which is a toolkit for building small limited purpose operating systems.

Luckily, it is possible to work around the above. But I couldn't find this documented outside a [few GitHub issues](https://github.com/falcosecurity/falco/issues/657), and even then without much context. Hence this post.


## Building the Kernel module

In order to build and install the Kernel module we're going to use Docker itself. Kernel modules need to align with the
running Kernel, so the following Dockerfile takes several build args.

* `ALPINE_VERSION` - the version of the Alpine operating system, this should match the Docker Desktop version you are using
* `KERNEL_VERSION` - again, this needs to match the version in use by Docker Desktop
* `FALCO_VERSION` - the version of Falco you plan on using
* `SYSDIG_VERSION` - the version of Sysdig used by Falco

For the Alpine and Kernel versions you can check the release notes of the Docker Desktop version you are running. You
can check this online [for Mac](https://docs.docker.com/docker-for-mac/release-notes/) and [for Windows](https://docs.docker.com/docker-for-windows/release-notes/) or you can access them from the About menu where you should find a _Release notes_ link.

The following Dockerfile has default values for the latest versions of everything, including Docker Desktop 2.1.0.1 (37199) which uses Alpine 3.10 and a 4.9.184 Kernel.

```dockerfile
ARG ALPINE_VERSION=3.10
ARG KERNEL_VERSION=4.9.184

FROM alpine:${ALPINE_VERSION} AS alpine

FROM linuxkit/kernel:${KERNEL_VERSION} AS kernel

FROM alpine
ARG FALCO_VERSION=0.17.0
ARG SYSDIG_VERSION=0.26.4

COPY --from=kernel /kernel-dev.tar /

RUN apk add --no-cache --update wget ca-certificates \
    build-base gcc abuild binutils \
    bc \
    cmake \
    git \
    autoconf && \
  export KERNEL_VERSION=`uname -r  | cut -d '-' -f 1`  && \
  export KERNEL_DIR=/usr/src/linux-headers-${KERNEL_VERSION}-linuxkit/ && \
  tar xf /kernel-dev.tar && \
  cd $KERNEL_DIR && \
  zcat /proc/1/root/proc/config.gz > .config && \
  make olddefconfig && \
  mkdir -p /falco/build && \
  mkdir /src && \
  cd /src && \
  wget https://github.com/falcosecurity/falco/archive/$FALCO_VERSION.tar.gz && \
  tar zxf $FALCO_VERION.tar.gz && \
  wget https://github.com/draios/sysdig/archive/$SYSDIG_VERSION.tar.gz && \
  tar zxf $SYSDIG_VERSION.tar.gz && \
  mv sysdig-$SYSDIG_VERSION sysdig && \
  cd /falco/build && \
  cmake /src/falco-$FALCO_VERSION && \
  make driver && \
  rm -rf /src && \
  apk del wget ca-certificates \
    build-base gcc abuild binutils \
    bc \
    cmake \
    git \
    autoconf

CMD ["insmod","/falco/build/driver/falco-probe.ko"]
```

Building an image from the above Dockerfile will compile the kernel module, and then running the resulting image will install it. Note that we need to run the container as `--privileged` in order to install the module on the host VM.

```
docker build -t falco-docker-desktop .
docker run -it --rm --privileged falco-docker-desktop
```

If you're using a newer version of DOcker Desktop, or want to use a specific version of Falco, you can use the same Dockerfile and pass in build arguments. For instance if you are running the Edge version of Docker Desktop (2.1.1.0) this has a newer kernel. Currently you would want to build the image like so:

```
docker build --build-arg KERNEL_VERSION=4.14.131 -t falco-docker-desktop .
```

## Running Falco

With the module installed we can now try out Falco locally. The following is a very simple demonstration, using the built-in rules and just using the Docker runtime.

```console
$ docker run -e "SYSDIG_SKIP_LOAD=1" -it --rm --name falco --privileged -v /var/run/docker.sock:/host/var/run/docker
.sock -v /dev:/host/dev -v /proc:/host/proc:ro -v /lib/modules:/host/lib/modules:ro -v /usr:/host/usr:ro falcosecurity/falco

2019-08-23T14:32:52+0000: Falco initialized with configuration file /etc/falco/falco.yaml
2019-08-23T14:32:52+0000: Loading rules from file /etc/falco/falco_rules.yaml:
2019-08-23T14:32:52+0000: Loading rules from file /etc/falco/falco_rules.local.yaml:
2019-08-23T14:32:52+0000: Loading rules from file /etc/falco/k8s_audit_rules.yaml:
2019-08-23T14:32:53+0000: Starting internal webserver, listening on port 8765
2019-08-23T14:32:53.499871000+0000: Notice Container with sensitive mount started (user=<NA> command=container:814be8a1ef88 k8s_snyk-monitor_snyk-monitor-68d5f8d85f-vxqbn_snyk-monitor_643e8eb1-c33d-11e9-b290-025000000001_28 (id=814be8a1ef88) image=snyk/kubernetes-monitor:latest mounts=/var/lib/kubelet/pods/643e8eb1-c33d-11e9-b290-025000000001/volumes/kubernetes.io~secret/snyk-monitor-token-5zpp7:/var/run/secrets/kubernetes.io/serviceaccount:ro:false:rprivate,/var/lib/kubelet/pods/643e8eb1-c33d-11e9-b290-025000000001/etc-hosts:/etc/hosts::true:rprivate,/var/lib/kubelet/pods/643e8eb1-c33d-11e9-b290-025000000001/containers/snyk-monitor/41e63105:/dev/termination-log::true:rprivate,/var/run/docker.sock:/var/run/docker.sock::true:rprivate,/var/lib/kubelet/pods/643e8eb1-c33d-11e9-b290-025000000001/volumes/kubernetes.io~secret/docker-config:/root/.docker:ro:false:rprivate)
2019-08-23T14:32:53.514614000+0000: Notice Privileged container started (user=<NA> command=container:7603e8ff28a7 falco (id=7603e8ff28a7) image=falcosecurity/falco:latest)
```

Note Falco noticed a container starting which was mounting sensitive information from the host, and also noticed a privileged container starting. In this case that container was Falco itself :)

