
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

**Answer.** `ping` is not a Set UID binary on a modern Ubuntu/Debian system; it is granted the single capability `cap_net_raw` on the file, which lets an unprivileged user open the `SOCK_RAW` (and `SOCK_PACKET`) sockets needed to build and send ICMP packets directly. We first confirmed the capability and then stripped it:

```
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep

# -r removes all file capabilities
$ setcap -r /usr/bin/ping          
    
# empty output – no capabilities left
$ getcap /usr/bin/ping
```

After removal, running `ping` **as a normal (non-root) user fails**, because the process can no longer create the raw socket it needs. The error looks like:

```
$ ping www.dei.isep.ipp.pt
ping: socktype: SOCK_RAW
ping: socket: Operation not permitted
ping: => missing cap_net_raw+p capability or setuid?
```

The failure is the `socket()`/`EPERM` at start-up, not a networking problem. The binary never gets far enough to send a single packet. (Running it with `sudo` still works, because root bypasses the capability check.)

**Question** **7.** **Can you make ping work again without adding any capability, and without running with** **sudo?** **Provide a video showing your solution before and after.**

**Video:** [question-07-video](/videos/question-07-video.mp4)

**Answer.** Yes. Modern `iputils` `ping` no longer strictly needs a raw socket: if it cannot open `SOCK_RAW` it falls back to an **ICMP datagram socket** (`socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP)`). The kernel decides which users (by GID) may open such a socket through the sysctl **`net.ipv4.ping_group_range`**. By default this range is often empty (`1 0`, i.e. no group allowed), so the fallback also fails. We simply widen the allowed group range so our user's GID is included:

```
# check current (empty) range
$ sysctl net.ipv4.ping_group_range
net.ipv4.ping_group_range = 1    0        # min(1) > max(0) => nobody

# allow all groups (or just your own GID) to use ICMP datagram sockets
# setcap -r /usr/bin/ping (capability still removed from Q6)
$ sysctl -w net.ipv4.ping_group_range="0 2147483647"
```

After this, as a **normal user and without sudo and without any file capability**, `ping` works again:

```
$ getcap /usr/bin/ping          # still empty – no capability
$ ping -c1 www.dei.isep.ipp.pt
64 bytes from ... icmp_seq=1 ttl=.. time=.. ms
```

This works because the privilege was moved from a *per-binary file capability* to a *kernel policy* (`ping_group_range`): the kernel now permits members of the listed groups to create ICMP sockets directly, so no raw-socket capability is required. To make it persist across reboots, add `net.ipv4.ping_group_range = 0 2147483647` to `/etc/sysctl.conf` (or a file in `/etc/sysctl.d/`). The video should show: (a) `ping` failing after Q6's removal, (b) the `sysctl -w` change, and (c) `ping` succeeding as the normal user with `getcap` showing no capability.

**Question 8. Can you make `passwd` work without being a Set-UID program? Detail how and why the solution works, and provide a video demonstrating `passwd` functioning without being a Set-UID program.**

**Video:** [question-08-video](/videos/question-08-video.mp4)

**Answer.** Yes — but doing it correctly reveals that `passwd` needs *more* than "permission to write a root-owned file," which makes this a good illustration of how hard true least-privilege actually is.

The reason `passwd` is Set-UID root is to modify root-owned files (`/etc/shadow`, `/etc/passwd`) that a normal user cannot touch. The natural first attempt is to drop the Set-UID bit and grant only the capability that bypasses the discretionary-access-control (DAC) permission checks, `cap_dac_override`:

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root ... /usr/bin/passwd

# remove the Set-UID bit
$ chmod u-s /usr/bin/passwd
$ ls -l /usr/bin/passwd
-rwxr-xr-x 1 root root ... /usr/bin/passwd

