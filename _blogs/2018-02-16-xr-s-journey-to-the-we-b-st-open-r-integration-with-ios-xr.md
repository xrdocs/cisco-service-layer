---
published: true
date: '2018-02-16 10:46 +0530'
title: 'XR''s journey to the web:  Open/R integration with XR'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - linux
  - servicelayer
  - service-layer
  - application
  - igp
  - gRPC
  - openr
  - open/r
  - open-r
  - rib
position: hidden
excerpt: >-
  This blog series chronicles IOS-XR's Journey to the web through a series of
  integrations with community tools. Open/R, open sourced by Facebook in 2017 is
  a great example of open source innovation driving the evolution of vendor
  network stacks to be more open and pluggable. In this blog, we understand the
  various integration points and expectations with Open/R and see how IOS-XR
  lives up to the task at hand, quite admirably.
---

{% include toc %}

## IOS-XR's Journey to the West (Web)


As part of this blog series titled **"XR's journey to the Web"**, I intend to candidly present the journey that we have so earnestly taken with the IOS-XR software stack since 2014. In line with the ethos of xrdocs, expect this series to be highly technical and heavily focused on showcasing how IOS-XR integrates with community tools and open-source software innovations - where does it excel, where does it falter, and what needs to be done to help it improve.    
  
<a href="/cisco-service-layer/images/journey_to_the_west.png"><img src="/cisco-service-layer/images/journey_to_the_west.png" alt="Chinese folklore - Journey to the west" class="align-right" width="300" height="300"></a> 

As an allegorical reference let me present to you the famous Chinese folklore called **"Journey to the west"** - a 16th Century Chinese novel by Wu Cheng-en.  

It chronicles the story of Xuanzang in the 6th Century AD and his journey to India (the "west") to seek out and bring back the teachings of Buddhism to a primarily Confucian China. As Buddhism began to spread through the east, a new movement began in China that sought to marry the practical Confucian core with the new intellectual and spiritual expectations arising out of Buddhism. This movement came to be known as neo-confucianism.  


<a href="/cisco-service-layer/images/xr-journey-to-the-web.png"><img src="/cisco-service-layer/images/xr-journey-to-the-web.png" alt="XR's journey to the web" class="align-left" width="500" height="300"></a>

I liken the changes that have happened to IOS-XR over the last few years to a similar creative reinterpretation of the core concepts of traditional networking. As the "Web" players (Google, Facebook, Apple, Amazon etc.) began to showcase how better efficiencies may be achieved in network operations through **automation at every stage of deployment**, and through the ability to **"run with scissors"** - it became clear that the traditional core of network stacks would have to evolve to meet the operational demands of these highly automated networks.  

