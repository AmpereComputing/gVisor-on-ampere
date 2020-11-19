![Ampere Computing](https://avatars2.githubusercontent.com/u/34519842?s=400&u=1d29afaac44f477cbb0226139ec83f73faefe154&v=4)

# gVisor on Ampere

## Table of Contents
* [Introduction](#introduction)
* [Requirements](#requirements)
  * [Operating System](#operating-system)
  * [Kernel 5.5. or Newer](#kernel-5-5-or-newer)
  * [Bazelbuild binaries](#bazelbuild-binaries)
* [Download the gVisor Code](#download-the-gvisor-code)
* [Build gVisor](#build-gvisor)
* [Make the compiled binary available for use:](#make-the-compiled-binary-available-for-use)
* [Update Docker for the gVisor Runtime](#update-docker-for-the-gvisor-runtime)
* [Run gVisor](#run-gvisor)
* [Running with KVM](#running-with-kvm)
* [Further Reading](REFERENCES.md)



## Introduction

Here at Ampere Computing we are always interested in emerging cloud native technologies as interesting workloads for our cloud optimized Ampere(R) Altra(TM) Arm64 processors. [gVisor](https://github.com/google/gvisor) is an active open source software project originally created by Google that provides an application kernel for containers. The [gVisor](https://github.com/google/gvisor) project ecosystem and community of [contributors](https://github.com/google/gvisor/graphs/contributors) continues to make regular improvements to gVisor for ARM64 platforms.  In fact, in recent months, [ARM64 improvements](https://github.com/google/gvisor/pulse) have been coming in almost weekly.  

From a technology perspective, [gVisor](https://github.com/google/gvisor) was written in [Golang](https://golang.org/) and is often used as a method to sandbox application containers by providing a substantial portion of the Linux operating system surface. Sandboxing can be thought of as an isolation boundary for greater security between the application and the Linux kernel running on the host.  This isolation boundary, or `sandbox` between the application and the host kernel is provided by gVisor's `runsc` binary.  The `runsc` binary implements an [Open Container Initiative (OCI)](https://opencontainers.org/) compliant runtime which integrates with Docker and Kubernetes, making it simple to run sandboxed containers in cloud native environments.

gVisor currently requires an abstraction which it calls a `platform` to implement the sandboxing mechanisms; currently available platforms are `ptrace` and `KVM`.  There are different tradeoffs between each `Platform` which generally are focused around performance and hardware requirements for running gVisor.  To describe it in an overly simplistic way, the platform implemention depends on the context in which `runsc` is executing, for example virtualized platforms would be limited to systems that do not require hardware virtualization.  The `ptrace` platform executes user code with out allowing it to execute host system calls, whereas the KVM platform uses kernel and hardware virtualization to allow gVisor to act as both guest OS and Virtual Machine Manager.  For more information regarding gVisor platforms please consult with upstream documenation on the subject located here:

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
Please note: using ptrace with older kernels can be used, and may work however your results and experience may vary.  Additionally please note that the current gVisor implementations only support kernels configured with 4K-page sizes.

### Bazelbuild binaries

Basibuild binaries for arm64 are needed prior to building gVisor.  The latest binary releases can found on the project releases page located here:

* [https://github.com/bazelbuild/bazel/releases/](https://github.com/bazelbuild/bazel/releases/)

At the time of this writing bazel 3.4.1 and 3.7.0 have been used during the processes decribed below.

## Download the latest gVisor Code

For the best experience with building gvisor, it is recommended that you pull the `master` branch directly from the [upstream source code](https://github.com/google/gvisor) located on GitHub.

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

## Build gVisor

You can now build gVisor by simply executing the following command, which will will launch a containerized Bazel build environment to generate the needed binary.

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

Once we have installed the gVisor binaries into a directory located within the system PATH, we can then update the docker daemon to utilize the gvisor runtime.   To change the configuration of the docker daemon you will need to update the file `/etc/docker/daemon.json` as follows:

```
{
  "runtimes": {
    "runsc-ptrace": {
      "path": "/usr/local/bin/runsc",
      "runtimeArgs": [
        "--platform=ptrace"
       ]
     },
     "runsc-kvm": {
       "path": "/usr/local/bin/runsc",
       "runtimeArgs": [
         "--platform=kvm"
        ]
     } 
}
```

After updating your docker daemon configuration file, restart the docker daemon to ensure the changes take effect.

```
sudo systemctl restart docker
```

## Run gVisor

```
docker run --runtime=runsc-ptrace hello-world
```

## Running with KVM
The current tip-of-tree for gVisor has a recent update which breaks gVisor's KVM functionality for ARM64.  A fix is in progress, but the following can be used to pull the code when it last supported KVM on ARM64:

```
git clone https://github.com/google/gvisor.git
git checkout 66d24bb692
```
Follow the build instructions as before.  To run with KVM, execute the following:

```
docker run --runtime=runsc-kvm hello-world
```
