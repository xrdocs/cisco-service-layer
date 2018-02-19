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


## Prelude

In December 2017, within a month of Facebook's open-source [announcement](https://code.facebook.com/posts/291641674683314/open-r-open-routing-for-modern-networks/){:target="_blank"} of Open/R, we released an integration of Open/R with IOS-XR. With the model-driven Service Layer APIs and application hosting capabilities, IOS-XR provided a pretty easy ride. The code for this integration is on Github (<https://github.com/akshshar/openr-xr>{:target="_blank"}) and is going through iterations and reviews before a pull request is sent out to the core code at <https://github.com/facebook/openr/>{:target="_blank"}.  

To see the demo of Open/R on XR in action, take a look at the following NFD17 presentation focused on IOS-XR's interplay with community tools:     

<iframe width="560" height="315" src="https://www.youtube.com/embed/HMvl9CzDIpQ?start=463" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
  
This blog will dive deeper into the demo as well as the integration points in code, so hopefully it is useful for folks starting off with open/R.

## IOS-XR's Journey to the West (Web)


As part of this blog series titled **"XR's journey to the Web"**, I intend to candidly present the journey we have undertaken with the IOS-XR software stack since 2014. In line with the ethos of xrdocs, expect this series to be highly technical and heavily focused on showcasing how IOS-XR integrates with community tools and open-source software innovations - where does it excel? where can it get better? and what needs to be done to help it get better?   
  
<a href="/cisco-service-layer/images/journey_to_the_west.png"><img src="/cisco-service-layer/images/journey_to_the_west.png" alt="Chinese folklore - Journey to the west" class="align-right" width="300" height="300"></a> 

As an allegory, I often refer back to the famous Chinese folklore: **"Journey to the west"** - a 16th Century Chinese novel by Wu Cheng-en.  

Through an extended account, it chronicles the story of Xuanzang in the 6th Century AD and his journey to India (the "west") to seek out and bring back the teachings of Buddhism to a primarily Confucian China. As Buddhism began to spread through the east, a new movement began in China that sought to marry the practical Confucian core with the new intellectual and spiritual expectations arising out of Buddhism. This movement came to be known as neo-confucianism.  


<a href="/cisco-service-layer/images/xr-journey-to-the-web.png"><img src="/cisco-service-layer/images/xr-journey-to-the-web.png" alt="XR's journey to the web" class="align-left" width="500" height="300"></a>

I liken the changes that have happened to IOS-XR over the last few years to a similar creative reinterpretation of the core concepts of traditional networking. As the "Web" players (Google, Facebook, Apple, Amazon etc.) began to showcase how better efficiencies may be achieved in network operations through **automation at every stage of deployment**, and through the ability to **"run with scissors"** - it became clear that the traditional core of network stacks would have to evolve to meet the operational demands of these highly automated networks.  

