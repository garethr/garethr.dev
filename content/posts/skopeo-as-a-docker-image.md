---
title: "Skopeo as a Docker Image"
date: 2019-06-02T12:15:57+01:00
---

[Skopeo](https://github.com/containers/skopeo) is a handy tool for interogating OCI registries. You can inspect the image manifests and copy images between various stores.
I found myself wantting to use Skopeo in the context of a container, and having searched on Hub mainly found either out-of-date images or images designed for a slightly different purpose.
Maintaining images is non-trivial, and sometimes it's better to just let folks know how to build there own. So for anyone else in need of such a thing, here is a Skopeo image based on Alpine linux.

```dockerfile
FROM golang:1.12-alpine AS builder

RUN apk add --no-cache \
    git \
    make \
    gcc \
    musl-dev \
    btrfs-progs-dev \
    lvm2-dev \
    gpgme-dev \
    glib-dev || apk update && apk upgrade

WORKDIR /go/src/github.com/containers/skopeo
RUN git clone https://github.com/containers/skopeo.git .
RUN make binary-local-static DISABLE_CGO=1


FROM alpine:3.7
run apk add --no-cache ca-certificates
COPY --from=builder /go/src/github.com/containers/skopeo/skopeo /usr/local/bin/skopeo
COPY --from=builder /go/src/github.com/containers/skopeo/default-policy.json /etc/containers/policy.json
ENTRYPOINT ["/usr/local/bin/skopeo"]
CMD ["--help"]
```

Using the image is straightforward if you're familiar with Skopeo and Docker. You can inspect an image like so:

```shell
$ docker run -it --rm garethr/skopeo inspect docker://docker.io/fedora
{
    "Name": "docker.io/library/fedora",
    "Digest": "sha256:2a60898a6dd7da9964b0c59fedcf652e24bfff04142e5488f793c9e8156afd33",
    "RepoTags": [
        "20",
        "21",
        "22",
        "23",
        "24",
        "25",
        "26-modular",
        "26",
        "27",
        "28",
        "29",
        "30",
        "31",
        "branched",
        "heisenbug",
        "latest",
        "modular",
        "rawhide"
    ],
    "Created": "2019-03-12T00:20:38.300667849Z",
    "DockerVersion": "18.06.1-ce",
    "Labels": {
        "maintainer": "Clement Verna <cverna@fedoraproject.org>"
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:01eb078129a0d03c93822037082860a3fefdc15b0313f07c6e1c2168aef5401b"
    ]
}
```

Copying is a little more complicated, assuming you want to save the image locally you'll need to mount an empty directory, or keep the image around and use `docker cp`. For instance:


```shell
$ docker run -it --rm -v $PWD/data:/data garethr/skopeo copy docker://alpine:latest oci:data/alpine:latest                                                                   Sun  2 Jun 12:24:39 2019
Getting image source signatures
Copying blob e7c96db7181b done
Copying config 8c79fc7093 done
Writing manifest to image destination
Storing signatures
```

That should have saved the image like so:

```shell
$ tree data/                                                                                                                                                                 Sun  2 Jun 12:27:14 2019
data/
└── alpine
    ├── blobs
    │   └── sha256
    │       ├── 63aec9aa7e327bf2359eca3a8345b1678b6a592241916427dbeb1d884eb3cda2
    │       ├── 8c79fc709348f9fdb30cc2dc10999b30e095fc7cad8aa9320e8834aca05f7740
    │       └── e7c96db7181be991f19a9fb6975cdbbd73c65f4a2681348e63a141a2192a5f10
    ├── index.json
    └── oci-layout

3 directories, 5 file
```