The evolution was comprised of some very interesting developments:

  *  **Linux-ization of the Stack**: IOS-XR moved from 32-bit QNX to 64-bit Linux to enable an 
     environment for scripting, [**hosting applications**](https://xrdocs.github.io/application-hosting/) and integration with tools in the DevOps space such as Ansible, Puppet, Docker, etc.  
     
  *  **Streaming Telemetry**: [**Real-time push-based Streaming Telemetry**](https://xrdocs.github.io/telemetry/) for monitoring, alerts and 
     remediation triggers/events, outperforming SNMP in every department - scale, ease-of-use, 
     cadence etc.  
     
  *  **Model Driven APIs at every layer of the stack**: This includes [**Yang Models at the Manageability layer**](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr) and [**tools to generate bindings in various languages**](http://ydk.io). Further,in September 2017, we took it a step further - we introduced model driven APIs over gRPC called the [**Service Layer APIs**](https://xrdocs.github.io/cisco-service-layer/) that provide programmatic access to the IOS-XR RIB, label switch database and notifications for interface and BFD events.

This blog series will focus on how a combination of the above enhancements should allow us to integrate with a wide variety of tools and software stacks in the networking community and let our users run with scissors when needed.



In this blog, we shall explore how [IOS-XR's service layer APIs](https://xrdocs.github.io/cisco-service-layer/) and [application hosting capabilities](https://xrdocs.github.io/application-hosting/) can be leveraged to host and integrate Open/R as an IGP on IOS-XR. We will also touch upon further enhancements to Open/R that may be possible with Service Layer APIs serving as the platform hooks with IOS-XR. 

## What is Open/R?

In November 2017, Facebook finally open sourced [Open/R](https://github.com/facebook/openr).  
As the Github description suggests, it is, and I quote, a "Distributed platform for building autonomic network functions". Pretty heavy description, so let's distill it a bit.

Much of the documentation for open/R can be found in the docs directory in the git repo:

><https://github.com/facebook/openr/tree/master/openr/docs>

It is laid out rather well and describes all the components of the code individually - their purpose, internal interactions, et al.

At a higher level, the components look something like this:

![openr_high_level]({{ site.baseurl }}/images/openr_high_level.png)

At the outset, the architecture is reminiscent of traditional link state routing protocols like ISIS - what with the initial Hellos used to identify neighbors (similar to IS-IS Hello Packets), establishment of adjacencies using 0MQ messages (similar to Link State PDUs (LSPs) in IS-IS) and the use of Djikstra's algorithm for SPF computations. It also borrows ideas from other protocols like BGP (the concept of originatorIDs for loop prevention, à la AS-PATH handling) and spanning tree for flood optimization of messages.

So is it really just an alternative to traditional IGPs ? Not quite. There are some design decisions taken to enable the architecture to be pluggable from the get-go:   

  *  **KV-Store**:  A Key-Value store that serves as a common database for the entire stack. Peering Messages to other routers are sent out over 0MQ channels. The internal  communication with other components such as the Prefix Manager, Decision module or the link monitor module are handled by sending and receiving thrift objects on sockets through an abstracted KvStoreClient.

  *  **Thrift based modeled APIs** between all the modules: for example, the FIB module implements a thrift client and the platform module implements a thrift server to receive route batches from the FIB before programming the underlying platform. These interaction RPCs and the data structures such as the route updates are modeled in thrift IDL files (See <https://github.com/facebook/openr/tree/master/openr/if>)

Consequently, integrations with existing stacks and platforms can cleanly occur at the lower platform layer abstracted through the modeled thrift interface. Newer functionalities that leverage the underlying platform's capabilities (like MPLS, BFD, SR etc.) can extend or implement a new thrift model and leverage the KVstore to store data locally and share information with other routers easily.  

  
### Where do I start?

The original developers at Facebook were gracious enough to release a netlink platform integration for Open/R to enable the community to take a look at how things tie in internally.

This netlink platform integration enables Open/R to run as a routing stack on top of a Linux kernel as the network stack. You can check out the relevant pieces of code here:  

>The **"Platform"** module code runs a thrift Server and receives route batches from the Fib module that runs a thrift client
><https://github.com/facebook/openr/tree/master/openr/platform>

This consists of two important abstractions:
  *  **NetlinkFibhandler**:  implements the FibService interface described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift> to handle the incoming route batches from the Fib module
  *  **NetlinkSystemHandler**: implements the SystemService interface again described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift> to detect interfaces and IPv6 neighbors in the kernel that may be used to send hellos and peering messages to neighbors.  
  
  
>The **"Netlink(nl)"** abstraction  handles actual interaction with the kernel through netlink using the libnl library. The Netlink platform handlers described above utilize this abstraction to program and fetch routes and get a list of ipv6 neighbors or links or associated events from the kernel.
><https://github.com/facebook/openr/tree/master/openr/nl>


### Vagrant Setup for Open/R on Linux

If you'd like to try a back-to-back setup with two linux instances on your laptop, I've published a vagrant setup with two ubuntu 16.04 instances (rtr1 and rtr2) connected through an ubuntu switch:  

><https://github.com/akshshar/openr-vagrant>

We will do a performance comparison between IOS-XR as a platform for Open/R (using service layer APIs) vs linux (using Netlink) later on.
{: .notice--info}

The vagrant provisioners will install open/R on "vagrant up" on both rtr1 and rtr2 and will setup the required "run" script for openr at `/usr/sbin/run_openr.sh` one each node.

The switch in the middle is a nice-to-have. It allows you to capture packets as the two nodes rtr1 and rtr2 exchange hellos and peering messages.

Clone the above git repo and issue a `vagrant up` inside the directory:

If you're behind a proxy, just populate the `<git repo directory>/scripts/http_proxy` `<git repo directory>/scripts/https_proxy` files before issuing a `vagrant up`.
{: .notice--warning} 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
cisco@host:~$ <mark> git clone https://github.com/akshshar/openr-vagrant </mark>
Cloning into 'openr-vagrant'...
remote: Counting objects: 31, done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 31 (delta 14), reused 26 (delta 12), pack-reused 0
Unpacking objects: 100% (31/31), done.
Checking connectivity... done.
cisco@host:~$ <mark>cd openr-vagrant/</mark>
cisco@host:~$ <mark>vagrant up</mark>
Bringing machine 'rtr1' up with 'virtualbox' provider...
Bringing machine 'switch' up with 'virtualbox' provider...
Bringing machine 'rtr2' up with 'virtualbox' provider...



</code>
</pre>
</div>

The provisioning scripts build open/R from scratch on rtr1 and rtr2, so expect this bringup to take a long time. You could parallelize the effort by commenting out the provisioners for rtr1 and rtr2  in the Vagrantfile, bring up the nodes, uncomment the provisioners and then run `vagrant provision rtr1` and `vagrant provision rtr2` in two separate terminals simultaneously.
{: .notice--warning}  

Once the devices are up, issue a `vagrant ssh rtr1` and `vagrant ssh rtr2` in separate terminals and start open/R (The run scripts added to each node will automatically detect the interfaces) and start discovering each other).  

Further, for rtr2, I've added a scaling python script that allows you to add up to 8000 routes by manipulating the `batch_size` and `batch_num` values in `<git repo directory>/scripts/increment_ipv4_prefix.py` before running /usr/sbin/run_openr.sh 
{: .notice--info}

```
vagrant@rtr1:~$ /usr/sbin/run_openr.sh 
/usr/sbin/run_openr.sh: line 98: /etc/sysconfig/openr: No such file or directory
Configuration not found at /etc/sysconfig/openr. Using default configuration
openr[10562]: Starting OpenR daemon.

......


vagrant@rtr2:~$ /usr/sbin/run_openr.sh 
/usr/sbin/run_openr.sh: line 98: /etc/sysconfig/openr: No such file or directory
Configuration not found at /etc/sysconfig/openr. Using default configuration
openr[10562]: Starting OpenR daemon.

......

```




### Capturing Open/R Hellos and Peering messages

On the switch start up a tcpdump capture on one or more of the bridges on the switch in the middle:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:openr-two-nodes akshshar$<mark> vagrant ssh switch </mark>

#######################  snip #############################

Last login: Thu Feb 15 11:04:25 2018 from 10.0.2.2
vagrant@vagrant-ubuntu-trusty-64:~$<mark> sudo tcpdump -i br0 -w /vagrant/openr.pcap </mark>
tcpdump: listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes


</code>
</pre>
</div>

Open up the pcap file in wireshark and you should see the following messages show up:

  *  **Hello Messages**: The hello messages are sent to UDP port 6666 to the All nodes IPv6 
     Multicast address ff02::1 and source IP = Link local IPv6 address of node. These messages are 
     used to discover neighbors and learn their link local IPv6 addresses.  
     
     ![Openr/R hello messages]({{site.baseurl}}/images/openr_hellos.png)
     {: .notice--info}

  *  **Peering Messages**: Once the link local IPv6 address of neighbor is known, 0MQ TCP messages 
     are sent out to create an adjacency with the neighbor on an interface. One such message is 
     shown below:  
     
     ![0MQ messages openr]({{site.baseurl}}/images/0mq_openr.png)
     {: .notice--info}

### Open/R breeze CLI

Once the peering messages go through, adjacencies should get established with the neighbors on all connected interfaces. These adjacencies can be verified using the "breeze" cli:  

```
vagrant@rtr1:~$ breeze kvstore adj

> rtr1's adjacencies, version: 12, Node Label: 42122, Overloaded?: False
Neighbor    Local Interface    Remote Interface      Metric    Weight    Adj Label  NextHop-v4    NextHop-v6                Uptime
vagrant     enp0s9             enp0s9                     7         1        50004  0.0.0.0       fe80::a00:27ff:fed1:ba15  0m2s
vagrant     enp0s8             enp0s8                     8         1        50003  0.0.0.0       fe80::a00:27ff:fe94:3015  0m4s
vagrant     enp0s10            enp0s10                    7         1        50005  0.0.0.0       fe80::a00:27ff:fe36:aec9  0m6s


```


Let's look at the fib state in Open/R using the breeze cli:

```
vagrant@rtr1:~$ breeze fib list

== rtr1's FIB routes by client 786  ==

> 100.1.1.0/24
via 0.0.0.0@enp0s10
via 0.0.0.0@enp0s8
via 0.0.0.0@enp0s9

> 100.1.10.0/24
via 0.0.0.0@enp0s10
via 0.0.0.0@enp0s8
via 0.0.0.0@enp0s9

> 100.1.100.0/24
via 0.0.0.0@enp0s10

......

```

If you used the default scaling script on rtr2, then rtr1 should now have about 1000 routes in its fib:  


```
vagrant@rtr1:~$ breeze fib counters

== rtr1's Fib counters  ==

fibagent.num_of_routes : 1004

vagrant@rtr1:~$ 

```

Great! These outputs should give you a fair gist of how Open/R works as a link state routing protocol.
{: .notice--success}


## Integrating Open/R with IOS-XR

### Requirements

Now that we understand how Open/R operates, let's codify the requirements for it work on a platform running Linux:  

  *  **API to get and set Routes**: On Linux, this API is Netlink, but it can be replaced with any 
     viable API that the networking stack on the platform offers.  
     
       In September 2017, we introduced Service Layer APIs - a highly performant and model driven 
       API into the network infrastructure layer (RIB, label switch database, interface and BFD 
       events) over gRPC. This API is ideal for platform integration with Open/R and as we'll see        later,achieves higher performance than Netlink due to its route batching capability. This 
       API is available on IOS-XR releases post 6.1.2.
       {: .notice--info}
  
  *  **Ability to host applications**: The Network OS must have the capability to host Linux 
     applications either natively or as a container (docker/lxc).
       
       With IOS-XR 6.1.2+, the capability to host linux applications on the box with docker was 
       introduced. In addition, applications can be hosted within LXC containers or even natively 
       (if compiled for WRL7). Further, use of network namespaces mapped to IOS-XR vrfs was 
       introduced in release 6.2.2+, allowing isolation of traffic in the kernel based on vrf 
       configuration.
       {: .notice--info}
  
  *  **Ability to exchange Hellos and Peering Messages**: Open/R should be able to run unomodified 
     on a platform and send its UDP hellos to port 6666 and ff02::1 and send/receive TCP peering 
     messages using link local IPv6 addresses of neighbors.
     
       Packet/IO capabilities have existed in IOS-XR since 6.0.0+, allowing applications to bind 
       to XR interfaces in the kernel, open up the TCP/UDP ports, transmit/receive TCP/UDP traffic 
       along with exception traffic such as icmp, icmpv6, IPv6 multicast etc.
       {: .notice--info}
  



  