The evolution was comprised of some very interesting developments:

  *  **Linux-ization of the Stack**: IOS-XR moved from 32-bit QNX to 64-bit Linux to enable an 
     environment for scripting, [**hosting applications**](https://xrdocs.github.io/application-hosting/tutorials){:target="_blank"} and integration with tools in the DevOps space such as Ansible, Puppet, Docker, etc.  
     
  *  **Streaming Telemetry**: [**Real-time push-based Streaming Telemetry**](https://xrdocs.github.io/telemetry/tutorials/){:target="_blank"} for monitoring, alerts and 
     remediation triggers/events, outperforming SNMP in every department - scale, ease-of-use, 
     cadence etc.  
     
  *  **Model Driven APIs at every layer of the stack**: This includes [**Yang Models at the Manageability layer**](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr){:target="_blank"} and [**tools to generate bindings in various languages**](http://ydk.io){:target="_blank"}. Further,in September 2017, we took it a step further - we introduced model driven APIs over gRPC called the [**Service Layer APIs**](https://xrdocs.github.io/cisco-service-layer/){:target="_blank"} that provide programmatic access to the IOS-XR RIB, label switch database and notifications for interface and BFD events.

This blog series will focus on how a combination of the above enhancements should allow us to integrate with a wide variety of tools and software stacks in the networking community and let our users run with scissors when needed.



In this blog in particular, we shall explore how [IOS-XR's service layer APIs](https://xrdocs.github.io/cisco-service-layer/){:target="_blank"} and [application hosting capabilities](https://xrdocs.github.io/application-hosting/){:target="_blank"} can be leveraged to host and integrate Open/R as an IGP on IOS-XR. We will also touch upon further enhancements to Open/R that may be possible with Service Layer APIs serving as the platform hooks to IOS-XR. 

## What is Open/R?

In November 2017, Facebook open sourced [Open/R](https://github.com/facebook/openr){:target="_blank"}.  
As the Github description suggests, it is, and I quote, a "Distributed platform for building autonomic network functions". Pretty heavy description, so let's distill it a bit.

Much of the documentation for open/R can be found in the docs directory in the git repo:

><https://github.com/facebook/openr/tree/master/openr/docs>{:target="_blank"}

It is laid out rather well and describes all the components of the code individually - their purpose, internal interactions, et al.

At a higher level, the components look something like this:

![openr_high_level]({{ site.baseurl }}/images/openr_high_level.png)

At the outset, the architecture is reminiscent of traditional link state routing protocols like IS-IS - what with the initial Hellos used to identify neighbors (similar to IS-IS Hello Packets), establishment of adjacencies using 0MQ messages (similar to Link State PDUs (LSPs) in IS-IS) and the use of Djikstra's algorithm for SPF computations. It also borrows ideas from other protocols like BGP (the concept of originatorIDs for loop prevention, à la AS-PATH handling) and spanning tree for flood optimization of messages.

So is it really just an alternative to traditional IGPs ? Not quite. There are some design decisions taken to enable the architecture to be pluggable from the get-go:   

  *  **KV-Store**:  A Key-Value store that serves as a common database for the entire stack. Peering Messages to other routers are sent out over 0MQ channels. The internal  communication with other components such as the Prefix Manager, Decision module or the link monitor module are handled by sending and receiving thrift objects on sockets through an abstracted KvStoreClient.

  *  **Thrift based modeled APIs** between all the modules: for example, the FIB module implements a thrift client and the platform module implements a thrift server to receive route batches from the FIB before programming the underlying platform. These interaction RPCs and the data structures such as the route updates are modeled in thrift IDL files (See <https://github.com/facebook/openr/tree/master/openr/if>{:target="_blank"})

Consequently, integrations with existing stacks and platforms can cleanly occur at the lower platform layer abstracted through the modeled thrift interface. Newer functionalities that leverage the underlying platform's capabilities (like MPLS, BFD, SR etc.) can extend an existing or implement a new thrift model and leverage the KVstore to store data locally and share information with other routers easily.  

  
### Where does one start?

I believe the best place to start is of course the documentation on Github I refer to [above](https://github.com/facebook/openr/tree/master/openr/docs){:target="_blank"}. However, not enough importance can be placed on the need to read through the structure of the code to understand the important touch points in each module. The developers at Facebook graciously released a netlink platform integration for Open/R to enable the community to take a look at how things tie in internally.

This netlink platform integration enables Open/R to run as a routing stack on top of a Linux kernel as the network stack. You can check out the relevant pieces of code here:  

>The **"Platform"** module code runs a thrift Server and receives route batches from the Fib module that runs a thrift client
><https://github.com/facebook/openr/tree/master/openr/platform>{:target="_blank"}

This consists of two important abstractions:
  *  **NetlinkFibhandler**:  implements the FibService interface described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift>{:target="_blank"} to handle the incoming route batches from the Fib module
  *  **NetlinkSystemHandler**: implements the SystemService interface again described in the thrift IDL here: <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift>{:target="_blank"} to detect interfaces and IPv6 neighbors in the kernel that may be used to send hellos and peering messages to neighbors.  
  
  
>The **"Netlink(nl)"** abstraction  handles actual interaction with the kernel through netlink using the libnl library. The Netlink platform handlers described above utilize this abstraction to program and fetch routes and get a list of ipv6 neighbors or links or associated events from the kernel.
><https://github.com/facebook/openr/tree/master/openr/nl>{:target="_blank"}

Well, this is good to know. But the question still lingers -

> How can one power through the all the concepts and get Open/R running?

The Open/R Github repo gives a nod to an emulator that might become available soon (<https://github.com/facebook/openr/blob/master/openr/docs/Emulator.md>{:target="_blank"}). However, that doesn't prevent us from using standard techniques such as Vagrant to bring up an environment to play with, right away.

### Open/R on Linux:Vagrant

If you'd like to try a back-to-back setup with two linux instances on your laptop, I've published a vagrant setup with two ubuntu 16.04 instances (rtr1 and rtr2) connected through an ubuntu switch:  

><https://github.com/akshshar/openr-vagrant>{:target="_blank"}


The vagrant provisioners will install open/R on "vagrant up" on both rtr1 and rtr2 and will setup the required "run" script for openr at `/usr/sbin/run_openr.sh` on each node.

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

Further, for rtr2, I've added a simple route-scaling python script that allows you to add up to 8000 routes by manipulating the `batch_size` and `batch_num` values in `<git repo directory>/scripts/increment_ipv4_prefix.py` before running /usr/sbin/run_openr.sh 
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

Start a tcpdump capture on one or more of the bridges on the switch in the middle:  

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

  *  **Hello Messages**: The hello messages are sent to UDP port 6666 to the All-Nodes IPv6 
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

If you used the default route-scaling script on rtr2, then rtr1 should now have about 1000 routes in its fib:  


```
vagrant@rtr1:~$ breeze fib counters

== rtr1's Fib counters  ==

fibagent.num_of_routes : 1004

vagrant@rtr1:~$ 

```

Great! These outputs should give you a fair gist of how Open/R works as a link state routing protocol.
{: .notice--success}


## Integrating Open/R with IOS-XR

### Does XR meet the Requirements?

Now that we understand how Open/R operates, let's codify the requirements for it to work on a platform running Linux:  

  *  **API to get and set Routes**: On Linux, this API is Netlink, but it can be replaced with any 
     viable API that the networking stack on the platform offers.  
     
       In September 2017, we introduced Service Layer APIs - a highly performant and model driven 
       API into the network infrastructure layer (RIB, label switch database, interface and BFD 
       events) over gRPC. This API is ideal for platform integration with Open/R and as we'll see        later,achieves great performance due to its route batching capability. This 
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
  
  *  **Ability to exchange Hellos and Peering Messages**: Open/R should be able to run unmodified 
     on a platform and send its UDP hellos to port 6666 and ff02::1 and send/receive TCP peering 
     messages using link local IPv6 addresses of neighbors.
     
       Packet/IO capabilities have existed in IOS-XR since 6.0.0+, allowing applications to bind 
       to XR interfaces in the kernel, open up the TCP/UDP ports, transmit/receive TCP/UDP traffic 
       along with exception traffic such as icmp, icmpv6, IPv6 multicast etc.
       {: .notice--info}
  



### Gaps and workaround
  
  
**Pending Issue:**  

While the primary requirements were met right away, I did find an issue with dynamic IPv6 neighbor discovery in the kernel. Neighbor Solicitation messages generated in the kernel were unable to exit the box. This issue is being looked at as part of the packet/IO plumbing code that connects the linux kernel with the XR networking stack on the box.
  
  
**Workaround**:  

However, since XR itself has no issues in handling the IPv6 neighbors, we utilize two capabilities in XR as a workaround:

  1. Enable **"ipv6 nd ra-unicast"** under the XR interfaces using XR CLI or Yang model. This 
     enables neighbors to be reachable without explicit traffic being originated in XR.

  2. Set up a client (currently separate from the Open/R binary, running as a parallel process) 
     that receives **IPv6 neighbor table from XR periodically using Streaming Telemetry over gRPC 
     and programs the kernel using Netlink**. Open/R then works with the programmed neighbors in 
     the kernel without any issues. 
     By reacting to Streaming Telemetry data for IPv6 neighbors, the entries in the kernel are 
     kept dynamic and in sync with network events like link flap or disabling IPv6 on interface.

**Resolved Issue: Patch Ready**:  

As part of the initial design of the packet/IO architecture, the presence of a default route through an interface called "fwdintf" and consequently, an fe80::/64 route through fwdintf alone, solved most use cases where an application needed to send traffic through data ports over the internal fabric.   
    However, as soon as we realized that Open/R utilizes TCP messages with link local IPv6 addresses of neighbors as the destination to establish adjacencies, it became obvious that we needed to allow fe80::/64 routes through all the exposed interfaces (Gig, TenGig, HundredGig etc.) in the kernel. This was a simple fix and an internal bug was filed,resolved and the patch is utilized in the integration that you see below. This patch will be released for public consumption as a SMU on top of IOS-XR release 6.2.25 and will be integrated into 6.3.2.
  
  
### Current Solution and implementation details

The current solution is shown below:

![Open/R integration with IOS-XR- current design]({{site.baseurl}}/images/openr_xr_integration_current.png)


The code for this implementation can be found here:  
><https://github.com/akshshar/openr-xr>{:target="_blank"}  

The touch points are described below:

  1.  [**CMakelists.txt**](https://github.com/akshshar/openr-xr/blob/openr20171212/openr/CMakeLists.txt){:target="_blank"} was extended to include grpc,
      protobuf and iosxrsl (IOS-XR Service Layer APIs compiled into a library) as target link
      libraries.
  
  2.  [**Platform module**](https://github.com/akshshar/openr-xr/tree/openr20171212/openr/platform){:target="_blank"} was extended to include 
      [**IosxrslFibHandler**](https://github.com/akshshar/openr-xr/blob/openr20171212/openr/platform/IosxrslFibHandler.cpp){:target="_blank"} 
      that implements the FibService interface described in the thrift IDL here: 
      <https://github.com/facebook/openr/blob/master/openr/if/Platform.thrift> to handle
      incoming route batches from the Fib module.
  
      It may be noted that the [**NetlinkSystemHandler**](https://github.com/akshshar/openr-xr/blob/openr20171212/openr/platform/NetlinkSystemHandler.cpp){:target="_blank"} code remains untouched and continues to register and react to link and neighbor information from the kernel in IOS-XR for now. In the future, as the above figure indicates, I will experiment by replacing the netlink hooks for link information with the IOS-XR Service Layer RPC for Interface events and by replacing the netlink hooks for IPv6 neighbor information with IOS-XR Telemetry stream for IPv6 neighbors. The goal will be to remove dependencies on libnl and determine if any efficiencies are gained as a result. For now, there is no immediate need to replace the NetlinkSystemHandler functionality.
      {: .notice--info}
  
  3. [**IOS-XR Service Layer (iosxrsl) abstraction**](https://github.com/akshshar/openr-xr/tree/openr20171212/openr/iosxrsl){:target="_blank"}: The iosxrsl directory in the git repo implements all the necessary initialization techniques to connect to IOS-XR Service-layer over gRPC, handles async thread for the init channel used by service layer and creates the necessary abstractions for Route batch handling for IOS-XR RIB (Routing information Base), making it easy to implement the IosxrslFibHandler.cpp code explained above. Eventually, the IOS-XR Service Layer Interface API abstraction will also go here.
  
  4. [**Main.cpp**](https://github.com/akshshar/openr-xr/blob/openr20171212/openr/Main.cpp){:target="_blank"}: Extended to accept new parameters for IOS-XR Service Layer IP address and port (reachable IP address in IOS-XR and configured gRPC port for service-layer). Further, it starts a Fibthrift thread that intializes the gRPC connection to IOS-XR and registers against a set of VRFs (**New**) for IPv4 and IPv6 operations.
  
  5. [**Docker Build**](https://github.com/akshshar/openr-xr/tree/openr20171212/docker){:target="_blank"}: As shown in the figure above, Open/R is spun up on IOS-XR as an application running inside a docker container. The [**Dockerfile**](https://github.com/akshshar/openr-xr/blob/openr20171212/docker/Dockerfile){:target="_blank"} is used to build Open/R with all its dependencies, along with the grpc, protobuf, and IOS-XR Service-Layer library inside an ubuntu 16.04 rootfs. The Dockerfile used is also shown below:
  
  <div class="highlighter-rouge">
  <pre class="highlight">
  <code style="white-space: pre;">
  FROM ubuntu:16.04 


  RUN apt-get update && apt-get install -y autoconf automake libtool curl make g++ unzip git  python-pip python-dev && git clone https://github.com/google/protobuf.git ~/protobuf && \
            cd ~/protobuf && \
            git checkout 2761122b810fe8861004ae785cc3ab39f384d342 && \
            ./autogen.sh && \
            ./configure && \
            make && \
            make install &&\
            ldconfig && make clean && cd ~/ && rm -r ~/protobuf 


  RUN git clone https://github.com/grpc/grpc.git ~/grpc && cd ~/grpc && \
            git checkout 80893242c1ee929d19e6bec5dc19a1515cd8dd81 && \
            git submodule update --init && \
            make && \
            make install && make clean && cd ~/ && rm -r ~/grpc

  RUN apt-get install -y pkg-config && git clone https://wwwin-github.cisco.com/akshshar/service-layer-objmodel ~/service-layer-objmodel && \
           cd ~/service-layer-objmodel/grpc/cpp && \
           ./build_libiosxrsl.sh &&  \
           cd ~/ && rm -r ~/service-layer-objmodel

  RUN git clone https://github.com/akshshar/openr.git /root/openr && cd /root/openr/ && git   checkout openr20171212 && cd /root/openr/build && ./build_openr_dependencies.sh

  RUN cd /root/openr/build && ./build_openr.sh && ./remake_glog.sh && cd /root/ && rm -r /root/openr

  COPY run_openr.sh /usr/sbin/run_openr.sh

  CMD /usr/sbin/run_openr.sh >/var/log/openr.log 2>&1
  
    
  </code>
  </pre>
  </div>

  **Tip:** When issuing the docker build command with this Dockerfile, make sure to use the --squash flag: `docker build --squash -it openr .`
   This is required to prevent the size of the docker image from spiraling out of control.
   {: .notice--warning}
    
    
  
### Deploying Open/R Docker image on NCS5500

IOS-XR utilizes a consistent approach towards the application hosting infrastructure across all XR platforms. This implies that all hardware platforms: 1RU, 2RU, Modular or even Virtual platforms would follow the same deployment technique described below:

In the demo, I will utilize two NCS5501s connected to each other over a HundredGig interface.
The basic configuration on the router is shown below:


**XR Configuration:**

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">

interface HundredGigE0/0/1/0
 ipv4 address 10.1.1.10 255.255.255.0
 <mark>ipv6 nd unicast-ra</mark>
 ipv6 enable
!
!
!
grpc
 <mark>port 57777</mark>
 service-layer
!
!
telemetry model-driven
 sensor-group IPV6Neighbor
  sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
 !
 subscription IPV6
  sensor-group-id IPV6Neighbor sample-interval 15000
 !
!
end


</code>
</pre>
</div>

As explained in an [earlier section](https://xrdocs.github.io/cisco-service-layer/blogs/2018-02-16-xr-s-journey-to-the-we-b-st-open-r-integration-with-ios-xr/#gaps-and-workaround), `ipv6 nd unicast-ra` is required to keep neighbors alive in XR while Open/R initiates traffic in the linux kernel. The `grpc` configuration starts the gRPC server on XR and can be used to subscribe to Telemetry data (subscription IPv6 as shown in the configuration above) and service-layer configuration allows Service-Layer clients to connect over the same gRPC port.



Once the Docker image is ready, set up a private docker registry that is reachable from the NCS5500 router in question and push the docker image to that registry. Setting up a private docker registry and pulling a docker image onto NCS5500 is explained in detail in the "Docker on XR" tutorial here:  <https://xrdocs.github.io/application-hosting/tutorials/2017-02-26-running-docker-containers-on-ios-xr-6-1-2/#private-insecure-registry>{:target="_blank"} 

Once the docker image is pulled successfully, you should see:

```
RP/0/RP0/CPU0:rtr1#bash
Fri Feb 16 22:46:52.944 UTC
[rtr1:~]$ 
[rtr1:~]$ [rtr1:~]$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
11.11.11.2:5000/openr   latest              fdddb43d9600        33 seconds ago        1.829 GB
[rtr1:~]$ 
[rtr1:~]$ 
```

Now, simply spin up the docker image using the parameters shown below:

```

RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#bash
Fri Feb 16 22:46:52.944 UTC
[rtr1:~]$ 
[rtr1:~]$ docker run -itd  --name openr --cap-add=SYS_ADMIN --cap-add=NET_ADMIN  -v /var/run/netns:/var/run/netns -v /misc/app_host:/root -v /misc/app_host/hosts_rtr1:/etc/hosts --hostname rtr1 11.11.11.2:5000/openr bash
684ad446ccef5b0f3d04bfa4705cab2117fc60f266cf0536476eb9506eb3050a
[rtr1:~]$ 
[rtr1:~]$ 
[rtr1:~]$ docker ps
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS               NAMES
b71b65238fe2        11.11.11.2:5000/openr   "bash -l"           24 secondss ago        Up 24 seconds                             openr
```

Instead of `bash` as the entrypoint command for the docker instance, one can directly start openr using `/root/run_openr_rtr1.sh > /root/openr_logs 2>&1`. I'm using `bash` here for demonstration purposes.
{: .notice--info}

Note the capabilities: `--cap-add=SYS_ADMIN` and `--cap-add=NET_ADMIN`. Both of these are necessary to ensure changing into a mounted network namespace(vrf) is possible inside the container.
{: .notice--info}


Once the docker instance is up on rtr1, we do the same thing on rtr2:

```
RP/0/RP0/CPU0:rtr2#bash
Fri Feb 16 23:12:53.828 UTC
[rtr2:~]$ docker ps
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS               NAMES
684ad446ccef        11.11.11.2:5000/openr   "bash -l"           8 minutes ago       Up 8 minutes                            openr
[rtr2:~]$ 
```

On rtr2, the file `/root/run_openr_rtr2.sh` is slightly different. It leverages `increment_ipv4_prefix.py` as a route-scaling script to increase the number of routes advertized by rtr2 to rtr1. Here I'll push a 1000 routes from rtr2 to rtr1 to test the amount of time Open/R on rtr1 takes to program XR RIB.


### Testing FIB Programming Rate

On rtr2, exec into the docker instance and start Open/R:

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
RP/0/RP0/CPU0:rtr2#bash
Fri Feb 16 23:12:53.828 UTC
[rtr2:~]$ 
[rtr2:~]$ docker exec -it openr bash
root@rtr2:/# /root/run_openr_rtr2.sh
/root/run_openr_rtr2.sh: line 106: /etc/sysconfig/openr: No such file or directory
Configuration not found at /etc/sysconfig/openr. Using default configuration
openr[13]: Starting OpenR daemon.
openr

....

</code>
</pre>
</div>

Now, hop over to rtr1 and do the same:


<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">

RP/0/RP0/CPU0:rtr1#bash
Fri Feb 16 23:12:53.828 UTC
[rtr1:~]$ 
[rtr1:~]$ docker exec -it openr bash
root@rtr2:/# /root/run_openr_rtr1.sh
/root/run_openr_rtr1.sh: line 106: /etc/sysconfig/openr: No such file or directory
Configuration not found at /etc/sysconfig/openr. Using default configuration
openr[13]: Starting OpenR daemon.
openr

I0216 23:50:28.058964   134 Fib.cpp:144] Fib: publication received ...
I0216 23:50:28.063819   134 Fib.cpp:218] <mark>Processing route database ... 1002 entries</mark>
I0216 23:50:28.065533   134 Fib.cpp:371] Syncing latest routeDb with fib-agent ... 
I0216 23:50:28.081434   126 IosxrslFibHandler.cpp:185] Syncing FIB with provided routes. Client: OPENR
I0216 23:50:28.105329    95 ServiceLayerRoute.cpp:197] ###########################
I0216 23:50:28.105350    95 ServiceLayerRoute.cpp:198] Transmitted message: IOSXR-SL Routev4 Oper: SL_OBJOP_UPDATE
VrfName: "default"
Routes {
  Prefix: 1006698753
  PrefixLen: 32
  RouteCommon {
    AdminDistance: 99
  }
  PathList {
    NexthopAddress {
      V4Address: 167837972
    }
    NexthopInterface {
      Name: "HundredGigE0/0/1/0"
    }
  }
}
Routes {



.....




Routes {
  Prefix: 1677976064
  PrefixLen: 24
  RouteCommon {
    AdminDistance: 99
  }
  PathList {
    NexthopAddress {
      V4Address: 167837972
    }
    NexthopInterface {
      Name: "HundredGigE0/0
I0216 23:49:00.923399    95 ServiceLayerRoute.cpp:199] ###########################
I0216 23:49:00.943408    95 ServiceLayerRoute.cpp:211] RPC call was successful, checking response...
I0216 23:49:00.943434    95 ServiceLayerRoute.cpp:217] IPv4 Route Operation:2 Successful
I0216 23:49:00.944533    95 ServiceLayerRoute.cpp:777] ###########################
I0216 23:49:00.944545    95 ServiceLayerRoute.cpp:778] Transmitted message: IOSXR-SL RouteV6 Oper: SL_OBJOP_UPDATE
VrfName: "default"
I0216 23:49:00.944550    95 ServiceLayerRoute.cpp:779] ###########################
I0216 23:49:00.945046    95 ServiceLayerRoute.cpp:793] RPC call was successful, checking response...
I0216 23:49:00.945063    95 ServiceLayerRoute.cpp:799] IPv6 Route Operation:2 Successful
I0216 23:27:01.021437    52 Fib.cpp:534] OpenR convergence performance. Duration=3816
I0216 23:27:01.021456    52 Fib.cpp:537]   node: rtr1, event: ADJ_DB_UPDATED, duration: 0ms, unix-timestamp: 1518823617205
I0216 23:27:01.021464    52 Fib.cpp:537]   node: rtr1, event: DECISION_RECEIVED, duration: 1ms, unix-timestamp: 1518823617206
I0216 23:27:01.021471    52 Fib.cpp:537]   node: rtr1, event: DECISION_DEBOUNCE, duration: 9ms, unix-timestamp: 1518823617215
I0216 23:27:01.021476    52 Fib.cpp:537]   node: rtr1, event: DECISION_SPF, duration: 22ms, unix-timestamp: 1518823617237
I0216 23:27:01.021479    52 Fib.cpp:537]   node: rtr1, event: FIB_ROUTE_DB_RECVD, duration: 12ms, unix-timestamp: 1518823617249
I0216 23:27:01.021484    52 Fib.cpp:537]   node: rtr1, event: FIB_DEBOUNCE, duration: 3709ms, unix-timestamp: 1518823620958
I0216 23:27:01.021488    52 Fib.cpp:537]   node: rtr1, event: <mark>OPENR_FIB_ROUTES_PROGRAMMED, duration: 63ms, unix-timestamp: 1518823621021</mark>


