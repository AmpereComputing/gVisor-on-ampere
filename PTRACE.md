![Ampere Computing](https://avatars2.githubusercontent.com/u/34519842?s=400&u=1d29afaac44f477cbb0226139ec83f73faefe154&v=4)

# gVisor on Ampere

## Table of Contents
* [Introduction](#introduction)
* [Requirements](#requirements)
  * [Operating System](#operating-system)
  * [Docker](#docker)
  * [Kernel 5.5. or Newer](#kernel-5-5-or-newer)
* [Download the gVisor Code](#download-the-gvisor-code)
* [Build gVisor-from-source](#build-gvisor-from-source)
* [Install the compiled binary](#install-the-compiled-binary)
* [Update Docker to use the gVisor runtimes](#update-docker-to-use-the-gvisor-runtimes)
* [Run a conttainer using runsc-ptrace](#run-a-container-using-runsc-ptrace)
* [Run a conttainer using runsc-kvm](#run-a-container-using-runsc-kvm)
* [All the steps in ansible](#all-the-steps-in-ansible)
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

### Docker

Docker version 17.09.0 or greater is required for use with gVisor.  Further reading on the docker requirements can be found in gVisor's `Docker Quick Start` located at the link below.

* [https://gvisor.dev/docs/user_guide/quick_start/docker/](https://gvisor.dev/docs/user_guide/quick_start/docker/)


### Kernel 5.5 or newer

As discussed earlier, gVisor requires a platform to implement the intercepting sandboxing mechanisms, and currently `ptrace` and `KVM` are implemented.
If you are planning on using gVisor with KVM, the Linux Kernel Virtual Machine, then it is necessary to use a specific Linux Kernel version which includes the necessary patches to enable usage on Arm64.  Currently the Linux Kernel 5.5 includes the necessary patch for KVM.  Further information and reading on the patches that for enabling usage of KVM as the gVisor platform please consult the information located here: 

* [https://lore.kernel.org/kvmarm/20191206020802.196108-1-justin.he@arm.com/t/#u](https://lore.kernel.org/kvmarm/20191206020802.196108-1-justin.he@arm.com/t/#u
)

Obviously because the patches are already included in kernels 5.5 and newer we strong suggest staying on newer kernels for the best results.
Please note: using ptrace with older kernels can be used, and may work however your results and experience may vary.  Additionally please note that the current gVisor implementations only support kernels configured with 4K-page sizes.

Currently Ubuntu LTS includes a 5.4 kernel.  Mainline kernel packages for Ubuntu can be found in the Ubuntu Mainline Kernel Archive and installed onto the sytsem. The Ubuntu Mainline Kernel Archive can be found here:

* [https://kernel.ubuntu.com/~kernel-ppa/mainline/?C=N;O=D](https://kernel.ubuntu.com/~kernel-ppa/mainline/?C=N;O=D)

## Download the latest gVisor code

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

## Build gVisor from source

You can now build gVisor by simply executing the `make runsc`, which will will launch a containerized Bazel build environment to generate the needed binary.
Output of the command will look similar to below.

```
# make runsc
make build OPTIONS="-c opt" TARGETS="//runsc"
make[1]: Entering directory '/usr/local/src/gvisor'
Error: No such container: gvisor-bazel-d1b91a5d
make[2]: Entering directory '/usr/local/src/gvisor'
make -C images load-default
make[3]: Entering directory '/usr/local/src/gvisor/images'
make pull-default || make rebuild-default
make[4]: Entering directory '/usr/local/src/gvisor/images'
docker pull  gcr.io/gvisor-presubmit/default_aarch64:b633fc901d737471
b633fc901d737471: Pulling from gvisor-presubmit/default_aarch64
Digest: sha256:50232d4b6ee2f5608bf1ed4de18ca1c8217296b1bb44c5cbf2543f8d88c6b716
Status: Image is up to date for gcr.io/gvisor-presubmit/default_aarch64:b633fc901d737471
gcr.io/gvisor-presubmit/default_aarch64:b633fc901d737471
make[4]: Leaving directory '/usr/local/src/gvisor/images'
docker tag gcr.io/gvisor-presubmit/default_aarch64:b633fc901d737471 gvisor.dev/images/default
make[3]: Leaving directory '/usr/local/src/gvisor/images'
docker run --user 0:0 --entrypoint "" --name gvisor-builder-d1b91a5d \
	gvisor.dev/images/default \
	sh -c "groupadd --gid 998 --non-unique docker-d1b91a5d && groupadd --gid 108 --non-unique kvm-d1b91a5d && \
	        \
	       if [[ -e /dev/kvm ]]; then chmod a+rw /dev/kvm; fi"
docker commit gvisor-builder-d1b91a5d gvisor.dev/images/builder
sha256:819d928bd7649ca98f09f21b5c231b8e89128adce89698c60fa8681a8879e590
gvisor-builder-d1b91a5d
# This command runs a bazel server, and the container sticks around
# until the bazel server exits. This should ensure that it does not
# exit in the middle of running a build, but also it won't stick around
# forever. The build commands wrap around an appropriate exec into the
# container in order to perform work via the bazel client.
docker run -d --rm --name gvisor-bazel-d1b91a5d \
	-v "/usr/local/src/gvisor:/usr/local/src/gvisor" \
	--workdir "/usr/local/src/gvisor" \
	--user 0:0 --entrypoint "" --init -v "/root/.cache/bazel:/root/.cache/bazel" -v "/root/.config/gcloud:/root/.config/gcloud" -v "/tmp:/tmp" -v "/var/run/docker.sock:/var/run/docker.sock" -v "/etc/docker/daemon.json:/etc/docker/daemon.json" --privileged --group-add 998 --device=/dev/kvm --group-add 108 \
	gvisor.dev/images/builder \
	sh -c "tail -f --pid=\$(bazel  info server_pid) /dev/null"
f01fb2b5e3b59324c14fa91e9511dad7069a8e3a9470565a932a73444d82a4b7
make[2]: Leaving directory '/usr/local/src/gvisor'
Another command holds the client lock: 
pid=7
owner=client
cwd=/usr/local/src/gvisor

Waiting for it to complete...
Another command (pid=7) is running.  Waiting for it to complete on the server...
DEBUG: /root/.cache/bazel/_bazel_root/13221ba0b7e9fde51c3391866c0ad16c/external/bazel_toolchains/rules/rbe_repo/version_check.bzl:68:14: 
Current running Bazel is ahead of bazel-toolchains repo. Please update your pin to bazel-toolchains repo in your WORKSPACE file.
DEBUG: /root/.cache/bazel/_bazel_root/13221ba0b7e9fde51c3391866c0ad16c/external/bazel_toolchains/rules/rbe_repo/checked_in.bzl:125:14: rbe_default not using checked in configs; Bazel version 3.4.1 was picked/selected but no checked in config was found in map {"0.20.0": ["8.0.0"], "0.21.0": ["8.0.0"], "0.22.0": ["8.0.0", "9.0.0"], "0.23.0": ["8.0.0", "9.0.0"], "0.23.1": ["8.0.0", "9.0.0"], "0.23.2": ["9.0.0"], "0.24.0": ["9.0.0"], "0.24.1": ["9.0.0"], "0.25.0": ["9.0.0"], "0.25.1": ["9.0.0"], "0.25.2": ["9.0.0"], "0.26.0": ["9.0.0"], "0.26.1": ["9.0.0"], "0.27.0": ["9.0.0"], "0.27.1": ["9.0.0"], "0.28.0": ["9.0.0"], "0.28.1": ["9.0.0"], "0.29.0": ["9.0.0"], "0.29.1": ["9.0.0", "10.0.0"], "1.0.0": ["9.0.0", "10.0.0"], "1.0.1": ["10.0.0"], "1.1.0": ["10.0.0"], "1.2.0": ["10.0.0"], "1.2.1": ["10.0.0"], "2.0.0": ["10.0.0"], "2.1.0": ["10.0.0"], "2.1.1": ["10.0.0", "11.0.0"], "2.2.0": ["11.0.0"], "3.0.0": ["11.0.0"], "3.1.0": ["11.0.0"]}
INFO: Analyzed target //runsc:runsc (325 packages loaded, 11571 targets configured).
INFO: Found 1 target...
Target //runsc:runsc up-to-date:
  bazel-out/aarch64-opt-ST-5e46445d989a/bin/runsc/runsc_/runsc
INFO: Elapsed time: 150.855s, Critical Path: 62.34s
INFO: 1613 processes: 1613 linux-sandbox.
INFO: Build completed successfully, 1649 total actions
make[1]: Leaving directory '/usr/local/src/gvisor'
```

At the end of the compile, the output will have a statement similar to the following,
which indicates the location of the freshly compiled runsc binary:

```
Target //runsc:runsc up-to-date:
  bazel-out/aarch64-opt-ST-5e46445d989a/bin/runsc/runsc_/runsc
```

## Install compiled binary

Copy the copiled binary to a location in the path and make it executible.

```
cp /usr/local/src/gvizor/bazel-out/aarch64-opt-ST-5e46445d989a/bin/runsc/runsc_/runsc /usr/local/bin
chmod 0777 /usr/local/bin/runsc
```

## Update Docker to use the gVisor runtimes

Once we have installed the gVisor binaries into a directory located within the system PATH, we can then update the docker daemon to utilize the gvisor runtime.   To change the configuration of the docker daemon you will need to update the file `/etc/docker/daemon.json` as follows:

```
{
    "runtimes": {
        "runsc-ptrace": {
            "path": "/usr/local/bin/runsc",
            "runtimeArgs": [
                "--platform=ptrace",
                "--debug-log=/tmp/runsc/",
                "--debug",
                "--strace"
            ]
        },
        "runsc-kvm": {
            "path": "/usr/local/bin/runsc",
            "runtimeArgs": [
                "--platform=kvm",
                "--debug-log=/tmp/runsc/",
                "--debug",
                "--strace"
            ]
        }
    }
}

```

After updating your docker daemon configuration file, restart the docker daemon to ensure the changes take effect.

```
sudo systemctl restart docker
```

Next verify the changes have taken by running the `docker info` command.  THe output should look as follows:

```
root@raptor:/etc/docker# docker info
Client:
 Debug Mode: false

Server:
 Containers: 36
  Running: 0
  Paused: 0
  Stopped: 36
 Images: 223
 Server Version: 19.03.13
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runsc-kvm runsc-ptrace runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 8fba4e9a7d01810a393d5d25a3621dc101981175
 runc version: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 init version: fec3683
 Security Options:
  apparmor
  seccomp
   Profile: default
 Kernel Version: 5.8.0-29-generic
 Operating System: Ubuntu 20.10
 OSType: linux
 Architecture: aarch64
 CPUs: 32
 Total Memory: 249.9GiB
 Name: raptor
 ID: TIMU:66I7:UQK4:YLPK:KZCL:RLBG:IZRC:N2UX:LTNP:5HAC:P3YL:UXHP
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Username: ppouliot
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

Take notice that the runtimes for runsc-kvm and runsc-ptrace show up in the running configuration by verifying the following line:

```
 Runtimes: runsc-kvm runsc-ptrace runc
 ```

## Run a container using runsc-ptrace

In order to use the gVisor runsc-ptrace we must pass in the `--runtime=runsc-ptrace` as part of the `docker run` command in order to select the runtime for use.
The following command is an example of running a hello-world container using the runsc gvisor runtime.

```
docker run --runtime=runsc-ptrace hello-world
```

Output from the above command would look as follows:

```
# docker run --runtime=runsc-ptrace hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
Now let's try running `dmesg` from within an ubuntu container using the runsc-ptrace runtime.

```
root@raptor:/usr/local/src/gvisor# docker run --runtime=runsc-ptrace ubuntu dmesg
[    0.000000] Starting gVisor...
[    0.249959] Generating random numbers by fair dice roll...
[    0.737573] Segmenting fault lines...
[    1.164732] Recruiting cron-ies...
[    1.388538] Adversarially training Redcode AI...
[    1.683637] Reticulating splines...
[    2.101409] Searching for needles in stacks...
[    2.112452] Checking naughty and nice process list...
[    2.362634] Committing treasure map to memory...
[    2.398640] Granting licence to kill(2)...
[    2.859065] Accelerating teletypewriter to 9600 baud...
[    2.975406] Ready!
```

Notice when attempting to use `runc` in this case we get a failure with the following output:

```
# docker run --runtime=runc ubuntu dmesg
dmesg: read kernel buffer failed: Operation not permitted
```




## Run a container using runsc-kvm

To use kvm instead of ptrace is simply a matter of substituting the name of the runtime as follows:

```
docker run --runtime=runsc-kvm hello-world
```
Output for the above command will be similar to using the runsc-ptrace runtime.

```
# docker run --runtime=runsc-kvm hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

As we did with the runsc-ptrace runtime, let's now try to use the runsc-kvm runtime to execute `dmesg` from within an ubuntu container.

```
# docker run --runtime=runsc-kvm ubuntu dmesg
[    0.000000] Starting gVisor...
[    0.251907] Singleplexing /dev/ptmx...
[    0.392448] Creating bureaucratic processes...
[    0.615196] Creating cloned children...
[    1.082841] Moving files to filing cabinet...
[    1.202539] Daemonizing children...
[    1.381653] Preparing for the zombie uprising...
[    1.778337] Consulting tar man page...
[    1.879892] Checking naughty and nice process list...
[    1.947322] Accelerating teletypewriter to 9600 baud...
[    2.434705] Adversarially training Redcode AI...
[    2.511895] Ready!
```

## All the steps in ansible

To make things easier the steps above have been captures as ansible.   Two files, the ansible code itself and a template for the docker deamon.json file are needed.

The ansible code is as follows:

```
#!/usr/bin/env ansible-playbook
---
- name: "Build and Install gvisor on linux aarch64"
  hosts: all
  any_errors_fatal: true
  gather_facts: true
  become: true
  become_method: sudo
  become_flags: '-E'
  any_errors_fatal: True
  order: sorted
  gather_facts: True
  vars:
    bin_path: "/usr/local/bin"
    src_path: "/usr/local/src"
    user: "root"
    group: "root"
    packages:
      - aptitude
      - screen
      - sudo
      - rsync
      - git
      - curl
      - byobu
      - asciinema
      - python3-dev
      - python3-pip
      - python3-selinux
      - python3-setuptools
      - python3-virtualenv
      - libffi-dev
      - gcc
      - g++
      - apt-transport-https
      - ca-certificates
      - gnupg-agent
      - software-properties-common
      - build-essential
      - golang
      - unzip
      - zip
    # Started w/ 3.4.1
    bazel_version: "3.4.1"
  handlers:
    - name: Restart Docker
      service:
        name: "docker"
        state: restarted
  tasks:
    - name: Installing Base Packages
      apt:
        name: "{{ packages }}"
        state: present
        update_cache: true

    - name: Use git to retrieve gvisor source
      git:
        repo: https://github.com/google/gvisor
        dest: "{{ src_path }}/gvisor"
        clone: true
        update: true
        force: true

    - name: Build Gvisor
      become: false
      command: 'make runsc'
      ignore_errors: yes
      args:
        chdir: "{{ src_path }}/gvisor"

    - name: "Copy runsc to {{ bin_path }}"
      copy:
        src: "{{ src_path }}/gvisor/bazel-out/aarch64-opt-ST-5e46445d989a/bin/runsc/runsc_/runsc"
        dest: "{{ bin_path }}/runsc"
        owner: "{{ user }}"
        group: "{{ group }}"
        mode: "0777"
        remote_src: yes

    - name: get the runsc version
      command:
        cmd: "/usr/local/bin/runsc --version"
      register: runsc_build_version

    - debug:
        msg: "{{ runsc_build_version }}"

    - name: "Enable gVisor runtime in docker config /etc/docker/daemon.json"
      template:
        dest: "/etc/docker/daemon.json"
        src: "docker.json.j2"
      notify:
        - Restart Docker

    - name: Flush Handlers
      meta: flush_handlers

    - name: Run hello-world with runsc-ptrace
      docker_container:
        docker_host: tcp://{{ ansible_default_ipv4.address }}:2375
        name: hello-world-runsc-ptrace
        image: hello-world
        runtime: runsc-ptrace

    - name: Run ubuntu dmesg with runsc-ptrace
      docker_container:
        docker_host: tcp://{{ ansible_default_ipv4.address }}:2375
        name: ubuntu-dmesg-runsc-ptrace
        image: ubuntu
        command: dmesg
        runtime: runsc-ptrace

    - name: Run hello-world with runsc-kvm
      docker_container:
        docker_host: tcp://{{ ansible_default_ipv4.address }}:2375
        name: hello-world-runsc-kvm
        image: hello-world
        runtime: runsc-kvm

    - name: Run ubuntu dmesg with runsc-kvm
      docker_container:
        docker_host: tcp://{{ ansible_default_ipv4.address }}:2375
        name: ubuntu-dmesg-runsc-kvm
        image: ubuntu
        command: dmesg
        runtime: runsc-kvm
```

The jinja template used for the docker daemon.json configuration:

```
{
    "runtimes": {
        "runsc-ptrace": {
            "path": "{{ bin_path }}/runsc",
            "runtimeArgs": [
                "--platform=ptrace",
                "--debug-log=/tmp/runsc/",
                "--debug",
                "--strace"
            ]
        },
        "runsc-kvm": {
            "path": "{{ bin_path }}/runsc",
            "runtimeArgs": [
                "--platform=kvm",
                "--debug-log=/tmp/runsc/",
                "--debug",
                "--strace"
            ]
        }
    }
}
```

Both files are included within this repository at the links below:

* [ansible/gvisor.yml](ansible/gvisor.yml)
* [ansible/docker.json.j2](ansible/docker.json.j2)


