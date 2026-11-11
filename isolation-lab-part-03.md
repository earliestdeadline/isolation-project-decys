
# Part 3: Docker Security Evasion

## Introduction

This is the third part of the *Isolation Lab*, where you are going to experiment with various aspects of software Isolation using Docker. If you haven't, start from Part I.

## Setup

For this part of the Lab, you will need a computer with Docker installed.

## Privileged Containers

### Scott is Back

Oh oh, they asked Scott to setup the container on one of SecureLabs’ servers. He was having some issues bringing up the container and ended up with doing this:

```
docker run -d --privileged -p 80:80 isepdei/insecurelabs03
```

**Question** **14.** **What are the capabilities of the container?**

**Answer:** With --privileged the container gets every capability the kernel defines. /proc/1/status shows:

```sh
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
```

The permitted, effective and bounding sets are all 000001ffffffffff, which means all 41 capabilities.

Decoding it with capsh lists all 41, including the ones Docker normally withholds:

- cap_sys_admin, the near-root catch-all (mount, namespaces and more)
- cap_sys_module, to load kernel modules
- cap_sys_rawio, for raw access to block devices and I/O ports
- cap_sys_ptrace, to trace and attach to any process
- cap_net_admin, to configure network interfaces, routes and firewall rules
- cap_sys_boot, cap_sys_time, cap_mac_admin, cap_mac_override, cap_bpf, cap_perfmon and others

So the container has the same capabilities as root on the host.

--privileged also does more than grant capabilities, as it disables the default seccomp profile and AppArmor/SELinux confinement, gives the container access to the host’s devices (e.g. /dev/sda), which is why fdisk matters in Q15, and makes /sys and /proc writable.

Taken together, root inside the container is effectively root on the host, and the isolation Docker normally provides is largely gone.

**Question** **15.** **Can you describe what can go wrong? Provide a detailed exploit of the issue, showcasing some compromise of the container’s host (for some reason, Scott installed `fdisk` on the container). Include a video of the exploit.**

## Advanced Escapes

There are a number of container escapes that rely on the availability of certain capabilities[^P31]. Have a look at the article. 

> Note that some of these exploits might depend on other system configuration, Docker versions, and other layers of isolation Docker relies on (e.g. namespaces, capabilities, seccomp,...).

**Question** **16.** **Can you replicate any of these escapes? Remember that some might not work anylonger. You should test them and explain how you proceeded to make it work. Provide a video demonstrating the exploit (demo).** (this question is for extra points)

**Question** **17.** **Finally, what are some good practices regarding Docker security?**

------

[^P31]: [https://www.panoptica.app/research/7-ways-to-escape-a-container](https://infosecwriteups.com/breaking-free-26-advanced-techniques-to-escape-docker-containers-530049816b55)