</code>
</pre>
</div>

As seen in the highlighted outputs, the total time taken to program 1002 route entries was about 63ms, giving us a route programming rate of about 16000 routes/second!
{: .notice--info}

Having said that, the FIB_DEBOUNCE rate is something to be looked at as I work through the code a bit more.

It goes without saying that the adjacencies were correctly established, but for the sake of public record, here are breeze outputs on rtr1 and rtr2:

**rtr1 breeze adj**:

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">

RP/0/RP0/CPU0:rtr1#bash
Sat Feb 17 00:18:07.974 UTC
[rtr1:~]$ 
[rtr1:~]$ docker exec -it openr bash
root@rtr1:/# 
root@rtr1:/# ip netns exec global-vrf bash
root@rtr1:/# 
root@rtr1:/# 
root@rtr1:/# breeze kvstore adj

&gt; rtr1's adjacencies, version: 2, Node Label: 36247, Overloaded?: False
Neighbor    Local Interface    Remote Interface      Metric    Weight    Adj Label  NextHop-v4    NextHop-v6                Uptime
<mark>rtr2        Hg0_0_1_0          Hg0_0_1_0                 10         1        50066  10.1.1.20     fe80::28a:96ff:fec0:bcc0  1m36s</mark>


root@rtr1:/# 

