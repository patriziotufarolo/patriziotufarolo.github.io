---
layout: post
title: 'IMUNES: how to export changes made to an experiment'
date: '2016-03-15 21:56:02'
tags:
- linux
- openvswitch
- docker
- imunes
- networking
- network-simulator
- networking-laboratory
- unimi
- crema
- bash
- export-imunes
- import-imunes
---

These are two snippets I've written in BASH in the last few days to allow IMUNES users to export changes made to an experiment in runtime, on Linux.
The former allows users to export changes, the latter allows them to load changes back to Docker containers.
They will be improved day by day, so it's recommended to consult this page to get an updated version of them, especially if you encounter problems.

The execution flow to use these scripts properly is:

1. Draw a topology
2. Run the experiment
3. Make changes you want to make (e.g. configurations, your files, etc.)
4. Use the first script to export those changes to your computer's file system
5. Stop the experiment
6. Save the topology
7. Close IMUNES

---
8. Open IMUNES again
9. Load the topology back
10. Execute the experiment
11. Use the second script to load your previously exported changes back
12. Enjoy!

Notes:

- Your Linux distribution needs to have the **zenity** package installed
- Both scripts need to be executed with **root privileges** or by a user that is assigned to the **docker** group.
Remember that if you run them with root privileges, the files created by them will be owned by root, so you could encounter permission problems.
- You can use these scripts passing them as argument to the BASH interpeter.
```
# bash imunes-export.sh
```
```
# bash imunes-import.sh
```
- You can either assign the executable flag to the scripts with
```
chmod +x imunes-export.sh ; chmod+x imunes-import.sh
```
and call them directly writing
```
./imunes-export.sh
```
or
```
./imunes-import.sh
```
in the directory where they are placed.
- Obviously you can put them (with the executable flag assigned) in a directory specified in the *PATH* environment variable (e.g. */usr/bin*) to be able to call them from anywhere.

<a href="https://gist.github.com/patriziotufarolo/e5003c3e04fa67fba697" title="View scripts" class="gist-link">View scripts</a>

You can report issues (and suggest corrections of course) using my own comment system or GitHub's one.
