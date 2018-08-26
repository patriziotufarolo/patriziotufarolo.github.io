---
layout: post
title: 'IMUNES: how to export changes made to an experiment - Part 2'
date: '2016-05-01 20:09:57'
tags:
- linux
- openvswitch
- docker
- imunes
- networking
- python
- iso-osi
- network-simulator
- networking-laboratory
- unimi
- crema
- export-imunes
- import-imunes
---

![Screenshot](/content/images/2016/05/screenshot.png)

Today I've written again from scratch the two BASH scripts to export changes made in IMUNES, this time using Python.
I've built a UI with GTK3 using Glade, and I've organized the code in a more clean and safe way.
Docker management is done via the official library docker-py.

##Install
The software is available on GitHub: https://github.com/patriziotufarolo/ImunesExperimentExporter

So:

* Clone the repository and browse it
```
git clone https://github.com/patriziotufarolo/ImunesExperimentExporter
cd ImunesExperimentExporter
```
* Install dependencies through Pip
```
pip install -r requirements.txt
```
* Install the software by calling
```
sudo python2 setup.py install
```
* Enjoy the software calling it with the command
```
imunes-export
```

**You need to have Docker configured for listening on UNIX socket (at the path /var/run/docker.sock). Please note that in a normal environment this socket file is owned by root and docker group, so it could be needed to run imunes-export with root or docker group privileges.**

##Export a topology
You can choose to export a single container or the whole experiment.
The execution flow is the same as the last time.

1. Draw a topology
2. Run the experiment
3. Make changes you want to make (e.g. configurations, your files, etc.)
4. Use my software to export those changes to your computer's file system
5. Stop the experiment
6. Save the topology
7. Close IMUNES

##Import a topology
You have to do this operation container by container, because it's safer.
IMUNES uses an odd file to save its topologies and I didn't want to implement a more intelligent way to export them in a more clean way :)

1. Open IMUNES again
2. Load the topology back, container by container, selecting the folder where you've saved it before.
3. Execute the experiment
4. Use my software load your previously exported changes back
5. Enjoy!

--

Please let me know if you find bug, issues and...patches :) 