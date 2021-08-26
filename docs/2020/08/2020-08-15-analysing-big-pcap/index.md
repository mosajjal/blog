# Parsing a massive DNS PCAP file efficiently



# The problem statement
Say you have a 30GB PCAP full of DNS data, and you want to analyse unusual activity on it. To make things simple, let's see how long it'll take to find a list of IPs that have accessed PSN's domain (`prppsn.com`) and the timestamp associated with them.

```sh
$ perf stat capinfos bigdnssample.pcap

File name:           bigdnssample.pcap
File type:           Wireshark/tcpdump/... - pcap
File encapsulation:  Ethernet
File timestamp precision:  microseconds (6)
Packet size limit:   file hdr: 65535 bytes
Number of packets:   222 M
File size:           31 GB
Data size:           28 GB
Capture duration:    1210.540079 seconds
First packet time:   *REDACTED* 
Last packet time:    *REDACTED* +20 minutes of first packet
Data byte rate:      23 MBps
Data bit rate:       185 Mbps
Average packet size: 125.93 bytes
Average packet rate: 184 kpackets/s
SHA256:              *REDACTED*
RIPEMD160:           *REDACTED*
SHA1:                *REDACTED*
Strict time order:   False
Number of interfaces in file: 1

 Performance counter stats for 'capinfos bigdnssample.pcap':

        235,264.47 msec task-clock:u              #    0.796 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
               347      page-faults:u             #    0.001 K/sec                  
   699,682,435,518      cycles:u                  #    2.974 GHz                    
 2,359,352,338,719      instructions:u            #    3.37  insn per cycle         
    36,761,315,016      branches:u                #  156.255 M/sec                  
       152,805,051      branch-misses:u           #    0.42% of all branches        

     295.708867452 seconds time elapsed

     209.632593000 seconds user
      26.655789000 seconds sys
```


## What's our hardware
My laptop:

OS: Arch Linux 
Kernel: x86_64 Linux 5.7.7-arch1-1
Disk: Timing cached reads:   25744 MB in  1.99 seconds = 12915.87 MB/sec, Timing buffered disk reads: 1314 MB in  3.00 seconds = 437.48 MB/sec
CPU: Intel Core i5-8350U @ 8x 3.6GHz
RAM: 32GB

## Wireshark
This one was easy to dismiss since it didn't even get to 10% of the packets before filling up the RAM and basically died. 

## Termshark
Terminal brother of Wireshark dies on 6%. But overall a quite nice binary. I'll probably add it to my binary collection since it's basically wireshark in terminal. Almost perfect mouse support as well.

## Tshark JSON output
From now on, the solutions are going to work in "stream", meaning they probably won't run out of RAM and the question becomes the speed of the solution rather than weather it'll work or not. For the purpose of benchmarking these, I'm gonna use compact JSON format output of `tshark`, and then pipe it to PV to measure lines/second being written to stdout. That way we can have a feeling of how long will it take to actually parse this massive PCAP

```
$ command | pv --line-mode --rate > /dev/null
```

```
TShark (Wireshark) 3.2.5 (Git commit ed20ddea8138)

Compiled (64-bit) with libpcap, with POSIX capabilities (Linux), with libnl 3,
with GLib 2.64.4, with zlib 1.2.11, without SMI, with c-ares 1.16.1, with Lua
5.2.4, with GnuTLS 3.6.14 and PKCS #11 support, with Gcrypt 1.8.6, with MIT
Kerberos, with MaxMind DB resolver, with nghttp2 1.41.0, with brotli, with LZ4,
with Zstandard, with Snappy, with libxml2 2.9.10.
```

```sh
$ tshark -T jsonraw -r bigdnssample.pcap | sed -e "s/^  },$/ },\r/g" | tr -d '\n' | tr -s '\r' '\n' | pv --line-mode --rate > /dev/null
~[3.5k/s]
```
Note that looking at my `gotop`, `sed` and `tr` are not the bottlenecks since `tshark` was filling up a core of CPU and it's a single-core binary


Now let's see how long it'll take for `tshark` to solve our problem

```sh
$ perf stat tshark -r bigdnssample.pcap -2 -R 'dns.qry.name matches prppsn.com'
...
```

sadly `tshark` died on me before giving any results (in ~900 seconds) since it filled up my RAM and crashed.. RIP!

## Packetbeat

packetbeat version:
`packetbeat version 7.7.1 (amd64), libbeat 7.7.1 [unknown built unknown]`

I've made a pretty simple `packetbeat.yml` just to demonstrate DNS traffic within the file

```yaml
packetbeat.protocols:
- type: dns
  ports: [53]

output.console:
  pretty: false
```

