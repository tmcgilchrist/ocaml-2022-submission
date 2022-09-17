OCaml workshop OBuilder presentation
==========

Good afternoon everyone, my name is Tim McGilchrist. I am please to be presenting this joint work from many people at Tarides. Any mistakes or inconsistencies are entirely my own.

Introduction 
----------

 * present a lightweight sandboxing solution that we call OBuilder
 * support for macOS, Windows and the BSDs without requiring virtualisation
 * this work is in production providing infrastructure for
   opam-health-check, opam-repo-ci and ocaml-ci.

Background
----------

 * running OCaml on many operating systems and architectures
 * various continuous integration and build systems
 * common build specification
 * support for Linux, macOS, Windows and the BSDs
 * multi-architecture (x86, ARM64, PPC64, s390x)

KC mentioned checks.ocamllabs.io as one example but there are many other systems.

Build specification aka build spec
----------
> OBuilder is designed to take a build script and perform the steps in it inside a sandboxed environment

``` common-lisp
((build dev
  ((from ocaml/opam:alpine-3.15-ocaml-4.14)
   (user (uid 1000) (gid 1000))
   (workdir /home/opam)
   (run (shell "echo 'print_endline {|Hello, world!|}' > main.ml"))
   (run (shell "opam exec -- ocamlopt -ccopt -static -o hello main.ml"))))
 (from alpine:3.15)
 (shell /bin/sh -c)
 (copy (from (build dev))
       (src /home/opam/hello)
       (dst /usr/local/bin/hello))
 (run (shell "hello")))
```

 * take a build spec and execute it inside a sandbox
 * create a snapshot and cache it for each line
 * compiles a simple hello world OCaml program using Alpine and OCaml 4.11
 * zooming out from the build spec, how does it get run? 

oCluster architecture
----------

                             OCluster
                          ┌────────────────────────┐
 ┌──────────┐             │                        │
 │ ocaml-ci ├────────────►│  ┌─────────────────┐   │
 └──────────┘             │  │    scheduler    │   │
                          │  └─┬───────────────┘   │
                          │    │                   │
          * submission    │    │      pools        │
            capability    │    │   ┌────────────┐  │
                          │    │   │  x86 linux │  │
          * obuilder spec │    ├───┤  worker    │  │
                          │    │   │            │  │
                          │    │   ├────────────┤  │
                          │    │   │  x86 macos │  │
 ┌────────────┐           │    └───┤  worker    │  │
 │opam-repo-ci├──────────►│        │            │  │
 └────────────┘           │        └────────────┘  │
                          │                        │
                          └────────────────────────┘
                          1 or more workers per pool

 * digress here and show the big picture
 * various clients require builds
 * ocluster providing a build cluster of pools
 * pools representing a group of workers with similiar capabilities
 * workers execute the build spec in a platform specific way

oBuilder architecture
----------

     OCluster worker
  ┌────────────────────────────────────┐
  │                                    │
  │ ┌────────────────────┐             │
  │ │  * scheduler capnp │             │
  │ │  * control logic   │             │
  │ └────────────────────┘             │
  │                                    │
  ├────────────────────────────────────┤
  │   OBuilder library                 │
  │    ┌───────────────┐               │
  │    │  build logic  │  ┌──────────┐ │
  │    │               ├──► fetcher  │ │
  │    └───┬────────┬──┘  └──────────┘ │
  │        │        │                  │
  │ ┌──────▼───┐ ┌──▼───────┐          │
  │ │ sandbox  │ │  store   │          │
  │ └──────────┘ └──────────┘          │
  │                                    │
  └────────────────────────────────────┘

 * responsible for executing a build spec
 * coordinates communication with the cluster
 * provides a sandbox, store and fetcher
 * workers are platform specifc, providing a native 
   implementation on that platform
 * Let's look at the various implementations

linux implemention
----------

     OCluster worker  (linux)
  ┌───────────────────────────────────────────────────────┐
  │                                                       │
  │ ┌────────────────────┐         sandbox                │
  │ │                    │   ┌──────────────────────────┐ │
  │ │  * scheduler capnp │   │                          │ │
  │ │  * control logic   │   │    * runc containers     │ │
  │ │  * build logic     │   │                          │ │
  │ │                    │   │                          │ │
  │ └────────────────────┘   └──────────────────────────┘ │
  │                                                       │
  │                                store                  │
  │       fetcher            ┌──────────────────────────┐ │
  │   ┌─────────────┐        │                          │ │
  │   │             │        │    * Btrfs filesystem    │ │
  │   │  * docker   │        │    * ZFS filesystem      │ │
  │   │             │        │                          │ │
  │   └─────────────┘        └──────────────────────────┘ │
  │                                                       │
  └───────────────────────────────────────────────────────┘

