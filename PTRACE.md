![Ampere Computing](https://avatars2.githubusercontent.com/u/34519842?s=400&u=1d29afaac44f477cbb0226139ec83f73faefe154&v=4)

# gVisor on Ampere

## Table of Contents
* [Introduction](#introduction)
* [Requirements](#requirements)
  * [Operating System](#operating-system)
  * [Kernel 5.5. or Newer](#kernel-5-5-or-newer)
* [Getting started](#getting-started)
  * [Download the gVisor Code](#download-the-gvisor-code)
  * [Modifications for Arm64](#modifications-for-arm64)
* [Build gVisor](#build-gvisor)
* [Make the compiled binary available for use:](#make-the-compiled-binary-available-for-use)
* [Update Docker for the gVisor Runtime](#update-docker-for-the-gvisor-runtime)
* [Run gVisor](#run-gvisor)
* [Further Reading](REFERENCES.md)



## Introduction

Here at Ampere Computing we are always interested in emerging cloud native technologies as interesting workloads for our cloud optimized Ampere(R) Altra(TM)Arm64 processors. [gVisor](https://github.com/google/gvisor) is an active open source software project originally created by Google that provides an application kernel for containers. The [gVisor](https://github.com/google/gvisor) project ecosystem and community of [contributors](https://github.com/google/gvisor/graphs/contributors) continues to make regular improvements to gVisor for ARM64 platforms.  In fact, in recent months, [improvements](https://github.com/google/gvisor/pulse)  have been coming in almost weekly.  

From a technology perspective, [gVisor](https://github.com/google/gvisor) was written in [Golang](https://golang.org/) and is often used as a method to sandbox application containers by providing a substantial portion of the Linux operating system surface. Sandboxing can be thought of as an isolation boundary for greater security between the application and the Linux kernel running on the host.  This isolation boundary, or `sandbox` between the application and the host kernel is provided by gVisor's `runsc` binary.  The `runsc` binary implements an [Open Container Initiative (OCI)](https://opencontainers.org/) compliant runtime which integrates with Docker and Kubernetes, making it simple to run sandboxed containers in cloud native environments.

gVisor currently requires an abstraction which it calls a `platform` to implement the sandboxing mechanisms, currently the avaiable platforms are `ptrace` and `KVM`.  There are different tradeoffs between each `Platform` which generally are focused around performance and hardware requirements for running gVisor.  To describe it in an overly simplistic way, the platform implemention depends on the context in which `runsc` is executing, for example virtualized platforms would be limited to systems that do not require hardware virtualization.  The `ptrace` platform executes user code with out allowing it to execute host system calls, whereas the KVM platform uses kernel and hardware virtualization to allow gVisor to act as both guest OS and Virtual Machine Manager.  For more information regarding gVisor platforms please consult with upstream documenation on the subject located here:

* [https://gvisor.dev/docs/architecture_guide/platforms/#implementations](https://gvisor.dev/docs/architecture_guide/platforms/#implementations)

In this post, we will discuss how to build, install and run [gVisor](https://github.com/google/gvisor) on Ampere(R) Altra(TM) Arm64 processors optimized for cloud workloads using `ptrace` as the gvisor platform.

## Requirements


### Operating System

At the time of this writing Ubuntu 20.04 was the ideal choice of OS gVisor based on the upstream documentation.  Installing Ubuntu is outside the scope of this document.  However the Ubuntu 20.04.1 server install image for Arm64 can be downloaded from the following URL:

* [http://cdimage.ubuntu.com/ubuntu/releases/20.04.1/release/ubuntu-20.04.1-live-server-arm64.iso](http://cdimage.ubuntu.com/ubuntu/releases/20.04.1/release/ubuntu-20.04.1-live-server-arm64.iso)

Additionally further instructions for installing Ubuntu on Arm64 can be found in the *Offical Ubuntu instalation Guide for Arm64* which can be found at the following url:

* https://help.ubuntu.com/lts/installation-guide/arm64/index.html

For more in depth details on the OS requirements and installation of gVisor please refer to the upstream documenation which can be found here:

* [https://gvisor.dev](https://gvisor.dev)


### Kernel 5.5 or newer

As discussed earlier, gVisor requires a platform to implement the intercepting sandboxing mechanisms, and currently `ptrace` and `KVM` are implemented.
If you are planning on using gVisor with KVM, the Linux Kernel Virtual Machine, then it is necessary to use a specific Linux Kernel version which includes the necessary patches to enable usage on Arm64.  Currently the Linux Kernel 5.5 includes the necessary patch for KVM.  Further information and reading on the patches that for enabling usage of KVM as the gVisor platform please consult the information located here: 

* [https://lore.kernel.org/kvmarm/20191206020802.196108-1-justin.he@arm.com/t/#u](https://lore.kernel.org/kvmarm/20191206020802.196108-1-justin.he@arm.com/t/#u
)

Obviously because the patches are already included in kernels 5.5 and newer we strong suggest staying on newer kernels for the best results.
Please note: using ptrace with older kernels can be used, and may work however your results and experience may vary.  Additinoally please note that the current gVisor implementations only support kernels configured with 4K pages sizes.

## Getting started

### Download the latest gVisor Code

For the best experience with building gvisor, it is recommend that you pull the `master` branch directly from the [upstream source code](https://github.com/google/gvisor) located on on GitHub.

```
$ git clone https://github.com/google/gvisor.git
Cloning into 'gvisor'...
remote: Enumerating objects: 194, done.
remote: Counting objects: 100% (194/194), done.
remote: Compressing objects: 100% (155/155), done.
remote: Total 91378 (delta 80), reused 85 (delta 39), pack-reused 91184
Receiving objects: 100% (91378/91378), 60.10 MiB | 25.60 MiB/s, done.
Resolving deltas: 100% (68220/68220), done.
```

### Modifications for Arm64

Some changes are necessary to sucessfully build Arm64 binaries of gVisor.
Locate the WORKSPACE file in the root level of the `gvisor` project directory you cloned with git.

```
$ ls -n
total 232
-rw-rw-r--  1 1000 1000   365 Nov  6 13:31 AUTHORS
-rw-rw-r--  1 1000 1000  3317 Nov  6 13:31 BUILD
-rw-rw-r--  1 1000 1000  4481 Nov  6 13:31 CODE_OF_CONDUCT.md
-rw-rw-r--  1 1000 1000  4328 Nov  6 13:31 CONTRIBUTING.md
-rw-rw-r--  1 1000 1000  5410 Nov  6 13:31 GOVERNANCE.md
-rw-rw-r--  1 1000 1000 12519 Nov  6 13:31 LICENSE
-rw-rw-r--  1 1000 1000 19957 Nov  6 13:31 Makefile
-rw-rw-r--  1 1000 1000  4510 Nov  6 13:31 README.md
-rw-rw-r--  1 1000 1000   466 Nov  6 13:31 SECURITY.md
-rw-rw-r--  1 1000 1000 42095 Nov  6 13:31 WORKSPACE             <-- EDIT THIS FILE
drwxrwxr-x  2 1000 1000  4096 Nov  6 13:31 debian
drwxrwxr-x  4 1000 1000  4096 Nov  6 13:31 g3doc
-rw-rw-r--  1 1000 1000  3238 Nov  6 13:31 go.mod
-rw-rw-r--  1 1000 1000 49126 Nov  6 13:31 go.sum
drwxrwxr-x 10 1000 1000  4096 Nov  6 13:31 images
-rw-rw-r--  1 1000 1000  9639 Nov  6 13:31 nogo.yaml
drwxrwxr-x 58 1000 1000  4096 Nov  6 13:31 pkg
drwxrwxr-x 13 1000 1000  4096 Nov  6 13:31 runsc
drwxrwxr-x  4 1000 1000  4096 Nov  6 13:31 shim
drwxrwxr-x 18 1000 1000  4096 Nov  6 13:31 test
drwxrwxr-x 15 1000 1000  4096 Nov  6 13:31 tools
drwxrwxr-x  2 1000 1000  4096 Nov  6 13:31 vdso
drwxrwxr-x  3 1000 1000  4096 Nov  6 13:31 webhook
drwxrwxr-x 11 1000 1000  4096 Nov  6 13:31 website
```
It is necessary to modify the WORKSPACE file located in the root of the source project
directory previously cloned from git sources with information needed to download arm64
binaries as part of the building process. Using a text editor open the WORKSPACE file.
To configure the download of the arm64 `Golang` binaries you must first locate the
following lines in the WORKSPACE file:

```
load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")

go_rules_dependencies()

go_register_toolchains(go_version = "1.15.2")
```

Replace the lines with the text below:

```
load("@io_bazel_rules_go//go:deps.bzl", "go_download_sdk", “go_rules_dependencies")

go_rules_dependencies()

go_download_sdk(name = "go_sdk", urls=["https://golang.org/dl/{}",],
 sdks = {
 "linux_arm64": ("go1.15.linux-arm64.tar.gz", "7e18d92f61ddf480a4f9a57db09389ae7b9dadf68470d0cb9c00d734a0c57f8d"),
 },
)
```

## Build gVisor

Once you have successfully edited the WORKSPACE file replacing the information needed to 
successfully retrieve the necessary arm64 binaries you are ready to begin building gVisor
on Ampere platforms.


```
make runsc
```

At the end of the compile, the output will have a statement similar to the following,
which indicates the location of the runsc binary:

```
Target //runsc:runsc up-to-date:
  bazel-out/aarch64-opt-ST-5e46445d989a/bin/runsc/runsc_/runsc
```

## Make the compiled binary available for use:

```
sudo rm -f /usr/local/bin/runsc
sudo ln -s $(pwd)/<binary location from above> /usr/local/bin
```

## Update Docker for the gVisor Runtime

Update /etc/docker/daemon.json as follows:

```
{
  "runtimes": {
    "runsc-ptrace": {
      "path": "/usr/local/bin/runsc",
      "runtimeArgs": [
        "--platform=ptrace"
       ]
     },
     “runsc-kvm": {
       "path": "/usr/local/bin/runsc",
       "runtimeArgs": [
         "--platform=kvm"
        ]
     } 
}
```
### Run gVisor

```
docker run --runtime=runsc-ptrace hello-world
```
