---
layout: post
title: How to configure PiHole in QubesOS (ProxyVM)
date: '2017-12-27 15:21:33'
tags:
- pihole
- raspberry-pi
- qubesos
- dns
- ad-blocker
- security
- dnsmasq
- debian
- fedora
- xen
- virtualization
- proxy
---

[Pi Hole](https://pi-hole.net/) is a framework that aims to block advertisements on your network using a sinkhole approach on known advertisements domains.

PiHole is based on Raspberry PI, a device that almost all computer enthusiasts have in their home, it is built on top of DNSMasq weaponized with community-maintained blacklist and can be backed with your favorite DNS servers.

But PiHole is not just and adblocker: it provides also a DHCP server and a dashboard for configuration, monitoring and tracking visited urls, allowing the PI owner to detect suspicious behaviors in the network.
Is very convenient to use it, because you have a central point in the network in which ads are conveyed avoiding the usage of addons that act browser-level, and preventing anti ad-blockers mechanisms.
Furthermore, it's open-source and [available on GitHub](https://github.com/pi-hole)
If you like to play with dedicated hardware, you can adopt a standard approach buying a cheap Raspberry Zero, installing PiHole on it, configuring it to be recognized as an USB ethernet adapter and play with NAT on your machine to give connectivity to him.

The annoying thing comes when you use to use a laptop: you don't have your Raspberry PI in your pocket and, if you have it, you really don't want to plug it everytime you browse the internet.

You can then adopt a software-based approach installing PiZero on your machine.
Of course you can also configure DNSMasq and related blacklists on your own, but why don't use a valid framework without reinventing the wheel?

There are three approaches you can use:

- install PiZero directly on your machine, polluting your system with custom dnsmasq configurations
- install PiZero in a Docker container (images are available on the Docker Hub) without smudging your machine
- virtualize PiZero

What??? **Virtualize???** Yep.
As some of my friends know, I've become a little paranoid about security, and I've been using QubesOS since a while as my primary OS both at home and work.
[QubesOS](https://www.qubes-os.org) is an operating system based on the XEN hypervisor that adopts the *security by isolation* paradigm. The Dom0 is based on Fedora and every application is ran in a dedicated virtual machine.

Here is how I integrated PiHole in QubesOS:

- I created a ProxyVM, using the stock Debian 8 template as base, and enabling the 'Standalone' flag. I configured the firewall virtual machine as NetVM for this one.
- I disabled systemd-resolved because I don't want it
```
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```
- I updated the software in the vm, typing the well known commands
```
apt update && apt upgrade
``` 
- Installed git and curl to clone the git repository (yes, I know, there's a more ~~practical~~ way to install PiHole , typing `curl -sSL https://install.pi-hole.net | bash`. Please, don't do this: piping curl and bash together could be very dangerous because you don't know which commands are being executed on your system; the content of the page could be altered by the server dynamically or by Mallory.
Use git and review the sources.)

```
apt install git curl
```

- Cloned the repository and installed the software:
```
git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
cd Pi-hole
# [...] source code review
cd automated\ install
bash basic-install.sh
```
Now I have Pi-hole installed on my virtual machine, but it's not enough: I have to make the other VMs use it as DNS server.
To do so, I configured the VMs I wanted to protect with PiHole to have the Proxy VM as their NetVM.

In this way when I will trigger a name resolution on those virtual machines, the DNS traffic will be routed through the PiHole proxy... ops it's not enough again: in fact, PiHole is not serving the DNS requests... but name resolution is currently working... WUT???

To understand the reason, I inspected the DNS implementation of Qubes OS: DNS requests are being managed by `netfilter` and natted through all hops from each AppVM or HVM to the Network VM.
In the NAT table of the PiHole ProxyVM, in fact, we find a chain named `PR-QBS` called from the `PREROUTING` chain. `PR-QBS` contains a DNAT rule that redirects the traffic arriving on the port 53 to the next hop; a matching rule has been inserted in the FORWARD chain of the filter table to allow the forwarding.
Cause of this configuration, the DNS request never reaches DNSMasq, so I reverted it.
To do so, I edited the file `/rw/config/qubes-firewall-user-script`; this script is called everytime the firewall configuration gets updated.
In that script I wrote the following commands:
```
# Flush the PR-QBS chain
iptables -t nat -F PR-QBS

# Add a rule that redirects all the DNS traffic to localhost:53
iptables -t nat -I PR-QBS -i vif+ -p udp --dport 53 -j DNAT --to-destination 127.0.0.1

# Add a rule that accepts the traffic coming to localhost
# from XEN's virtual interfaces on port 53

iptables -I INPUT -i vif+ -p udp --dport 53 -d 127.0.0.1 -j ACCEPT

# Enable the traffic coming from the virtual interfaces
# to be forwarded to the loopback interface
# enabling the route_localnet flag on them

find /proc/sys/net/ipv4/conf -name "vif*" -exec bash -c 'echo 1 | sudo tee {}/route_localnet' \;
```
I then configured DNSMasq to listen only on 127.0.0.1 and removed the comment form the `bind-interfaces` option in `/etc/dnsmasq.conf'.

Here's the current output of  `grep -v -E '^(#|$)' /etc/dnsmasq.conf`:
```
interface=lo
bind-interfaces
conf-dir=/etc/dnsmasq.d
```
Did the 'reboot test' on all vms and...it's working!

Have fun and great Christmas holidays :) 
