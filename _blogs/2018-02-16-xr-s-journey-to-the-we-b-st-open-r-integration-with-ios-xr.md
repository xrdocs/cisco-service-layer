---
published: true
date: '2018-02-16 10:46 +0530'
title: 'XR''s journey to the west:  Open/R integration with XR'
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
  integrations with community tools. Open/R, open sourced by Facebook in 2017
  is a great example of open source innovation driving the evolution of vendor
  network stacks to be more open and pluggable. In this blog, we understand the
  various integration points and expectations with Open/R and see how IOS-XR
  lives up to the task at hand, quite admirably.
---

{% include toc %}

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

  *  **KV-Store**:  A Key value store that serves as a common database for the entire stack. Peering Messages to other routers are sent out over 0MQ channels. The internal  communication with other components such as the Prefix Manager, Decision module or the link monitor module are handled by sending and receiving thrift objects on sockets through an abstracted KvStoreClient.

  *  **Thrift based modeled APIs** between all the modules: for example, the FIB module implements a thrift client and the platform module implements a thrift server to receive route batches from the FIB before programming the underlying platform. These interaction RPCs and the data structures such as the route updates are modeled in thrift IDL files (See <https://github.com/facebook/openr/tree/master/openr/if>)

Consequently, integrations with existing stacks and platforms can cleanly occur at the lower platform layer abstracted through the thrift interface. Newer functionalities that leverage the underlying platform's capabilities (like MPLS, BFD, SR etc.) can leverage the KVstore to store data locally and share information with other routers easily.




## IOS-XR's Journey to the We(b)st

As part of a series of blogs showcasing the

![IOS-XR Journey to the we(b)st]({{ site.baseurl }}/images/iosxr_journey_to_the_webst.png)
