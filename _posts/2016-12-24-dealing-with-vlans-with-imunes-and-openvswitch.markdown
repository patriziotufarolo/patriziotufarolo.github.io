---
layout: post
title: Dealing with VLANs with IMUNES and OpenVSwitch
date: '2016-12-24 17:33:17'
---

Hello,

it has been a long time since my last post.
I'm writing this to make a brief recap on how to deal with VLANs in IMUNES on Linux, using the Open-vSwitch CLI tools.

The purpose is to give a reference point to students who are approaching the final project for the Computer Networking classroom @ Unimi.
As I've discussed during the class (see [Lesson 3, Bridges and Switches](https://github.com/patriziotufarolo/NetworkingLabSlides/blob/master/Lez3.pdf)), IMUNES doesn't come with built-in support for VLAN tags at switch level.
You can set up VLAN tags only on interfaces declared with iproute2, I think for a compatibility reason between the BSD version and the Linux version of the product.

However we should keep in mind that IMUNES makes use of Open vSwitch to manage bridges on Linux. That is to say that one can use Open vSwitch CLI tools to accomplish advanced tasks, such as assigning VLAN tags directly on the switch ports through Open vSwitch.

Another important thing we have to consider before proceeding, is the mapping between IMUNES UI and real bridges names.

## What happens when the user runs the experiment?
IMUNES spawns a Docker container for each host or L3 device (router) declared in the topology.
Each L2 device (switch) is mapped to an OVS bridge.

## How are names generated?
All names are generated starting from the experiment id. You can list experiments and all related containers and bridges by running  `# himage -l`.
**Example output**:
```
i4d471 (router1 switch1 switch2)
```
Here `i4d471` represents the experiments, while `router1 switch1 switch2` are the devices declared in the topology.

Devices names are built following this template:
```
$eid[separator]n[progressive-number]
```
The separator can be both a dot or a dash (it depends on your IMUNES build).

For switch port names, IMUNES joins to this template the following:
```
[separator][port-number]
```
where `port-number` is the same as shown in IMUNES's UI (eg. `e0`).

You can list all Open-vSwitch bridges typing: `# ovs-vsctl show`.

#How to manage VLANs?
The command to assign a Vlan ID to a port in Open-vSwitch is:
```
 # ovs-vsctl set port $port-name tag=$vid
```

Only the peers connected to the same VLAN will be able to communicate at layer 2.

A port can also be a trunk, to allow the connected peer (either L2 or L3 devices) to receive traffic tagged with more of one VLAN id.
This is useful for particular configurations like, for example, the so called *router-on-a-stick*, or to transport VLAN tags over more than one switch.
The command is:
```
 # ovs-vsctl set port $port-name trunks=[$vid1,...$vidN]
```
To inspect the current status of a port you can trigger:
```
# ovs-vsctl list port $port-name 
```

In both situations, you have to remember to tag also the traffic coming from the other peer of the communication.
In the *router-on-a-stick* case, the router has to tag packets by itself.
This can be done with three commands:
```
# ip link add link <interface> name <new-link-name> type vlan id <vid>
```
to create a new link with an arbitrary name (`<new-link-name>`) communicating through the interface `<interface>` on the VLAN `<vid>`,
```
# ip link set up dev <new-link-name> 
```
to activate the link and
```
# ip addr add <ip>/<prefix> dev <new-link-name>
```
to eventually set an IP address for the new interface.
In the *VLAN transport* case, instead, you have to remember to set the *trunks* attribute on the trunk port of each switch involved.

Have fun :)

