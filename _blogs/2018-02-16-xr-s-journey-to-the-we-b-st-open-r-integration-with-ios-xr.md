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


As part of this blog series titled **"XR's journey to the Web"**, I intend to candidly present the journey that we have so earnestly taken with the IOS-XR software stack since 2014. In line with the ethos of xrdocs, expect this series to be highly technical and heavily **focused on showcasing how IOS-XR integrates with community tools and open-source software innovations - where does it excel, where does it falter, and what needs to be done to help it improve**.    
  
<a href="/cisco-service-layer/images/journey_to_the_west.png"><img src="/cisco-service-layer/images/journey_to_the_west.png" alt="Chinese folklore - Journey to the west" class="align-right" width="300" height="300"></a> 

The allegorical reference that I like to allude to as we embark on this journey is a famous Chinese folklore called **"Journey to the west"** - a 16th Century Chinese novel by Wu Cheng-en.  

It chronicles the story of Xuanzang in the 6th Century AD and his original journey to India (the "west") to seek out and bring back the teachings of Buddhism to a primarily Confucian China. I may be oversimplifying the events, however, as Buddhism began to spread through the east, a new movement began that sought to marry the practical Confucian core with the new intellectual and spiritual expectations arising out of Buddhism. This movement came to be known as neo-confucianism.  



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


## Integrating Open/R with IOS-XR

In this blog, we shall explore how [IOS-XR's service layer APIs](https://xrdocs.github.io/cisco-service-layer/) and application hosting capabilities can be leveraged to host and integrate Open/R as an IGP on IOS-XR. We will also touch upon further enhancements to Open/R that may be possible with Service Layer APIs serving the platform hooks with IOS-XR. 

### What is Open/R?

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

  
### Where do it start?

The original developers at Facebook were gracious enough to release a netlink platform integration for Open/R to enable the community to take a look at how things tie in internally.

This netlink platform integration enables Open/R to run as a routing stack on top of a Linux kernel as the network stack. You can check out the relevant pieces of code here:  

>The **"Platform"** module code that runs a thrift Server and receives route batches from the Fib module running a thrift client
><https://github.com/facebook/openr/tree/master/openr/platform>

This consists of two important abstractions:
  *  **NetlinkFibhandler**:  implements the FibService interface described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift> to handle the incoming route batches from the Fib module
  *  **NetlinkSystemHandler**: implements the SystemService interface again described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift> to detect interfaces and IPv6 neighbors in the kernel that may be used to send hellos and peering messages to neighbors.  
  
  
>The **"Netlink(nl)"** abstraction that handles actual interaction with the kernel using the libnl3 library
><https://github.com/facebook/openr/tree/master/openr/nl>


  