$ setcap cap_dac_override+ep /usr/bin/passwd
$ getcap /usr/bin/passwd
/usr/bin/passwd cap_dac_override=ep
```

**This single capability is not enough.** Running `passwd` as a normal user now fails:

```
$ passwd
Changing password for rita.
Current password:
New password:
Retype new password:
passwd: Authentication token manipulation error
passwd: password unchanged
```

**Why the naive version fails.** `passwd` does not edit `/etc/shadow` in place. Through PAM's `pam_unix` module it performs an **atomic replace**: it creates a temporary file, copies every entry across (rewriting the user's hash), **restores the original owner, group and mode** on the new file (`root:shadow`, `0640`), then `rename()`s it over `/etc/shadow`. That ownership/mode restoration needs privileges `cap_dac_override` does **not** provide:

- changing a file's owner **to root** requires `CAP_CHOWN`;
- changing the mode/attributes of a file you **do not own** requires `CAP_FOWNER`.

`cap_dac_override` only skips the read/write/execute permission-bit checks (it lets the process *open* `/etc/shadow`); it grants none of the above. So PAM aborts and reports the generic *"Authentication token manipulation error."*

**The working solution** is to grant the full set of capabilities the operation genuinely uses:

```
# setcap cap_dac_override,cap_chown,cap_fowner+ep /usr/bin/passwd
$ getcap /usr/bin/passwd
/usr/bin/passwd cap_dac_override,cap_chown,cap_fowner=ep
```

With these three capabilities the (non-Set-UID) binary can rewrite `/etc/shadow` *and* restore its ownership and mode, so a normal user can change their own password.

**Why the solution works (and what it teaches).** Replacing Set-UID with capabilities works because Set-UID hands the process the entire root *identity*, whereas capabilities let us grant only specific privileged *operations*. The lesson is that enumerating those operations precisely is non-trivial: changing a password is not one privilege but several, bypass DAC to open the file (`cap_dac_override`), change its owner (`cap_chown`), fix its mode (`cap_fowner`), plus reliance on a separately-privileged helper for the password check. This is exactly why the shipped binary simply takes full root via Set-UID: it is the blunt but simple way to cover every one of those sub-privileges at once. The capability version is more faithful to the principle of **least privilege**, at the cost of having to know precisely what the program does.

**What the video shows:** `whoami` (an unprivileged user), `ls -l /usr/bin/passwd` with **no `s` bit**, `getcap /usr/bin/passwd` showing `cap_dac_override,cap_chown,cap_fowner=ep`, and finally a **successful** `passwd` run changing the user's own password.

**Question** **9.** **Discuss why do you think that, by default,** **passwd is a Set UID program and** **ping is not?**

**Answer.** The difference comes down to **how narrowly each program's privilege need can be expressed, and the legacy behind each tool**:

- **`ping` needs exactly one, well-defined privilege:** the ability to open a raw/ICMP socket. That maps cleanly onto a single capability, `cap_net_raw`. Granting that one capability gives `ping` everything it needs and *nothing more*, so distributions moved it off Set UID root. If `ping` were compromised, the attacker only inherits "can craft raw packets," not full root.

- **`passwd` needs a privilege that is harder to reduce to one capability:** it must modify root-owned, security-critical files (`/etc/shadow`, `/etc/passwd`), coordinate locking, run PAM modules, and historically relied on *being* UID 0. Capturing all of that cleanly in a single capability is awkward (`cap_dac_override` bypasses *all* file permissions — nearly as dangerous as root anyway). Set UID root is the simplest way to give it the full file-ownership context it expects. It also predates Linux capabilities, so Set UID was the only mechanism available when it was written.

- **Blast radius / least privilege:** `ping`'s one capability is low-risk and a natural fit for the capabilities model, so it was a prime candidate to drop Set UID. `passwd`'s requirement is broad and inherently sensitive, so splitting it into capabilities buys little security (any capability that lets it rewrite `/etc/shadow` already implies effective root), and Set UID remains the conventional, well-audited choice.

In short: `ping` keeps Set UID *off* because a single minimal capability covers its need; `passwd` keeps Set UID *on* because its need is broad, identity-bound, and historically entrenched, so the capabilities model offers little practical gain.

**Question** **10.** **Can you get familiar with other capabilities? On you report explain ten more capabilities of your choice.**

**Answer.** Ten capabilities (other than `cap_net_raw`) from `capabilities(7)`:

1. **`CAP_CHOWN`** — Change the ownership (UID/GID) of files arbitrarily, bypassing the normal rule that only root may `chown` to another user.
2. **`CAP_DAC_OVERRIDE`** — Bypass all discretionary read/write/execute permission checks on files and directories (effectively read/write any file regardless of its mode bits).
3. **`CAP_DAC_READ_SEARCH`** — Bypass read permission checks on files and read/execute (search) checks on directories; weaker than `DAC_OVERRIDE` (read/traverse only, no write).
4. **`CAP_FOWNER`** — Bypass permission checks that normally require the process UID to match the file owner (e.g. setting file flags, changing mode, updating timestamps on files you don't own).
5. **`CAP_SETUID`** — Make arbitrary changes to the process's UIDs (`setuid`, `setreuid`, etc.) and forge UID in passed credentials — i.e. become any user.
6. **`CAP_SETGID`** — The group equivalent of `CAP_SETUID`: change GIDs arbitrarily and manipulate the supplementary group list.
7. **`CAP_NET_BIND_SERVICE`** — Bind a socket to a privileged (well-known) TCP/UDP port below 1024, which is otherwise reserved for root.
8. **`CAP_NET_ADMIN`** — Perform network administration: configure interfaces, routing tables, firewall (netfilter) rules, set interfaces promiscuous, etc.
9. **`CAP_KILL`** — Send signals to any process, bypassing the usual rule that you can only signal processes with a matching UID.
10. **`CAP_SYS_ADMIN`** — A very broad "catch-all" capability covering mount/umount, setting the hostname, many filesystem and namespace operations; so powerful it is often described as nearly equivalent to root. (Bonus: **`CAP_SYS_TIME`** sets the system clock; **`CAP_SYS_BOOT`** allows `reboot()`; **`CAP_SYS_PTRACE`** allows tracing/attaching to arbitrary processes; **`CAP_SYS_MODULE`** loads/unloads kernel modules.)

**Question** **11.** **After a program (running as normal user) disables a** **`cap_dac_override` capability, it is compromised by a buffer-overflow attack. The attacker successfully injects his malicious code into this program’s stack space and starts to run it. Can this attacker use the** **`cap_dac_override` capability? What if the process deleted the capability, can the attacker use the capability?**

**Answer.** The answer depends on the crucial difference between **disabling** a capability and **deleting (dropping)** it, which maps onto the *effective* vs *permitted* capability sets:

- **The process only *disabled* it (removed from the EFFECTIVE set, but kept it in the PERMITTED set).** This is the usual "temporarily lower privilege" pattern: a capability must be in the effective set to be used, but as long as it remains in the permitted set the process is allowed to re-raise it into the effective set at any time (via `capset()` / `prctl(PR_CAPBSET...)`-style calls). The injected shellcode runs **inside the same process**, with that same permitted set, so it can simply re-enable `cap_dac_override` and then use it. **Yes — the attacker can use the capability.** Disabling (clearing the effective bit) is *not* a security boundary against code running in the same process.

- **The process *deleted* it (dropped it from the PERMITTED set, not just the effective set).** Once a capability is removed from the permitted set, the kernel will **not** let the process add it back to the effective set — the drop is irreversible for the lifetime of that process. The attacker's injected code inherits exactly this reduced permitted set, so there is no capability to re-raise. **No — the attacker cannot use the capability.**

**Lesson:** clearing a capability from the *effective* set only protects against accidental misuse by the program's own correct code; it offers no protection against a code-injection/overflow attacker in the same process, because the capability is still *permitted* and can be re-raised. To genuinely defend against such an attacker, a program should **drop the capability from its permitted set** as soon as it no longer needs it (and ideally from the bounding set), making the privilege unrecoverable even by injected code.

### Trying Linux Capabilities with Docker

Let's explore how Docker uses Capabilities. Note however that Capabilities are only one of the several Linux kernel features Docker relies on to isolate and limit the behavior of containers. **Namespaces** provide separation of resources like processes, networking, and file systems, giving each container its own isolated environment. Control groups (**cgroups**) limit and monitor the container's resource usage (CPU, memory, etc.), preventing any single container from consuming all system resources. Docker uses **seccomp** (secure computing mode) to restrict access to potentially dangerous system calls, reducing the attack surface. In systems where it is available, Docker also uses AppArmor (a **mandatory access control mechanism**) to define what resources a container is allowed to access on the host.

By default, Docker assigns a few default capabilities to containers. We created a ubuntu-based Docker image with the capability library binaries (libcap2-bin) for the purpose of running the Linux capabilities utility programs inside a container and check the container’s capabilities. It is available in Docker Hub:

- <https://hub.docker.com/r/isepdei/capabilities01>

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

**Answer**: For the container’s main process (PID 1), /proc/1/status shows:

```sh
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
```

The inheritable set is empty, while the permitted and effective sets are identical (a80425fb), as is the bounding set. Decoding the value with capsh --decode=00000000a80425fb gives 14 capabilities:

- cap_chown
- cap_dac_override
- cap_fowner
- cap_fsetid
- cap_kill
- cap_setgid
- cap_setuid
- cap_setpcap
- cap_net_bind_service
- cap_net_raw
- cap_sys_chroot
- cap_mknod
- cap_audit_write and cap_setfcap

capsh --print shows the same data already decoded: Current: ...=ep lists those 14 as effective and permitted (no i, so inheritable is empty), the Bounding set lists the same 14, and the Ambient set is empty. The Current IAB line lists what is missing (ex: cap_sys_admin, cap_net_admin, cap_sys_module). So /proc gives raw bitmasks and capsh gives names, but they agree. The process runs as uid=0 yet holds only a subset of root’s capabilities, which is Docker’s default hardening.

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

**Answer:** Yes. After --cap-drop all, ping fails immediately:

```sh
sh: 1: ping: Operation not permitted
```

/proc/1/status shows all capability sets as 0000000000000000, so the container has no capabilities, not even cap_net_raw.

Ping is not Set UID, it relies on the file capability cap_net_raw=ep. When the kernel executes a file with capabilities, what it can grant is limited by the process’s bounding set. With --cap-drop all that set is empty, so the kernel cannot give ping the capability it requires and refuses. That is why the error comes from sh: ping never starts. Being uid=0 doesn’t help, because root’s power inside a container comes from its capabilities.

To fix this we can add back only the capability it needs:

```sh
docker run --name captest --cap-drop all --cap-add net_raw -d -p 8000:80 isepdei/capabilities01
```

Ping now works and CapPrm, CapEff and CapBnd are all 0000000000002000, which is cap_net_raw only.

```sh
CapInh: 0000000000000000
CapPrm: 0000000000002000
CapEff: 0000000000002000
CapBnd: 0000000000002000
CapAmb: 0000000000000000
```

The container holds one capability instead of Docker’s default 14, following least privilege.

------

[^P21]: OS Boxes provides pre-build images for VirtualBox, e.g.: <https://www.osboxes.org/ubuntu-server/>

[^P22]: <https://man7.org/linux/man-pages/man7/capabilities.7.html>

[^P23]: Note, however that, by default, recent Docker versions alter the OS policy to allow containers to bind to any port.

[^P24]: <https://man7.org/linux/man-pages/man1/capsh.1.html>
