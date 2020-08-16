---
layout: post
title:  "Why is Docker insecure and what can we do about it"
comments: true
draft: true
aliases: [/dockersecurity]
date:   2020-05-30 10:24:00
tags:
- docker
- podman
- openshift
- containers
- kubernetes
description: 'Docker Inherent Security Issues'
images:
- /img/dockersecurity/wreckedship.jpg
categories:
- security
- containers
--- 

![image](/img/dockersecurity/wreckedship.jpg)

## Client/Server Model vs Fork/exec Model

As the name suggests, Client/Server model has a running daemon acting as a server, and one or more clients connecting to it, sending and receiving commands and generally dictating how the server behaves. Docker typically uses a Unix socket to have that shared communication pipe between client(s) and the server.

This approach has been embraced by Docker at its fullest. Docker server runs and exposes a socket and/or a HTTP endpoint for communication. Having this model allows Docker server and client to be further away from each other. The same approach has been taken by `Xorg` project as well as `cups` printing service.

In the container world, client/server model has some disadvantages. In my opinion, one of the biggest ones is re-inventing the wheel when it comes to managing processes, containers, etc. each container will be handled by a single process, which means dropping privilege of a container to user permission rather than root is next to impossible. Also, when everything relies on your Docker server to be in running state, whatever kills the daemon process, can kill all the containers, which leads to a DoS opportunity on an OS level.

Fork/exec method takes advantage of what the OS already has. 

### Why 


## docker group vs wheel group permission
## auditd log problem
## Docker == initd inside another initd, redundant
## What Syscalls are available in which platform
## Frequent Updates and Changes

test text



### resources

* [Podman: A more secure way to run containers](https://opensource.com/article/18/10/podman-more-secure-way-run-containers)
* [Dockerless, part 1: Which tools to replace Docker with and why](https://mkdev.me/en/posts/dockerless-part-1-which-tools-to-replace-docker-with-and-why)