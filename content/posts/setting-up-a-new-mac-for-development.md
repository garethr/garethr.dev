---
title: "Setting Up a New Mac For Development"
date: 2019-05-26T16:50:17+01:00
---

With a new job (at [Snyk](https://snyk.io/)) comes the opportunity to setup a new machine from scatch. I've always taken a perverse pleasure in building development machines for myself. It's also the first time I've been back on a Mac (a MacBook Pro 13" to be precise) for a while so always new things to play with.


## Homebrew

The reality is I don't use much beyond a web browser and a [terminal](https://www.iterm2.com/) (partly a concious decision, it makes it easier to move between different operating systems and computers). On Mac [Homebrew](https://brew.sh/) is my go-to starting point. Here's a list of the various packages I have installed to start with.


```shell
$ brew list
adns                    libassuan               pcre2
asdf                    libevent                pinentry
autoconf                libffi                  pkg-config
automake                libgcrypt               python
bats                    libgpg-error            readline
conftest                libidn2                 rlwrap
coreutils               libksba                 skaffold
curl                    libtasn1                snyk
fish                    libtool                 sqlite
gdbm                    libunistring            terraform
gettext                 libusb                  tilt
git                     libxml2                 tmux
git-credential-netlify  libxslt                 tree
git-lfs                 libyaml                 unbound
gmp                     ncurses                 unixodbc
gnupg                   nettle                  unzip
gnutls                  npth                    wget
goreleaser              opa                     xz
hugo                    openssl                 zlib
kubeval                 p11-kit
```

Some of these are dependencies of other packages, or small system tools. The others of interest are:

* [fish](https://fishshell.com/) - I'm a convert to the Fish shell, mainly for the excellent defaults which keep configuration to an absolute minimum 
* [tmux](https://github.com/tmux/tmux) - TMUX is my go-to shell environment
* [bats](https://github.com/sstephenson/bats) - I still turn to bats regularly for high-level acceptance tests
* [terraform](https://www.terraform.io/) - I'm not a heavy Terraform user, but I do have an interest in Terraform tooling
* [snyk](https://snyk.io/) - :)
* [conftest](https://github.com/instrumenta/conftest) - My latest open source project, using Open Policy Agent to test structured data
* [kubeval](https://github.com/instrumenta/kubeval) - Another of my projects, Kubeval helps validate Kubernetes configurations using the upstream schemas
* [hugo](https://gohugo.io/) - I use Hugo for building this site, and a few other small sites I maintain
* [goreleaser](https://github.com/goreleaser/goreleaser) - GoReleaser is a fantastic tool for releasing Go projects
* [opa](https://www.openpolicyagent.org/) - I'm experimenting with Open Policy Agent for a few things at the moment, including conftest above
* [tilt](https://tilt.dev/) - A very handy tool for local development against Kubernetes


## ASDF

For installing various programming language environments I took a go at using [asdf](https://asdf-vm.com/#/) and I've been very impressed. So far I've installed the following.

```console
$ asdf list
clojure
  1.10.0
golang
  1.12.5
java
  openjdk-11.0.1
kotlin
  1.3.31
lua
  5.3.5
nodejs
  12.3.1
python
  3.7.3
racket
  7.3
ruby
  2.6.3
rust
  stable
```

The asdf version manager provides a similar interface to all of those platform specific tools (like `rbenv`, `pyenv`, `gvm`, `nvm`, etc.) but has a plugin system, and has plugins for most language environments you might be interested in.

```shell
$ asdf plugin-add lua https://github.com/Stratus3D/asdf-lua.git
# installs the plugin for managing lua
$ asdf list-all lua
# lists all available versions on lua that can be installed
$ asdf install lua 5.3.5
# installs a specific version
$ asdf global lua 5.3.5
# sets the version to be used everywhere that doesn't have a local override
```

## Unpackaged

On top of the command line tools and language toolchains I installed [Docker Desktop](https://www.docker.com/products/docker-desktop) (obviously) to provide a nice Docker environment. I also installed [Spectacle](https://www.spectacleapp.com/) as a simple keyboard-powered windows manager.

I installed a few things globally within the above development environments. Where I can avoid it I prefer not to do so, I'd much rather have standalone tools, but some things are new enough that they haven't release packages outside a development toolchain yet.

* [Krew](https://github.com/kubernetes-sigs/krew) - Krew provides a plugin manager for `kubectl` (custom installer)
* [Kind](https://kind.sigs.k8s.io/) - Running ephemeral Kubernetes clusters on top of Docker is fantastic for testing (Go module)
* [CUE](https://cue.googlesource.com/cue/) - A data constraint language well-suited to generating configuration (Go module)
* [Netlify](https://www.netlify.com/docs/cli/) - This and other sites I maintain run on netlify, so having the CLI installed is handy for management (NPM module)  
* [TypeScript](https://www.typescriptlang.org/) - The compiler and language toolchain builds atop the NodeJS tools (NPM module)



I'm sure I'll install more things as I go along, and I'm sure I'll have missed something that I'll remember the moment I publish this post, but all-in-all I'm up and running with a nice development machine quickly and fairly painlessly thanks to good package management tools and the work of lots of folks to package up and maintain packages for the wide various of software I use. 

