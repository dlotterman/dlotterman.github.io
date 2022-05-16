---
layout: post
title:  "-rehost- Networking: Using Linux Traffic Control for Fun and Profit Loss Prevention"
date:   2013-11-19
tags: networking linux tooling bitly
description: "Here at bitly, we are big fans of data, tubes and especially tubes that carry data."
---

*This is a rehost of a blog post I wrote for the [Bitly Engineering blog](https://word.bitly.com/post/67486390974/networking-traffic-control) back in 2013. I'm rehosting it here for my own context and control. There is some language clean-up I intend to do as well.*


Here at bitly, we are big fans of data, tubes and especially tubes that carry data.

This is a story about asking tubes to carry too much data.

A physical [hadoop](https://bitly.com/nw6gyy) cluster has been a significant part of bitly’s infrastructure and a core tool of the bitly Data Science and Ops/Infra teams for some time. Long enough that we needed to cycle in a new cluster, copy data, and fold the old into the new.

Branches were opened, work was done, servers provisioned and the new cluster was stood up. Time to take a step back from the story and flesh out some technical details:

* bitly operates at a consequential scale of data: At the time of this migration, bitly's hadoop cluster was just over 150TB consumed disk space (before downstream denormalization) of compressed stream data, that is data that is the valuable output of our various applications after having been manipulated and expanded on by other applications.

* bitly’s physical presence is collocated with our data center partner. There are three physical chassis classes (application, storage and database) racked together in contiguous cabinets in rows. At the time of this story each chassis had three physical 1Gb Ethernet connections (each logically isolated by VLANs), frontlink, backlink and lights-out (for out of band management of the server chassis). Each connection, after a series of cabinet specific patch panels and switches, connects to our core switches over [10Gb glass](https://bit.ly/18szUaI) in a hub and spoke topology.

* While bitly also operates at a consequential physical scale (hundreds of physical server chassis), we depend on our data center partner for network infrastructure and topology. This means that within most levels of the physical networking stack, we have severely limited control and visibility.

Back to the story:

The [distcp tool bundled with hadoop](https://hadoop.apache.org/docs/current/hadoop-distcp/DistCp.html) allowed us to quickly copy data from one cluster to the other. Put simply, the `distcp` tool creates a mapreduce job to shuffle data from one hdfs cluster to another, in a *many to many node* copy. Distcp was fast, which was good, then:

*bitly broke, and the Ops/Infra team was very sad.*

Errors and latent responses were being returned to website users and api clients. We discovered that services were getting errors from other services, database calls were timing out and even DNS queries internal to our network were failing. We determined that the copy had caused unforeseen duress on the network tubes, particularly the tubes of the network that carried traffic cross physical cabinets. However the information given to us by our data center partner only added to the confusion: no connection, not even cabinet to core, showed signs of saturation, congestion or errors.

We now had two conflicting problems: a need to continue making headway with the hadoop migration as well as troubleshooting and understanding our network issues.

We limited the number of mapreduce mappers that `distcp` used to copy data between clusters, which artificially throttled throughput for the copies, allowing us to resume moving forward with the migration. Eventually the copies completed, and we were able to swap the new cluster in for the old.

The new cluster had more nodes, which meant that hadoop was faster:

*The Data Science team was happy.*

Unfortunately, with hadoop being larger and faster, more data was getting shipped around to more nodes during mapreduce workloads, which had the unintended effect of:

*bitly broke, a lot. The Ops/Infra team was very sad.*

The first response action was to turn hadoop off:

*The Data Science team was sad.*

A turned off hadoop cluster is bad (just not as bad as breaking bitly), so we time warped the cluster back to 1995 by forcing all NICs to re-negotiate at 100Mbps (as opposed to 1Gbps) using `ethtool -s eth1 speed 100 duplex full autoneg on`. Now we could safely turn hadoop on, but it was painfully slow.

*The Data Science team was still sad.*

In fact it was so slow and congested, that data ingestion and scheduled ETL/reporting jobs began to fail frequently, triggering alarms that woke up Ops/Infra team members up in the middle of the night:

*The Ops/Infra team was still sad.*

Because of our lack of sufficient visibility into the state of the network, working through triage and troubleshooting with our data center partner was going to be an involved and lengthy task. Something had to be done to get hadoop into a usable state, while protecting bitly from breaking.

Time to take another step back:

Some tools we have at bitly:

* `roles.json` : A list of servers (app01, app02, userdb01, hadoop01 etc), roles (userdb, app, web, monitoring, hadoop_node etc), and the mapping of servers into roles (app01,02 -> app, hadoop01,02 -> hadoop_node etc).

* `$datacenter/jsons/*` : A directory containing a json file per logical server, with attributes describing the server such as ip address, names, provisioning information and most importantly for this story; cabinet location.

* `linux : Linux.`

Since we could easily identify what servers do what things, where that server is racked and can leverage all the benefits of Linux, this was a solvable problem, and we got to work.

*And the Ops/Infra team was sad.*

Because Linux’s networking Traffic Control (tc) syntax was clunky and awkward and its documentation intimidating. After much swearing and keyboard smashing, perseverance paid off and working examples of tc magic surfaced. Branches were opened, scripts were written, deploys done, benchmarks run and finally some test nodes were left with the following:

```
$ tc class show dev eth1
class htb 1:100 root prio 0 rate 204800Kbit ceil 204800Kbit burst 1561b
    cburst 1561b
class htb 1:10 root prio 0 rate 819200Kbit ceil 819200Kbit burst 1433b 
    cburst 1433b
class htb 1:20 root prio 0 rate 204800Kbit ceil 204800Kbit burst 1561b 
    cburst 1561b

$ tc filter show dev eth1
filter parent 1: protocol ip pref 49128 u32 
filter parent 1: protocol ip pref 49128 u32 fh 818: ht divisor 1 
filter parent 1: protocol ip pref 49128 u32 fh 818::800 order 2048 key 
    ht 818 bkt 0 flowid 1:20 
    match 7f000001/ffffffff at 16
filter parent 1: protocol ip pref 49129 u32 
filter parent 1: protocol ip pref 49129 u32 fh 817: ht divisor 1 
filter parent 1: protocol ip pref 49129 u32 fh 817::800 order 2048 key 
    ht 817 bkt 0 flowid 1:10 
    match 7f000002/ffffffff at 16
filter parent 1: protocol ip pref 49130 u32 
filter parent 1: protocol ip pref 49130 u32 fh 816: ht divisor 1 
filter parent 1: protocol ip pref 49130 u32 fh 816::800 order 2048 key 
    ht 816 bkt 0 flowid 1:20 
    match 7f000003/ffffffff at 16
<snipped>

$ tc qdisc show
qdisc mq 0: dev eth2 root 
qdisc mq 0: dev eth0 root 
qdisc htb 1: dev eth1 root refcnt 9 r2q 10 default 100 
    direct_packets_stat 24
```

In plain English, there are three traffic control classes. Each class represents a logical group, to which a filter can be subscribed, such as:

`class htb 1:100 root prio 0 rate 204800Kbit ceil 204800Kbit burst 1561b cburst 1561b`

Each class represents a ceiling or throughput limit of outgoing traffic aggregated across all filters subscribed to that class.

Each filter is a specific rule for a specific ip (unfortunately each IP is printed in hex) , so the filter

```
filter parent 1: protocol ip pref 49128 u32 
filter parent 1: protocol ip pref 49128 u32 fh 818: ht divisor 1 
filter parent 1: protocol ip pref 49128 u32 fh 818::800 order 2048 key 
    ht 818 bkt 0 flowid 1:20 
    match 7f000001/ffffffff at 16
```

can be read as “subscribe `hadoop14` to the class `1:20`” where `7f000001` can be read as the IP for `hadoop14` and `flowid 1:20` is the class being subscribed to.

Finally there is a qdisc, which is more or less the active queue for the `eth1` device. That queue defaults to placing any host that is not otherwise defined in a filter for a class, into the `1:100` class.

`qdisc htb 1: dev eth1 root refcnt 9 r2q 10 default 100 direct_packets_stat 24`

With this configuration, any host, hadoop or not, that is in the same cabinet as the host being configured gets a filter that is assigned to the `1:10` class, which allows up to ~800Mbps for the class as a whole. Similarly, there is a predefined list of roles that are deemed “roles of priority hosts”, which get a filter created on the same `1:100` rule. These are hosts that do uniquely important things, like running the hadoop namenode or jobtracker services, and also our monitoring hosts.

Any other hadoop host that is not in the same cabinet is attached to the `1:20` class, which is limited to a more conservative ~200Mbps limit.

As mentioned before, any host not specified by a filter gets caught by the default class for the `eth1` qdisc, which is `1:100`.

What does this actually look like? Here is a host that is caught by the catch all `1:100` rule:

```
[root@hadoop27 ~]# iperf -t 30 -c NONHADOOPHOST
------------------------------------------------------------
Client connecting to NONHADOOPHOST, TCP port 5001
TCP window size: 23.2 KByte (default)
------------------------------------------------------------
[  3] local hadoop27 port 35897 connected with NONHADOOPHOST port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-30.1 sec   735 MBytes   205 Mbits/sec
```

Now when connecting to another host in the same cabinet, or the `1:10` rule:

```
[root@hadoop27 ~]# iperf -t 30 -c CABINETPEER
------------------------------------------------------------
Client connecting to CABINETPEER, TCP port 5001
TCP window size: 23.2 KByte (default)
------------------------------------------------------------
[  3] local hadoop27 port 39016 connected with CABINETPEER port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-30.0 sec  2.86 GBytes   820 Mbits/sec
```

Now what happens when connecting to two servers that match the `1:10` rule?

```
[root@hadoop27 ~]# iperf -t 30 -c CABINETPEER1
------------------------------------------------------------
Client connecting to CABINETPEER1, TCP port 5001
TCP window size: 23.2 KByte (default)
------------------------------------------------------------
[  3] local hadoop27 port 39648 connected with CABINETPEER1 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-30.0 sec  1.47 GBytes   421 Mbits/sec

[root@hadoop27 ~]# iperf -t 30 -c CABINETPEER2
------------------------------------------------------------
Client connecting to 10.241.28.160, TCP port 5001
TCP window size: 23.2 KByte (default)
------------------------------------------------------------
[  3] local hadoop27 port 38218 connected with CABINETPEER2 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-30.0 sec  1.43 GBytes   408 Mbits/sec
```

So the traffic got halved? Sounds about right.

Even better, trending the data was relatively easy by mangling the stats output to our trending services:

```
$ /sbin/tc -s class show dev eth1 classid 1:100
class htb 1:100 root prio 0 rate 204800Kbit ceil 204800Kbit 
    burst 1561b cburst 1561b 
Sent 5876292240 bytes 41184081 pkt (dropped 0, overlimits 0 requeues 0) 
rate 3456bit 2pps backlog 0b 0p requeues 0 
lended: 40130273 borrowed: 0 giants: 0
tokens: 906 ctokens: 906
```

After testing, we cycled through hadoop hosts, re-enabling their links to 1Gb after applying the traffic control roles. With deploys done, hadoop was use-ably performant:

*The Data Science team was happy.*

The Ops/Infra team could begin tackling longer term troubleshooting and solutions while being able to sleep at night, knowing that bitly was not being broken:

*The Ops/Infra team was happy.*

### Take aways:

* In dire moments: your toolset for managing your environment is as important as the environment itself. Because we already had the toolset available to holistically control the environment, we were able to dig ourselves out of the hole almost as quickly as we had fallen into it.

* Don’t get into dire moments: Understand the environment that you live in. In this case, we should have had a better understanding and appreciation for the scope of the hadoop migration and its possible impacts.

* Linux TC is a high cost, high reward tool. It was almost certainly written by people with the very longest of beards, and requires time and patience to implement. However we found it to be an incredibly powerful tool that helped save us from ourselves.

* `linux`: Linux

### EOL

This story is a good reminder of the [“Law of Murphy for devops”](https://twitter.com/DEVOPS_BORAT/status/281242898165538818). Temporary solutions like those in this story afforded us the time to complete troubleshooting of our network and implement permanent fixes. We have since unthrottled hadoop and moved it to its own dedicated network, worked around shoddy network hardware to harden our primary network and much more. Stay tuned.
