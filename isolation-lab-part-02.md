
# Part 2: Docker and Linux Capabilities

## Introduction

This is the second part of the *Isolation Lab*, where you are going to experiment with various aspects of software Isolation using Docker. If you haven't, checkout Part I for a longer intro and the first set of experiments.

## Setup

You will need a modern Linux Distribution and Docker. We suggest using a prebuilt Linux VM[^P21] and install Docker on the VM. Windows WSL should be able to support this lab but was not tested.

## Intro to Linux Capabilities

Traditional UNIX systems categorize processes into two types: **privileged processes**, which have an effective user ID (UID) of 0 and are commonly referred to as superuser or root, and **unprivileged processes**, which have a nonzero effective UID. A special type of executables, called **Set UID** (Set User ID), allow users to run the program with the file owner's privileges, rather than their own. This is useful in scenarios where certain actions require higher privileges, such as administrative tasks, but users should not have full access to those elevated privileges. A common example of a Set UID program is the `passwd` command, which allows a regular user to change their password. The system password file (like `/etc/passwd` or `/etc/shadow`) is typically only writable by root, but when a user runs `passwd`, it executes with root privileges (thanks to Set UID), allowing the password change while maintaining system security. Because Set UID programs run with elevated privileges, they pose security risks if not implemented carefully. Set UID programs must be carefully scrutinized.  

You can see that a program has Set UID by looking at its permissions. The program **passwd**, needs Set UID permission (notice the ‘**s**’ in the user permissions):

```
# ls -la /usr/bin/passwd
-rw**s**r-xr-x 1 root root 55544 fev 6 2024 /usr/bin/passwd
```

Instead of granting full privileged access (as Set UID does), we can split the privileges once reserved for the superuser into smaller, discrete units called **Capabilities**[^P22]. Capabilities were introduced in the Linux Kernel since a while back (Kernel version 2.2). For example, Capabilities, such as CAP_NET_BIND_SERVICE to allow binding to ports below 1024 (something typically reserved for privileged services[^P23]), or CAP_SYS_BOOT to allow rebooting the system (there are many more; see them in references given in the footnotes). **Linux Capabilities are widely used in container platforms (such as Docker) to restrict what a process inside the container can do, improving security isolation**. 

One example of a program that uses Capabilities is **ping**. It needs RAW sockets because it directly constructs and sends ICMP (Internet Control Message Protocol) packets, bypassing transport layer protocols like TCP or UDP. Use `getcap` to see a program’s capabilities (**cap_net_raw** is the name of the capability and **ep** refers to the Effective (e) and Permitted (p) capability sets):

```
# getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep  
```

Please take a few moments to read the reference man pages and learn more about capability sets, processes, files, users, and ambient capabilities.

**Question** **6.** **What happens if you remove** **`cap_net_raw` from ping? Describe how you removed the capability.**

**Question** **7.** **Can you make ping work again without adding any capability, and without running with** **sudo?** **Provide a video showing your solution before and after.**

**Question** **8.** **Can you make** **passwd work without being a Set UID program? Detail how and why the solution works, and provide a video demonstrating passwd functioning without being a Set UID program.**

**Question** **9.** **Discuss why do you think that, by default,** **passwd is a Set UID program and** **ping is not?**

**Question** **10.** **Can you get familiar with other capabilities? On you report explain ten more capabilities of your choice.**

**Question** **11.** **After a program (running as normal user) disables a** **`cap_dac_override` capability, it is compromised by a buffer-overflow attack. The attacker successfully injects his malicious code into this program’s stack space and starts to run it. Can this attacker use the** **`cap_dac_override` capability? What if the process deleted the capability, can the attacker use the capability?**

### Trying Linux Capabilities with Docker

Let's explore how Docker uses Capabilities. Note however that Capabilities are only one of the several Linux kernel features Docker relies on to isolate and limit the behavior of containers. **Namespaces** provide separation of resources like processes, networking, and file systems, giving each container its own isolated environment. Control groups (**cgroups**) limit and monitor the container's resource usage (CPU, memory, etc.), preventing any single container from consuming all system resources. Docker uses **seccomp** (secure computing mode) to restrict access to potentially dangerous system calls, reducing the attack surface. In systems where it is available, Docker also uses AppArmor (a **mandatory access control mechanism**) to define what resources a container is allowed to access on the host.

By default, Docker assigns a few default capabilities to containers. We created a ubuntu-based Docker image with the capability library binaries (libcap2-bin) for the purpose of running the Linux capabilities utility programs inside a container and check the container’s capabilities. It is available in Docker Hub: 

- https://hub.docker.com/r/isepdei/capabilities01

You can start a container named **captest** (**`--name captest`**) in background (**`-d`**) with this image (and map container port 80 to host’s port 8000;**`-p 8000:80`**)

```
docker run --name captest -d -p 8000:80 isepdei/capabilities01
```

The container runs a webserver you can test with, e.g., **`curl`**:

```
curl localhost:8000
```

Now, use **`capsh`**[^P24] installed in the container to see the capabilities of a process inside the container (**`docker exec <container_name> <command>`** executes the given command inside the named container):

```
docker exec captest sh -c 'capsh --print'
```

You can also directly check the contents of the container’s **`proc`** filesystem (**`/proc/1/status`** is the status the main container process, with PID 1); this is useful to see quickly all the different capability sets and visually compare the values:

```
docker exec captest sh -c 'grep Cap /proc/1/status'
```

And decode the given values using **`capsh`**:

```
docker exec captest sh -c 'capsh --decode=00000000a80425fb'
```

**Question** **12.** **What are the inheritable, permitted and effective capabilities of a process running in the container (look at the `proc` filesystem output)? Compare to the output of** **`capsh –-print`.**

Try to use the container to perform a **`ping`**:

```
docker exec captest sh -c 'ping www.dei.isep.ipp.pt'
```

Now, let’s stop the container, and start it again dropping all capabilities (**--cap-drop all**):

```
docker stop captest; docker rm captest
docker run --name captest --cap-drop all -d -p 8000:80 isepdei/capabilities01
```

**Question** **13.** **Do you notice any issue when you try to do a `ping` again? Explain why and how can you fix it while still running with** **`--cap-drop` all.**

------

[^P21]: OS Boxes provides pre-build images for VirtualBox, e.g.: https://www.osboxes.org/ubuntu-server/

[^P22]: https://man7.org/linux/man-pages/man7/capabilities.7.html

[^P23]: Note, however that, by default, recent Docker versions alter the OS policy to allow containers to bind to any port.

[^P24]: https://man7.org/linux/man-pages/man1/capsh.1.html

