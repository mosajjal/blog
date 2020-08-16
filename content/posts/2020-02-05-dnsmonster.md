---
layout: post
title:  "Monitoring 200K DNS Queries per second using ClickHouse"
comments: true
aliases: [/dnsmonster]
date:   2020-02-07 10:24:00
tags:
- dns
- networking
- nsm
description: 'Passive DNS monitoring using ClickHouse and Grafana'
images:
- /img/dnsmonster/overview.jpg
categories:
- security
- infosec
- big data
--- 

![image](/img/dnsmonster/dnsmonster-icon.png)

## Why Monitor DNS traffic

As a member of a Security Incident Response Team, I’ve seen that the normal security practices like deploying firewalls, IDS sensors, NSM Toolkit and Netflow Monitoring. SIRT is increasingly looking to the network as a data source. For years infosec has been network-address-centric and the attackers have adapted. Today it is very common to see malware command and control (C&C) use domain generation algorithms (DGAs), Peer-to-Peer (P2P), or even fast-flux DNS to evade IP address-based detection and blocking. It's very clear that monitoring DNS activity inside the network and trying to find anomalies associated with it is an essential part of SIRT toolkit.


## Exploring Options: What should I choose?

Naturally, this question has a very simple (and wrong) answer: "why don't you push the data to Elasticsearch or Splunk and query it there?" 

First let's take Splunk, and probably any other commercial tool, out of the way. Remember, we're talking about at least 1TB of data per day. Splunk charges $1800/GB on annual license, but their pricing drops if you push more data into it. Let's assume Splunk gives a whopping 90% discount on 3TB/day traffic. `$1800/GB * 1024GB * (100 - 90) / 100` = `$184,320` per year. So unless you have a quarter million USD burning a hole in your pocket, let's move on :-)

Elasticsearch is the next obvious choice if you have a generic JSON-formatted data you want to search on. So this was my first attempt. In addition to an ES node, a choice must have been made about which solution is going to be used to pull data out of the SPAN session and push it to ES. The choices were:

* [Packetbeat](https://www.elastic.co/products/beats/packetbeat) pushing data to ES
* Packetbeat pushing data to [Logstash](https://www.elastic.co/products/logstash) and to ES
* [Passivedns](https://github.com/gamelinux/passivedns) saving raw DNS JSON in a folder and Filebeat/Logstash pushing them to ES

These solutions, along with a bunch of similar solutions, had a big drawback: ES was being overwhelmed and was constantly crashing. Mind you, the configuration of the SPAN probe host wasn't bad at all:

- OS: Red Hat Enterprise Linux 7.7 Maipo
- Kernel: Linux 3.10
- Disk: 44TB HDD
- CPU: Intel Xeon Gold 5122 @ 16x 3.7GHz
- RAM: 192GB

Having your ES (and sometimes Logstash) crash under no hardware bottleneck, meant only two things: ES is not made for a lot (150K/s) of small JSON lines, or I haven't configured it right and ES itself is the bottleneck. I spent a couple of months trying every possible scenario to get it right, with no success.

## Introducing ClickHouse

In walks ClickHouse. Borrowing a short description of the product from [George](https://hackernoon.com/@george3d6), Clickhouse is used to run fast analytics on very large amount of data. It’s rather bad as a transactional database, but it can run aggregate queries on billions of rows in sub second times.

Clickhouse is designed in a way that it is an extremely faster writer but it's a relatively slow reader, which is perfect when you're writing a huge amount of DNS queries but you don't need to read them as fast as you write them. In our little project, the worst case write to read ratio is 1 million to one, meaning for every 1 million write to the DB, we're issuing one SELECT query to fetch a subsection of the data to show on our dashboard, which we'll get to later. 

Cloudflare has already put an amazing [blogpost](https://blog.cloudflare.com/how-cloudflare-analyzes-1m-dns-queries-per-second/) talking about how they're leveraging the amazing power of ClickHouse to monitor 1M QPS. Cloudflare has also released a really useful ClickHouse dialect for SQLAlchemy for all or you py-heads. With an standard ORM talking to ClickHouse, developing a simple dashboard using Flask/Django becomes much easier. But I'm not going to talk about that, this post will provide instructions to set up your passive DNS monitoring system without having to write one line of code. 

## DNSmonster to collect and send

As I mentioned earlier, there are several tools used to listen on a network interface and create a JSON document based on the DNS query and responses coming to/from that interface. The most famous solution out there is [Passivedns](https://github.com/gamelinux/passivedns). `passivedns` works fairly well if you're looking at a relatively low rate of QPS and dumps DNS conversations into raw JSON files. the solution has a major drawback though: I can only leverage 1 CPU core therefore it'll lose some traffic if it's overwhelmed.

The second interesting project that I looked at, was `gopassivedns`. This one builds up on `passivedns` and adds that Golang streams goodness to it, which means the programmming language will not be a bottleneck in processing as much data as coming it. That being said, this solution also had some drawbacks:

- not being 100% complete (Kafka support was missing)
- not being directly connected to an output (Elastic etc.) meant a lot of HDD overhead.
- didn't support Dot1Q packets

so the effort was stuck here. Between a dead project and an alpha codebase with a lot of promise. Until I saw an interesting video on Youtube coming from Chile's NIC and how they did passive DNS monitoring and saw their project, which was in fact another fork of `gopassivedns`: `dnszeppelin`. This one solved one big issue. It connected directly to ClickHouse therefore it didn't need Kafka or any data bus in between. But interestingly, it still had the Dot1Q support issue. `dnszeppelin` was also two projects on its own, one was a direct fork of `gopassivedns`, another project trying to connect it to ClickHouse.

So I decided to combine all the mentioned solutions and I ended up with `dnsmonster`. this little code:

- Directly supports ClickHouse
- It generates a single binary
- Supports Dot1Q packets
- Is exteremly light and scalable

The project is [available](https://gitlab.com/mosajjal/dnsmonster) on Gitlab for free under MIT license. 


So basically, the setup would be:

- A `dnsmonster` binary running on the system listening on the traffic
- A ClickHouse container running on the same machine
- A cronjob to provide retention policy (basically delete old stuff)
- Firewall and Service configuration to keep it running on startup
- A Grafana instance to provide a nice dashboard

Grafana provides an intuitive interface to any ClickHouse database. It's free and open source and sharing your dashboard in it is a breeze. I managed to get some really useful data out of passiveDNS using this dashboard.

The entire setup process as well as a dashboard template is available in the Gitlab repository, after setting it up, it should look something like this

![image](/img/dnsmonster/overview.jpg)

The project has worked fairly well, but it obviously hasn't been through many tests. If you see anything not working perfectly, Issues/PRs are more than welcome.

Enjoy!

