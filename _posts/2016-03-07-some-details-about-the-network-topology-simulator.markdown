---
layout: post
title: Some details about the network topology simulator
date: '2016-03-07 14:06:44'
tags:
- linux
- gre
- sdn
- openvswitch
- docker
- imunes
- cloud-computing
- networking
- python
- flask
- vxlan
- iproute
- coreos
- iso-osi
- network-simulator
- networking-laboratory
- unimi
- crema
---

In my last post I told you I want to develop a network topology simulator, but I haven't given you any technical details about it, except for the fact I want to use **Docker** and **OpenvSwitch**, just like *IMUNES* already does.
The whole project was only on my mind, so I decided to take some notes about the components I have to build and about how I want to implement things.
Now, I'm sharing those notes with you.

To simulate a network topology I would have to reproduce the behavior of ISO/OSI's level 2 and level 3.
To implement level 2, I'd use *OpenVSwitch*, that is a *SDN-based* layer 2/3 switch. Links would be realized with Linux kernel's iproute, and associated with *ports* on *OVS*.
I also want to let the user specify some informations about the links, such as *bandwidth*, *delay*, *MTU*, etc.
Level 3 and above features are going to be implemented with *Docker*: containers can represent hosts running services and network devices (such as firewalls and routers).
To do this, I think I'd need to run Docker containers with all capabilities; this could represent a security issue, but I will address this later.
Since Docker networking features aren't matching my needs, I'd disable them and move iproute generated interfaces directly to namespaces of containers.

![](/content/images/2016/03/Untitled.png).
The architecture I was thinking consists of a controller node and many compute nodes, just like any already existing cloud infrastructure manager (e.g. OpenStack).
Images for containers are going to be stored on an external Docker image registry (that could rely on external object storage or file system, but I don't care about that).
There could be also another optional image registry to save persistent snapshots of containers.

####Controller
On the controller node we've got:

- ***the graphical user interface*** to which the users connect to.
Here they can build their own network graph, configure hosts, switches and links between each other, and deploy everything to the underlying architecture.
- ***REST APIs***, on which the GUI leans
- ***A database***, to track the ownership of the resources created by users (once again, containers, *OVS* bridges, interfaces), and to manage VLAN ids across hosts.
The VLAN thing is not that simple. Users need to specify their own VLAN tags, so I could have problems when two users choose to use the same VLAN tag. Theoretically speaking, there would be no problem, because bridges created by different users cannot be linked together, but it's too soon to say that.
- ***An identity service*** that implements identity management, provides authentication and integration with external authentication providers, resource isolation and security. I could use the awesome *Keystone*, from OpenStack :)
- ***A network service***, built on top of OpenvSwitch, that manages VXLAN tunneling between nodes
- ***A scheduler***, to choose the most proper host to deploy containers in, when the user "runs" its own topology. This is maybe the most complex part because I have to set a policy to choose that host.
- ***PTY Proxy***, that provides shell access for containers through the user's browser via websockets, using cool things like *tty.js* with something behind, that I don't know yet how to implement, but it should be pretty easy.



####Hosts
We can have multiple hosts, to scale horizontally. On each host there will be installed Docker and Open vSwitch.
 Each host needs to have:

- an agent that:
    - periodically collects status information and sends them to the controller's scheduler
    - exposes an interface to:
        - manage Docker containers
        - manage OpenvSwitch (VXLAN tunnel endpoints for shared connectivity with other hosts and controller)
        - manage iproute and network namespaces
- PTY proxy endpoint
    - the endpoint for the PTY proxy to serve shell control to users

####Images
Like I said before, images are saved on an external registry.
I will have a plain Docker image called "host", that represents a client or a server. On top of this the user can install and configure as many services as he wants.
Then I can define many *Dockerfile*s to obtain for example:

- a **router**, that is simply an host that allows IP forwarding, and comes with softwares like quagga or similar pre-installed
- a **dhcp server**, that is an host that comes with *dhcpd* preinstalled
- a **dns server**, that is an host that comes with *bind9*
...and so on.

####What else?
Some problems I haven't faced yet are:

- how to allow file uploads to containers
- how to (and if) permit external access to users except for the PTY console
- how to provide internet access

So, what do you think about all this? Am I missing something else? Suggestions, helps, criticism, insults are welcome ;)