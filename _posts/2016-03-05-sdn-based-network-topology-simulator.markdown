---
layout: post
title: 'New project: SDN-based network topology simulator'
date: '2016-03-05 14:57:17'
tags:
- sdn
- openvswitch
- docker
- imunes
- cloud-computing
- networking
- python
- flask
---

Next week I'm going to start a tutoring for the Networking laboratory class at University of Milan, in Crema. 
One of the most difficult tasks I had to accomplish, has been to find the right appliance to be used to teach networking concepts to students. 
For the moment I've decided to use IMUNES, a software that builds network topologies on top of Docker containers and OpenvSwitch. 
IMUNES is a great software, but it lacks of some functionalities I need to use (such as VLAN tagging on OpenvSwitch ports, trunking, and storage persistence for Docker containers running services), and even if it can be set up easily, its deployment required some effort due to some our environmental characteristics. 
The result is that I decided to start a new project: my own network topology simulator, built on the same technologies as IMUNES, web based, multi-user and cloud ready. 
I want to develop a software that developers, organizations and universities can use for testing, research, experiments and teaching purposes; it has to be elastic, scalable (cause it's cool too say to someone "my software scales well dude!"), as-a-service and obviously open-source. 
Maybe I will fail, maybe it won't never work, maybe the project will die, maybe it's too ambitious or maybe I am just crazy. 
But I want to try.