</code>
</pre>
</div>




**rtr2 breeze adj**:

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">

RP/0/RP0/CPU0:rtr2#
RP/0/RP0/CPU0:rtr2#bash
Sat Feb 17 00:24:11.960 UTC
[rtr2:~]$ 
[rtr2:~]$ docker exec -it openr bash
root@rtr2:/# 
root@rtr2:/# ip netns exec global-vrf bash
root@rtr2:/# breeze kvstore adj

&gt; rtr2's adjacencies, version: 3, Node Label: 1, Overloaded?: False
Neighbor    Local Interface    Remote Interface      Metric    Weight    Adj Label  NextHop-v4    NextHop-v6                Uptime
<mark>rtr1        Hg0_0_1_0          Hg0_0_1_0                  9         1        50066  10.1.1.10     fe80::28a:96ff:fed3:18c0  1m38s</mark>


</code>
</pre>
</div>

### Check IOS-XR RIB State

Well, this is the moment of truth. Open/R  instances were able to run inside docker containers on two back-to-back NCS5501 devices, connected over a HundredGig port. Now, let's see what happened to rtr1's RIB since we injected 1000 routes into this open/R instance:

```

RP/0/RP0/CPU0:rtr1#show route
Sat Feb 17 00:28:36.838 UTC

Codes: C - connected, S - static, R - RIP, B - BGP, (>) - Diversion path
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - ISIS, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, su - IS-IS summary null, * - candidate default
       U - per-user static route, o - ODR, L - local, G  - DAGR, l - LISP
       A - access/subscriber, a - Application route
       M - mobile route, r - RPL, (!) - FRR Backup path

Gateway of last resort is 10.1.1.20 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 10.1.1.20, 1d02h
               [1/0] via 11.11.11.2, 1d02h
C    10.1.1.0/24 is directly connected, 3w0d, HundredGigE0/0/1/0
L    10.1.1.10/32 is directly connected, 3w0d, HundredGigE0/0/1/0
L    10.10.10.10/32 is directly connected, 3w0d, Loopback1
C    11.11.11.0/24 is directly connected, 1d02h, MgmtEth0/RP0/CPU0/0
L    11.11.11.23/32 is directly connected, 1d02h, MgmtEth0/RP0/CPU0/0
L    50.1.1.1/32 is directly connected, 3w0d, Loopback0
a    60.1.1.1/32 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.1.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.2.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.3.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.4.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.5.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.6.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.7.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.8.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.9.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.10.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.11.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.12.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.13.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0
a    100.1.14.0/24 [99/0] via 10.1.1.20, 00:03:14, HundredGigE0/0/1/0

....


RP/0/RP0/CPU0:rtr1#show route summary
Sat Feb 17 00:29:47.961 UTC
Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            4          0          0           960          
connected                        2          2          0           960          
dagr                             0          0          0           0            
static                           1          0          0           352          
bgp 65000                        0          0          0           0            
application Service-layer        1002       0          0           240480       
Total                            1009       2          0           242752       

RP/0/RP0/CPU0:rtr1#

```

There you go! There are 1002 service layer routes in the RIB, all thanks to Open/R acting as an IGP, learning routes from its neighbor and programming the IOS-XR RIB on the local box over gRPC.
{: .notice--success}



## What else can we do?

Integration of Open/R with IOS-XR Service Layer APIs opens up a whole host of possibilities.
This particular integration was centered around Route handling and hence the RIB API was sufficient.

The Service Layer API however offers access to the Label Switch Database on IOS-XR so that a client may program label blocks and label-prefix mappings based on some application logic.
This may be leveraged to provide label handling (MPLS) capabilities to Open/R.

Further, Open/R currently hooks into interface events and IPv6 neighbor events. With the Service Layer API it is also possible to get a stream of BFD notifications, allowing faster response to link-down event of the neighbor, if the neighbor is connected through a LAN/switch.

As Service Layer APIs evolve further and get SR-TE (label stack) capabilities and L2 capabilities, there is a stream of new possibilities for Open/R in the near future.