Sandbox
 * provided by runc and Linux's native containerisation
 * based on Linux’s native namespaces and cgroups functionality
 * the key feature of namespaces is that they isolate processes
 * while cgroups control how much of a given key resource a process has access to
 
Store
 * provided by either ZFS or Btrfs
 * Btrfs store uses Btrfs subvolumes to snapshot and restore filesystem state
 * ZFS store uses the native snapshotting support in ZFS
 
 * implementation is obvious

macOS implementation
----------

``` ocaml

    OCluster worker   (macOS)
 ┌───────────────────────────────────────────────────────┐
 │                                                       │
 │ ┌────────────────────┐         sandbox                │
 │ │                    │   ┌──────────────────────────┐ │
 │ │  * scheduler capnp │   │                          │ │
 │ │  * control logic   │   │    * macOS user account  │ │
 │ │  * build logic     │   │    * FUSE filesystem     │ │
 │ │                    │   │                          │ │
 │ └────────────────────┘   └──────────────────────────┘ │
 │                                                       │
 │                                store                  │
 │       fetcher            ┌──────────────────────────┐ │
 │   ┌─────────────┐        │                          │ │
 │   │             │        │    * rsync file copy     │ │
 │   │  * docker   │        │                          │ │
 │   │             │        │                          │ │
 │   └─────────────┘        └──────────────────────────┘ │
 │                                                       │
 └───────────────────────────────────────────────────────┘

```

 * macOS is not a server operating system, terrible support for 
   server features.

Sandbox
 * provided by user accounts
 * complications with opam and homebrew
 * opam supports multiple opam roots on a system
 * homebrew tricked by file system redirection, per user homebrew install

Store
 * rejected option of ZFS, FUSE and ZFS bugs
 * store provided by rsync and file copying
 * snapshots store in a directory with keys

Windows implementation
----------

```
    OCluster worker   (windows)
 ┌───────────────────────────────────────────────────────┐
 │                                                       │
 │ ┌────────────────────┐          sandbox               │
 │ │                    │   ┌──────────────────────────┐ │
 │ │  * scheduler capnp │   │                          │ │
 │ │  * control logic   │   │ * windows base container │ │
 │ │  * build logic     │   │                          │ │
 │ │                    │   │                          │ │
 │ └────────────────────┘   └──────────────────────────┘ │
 │                                                       │
 │                                store                  │
 │       fetcher            ┌──────────────────────────┐ │
 │   ┌─────────────┐        │                          │ │
 │   │             │        │  * docker tag            │ │
 │   │  * docker   │        │  * docker image          │ │
 │   │             │        │                          │ │
 │   └─────────────┘        └──────────────────────────┘ │
 │                                                       │
 └───────────────────────────────────────────────────────┘
 
```

We could have used user accounts but chose something else.
Native Windows containerisation.

Sandbox
 * uses a Windows docker container with opam and ocaml 
 * runs the build spec inside the container

Store
 * docker tag the running container
 * store the resulting image under the computed key for that build step
 
Many bugs in LWT, Standard Library and GNU tar. 
Future work
----------

For macOS we are submitting builds from opam-repo-ci, these builds turn up as distributions in the CI report.
Our next step is to include ocaml-ci and provide builds for macOS Monterey on x86. This will start stressing the current MacMini machines we are using and we can optimise the current setup. Looking at supporting more than 1 concurrent build per machine and revisit the ZFS support perhaps see if we can isolate the stability issues there.

For the Windows builders, full opam-repo-ci support is the next step. These will again show up as a specific Windows variant on x86 and provide validation that submitted opam packages do work on Windows if they claim to. Then again rolling out support for ocaml-ci to submit Windows jobs. Windows being a very different architecture non-Unix to out other supported platforms will provide more interesting support. 

Finally I mentioned BSD support, we have looked at using runj, which provides an OCI container environment 
on FreeBSD using the native jails and resource controls. Coupled with the ZFS store we already have in obuilder. This looks like a promising solution. runj is experimental but we hope it will become more solid and we can use that for obuilder.









