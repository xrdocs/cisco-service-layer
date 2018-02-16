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


## What is Open/R?

In November 2017, Facebook finally open sourced [Open/R](https://github.com/facebook/openr).  
As the Github description suggests, it is, and I quote, a "Distributed platform for building autonomic network functions". Pretty heavy description, so let's distill it a bit.

Much of the documentation for open/R can be found in the docs directory in the git repo:

><https://github.com/facebook/openr/tree/master/openr/docs>

It is laid out rather well and describes all the components of the code individually - their purpose, their internal interaction, et al.

At a higher level, the components look something like this:

![openr_high_level]({{site.baseurl}}/images/openr_high_level.png)





## IOS-XR's Journey to the We(b)st

As part of a series of blogs showcasing the 

![IOS-XR Journey to the we(b)st]({{site.baseurl}}/images/iosxr_journey_to_the_webst.png)

