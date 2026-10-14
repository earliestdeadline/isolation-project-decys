
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

**Question** **15.** **Can you describe what can go wrong? Provide a detailed exploit of the issue, showcasing some compromise of the container’s host (for some reason, Scott installed `fdisk` on the container). Include a video of the exploit.**

## Advanced Escapes

There are a number of container escapes that rely on the availability of certain capabilities[^P31]. Have a look at the article. 

> Note that some of these exploits might depend on other system configuration, Docker versions, and other layers of isolation Docker relies on (e.g. namespaces, capabilities, seccomp,...).

**Question** **16.** **Can you replicate any of these escapes? Remember that some might not work anylonger. You should test them and explain how you proceeded to make it work. Provide a video demonstrating the exploit (demo).** (this question is for extra points)

**Question** **17.** **Finally, what are some good practices regarding Docker security?**

------

[^P31]: [https://www.panoptica.app/research/7-ways-to-escape-a-container](https://infosecwriteups.com/breaking-free-26-advanced-techniques-to-escape-docker-containers-530049816b55)
