---
layout: post
title:  "Dissection of Winbox critical vulnerability"
comments: true
aliases: [/2018-05-21-winbox-bug-dissection]
date:   2018-05-21 18:18:00
tags:
- Mikrotik
- Winbox
- Path Traversal
description: ''
categories:
- Reverse Engineering 
images:
- /n0p-blog/img/mikrotik-winbox/morphus-meme.jpg
---

![Morphus](/n0p-blog/img/mikrotik-winbox/morphus-meme.jpg)

On April 23rd 2018, Mikrotik fixed a vulnerability "that allowed gaining access to an unsecured router". myself and [@yalpanian](https://twitter.com/yalpanian) of [@BASUCERT](https://twitter.com/BASUCERT) (part of [IR CERT](https://cert.ir)) reverse engineering lab tried to figure out what exactly got fixed, what was the problem in the first place and how severe was the impact of it.

UPDATE: full PoC is now available on [Github](https://github.com/BasuCert/WinboxPoC).

UPDATE: CVE-2018-14847 has been assigned to this vulnerability and there should be a MetaSploit module related to this bug soon.


## What is Mikrotik
According to the official website, MikroTik is a Latvian company which was founded in 1996 to develop routers and wireless ISP systems. MikroTik now provides hardware and software for Internet connectivity in most of the countries around the world.

RouterOS is the operating system of most Mikrotik devices. The vulnerability affects all versions of RouterOS from 6.29 (release date: 2015/28/05) to 6.42 (release date 2018/04/20)

## The Diff

First things first, we had to see which binaries was changed before and after the patching. RouterOS is written on top of Linux Kernel so a lot of kernel modules will be different in each version. What we did was downloading RouterOS 6.40.7 and 6.40.8 npk files and look through each file to find the differences. I just named them "after" and "before" on my laptop. I hashed every file inside those packages to see the difference, and long behold I found a few. As you can see, mproxy binary has changed. Which makes sense because mproxy binary handles all Winbox requests.

<img src="{{ "/assets/img/mikrotik-winbox/mproxy-diff.png"}}" alt="">


## Let's Dive Into "mproxy"

mproxy is a 63K binary with no (except for a simple NX section) security measurements. So this bug could've been anything memory related. we ran bindiff on the files and found some differences between the two versions, but there was a simple "if" statement which was added to the file.

<img src="{{ "/assets/img/mikrotik-winbox/bindiff.jpg"}}" alt="">

## Jailbreak The Device and Turn on GDB

So, how can we confirm this? We don't have GDB and we don't have a shell (yet). After some digging around, we found an interesting project <a href="https://github.com/0ki/mikrotik-tools">"mikrotik-tools"</a>. It shows an easy way to get a bash shell inside a RouterOS and copy additional binaries to its filesystem. So we added a new and big BusyBox in addition to a gdbserver to do the job for us.
It was pretty easy on VM since all you need to do is mount RouterOS vmdk somewhere and add additional files. After that the RouterOS will boot and it'll login with "devel" account with the same password as your admin account. In that telnet session, there's no RouterOS shell, instead, it's bash! easy :-)

After putting a breakpoint on the "list" string, we started digging around and finally got the string which bypasses the first condition without violating any rules. Since the code scans the strings 4 bytes at a time, we could place our dots between them so it couldn't be discovered by the code.


## Sending the Packets

now let's get started on sending some packets across! First, we turn off secure communication between the Router and Winbox just because we can! When looking at the packets, we'll see a tiny conversation between the Router and Winbox and then we'll see the list file being sent to Winbox. This file includes all the necessary DLLs for Winbox to use in order to communicate with the Router. Interestingly, this approach has had more than a couple security flaws. Just think about it, you download an unknown DLL from a remote host and you run it under Winbox, a signed executable! But that's for another blog post. Right now, we want to switch off this list with our crafted string. Here's the original conversation:

<img src="{{ "/assets/img/mikrotik-winbox/wireshark-list.png"}}" alt="">


So what's so interesting about this conversation? If you look at it closely, you'll see the highlighted packets are not the first in the conversation. First Winbox authenticates with the Router and then tries to get the list file. Pretty safe, right? We thought so too at first, especially since we repeated the same packets and got a "bad session id" result back from RouterOS. But we noticed something interesting. A single byte is changing each time we send the same request to the Router, and each time we sent that exact same byte over to the Router in our next request.. Session ID? YES! So we did our repeat attack with this tiny modification. We sent the request, waited for the Router to respond, switched that one byte by the byte received from the response, and then we sent the second request. and Voilà! We got our file!! We didn't even need to authenticate! Just by sending the highlighted packets in the right order and switching the byte, the Router folds. 

A general rule of thumb: don't trust the client you shipped to users. Trust your own service to do proper validation :)

## Decoding the User Database

Now we can read ANY file from the Router! Which files are useful? In our previous talk on APA3 conference, we talked about just how insecure Mikrotik is, especially when it comes to handling credentials. In short, Mikrotik uses a very weak encoding (no hash and salt) to store passwords to an index file. So it's entirely possible to download the credential database and extract every username and password stored inside it and that's exactly what we did for this PoC.

After downloading the idx file, we used another great tool from mikrotik-tools to decrypt the username and password database and dump everything IN PLAIN TEXT !!!

## What did we Learn?

Simply put, don't use Mikrotik in an enterprise environment. Not only in poses a security concern for an organization, it will put IT manager's computer at a great risk because of all the external DLLs the winbox.exe binary downloads and executes on the computer (for more info on that check out Slingshot malware). In addition to all this, Mikrotik saves your passwords in easily decryptable ciphers (as you saw in the PoC), which might have an effect on your entire network infrastructure. For example, SNMP passphrases are usually shared across multiple network devices including the Mikrotik router. So if an attacker retrieves your SNMP passphrase, they can launch multiple attacks on your Cisco, Juniper and more secure layer 2/3 devices. As they say: "a chain is as strong as its weakest link"
