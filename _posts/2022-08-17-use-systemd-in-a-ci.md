---
layout: post
title: "Run Systemd in a Ci tool"
date: 2022-08-17 17:20:00 +0200
categories: physics
---


## Summary

In this tutorial I explain how to start systemd and execute services in a container running in a CI tool, such as Gitlab CI. I tested the solution on CentOS 7 and Debian 10 images. 


## Solution

The solution involves modifying the docker image to be used as well as some bash commands to init systemd and start services. To know the details go to [Finding the solution](#finding-the-solution).

### Configuring the image 
This steps are necessary in order to not crash the host (true story). They are used in the official `centos/systemd` image. The container will be executed with `--privileged` and for some reason I guess it makes systemd able to interact with the host.

The following steps are based, as said before, in the official [centos7/systemd](https://github.com/CentOS/CentOS-Dockerfiles/blob/master/systemd/centos7/Dockerfile). This steps hold for Debian 10, except that systemd binary is in a different path and some missing package.

```dockerfile
FROM centos:7

ENV container docker

RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;


CMD ["/bin/bash"]
```

### Runing services

In the container run:

```bash
unshare --pid --mount-proc --fork /usr/lib/systemd/systemd & #
pid=$(lsns --raw | grep "root /usr/lib/systemd/systemd" | cut -d' ' -f 4 | tr -d "\n")
nsenter -t $pid -m -p systemctl start ${SOME_SERVICE}
nsenter -t $pid -m -p systemctl status ${SOME_SERVICE}
```


## Finding the solution

The starting point was [centos7/systemd](https://github.com/CentOS/CentOS-Dockerfiles/blob/master/systemd/centos7/Dockerfile) official image. It worked, but it was not useful in a CI, where an interactive prompt (`bash`) is needed. I tried to make bash the entrypoint, but then executing `/usr/sbin/init` within `bash` did not work.
I suspected that this was due to the fact that `systemd` didn't have `PID=1`, as it was not the entrypoint anymore. Executing `exec /usr/sbin/init` did work, but I lost again the bash prompt.

To solve this, I used [Linux namespaces](https://blog.quarkslab.com/digging-into-linux-namespaces-part-1.html). In particular, **PID namespaces**. Inside a **PID namespace**, the PID count is reseted, and so the first process to be executed there will have PID=1 (viewed from inside). We are ticking systemd.

To execute a process in a new ** PID namespace** we need to use:
```bash
unshare --pid --mount-proc --fork ${process_to_be_executed}
```
So this starts systemd in a container, having bash as the entrypoint:
```bash
unshare --pid --mount-proc --fork /usr/sbin/init
```

I made two changes:
```bash
unshare --pid --mount-proc --fork /usr/lib/systemd/systemd &
```
* I replaced the binary path because the former is a link to the later and in debian it does not exist. Although in Debian 10 the path is another one anyway (`/lib/systemd/systemd`)
* I added `&` because if not it's needed to use `ctrl+C` to recover the control of the terminal, which is not possible or not straighforward in a CI.


Once we have executed `systemd`, we need to be able to execute services. If you try `systemctl start ${service}` it will not work. We need to execute it inside the namespace `systemd` is running in. We are going to use `nsenter` to do so:

```bash
nsenter -t $pid -m -p systemctl start ${SOME_SERVICE}
```
Before executing the command we need a PID (`$pid`) of a process running in the target namespace. We can use `lsns` (`ls` for namespaces). Select the PID associated with `/usr/lib/systemd/systemd`. Or you can execute:

```bash
pid=$(lsns --raw | grep "root /usr/lib/systemd/systemd" | cut -d' ' -f 4 | tr -d "\n")
```
And then
```bash
nsenter -t $pid -m -p systemctl start ${SOME_SERVICE}
```

### Debian 10

For Debian 10 the main differences is that:
1. systemd must be installed: `apt-get install -y systemd.sysv`
2. Systemd path is `/lib/systemd/systemd` instead.

A part from that, it will work if you apply all the steps used in the `centos7/systemd` image.