```sh
$ packetbeat -c packetbeat.yml -I bigdnssample.pcap |  pv --line-mode --rate > /dev/null
~[6.5k/s]
```

Getting a bit better, almost twice as fast a `tshark`! Although I should mention that this is technically cheating since `tshark` does ALL protocols + a lot of packet info that `packetbeat` misses. 

So now the logical next step would be to push this to ES and search it, right? wrong! At this stage I'm not very interested in benchmarking ES, only the packet parser. hence I'm gonna rely on `rg` to see how long it'll take to dish out what I need. *Later in this blogpost, I'll compare the search engines*

```json
perf stat packetbeat -c packetbeat.yml -I bigdnssample.pcap | rg 'prppsn.com'

...
{"@timestamp":"2020-08-09T05:11:59.487Z","@metadata":{"beat":"packetbeat","type":"_doc","version":"7.7.1"},"network":{"community_id":"1:pBBz9d/Bym26AyfI+91kH6HRfYs=","bytes":952,"type":"ipv4","transport":"udp","protocol":"dns"},"method":"QUERY","source":{"ip":"x.x.x.x","port":24609,"bytes":42},"host":{"name":"ali-pc"},"destination":{"ip":"y.y.y.y","port":53,"bytes":910},"client":{"ip":"x.x.x.x","port":24609,"bytes":42},"status":"OK","agent":{"id":"a28299f5-058a-43d1-9d43-eaafc45c112d","version":"7.7.1","type":"packetbeat","ephemeral_id":"7b8a8b2a-9a5e-4f69-ab78-bd48e2e4d7a6","hostname":"ali-pc"},"resource":"p5.prppsn.com","query":"class IN, type A, p5.prppsn.com","server":{"bytes":910,"ip":"y.y.y.y","port":53},"type":"dns","event":{"kind":"event","category":"network_traffic","dataset":"dns","duration":511439322,"start":"2020-08-09T05:11:59.487Z","end":"2020-08-09T05:11:59.999Z"},"dns":{"type":"answer","header_flags":["DO"],"answers_count":0,"id":23793,"response_code":"NOERROR","opt":{"do":true,"version":"0","udp_size":4096,"ext_rcode":"NOERROR"},"additionals_count":12,"authorities_count":6,"op_code":"QUERY","flags":{"recursion_desired":false,"recursion_available":false,"authentic_data":false,"checking_disabled":false,"authoritative":false,"truncated_response":false},"question":{"etld_plus_one":"prppsn.com","registered_domain":"prppsn.com","top_level_domain":"com","subdomain":"p5","name":"p5.prppsn.com","type":"A","class":"IN"}},"ecs":{"version":"1.5.0"}}
...

Gave up after 6200 seconds, 32 results are returned 
```

## PassiveDNS

version:
```sh
[*] PassiveDNS 1.2.1
[*] By Edward Bjarte Fjellskål <edward.fjellskaal@gmail.com>
[*] Using libpcap version 1.9.0-PRE-GIT (with TPACKET_V3)
[*] Using ldns version 1.7.0
```

```sh
$ pdns -r bigdnssample.pcap -j -l /dev/stdout -L /dev/null |  pv --line-mode --rate > /dev/null
~[3k/s]
```
`passivedns` turned out to be a big disappointment, mainly due to the fact that it's single-core and single threaded. But let's go through the `prppsn.com` search to see what happens anyway

```json
$ perf stat pdns -r bigdnssample.pcap -j -l /dev/stdout -L /dev/null |  rg 'prppsn.com'

...
{"timestamp_s":*REDACTED*,"timestamp_ms":50069,"client":"x.x.x.x","server":"y.y.y.y","class":"IN","query":"p8.prppsn.com.","type":"CNAME","answer":"i5u70.drt.cdn13.com.","ttl":300,"count":1}
...
Gave up after 6000 seconds, 10 results are returned during this time

```


## GoPassiveDNS

version:

since `gopassivedns` doesn't have a versioning system, I'll put the last commit hash here for reference: `9397838f12864410c26f6e7b6b2a1b59f3746698`

```sh
$ gopassivedns -pcap bigdnssample.pcap |  pv --line-mode --rate > /dev/null
~[100k/s]
```
Ok ok looks promising. Let's do our test against `gopassivedns` and see what happens

