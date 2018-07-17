---
published: true
date: '2017-09-25 12:48 -0700'
title: Using Service Layer APIs with Vagrant IOS-XR
author: Akshat Sharma
tags:
  - vagrant
  - iosxr
  - cisco
  - grpc
  - service-layer
---
{% include toc icon="table" title="IOS-XR Service Layer APIs" %}
{% include base_path %}
{% include github_org %}

## Introduction
  
  
If you've haven't played around with the vagrant IOS-XR box yet, now might be a good time to take a look at the following tutorials and get your environment set up:

* [Generate API Key]({{ site.url }}/getting-started/steps-download-iosxr-vagrant): Generate an API Key using your CCO (cisco.com) ID to download the Vagrant box for IOS-XR with SL-API support.

* [Vagrant IOS-XR Quick Start]({{ site.url }}/application-hosting/tutorials/iosxr-vagrant-quickstart): Use the downloaded box and learn how to boot it on your laptop and play with a couple of sample topologies. 

* [Bootstrap XR Configuration with Vagrant]({{ site.url }}/application-hosting/tutorials/iosxr-vagrant-bootstrap-config) **(optional)**: Learn how a simple shell provisioner can be used to apply a configuration on boot with a Vagrant IOS-XR box.

Once you have everything set up, you should be able to see the IOS-XRv vagrant box in the `vagrant box list` command:  


```shell
  AKSHSHAR-M-K0DS:~ akshshar$ vagrant box list
  IOS-XRv (virtualbox, 0)
  AKSHSHAR-M-K0DS:~ akshshar$ 
```

This tutorial is meant to get you up and running with an environment to play with the Service Layer APIs on IOS-XR. An introduction to these APIs can be found here:  

><{{ base_path }}/tutorials/service-layer-intro/>
{: .notice--info}

To get more details on the APIs, check out the API-Docs section:  

><{{ base_path }}/apidocs/>
{: .notice--info}

In this tutorial we set up a GRPC/python environment that may be used to build your own python client to interact with the SL APIs. 

A go client binary is also included as part of the "service-layer-objmodel" code itself at the following link: 
(You will also find a quick-start go client code located there) 

><https://github.com/{{ github_org }}/service-layer-objmodel/tree/master/grpc/go/src/tutorial>
{: .notice--info}


## Clone the development topology  

We'll use a simple Vagrant topology as shown below:  

