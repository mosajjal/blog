---
layout: post
title:  SSH MFA using Slack/Teams/Discord
comments: true
aliases: [/ssh-mfa-webhook]
image: ''
date:   2020-09-28 00:00:00
tags:
- linux
- ssh
- mfa
description: ''
categories:
- Linux 
- Security
---

# What is SSH MFA

Let's start from the basics. We're all familiar with Multi-Factor Authentication (MFA) and we're (hopefully) using it everywhere: Twitter, Gmail, Slack, etc. On the other hand, we don't usually use it for our infrastructure access. RDP, SSH, Telnet are among those services that are not easy to secure on the open internet, so we usually tuck them away under a bastion host or a "gateway" of sorts that can provide this MFA functionality. Unfortunately, those services are quite expensive and according to Shodan, we have ~23 MILLION SSH servers exposed to the open internet. Those services might not be any of my concern now, but when (not if) they're compromised, they're gonna form a zombie army for a botnet and then it becomes everyone's problem.

To that end, I decided to do a cost-benefit analysis of different methods to secure a SSH server exposed to the Internet. We covered the first and more cumbersome logical answer: using gateways and bastions. The next step, which has been around for years and it's working quite well, is authentication using SSH keypair. SSH keys don't allow interactive logins altogether and use a keypair to authenticate a user. It's been around for ages and it works really really well. I can't recommend it enough! But, in practice, even if the server is configured to authenticate using SSH key, it's not always enforced and the user can just log on using a regular password. There's also always a risk of key-sharing between the team members within the organization. So having a second factor still makes a lot of sense.

# Why Webhook

Ideally, the second factor of any authentication shouldn't use the medium that the first factor is using. eg having MFA sent over the internet still means we trust the internet and TLS/PKI. There are several protocols that implement an agreed timestamped approach with the key sync happening only once at the start of the key exchange, and there's no need for the "comms" to happen outside of that short period of time. The main implementation of this protocol is RFC 6238 (TOTP). TOTP is used quite extensively to build MFA. Google Authenticator, Lastpass Authenticator, Cisco Duo and Microsoft Auth all support RFC6238. But there are two main drawbacks of using this in a shared production environment

* Losing your private key means losing your access. If you drop your phone in the toilet, no SSH for you until you console back into the machine and re-configure the keys with your new phone
* Definitely not ideal for teams. One key is for one user and one user only. there's no good way to share the key between teams and have audit in place for everyone.

As I mentioned before, having a "shared" environment is not ideal at all. It's not the best practice and it's not recommended at all. But in reality it happens more often than not, especially in staging/testing instances. This is where making those corner-cases secure becomes important, and this is where Webhooks come into play.

Webhook MFA offers a simple yet effective MFA solution. in your team, you probably have a strict Slack/Teams Authentication policy. Enorced MFA, short session timeouts and managing sessions per device are easily possible with the current collaboration tools. All your comms are already in a centralized place, why not have a channel for your SSH MFA tokens and make them accessible to everyone, keep a log of which IP is trying to access which server at which time and more importantly, share this within a team rather than having it per individual?


# Implementation details

The implementation is actually pretty simple. The PAM module is a single `.c` file that compiles into a shared object (`.so`) that can be used in `auth` section of PAM. The `.so` file needs an ini configuration file path as an argument to make it as customizable as possible. The ini file has the URL, the JSON data, the details of how many digits should each OTA code be as well as a very simple template engine to make sure the message send to your work collaboration tool is suited to what you need. When an authentication attempt happen, depending on your `/etc/pam.d/sshd` config, you can put this authentication factor wherever in the process. I recommend putting it as a last step in auth process, especially for the internet-facing servers otherwise the spam in your Slack channel will be real! For internal-facing servers, you might consider putting this factor of auth in the beginning of the auth process just to make it more audit-able from your point of view, since internal servers are not meant to be touched/accessed all the time anyway. 

You can also configure `sshd` to ignore this factor of auth when you're logging in via SSH keys. That's customizable through `sshd` conf and it's covered in this amazing doc from [Arch Wiki](https://wiki.archlinux.org/index.php/OpenSSH#Two-factor_authentication_and_public_keys).

I've covered a bit of configuration and implementation details within the [project README](https://github.com/mosajjal/webhookpam/blob/master/README.md) as well.


# Declaimer 

Big Disclaimer: This project is at beta at best. Be careful with it otherwise you'll lock yourself out! you can take a look at PAM's `optional` module for this just to do some stress tests in different scenarios and see if it works first before pushing it to your prod instances. There are also a lot of error control and bits and pieces missing from the implementation. Feel free to raise any issues to the [Github repo](https://github.com/mosajjal/webhookpam) so we can work together to fix them and make this a viable option for everyone. 