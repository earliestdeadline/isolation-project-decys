
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

**Answer:** As stated on the previous answer, the container, not only doesn't have the seccomp profile and AppArmor/SELinux confinement, which gives it access to the host's devices, it also has the same capabilities as root on **the host**.

Since the host's block devices are exposed unfiltered inside the container, `fdisk -l` reveals them directly, including `/dev/sdd`, the host's main disk.
With the `cap_sys_admin` capability, the container can mount it without restriction:

```bash
mkdir -p /mnt/filesystem
mount /dev/sdd /mnt/filesystem
```

At this point, the container has the host's root filesystem mounted with **full read/write access as root**, with no MAC policy standing in the way. So, any file on the host can be read, modified or deleted from inside what *should have been* an isolated container.
The following video demonstrates this exploit by writing directly into the host filesystem: [Exploit Video](question-15-video.mp4)

## Advanced Escapes

There are a number of container escapes that rely on the availability of certain capabilities[^P31]. Have a look at the article. 

> Note that some of these exploits might depend on other system configuration, Docker versions, and other layers of isolation Docker relies on (e.g. namespaces, capabilities, seccomp,...).

**Question** **16.** **Can you replicate any of these escapes? Remember that some might not work anylonger. You should test them and explain how you proceeded to make it work. Provide a video demonstrating the exploit (demo).** (this question is for extra points)

**Answer:** Unlike the privileged container escape in Q15, this escape requires *no Linux capabilities*. The misconfiguration here is the **mounting of the host's Docker socket into the container:**

```bash
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock ubuntu bash
```

The docker daemon runs as root on the host and listens on `/var/run/docker.sock/`. When this socket is mounted inside a container, any process inside can communicate with the daemon directly as if it were the host itself. From inside the container, the socket is used to instruct the daemon to spawn a new privileged container with the host's root filesystem mounted:

```bash
docker -H unix:///var/run/docker.sock run -v /:/host -it ubuntu chroot /host
```

Resulting in a shell chrooted to the host's file system, launched outside of any container isolation boundary. This misconfiguration isn't present in Scott's container, however, it is a distinct and more dangerous class of escape since it can effect containers that appear hardened.
The following video demonstrates this exploit by writing directly into the host filesystem through a nested container: [Exploit Video](question-16-video.mp4)

**Question** **17.** **Finally, what are some good practices regarding Docker security?**

**Answer:** Good Docker security practice covers four areas: the image, the container runtime, the host and the running service.

**In regards to image security:** as demonstrated in Q1-Q5, secrets must never enter the image build context. By using multi-stage builds, credentials used during build steps never appear in a published layer. `.env` and credential files should be added to `.dockerignore` and, images that had a secret in any layer, should be treated as permanently compromised. Use minimal base images and regularly scan images for known CVEs.

**In regards to container runtime:** never use --privileged, as shown in Q14 and Q15, it removes every isolation boundary. Instead, start from `--cap-drop=ALL` and add back only the specific capability the application needs, as shown in Q13. Keep the default seccomp profile and AppArmor/SELinux confinement active to provide a secondary enforcement layer, even when a capability is present.

**In regards to the Host:** never mount `/var/run/docker.sock` into a container, as shown in Q16. Avoid mounting sensitive host paths as volumes. Keep the docker daemon containerd and runc updated as several of the CVEs in the references article are **runc vulnerabilities** that are exploitable on outdated hosts.

**In regards to service:** always apply the principle of least privilege, at every level. Only expose the ports the service actually needs, use read-only filesystems and use Docker's user namespace remapping to ensure that even a container root maps to an unprivileged host UID. Logging and anomaly monitoring should also be enabled so unexpected syscalls or mount operations are detectable.

------

[^P31]: [https://www.panoptica.app/research/7-ways-to-escape-a-container](https://infosecwriteups.com/breaking-free-26-advanced-techniques-to-escape-docker-containers-530049816b55)