```json
$ perf stat gopassivedns -pcap bigdnssample.pcap |  rg 'prppsn.com'

...
{"query_id":48129,"rcode":0,"q":"p0.prppsn.com","qtype":"A","a":"i5u70.drt.cdn13.com","atype":"CNAME","ttl":121,"dst":"y.y.y.y","src":"x.x.x.x","tstamp":"2020-08-09 05:13:11.384622396 +0000 UTC","elapsed":103779687478393,"sport":"28159","level":"","bytes":135,"protocol":"udp","truncated":false,"aa":false,"rd":true,"ra":false}
...

 Performance counter stats for 'gopassivedns -pcap bigdnssample.pcap':

      8,273,326.76 msec task-clock:u              #    1.556 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
            53,381      page-faults:u             #    0.006 K/sec                  
16,854,776,994,690      cycles:u                  #    2.037 GHz                    
21,366,259,885,030      instructions:u            #    1.27  insn per cycle         
 4,717,723,735,096      branches:u                #  570.233 M/sec                  
    53,260,241,533      branch-misses:u           #    1.13% of all branches        

    5317.668579857 seconds time elapsed

    7181.536296000 seconds user
    1316.158187000 seconds sys

106 results returned.
```


so, our fastest tool, took 5317 seconds to crunch 1200 seconds worth of DNS data. Nowhere near good enough.


### umpteen minutes later, getting bored

Biggest takeaway from the previous tools: By design, they can't handle traffic thrown at them with this rate. ~20 minutes worth of traffic takes well over ~20 minutes to be parsed by any of these tools and except for `tshark`, none of the tools fully utilized the resources thrown at them, and all 4 maxed out at using 20% of my CPU capacity. 

Max RAM usage of `packetbeat` was 0.3%, while `gopassivedns` was at 0.2% and `passivedns` was sitting at 3.7% max. For those who are wondering, disk I/O was well under 10% after the initialization of all of these platform. The next logical step would be to turn to solutions that are designed to be equipped to deal with large quantities of data at high rate: IDS/IPS solutions.

## Suricata

Suricata is supposed to be very fast at parsing packets, leveraging all full CPU capacity and utilizing a very fast Lua JIT as well as some Rust enhancement to the code. Let's try the following version:

```sh
$ suricata -V

This is Suricata version 6.0.0-dev (db75675f4 2020-07-09)
```

Suricata rule:

```
alert udp any any -> any 53 ( msg:"Is this really PSN?";  content:"prppsn"; nocase; priority:3; )
```

Command:

After making a Suricata folder to keep all the Suricata log outputs separated, I used the following to initiate the packet process

```sh
$ perf stat sudo  suricata -c /etc/suricata/suricata.yaml -r ../bigdnssample.pcap

[775501] 9/8/2020 -- 20:09:21 - (suricata.c:1064) <Notice> (LogVersion) -- This is Suricata version 6.0.0-dev (db75675f4 2020-07-09) running in USER mode
[775501] 9/8/2020 -- 20:09:21 - (output-json-stats.c:465) <Error> (OutputStatsLogInitSub) -- [ERRCODE: SC_ERR_STATS_LOG_GENERIC(278)] - eve.stats: stats are disabled globally: set stats.enabled to true. See https://suricata.readthedocs.io/en/latest/configuration/suricata-yaml.html#stats
[775501] 9/8/2020 -- 20:09:21 - (tm-threads.c:1887) <Notice> (TmThreadWaitOnThreadInit) -- all 9 packet processing threads, 2 management threads initialized, engine started.

[775501] 9/8/2020 -- 21:16:52 - (suricata.c:2617) <Notice> (SuricataMainLoop) -- Signal Received.  Stopping engine.
[775503] 9/8/2020 -- 21:17:11 - (source-pcap-file.c:371) <Notice> (ReceivePcapFileThreadExitStats) -- Pcap-file module read 1 files, 222780376 packets, 28054893332 bytes

 Performance counter stats for 'sudo suricata -c /etc/suricata/suricata.yaml -r ../bigdnssample.pcap':

    4079.763122836 seconds time elapsed

   30275.549690000 seconds user
     508.539254000 seconds sys


```
The CPU almost immediately jumps to 100% and 22% of my RAM is occupied by the Suricata process, and it finishes in ~4000 seconds. Given the fact that my CPU was fully utilized, I assume if I tweak the config of Suricata and/or throw better hardware at it, it'll be scalable enough to handle these numbers fairly easily. 

## Snort

I used `vimagick`'s snort docker image and created a rule just like the Suricata one to alert on the `prppsn` string. The result was surprisingly good!

Version:

```
   ,,_     -*> Snort! <*-
  o"  )~   Version 2.9.16 GRE (Build 118) 
   ''''    By Martin Roesch & The Snort Team: http://www.snort.org/contact#team
           Copyright (C) 2014-2020 Cisco and/or its affiliates. All rights reserved.
           Copyright (C) 1998-2013 Sourcefire, Inc., et al.
           Using libpcap version 1.5.3
           Using PCRE version: 8.32 2012-11-30
           Using ZLIB version: 1.2.7
```


