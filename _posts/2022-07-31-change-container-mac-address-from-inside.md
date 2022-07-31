---
layout: post
title: Change container MAC address from inside
date: 2022-07-31 17:20:00 +0200
categories: devops
---

## Summary 

This posts explains how to overwrite the MAC address of a docker container when is not possible to use docker run flags to do so, preserving network connectivity. This can be useful. for example, in some CI platform. You can go directly to the [solution](#solution)


## Finding the solution

**Note:** I used a debian 10 image.

After searching not for so long I found that you can rewrite a MAC within a container with `ifconfig` as follows:

```bash
ifconfig eth0 down
ifconfig eth0 hw ether ${NEW_MAC}
ifconfig eth0 up
```

**Note** that `ifconfig` needs the container to be executed with `--privileged` flag.

The problem that I encountered then is that, after re-enabling the network interface (`eth0`), the internet connectivity did not work. Executing `ping` inside the container returned `Network unreacheable`:

```bash
ping 8.8.8.8
connect: Network is unreachable
```

This happened also without doing any changes on the interface, just by executing `down` and `up`.

Clearly there was some configuration script missing or that was not working properly. After searching for a while I found that after disabling the network interface, the **routing table** was dumped. However, it was not recovered once the interface was up again. As you can see, there was no default gateway:
```bash
ip r s
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```
It was normal then, than network was unrecheable.

After executing:
```bash
ip r a default via 172.17.0.1 dev eth0
# Route anything (default) to the host 172.17.0.1 from eth0)
```
The connectivity returned again.

**Note:** I obtained the IP 172.17.0.1 from another container executed in my laptop. As I said, the routing table was dumped, so I could no longer recover the contents.

So, after knowing all this, what I do now is to save the default gateway before re-writing the MAC address. See the script below.

**Note** that with this aproximation I'm not saving other routing rules if present.

## Solution

```bash
gateway=$(ip r s default | cut -d' ' -f3 | tr -d '\n')
ifconfig eth0 down
ifconfig eth0 hw ether ${NEW_MAC}
ifconfig eth0 up
ip r a default via $gateway dev eth0
```

## Observations

By default, any docker container is attached to the `bridge` network. Once you start a container you can see with `docker network inspect bridge` the container MAC address. After changing the MAC address within the container, its value in `docker network inspect bridge` does not change. It's not clear if this can have any side effect.

Also, I tested that, in practice, it's possible to execute multiple containers with the same MAC without breaking internet connectivity (when not sharing the same network, of course). I tested it by using the method in this post to configure two containers with the same MAC and executing `ping` concurrently. Latencies did not seem to get affected.

