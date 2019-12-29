---
layout: post
title:  "Hide in Plain Sight: Protocol Multiplexers and Optional Authentication"
comments: true
aliases: [/2018-12-13-hide-in-plain-sight]
date:   2018-12-13 18:18:00
tags:
- Linux
- Network
- OSINT
- Infosec
description: ''
categories:
- Infosec
- Learn Linux 
---

![test](/img/hide-in-plain-sight/meme.jpeg)

Almost every Internet-connected device on the planet comes with a nice web interface to interact with. And some of them like routers and APs come with their own little firewall to prevent backdoors and whatnot. So what if one of these devices or even servers gets compromised? Where do you look at to find IoC (indication of compromise) in them?

I don't think I need to explain why IoT is a huge security challenge for every organization since everyone at least has a "smart" printer lying around somewhere. Lately, I was looking at ways to hide the traffic within another traffic type or regular TCP port and I stumbled upon two great ways to make it happen. 

## What the hell am I talking about

Let's put on our black hat and think about it for a second. I just pwned an internet connected device, but the only port we have is 80 (443 as well if we get lucky!). I can potentially open up another port, do a reverse connection to my C2 and exploit the firewall as well, but I wanna be as stealthy as possible. What are my options? 

I want to show you two methods that you must be aware of if you're hunting for these types of advanced threats. CIA (or maybe NSA, idk) has been reported using these methods to hide their attacks from their targets so they must be pretty dope. Let's dive in.

## Optional Authentication

Have you seen those annoying old school basic authentication schemes (which are rarely used nowadays) in very old devices? The auth pop-up is handled by the browser and if it's successful, you're redirected to the page and if you're not, you'd get a 401 error. 

![Basic Authentication](https://crossbrowsertesting.com/images/faq/basic-auth-example.png)

So how does your browser knows it's time for authentication? It's pretty simple. when you create a page with basic auth, the first time you do a GET request, your browser is greeted with a 40x and a `www-authenticate` message. This way, your browser pops up a dialog and asks for credentials. But here's where it gets very interesting. What if the server doesn't send that `www-authenticate` message? What if the server responds with a 200 with some content. In fact, what if the server responds normally to it and shows you the website? 

You might suggest that okay, what is the point then? You just described a regular visit to a website with the excitement of a 5-year-old. So what? Here's where it gets really cool. What if, the server offers TWO 200 responses, one for giving it auth, one for no-auth page handling. Let me explain, I wrote a simple [Bottle](https://bottlepy.org) code to demonstrate this. 

```py
from bottle import route, run, request

def check(user, pw):
    if user == "ali":
        if pw == "ali":
            return True
    return False

def auth_optional(check, realm="private", text="Access denied"):
    ''' Callback decorator to require HTTP auth (basic).
        TODO: Add route(check_auth=...) parameter. '''
    def decorator(func):
        def wrapper(*a, **ka):
            user, password = request.auth or (None, None)
            if not(user is None or not check(user, password)):
                return text
            return func(*a, **ka)
        return wrapper
    return decorator

@route('/')
@auth_optional(check, text="Hidden Message")
def index():
    return "Regular old website"

run(host='localhost', port=8080)
```

in the following code, the site won't give an error if you don't provide a credential. But if you do it with `ali:ali` as username and password, you'll get the "hidden message". It's pretty cool, isn't it? Now think about this: will your automated web vulnerability scanning tool detect this or not? I didn't think so either. 

It's not all text either, with the right code, you can write an HTTP proxy with Auth on a functioning web application. That would be pretty cool as well. But this is just one way of hiding your service where no one can find. You can also do something called "process multiplexing".

## Serve multiple services on the same port

So we had to change a function in the web app to add our own stuff in there. It's cool but it has a problem. If the code is managed by a version control software (e.g. git), the file would become an unstaged file and the injected code will be recognizable for the developer. Although you can always change the library itself, which most of the time isn't checked by the version control software. But there's another way! There's a very interesting tool called [sslh](https://github.com/yrutschle/sslh). They describe it as "Applicative Protocol Multiplexer". I think their Github page describes them perfectly:

`sslh` accepts connections on specified ports and forwards them further based on tests performed on the first data packet sent by the remote client.

So, sslh listens on port 80 and forwards everything to the old port 80 (which must be changed now to something like 8000 but only on localhost). This doesn't make any difference to the end user but opens up everything for the attacker. the attacker could bring her own OpenVPN server, configures it and make it listen on port 1194 (again, localhost). But configures `sslh` to forward OpenVPN traffic to 1194 from 80.

Port 80 will now serve both WWW and OpenVPN. Pretty neat, right? 


## If the server supports SSL/TLS

That's heaven right there. Why? you can encapsulate every protocol to TLS using `stunnel` and `proxytunnel`, right? So you can make EVERYTHING look like SSL/TLS traffic so even the more advanced IDS/IPS devices in the way won't notice anything other than "normal" traffic. The attacker can (and probably will) also steal the server certificate and makes the connection even more trustworthy as well. 

## How do I protect myself against this attack?

In IoT, you're in bad luck. IoT devices don't offer a lot of tools to monitor their OS. Their shell is sometimes limited and nothing is very accessible. But, if you have a root account on any devices and you also have an RS232 or SSH connection to it (basically a root shell), you can monitor your devices by calculating md5 hashes of every file on its filesystem to see if anything changes over time. 

In servers, you can use a File Integrity Monitoring/FIM on your farm to see if any file is changing over time. pulsar.py is a very neat code that does that automatically as long as you configure `hubblestack` on your infrastructure. `osquery` is also used to do these kinds of monitoring by providing a SQL interface to all the bits and pieces of your server. Take a look at them.

