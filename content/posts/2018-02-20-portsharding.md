---
layout: post
title:  "What is port sharding in Linux and why should I care"
comments: true
aliases: [/portsharding]
date:   2018-02-20 18:18:00
tags:
- linux
- networking
description: ''
images:
- author.jpg
categories:
- Linux 
---


![image](https://memegenerator.net/img/instances/12831919/does-devnull-support-sharding.jpg)

## It's actually called SO_REUSEPORT

Kernel 3.9 introduced a new cool feature in SOCKET interface called SO_REUSEPORT. So what is it? 

As the [official documentation](http://man7.org/linux/man-pages/man7/socket.7.html) says, it allows multiple AF_INET or AF_INET6 sockets to be bound to an identical socket address. before binding a socket to an interface, each one should have this option enabled. This way, mutiple processes can listen on the same port at the same time! 

## How Can This Possibly Be Secure?

That was my first concern. So we have a process (let's say it's Nginx) listening on port 8080 with this option enabled. Worth mentioning they have this option since NGINX [release 1.9.1](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/). So, what stops me from listening on the same port and hijack their traffic?

Well, Linux has a clever way of preventing these kind of attacks. Essensially, each socket who wants to listen to the same port, should have the same effective UID as the first openned socket on that port. So if your Nginx is running as www-user, a new process can listen to the same port only if they're owned by www-user.

## So.. What is it good for?

### Performance

Anyone who has ever configured Nginx, Gunicorn, Sanic, Apache or most of other web servers, know that having multiple threads for each service provides a better performance than having a single process handling all the requests. in the past, we had two main approches to have multiple threads sharing the load of the incoming network traffic:

+ Race for `accept` syscall between all the threads
+ a thread for accepting the connection and giving it to other threads

The second approch is not bad except it doesn't work very well with UDP and the first approch is basically chaos. With port sharding, all threads can listen on a port at the same time and get the traffic in a truly balanced manner. This can reduce lock contention between workers accepting new connections, and improve performance on multicore systems.


![image](/img/port-sharding/nginx-before-after.png)
As depicted in the figure, when the SO_REUSEPORT option is not enabled, a single listening socket notifies workers about incoming connections, and each worker tries to take a connection. With the SO_REUSEPORT option enabled, there are multiple socket listeners for each IP address and port combination, one for each worker process


Benchmarking Performance with reuseport from Nginx official website


![image](/img/port-sharding/reuseport-benchmark.png)
I ran a wrk benchmark with 4 NGINX workers on a 36‑core AWS instance. To eliminate network effects, I ran both client and NGINX on localhost, and also had NGINX return the string OK instead of a file. I compared three NGINX configurations: the default (equivalent to accept_mutex on), with accept_mutex off, and with reuseport. As shown in the figure, reuseport increases requests per second by 2 to 3 times, and reduces both latency and the standard deviation for latency.


### Zero Downtime Updates

Let's move on from Nginx and get into HAProxy. Something designed to be 100% up all the time. 


![image](/img/port-sharding/ha-fork.png)
In this diagram, we see the initial stage of an HAProxy reload starting with a single process (left) and then causing a second process to start (right) which binds to the same IP and port, but with a different socket..


This works great so far, until the original process terminates. HAProxy sends a signal to the original process stating that the new process is now accept()ing and handling connections, which causes it to stop accepting new connections and close its own socket before eventually exiting once all connections complete

This way, there's only a very small amount of downtime (in the region of microseconds) in HAProxy before it reloads completely. Yelp has pruposed a set of [workarounds](https://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html) for that to be completely eliminated as well!

## How do I implement this

Now we're talking code ;) first of all, this can only be enabled in Linux so if you're in Windows, you're out of luck.

First let's see a simple Python program listening on a port with REUSEPORT enabled:

```python
from os import getpid
from socket import *

port = 1234

server = socket(AF_INET, SOCK_STREAM)
server.setsockopt(SOL_SOCKET, SO_REUSEPORT, 1)
server.bind(('', port))
server.listen(0)

print('[python pid:{}] listening on port {}...'.format(getpid(), port))

while True:
    client, addr = server.accept()
    print('[python pid:{}] got connection from {}'.format(
          getpid(), client.getpeername()))
    client.send('Hello from Python process {}\n'.format(getpid()))
    client.close()
```


simple enough? Let's also see how to enable REUSEPORT in Nginx:

```c
http {
     server {
          listen 80 reuseport;
          server_name  localhost;
          # ...
     }
}

stream {
     server {
          listen 12345 reuseport;
          # ...
     }
}
```

![image](https://media.giphy.com/media/Hi3pr5mmHrbbi/giphy.gif)


## Sources

+ [https://stackoverflow.com/questions/23742368/can-so-reuseport-be-used-on-unix-domain-sockets](https://stackoverflow.com/questions/23742368/can-so-reuseport-be-used-on-unix-domain-sockets)
+ [https://www.ebayinc.com/stories/blogs/tech/zero-downtime-instant-deployment-and-rollback/](https://www.ebayinc.com/stories/blogs/tech/zero-downtime-instant-deployment-and-rollback/)
+ [https://engineeringblog.yelp.com/2017/05/taking-zero-downtime-load-balancing-even-further.html](https://engineeringblog.yelp.com/2017/05/taking-zero-downtime-load-balancing-even-further.html)
+ [https://githubengineering.com/glb-part-2-haproxy-zero-downtime-zero-delay-reloads-with-multibinder/](https://githubengineering.com/glb-part-2-haproxy-zero-downtime-zero-delay-reloads-with-multibinder/)
+ [https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/)
+ [http://man7.org/linux/man-pages/man7/socket.7.html](http://man7.org/linux/man-pages/man7/socket.7.html)
+ [https://github.com/joewalnes/port-sharding](https://github.com/joewalnes/port-sharding)


## Questions?

hit me up [@_n0p_](https://twitter.com/_n0p_) on Twitter :-)