```sh
Performance counter stats for 'sudo docker run --rm vimagick/snort -c /etc/snort/snort.conf -r /input.pcap -A full':

     380.446791709 seconds time elapsed

       0.063899000 seconds user
       0.034766000 seconds sys
```

380 Seconds! Incredibly fast. Although to be fair, Snort doesn't log the actual packets in a good format. However, It does create a `snort.log` with the content of the packets so it can later on be used to re-create those packets or parse them using `idstools` package. Also, Snort just used 1 CPU core and almost ~5% RAM. Impressive!

## DNSMonster and ClickHouse

Now let's talk about indexing and making DNS data searchable. I've talked about `dnsmonster` in detail in [another blogpost](///2020/02/2020-02-05-dnsmonster/). Let's quickly push this data to `dnsmonster` and into my local Clickhouse instance and see how long it'll take to index the whole file and make it searchable:

```sh
$ dnsmonster -serverName=pcaptest -pcapFile bigdnssample.pcap -clickhouseAddress=127.0.0.1:9000 -batchSize=1000000
```

The operation took ~15 minutes, giving the 20 minute `pcap` a run for its money. And now everything becomes much much more interesting! Let's fire up our Clckhouse client and issue a couple of queries to see how long it'll take to get some results:

```sql
SELECT COUNT(*)
FROM DNS_LOG
WHERE Question LIKE '%prppsn.com%'

┌─COUNT()──┐
│     136  │
└──────────┘

1 rows in set. Elapsed: 1.437 sec. Processed 222.42 million rows, 7.23 GB (154.79 million rows/s., 5.03 GB/s.) 

SELECT *
FROM DNS_LOG
WHERE Question LIKE '%prppsn%'
FORMAT CSV

"2020-08-08","2020-08-08 00:20:31","pcaptest",4,0000001111,"udp",0,0,1,1,0,0,0,"p5.prppsn.com.",31
"2020-08-08","2020-08-08 00:20:31","pcaptest",4,0000002222,"udp",0,0,1,1,1,1,0,"p5.prppsn.com.",42
"2020-08-08","2020-08-08 00:20:31","pcaptest",4,0000003333,"udp",1,0,1,1,1,1,0,"p5.prppsn.com.",910
...

136 rows in set. Elapsed: 1.327 sec. Processed 222.42 million rows, 7.23 GB (167.65 million rows/s., 5.45 GB/s.) 



```

under two seconds and all results returned!

{{< echarts >}}
{
  "tooltip":{
    "trigger":"axis",
    "axisPointer":{"type":"cross","crossStyle":{"color":"#999"}}
  },
  "toolbox":{
    "feature":{
      "saveAsImage":{"title": "Save as Image"}
    }
  },
  "legend":{"data":["蒸发量","降水量","平均温度"]},
  "xAxis":[
    {
      "type":"category",
      "data":["PassiveDNS","Packetbeat","GoPassiveDNS","Suricata","Snort","DNSMonster"],
      "axisPointer":{"type":"shadow"}
    }
  ],
  "yAxis":[
    {
      "type":"value",
      "name":"Completion Time",
      "min":0,
      "max":10000,
      "interval":1000,
      "axisLabel":{"formatter":"{value}"}
    },
    {
      "type":"value",
      "name":"CPU Usage",
      "min":0,
      "max":100,
      "interval":10,
      "axisLabel":{"formatter":"{value} %"}
    }
  ],
  "series":[
    {
      "name":"Seconds to Index",
      "type":"bar",
      "data":[6000,6200,5317,4079,380,835]
    },
    {
      "name":"CPU Usage (%)",
      "type":"line",
      "yAxisIndex":1,
      "data":[12.5,40,35,99,12.5,30]
    }
  ]
}

{{< /echarts >}}
## Conclusion

Depending on your use case, Wireshark might not be the best idea to deal with large `pcap` files. It's merely designed to represent the traffic flow in a nice manner with one of the most comprehensive packet parsers there is. My recommandetion is to use Wireshark and its family in you lab env and with a small `pcap` to come up with what you want to solve, and then leverage something like `dnsmonster` or on a larger scale, `moloch` to index and search across a giant network dump. 


The next step for me is to try and connect `dnsmonster` with something like `scikit` to auto-detect anomalies in a massive pcap file with Clickhouse as a search and index middleware. That way, the "intersting" packets will pop up automatically so you don't have to deal with milliions of ordinary packets. But that's a different experiment for another day :) 