![grpc-sl-apis-topo](https://xrdocs.github.io/xrdocs-images/assets/images/grpc-sl-apis-topo.png)


Clone the vagrant-examples repo to get started:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>

AKSHSHAR-M-K0DS:~ akshshar$<mark> git clone https://github.com/{{ github_org }}/vagrant-examples.git</mark>
Cloning into 'vagrant-examples'...
remote: Counting objects: 78, done.
remote: Compressing objects: 100% (42/42), done.
remote: Total 78 (delta 13), reused 78 (delta 13), pack-reused 0
Unpacking objects: 100% (78/78), done.
Checking connectivity... done.
AKSHSHAR-M-K0DS:~ akshshar$ 
AKSHSHAR-M-K0DS:~ akshshar$<mark>cd vagrant-examples/iosxr-grpc-setup/ </mark>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ 
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ ls
<mark>Vagrantfile	configs		scripts </mark>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$  
</code>
</pre>
</div>


### Understand the Vagrantfile

The Vagrantfile located under `vagrant-examples/iosxr-grpc-setup/` is shown in its entirety below:

{% capture "info-txt" %}

1.  **Notice the port forwarding enabled for the rtr node (IOS-XR) which corresponds to the grpc
server port (57344) configured as part of the bootstrap script. This makes the port accessible 
   on the management port (nat network for Vagrant).**  
   
2.  **Further, eth1 of the devbox is connected to GigabitEthernet0/0/0/0 of the IOS-XR vagrant node,making the grpc server available on this path as well.**  

{% endcapture %}

<div class="notice--info">
{{ info-txt | markdownify }}
</div>  

```ruby
Vagrant.configure(2) do |config|
  

   config.vm.define "rtr" do |node|
      node.vm.box =  "thinxr-aug3"

      node.vm.network "forwarded_port", guest: 57344, host: 57344 
      # gig0/0/0 connected to "link1"
      # auto_config is not supported for XR, set to false

      node.vm.network :private_network, virtualbox__intnet: "link1", auto_config: false


      #Source a config file and apply it to XR

      node.vm.provision "file", source: "configs/rtr_config", destination: "/home/vagrant/rtr_config"

      node.vm.provision "shell" do |s|
          s.path =  "scripts/apply_config.sh"
          s.args = ["/home/vagrant/rtr_config"]
      end

    end

 
    config.vm.define "devbox" do |node|
      node.vm.box =  "ciscoxr/grpc-ubuntu-16.04"

      # eth1 connected to link1
      # auto_config is supported for an ubuntu instance

      node.vm.network :private_network, virtualbox__intnet: "link1", ip: "11.1.1.20"

    end

end

```  


The following configuration is applied to the IOS-XR instance on boot:  
(located @  `vagrant-examples/iosxr-grpc-setup/configs/rtr_config`)  

```
!! XR configuration
!
interface GigabitEthernet0/0/0/0
ip address 11.1.1.10/24
no shutdown
!
grpc 
  port 57344
  address-family ipv4
  service-layer
!
!
end

```

**This configuration is in addition to a dhcp client configured on the MgmtEth0/RP0/CPU0/0 port and a user with credentials: `username/password = vagrant/vagrant`. These are set up in the vagrant box by default.**
{: .notice--info}  




## GRPC/Python environment (devbox) 

### Use our Vagrant box
You'll need a few dependencies installed to construct your own grpc python client. To make things easier we've already created an ubuntu-14.04 vagrant box with everything installed.
If you notice in the above Vagrantfile, we specify a name for a special Vagrant box:  

```ruby

 config.vm.define "devbox" do |node|
      node.vm.box =  "ciscoxr/grpc-ubuntu-16.04"

      # eth1 connected to link1
```

[ciscoxr/grpc-ubuntu-16.04](https://app.vagrantup.com/ciscoxr/boxes/grpc-ubuntu-16.04) is up on Atlas for your convenience and you can include it in your Vagrantfiles as shown above.  


To bring up the devbox, simply issue a "vagrant up devbox" inside the cloned directory:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ pwd
/Users/akshshar/vagrant-examples/iosxr-grpc-setup
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ 
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$<mark> vagrant up devbox </mark>
Bringing machine 'devbox' up with 'virtualbox' provider...
==> devbox: Importing base box 'ciscoxr/grpc-ubuntu-16.04'...

----------------------- snip output --------------------------


</code>
</pre>
</div>

Hop into the devbox once up:  

```
vagrant ssh devbox
```

All dependencies should be installed already:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
vagrant@vagrant-ubuntu-trusty-64:~$<mark> protoc --version</mark>
<mark>libprotoc 3.0.0</mark>
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$<mark> pip show grpcio </mark>
---
<mark>Name: grpcio
Version: 0.13.1 </mark>
Location: /usr/local/lib/python2.7/dist-packages
Requires: six, enum34, futures, protobuf
vagrant@vagrant-ubuntu-trusty-64:~$ 
</code>
</pre>
</div>


### Build your own? 

If you'd much rather build your own devbox environment, then let's use a pristine Ubuntu 14.04
box. Make the following change in the Vagrantfile:  

```ruby

 config.vm.define "devbox" do |node|
      node.vm.box =  "ubuntu/trusty64"

      # eth1 connected to link1
```

You will need the following steps:  

*  Bring up the devbox first:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ pwd
/Users/akshshar/vagrant-examples/iosxr-grpc-setup
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ 
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$<mark> vagrant up devbox </mark>
Bringing machine 'devbox' up with 'virtualbox' provider...
==> devbox: Importing base box 'ciscoxr/grpc-ubuntu-16.04'...

----------------------- snip output --------------------------


</code>
</pre>
</div>

* Hop into the devbox

```
vagrant ssh devbox
```

*  Install basic dependencies:  

```
sudo apt-get update
sudo apt-get -y install autoconf automake libtool curl make g++ unzip git python-pip python-dev 
```  

*  Clone and build google protobuf:

```
git clone https://github.com/google/protobuf.git ~/protobuf
cd ~/protobuf/
./autogen.sh
./configure
make
sudo make install
sudo ldconfig
```

*  Clone and build grpc

```
git clone https://github.com/grpc/grpc.git ~/grpc
cd ~/grpc/
git submodule update --init
make
sudo make install
```

* Install the python grpc package and other dependencies

```
sudo pip install six grpcio=='0.13.1' py2-ipaddress=='3.4'

```


**That's it! You're now ready to launch the router and test things out.**
{: .notice--success}  
 
 
## Clone the Object Model Code into Devbox  

By now, you should have your devbox up and running with the dependencies installed.  
Clone the Object Model code:  

**The following command must be run inside the devbox**
{: .notice--info}

```
git clone https://github.com/{{ github_org }}/service-layer-objmodel.git ~/service-layer-objmodel
```  

## Bring up the Router  

**On your laptop**, bring up the router connected to the devbox:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ pwd 
<mark>/Users/akshshar/vagrant-examples/iosxr-grpc-setup </mark>
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ ls
Vagrantfile	configs		scripts
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$<mark> vagrant up rtr </mark>
Bringing machine 'rtr' up with 'virtualbox' provider...
==> rtr: Importing base box 'IOS-XRv'...

----------------------- snip output --------------------------

</code>
</pre>
</div>


## Check connectivity from the devbox

Once the router is up, you should be able to issue `vagrant status` to see both nodes running on your laptop:  

```
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ pwd
/Users/akshshar/vagrant-examples/iosxr-grpc-setup
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ 
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ vagrant status
Current machine states:

rtr                       running (virtualbox)
devbox                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
AKSHSHAR-M-K0DS:iosxr-grpc-setup akshshar$ 

```  

Jump into the devbox to see that:  

```
vagrant ssh devbox
```

*  You're able to ping the directly connected interface of the router (Gig0/0/0/0 at 11.1.1.10):  
  
   ```
   vagrant@vagrant-ubuntu-trusty-64:~$ ping 11.1.1.10
   PING 11.1.1.10 (11.1.1.10) 56(84) bytes of data.
   64 bytes from 11.1.1.10: icmp_seq=1 ttl=255 time=1.48 ms
   64 bytes from 11.1.1.10: icmp_seq=2 ttl=255 time=2.12 ms
   64 bytes from 11.1.1.10: icmp_seq=3 ttl=255 time=2.49 ms

   ```

*  You're able to connect to port 57344 (opened up by the grpc server) on the router:

   * **Via Gig0/0/0/0**

   ```
   vagrant@vagrant-ubuntu-trusty-64:~$ telnet 11.1.1.10 57344
   Trying 11.1.1.10...
   Connected to 11.1.1.10.
   Escape character is '^]'.

   
   ```

   * **Via MgmtEth0/RP0/CPU0/0**  
   
   ```
   
   vagrant@vagrant-ubuntu-trusty-64:~$ telnet 10.0.2.2 57344
   Trying 10.0.2.2...
   Connected to 10.0.2.2.
   Escape character is '^]'.

   
   ```
   
 **Perfect! You're now all set to try out the python grpc client code.**
 {: .notice--success}  

 
 
## Run the python unit-tests
 
Still inside the devbox, set the SERVER_IP and SERVER_PORT variables:  
 
{% capture "info-txt" %}
You have two options:  
 
*  If you wish to use the Management Network, then `SERVER_IP=10.0.2.2`
*  If you wish to use the Gig0/0/0/0 Network, then `SERVER_IP=11.1.1.10`
 
{% endcapture %}
 
<div class="info-txt">
{{ info-txt | markdownify }}
</div>
 
For example, if we use the Management Network:  
 
```
vagrant@vagrant-ubuntu-trusty-64:~$ 
vagrant@vagrant-ubuntu-trusty-64:~$ export SERVER_IP=10.0.2.2
vagrant@vagrant-ubuntu-trusty-64:~$ export SERVER_PORT=57344
vagrant@vagrant-ubuntu-trusty-64:~$ 
```  

Now, run the python unit-tests to verify that everything is fine:  
 
```
cd ~/service-layer-objmodel/grpc/python/src/
python -m unittest -v tests.test_lindt  

```


You will see a slew of messages pass by on the screen, as the unit-tests program routes into the router using the Service Layer APIs:  


<div class="highlighter-rouge">
<pre class="highlight">
<code>
 
 D0808 12:23:28.989076624    1983 ev_posix.c:101]             Using polling engine: poll
test_000_global_init (tests.test_lindt.TestSuite_000_Global) ... Waiting to hear from Global event...
Server Version 0.0.0
Global Event Notification Received! Waiting for events...
ok
test_001_get_globals (tests.test_lindt.TestSuite_000_Global) ... Max VRF Name Len     : 33
Max Iface Name Len   : 64
Max Paths per Entry  : 128
Max Prim per Entry   : 64
Max Bckup per Entry  : 64
Max Labels per Entry : 3
Min Prim Path-id     : 1
Max Prim Path-id     : 64
Min Bckup Path-id    : 65
Max Bckup Path-id    : 128
Max Remote Bckup Addr: 2
ok
test_000_get_globals (tests.test_lindt.TestSuite_001_Route_IPv4) ... Max v4 VRF Reg Per VRF Msg : 512
Max v4 Routes per Route Msg: 1024
ok
test_001_vrf_registration_add (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_002_route_add (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_00_route_update (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_01_route_update_nhlfe_connected (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_02_route_update_nhlfe_ecmp (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_03_route_update_nhlfe_non_connected (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_04_route_update_route_connected (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok
test_003_05_route_update_route_ecmp (tests.test_lindt.TestSuite_001_Route_IPv4) ... ok

------------------------- snip output ----------------------------


----------------------------------------------------------------------
Ran 139 tests in 22.700s

OK

</code>
</pre>
</div>


## Writing your own Python-GRPC client

You now have a development environment up and running on your laptop.  
  
To understand how to build your own python-grpc client to interact with Service Layer APIs, head over to API-Docs section where you can find more in-depth information on the available Services, Messsages and Error codes:  

> <{{ base_path }}/apidocs/index.html>
