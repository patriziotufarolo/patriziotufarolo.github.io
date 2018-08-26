---
layout: post
title: Dynamic multi-point VPN with OpenNHRP powered linux hub
date: '2015-12-07 16:01:45'
tags:
- vpn
- nhrp
- cisco
- linux
- opennhrp
- dynamic-multipoint
- centos
- dmvpn
- ipsec
- gre
---

# Introduction
This post aims to explain how to configure a **dynamic multi-point** site-to-site VPN over IPSEC between CISCO routers and a Linux machine using the NHRP protocol.
For our deployment I used a Linux machine as hub and many Cisco 8X7 devices as spokes.
If you are reading this, I think that you already know what IPSec protocol is and how it works. If don't, go read [this](https://en.wikipedia.org/wiki/IPSec).

Most interesting to explain are the NHRP protocol properties.
NHRP is a protocol that can be used to improve the efficiency of the routing protocols in a NBMA network. The purpose is to permit communication between two devices using the most direct route (e.g. the route with the fewest number of hops).
It is based on a query-and-reply mechanism in which all parties cooperate to build a "network knowledge table", to be used to send packets directly to the destination devices (if the devices are on the same subnet) or to an egress router linked to it.
The benefit that the NHRP protocol provides is that it reduces the number of hops that a packet has to pass through enhancing the performance of the network.

# DMVPN
A dynamic multi-point virtual private network is based on a spoke-hub distribution paradigm (also known as star network in telecommunications), so it's orchestrated by the cooperation of:

* 1 central node called ***HUB***
* *n* peripheral nodes called ***SPOKES***

**NHRP** is used to dynamically establish IPSec-encrypted **GRE** tunnels between the nodes involved in a Virtual Private Network.
This allow us not to route all the traffic through the VPN concentrator by creating a fully meshed network between the nodes in a dynamic way, with no configuration effort on the nodes.
Technologies involved are:

- ***GRE***, *Generic Routing Encapsulation*, [RFC1701](https://tools.ietf.org/rfc/rfc1701.txt), to establish dynamic tunnels between spokes
- ***IPSec***, *Internet Protocol Security*, to encrypt those tunnels
- ***NHRP***, *Next-hop resolution protocol*, [RFC 2332](https://tools.ietf.org/rfc/rfc2332.txt), to dynamically discover the endpoint of the tunnel
- ***RIPv2***, *Routing Information Protocol*, [RFC 2453](https://tools.ietf.org/rfc/rfc2453.txt), to propagate internal routes between the nodes. Yes, it's better and easier to use OSPF, but our CISCO routers didn't support it **:-(** so I relied on RIPv2.

# SETUP
*For security reasons, I won't provide any configuration related to security mechanisms involved. You can implement whatever you want.*

## Hub
The hub I used is a CentOS 7 Linux machine.
To support the NHRP protocol I used **[OpenNHRP](http://sourceforge.net/projects/opennhrp/)**, an open-source implementation of the NHRP protocol. To bring up the IPSec tunnels, I used **[racoon](http://ipsec-tools.sourceforge.net)** with pre-shared key based authentication.
As the IP addresses of my spokes are assigned dynamically by their internet provider, I had to patch ipsec-tools 0.8.2 to support the wildcard character in the *psk* file.
The patch I used is the following:

```
--- ipsec-tools-0.8.2.orig/src/racoon/localconf.c	2012-08-23 13:10:45.000000000 +0200
+++ ipsec-tools-0.8.2/src/racoon/localconf.c	2015-11-18 22:25:00.902843868 +0100
@@ -207,7 +207,7 @@
 		if (*p == '\0')
 			continue;	/* no 2nd parameter */
 		p--;
-		if (strncmp(buf, str, len) == 0 && buf[len] == '\0') {
+		if ((strcmp(buf, "*") == 0) || (strncmp(buf, str, len) == 0 && buf[len] == '\0')) {
 			p++;
 			keylen = 0;
 			for (q = p; *q != '\0' && *q != '\n'; q++)
```
If you need, I can provide the *rpmbuild* directives to compile *ipsec-tools* with these directives on CentOS. Just mail me through my [website](https://patrizio.tufarolo.eu).

RIP routing is managed by **[quagga](http://www.nongnu.org/quagga/)**, a fork of GNU Zebra that implements a CISCO-like command line interface with many network daemons.
Let's go through the configurations.

First of all, I have to setup a GRE  tunnel.
It's pretty easy to do so in CentOS Linux:

**/etc/sysconfig/network-scripts/ifcfg-dmvpn**
```
DEVICE=dmvpn
BOOTPROTO=none
ONBOOT=yes
TYPE=GRE
MY_INNER_IPADDR="10.254.254.1/24"
KEY=#ChooseYourGREKeyHere#
TTL=64
```
Replace #ChooseYourGREKeyHere# with a key of your choice.
Then I have to set broadcast address and enable multicast.
Let's create a script in */sbin* called *ifup-local*.

**/sbin/ifup-local** 

```
#!/bin/sh
if [[ "$1" == "dmvpn" ]]
then
ifconfig dmvpn broadcast 10.254.254.255
ifconfig dmvpn multicast
fi
```
Remember to make it executable running:
```
# chmod u+x /sbin/ifup-local
```

The next step it's to install OpenNHRP. It is not provided as CentOS package, so you have to download it from the link I provided and compile it manually.
```
$ tar xfv opennhrp-0.14.1.tar.bz2
$ cd opennhrp-0.14.1
$ make
# make install
```
A kernel version higher than 3.12 is recommended because I got some strange issues in IP handling with kernel 3.10, that have been patched in the later releases.
Let's configure OpenNHRP:

**/etc/opennhrp/opennhrp.conf**
```
interface dmvpn 
  holding-time 3600
  multicast dynamic
  shortcut
  redirect
  non-caching
```
Here I am using dynamic multicast to advertise routes through the RIP protocol. For further and more detailed configurations read the *opennhrp* manpage.

Let's configure racoon:
**/etc/racoon/racoon.conf**
All the encryption parameters have to be coherent with the spokes configuration, of course.

```
path include "/etc/racoon";
path pre_shared_key "**************";
path script "/etc/racoon/scripts";


remote anonymous {
      exchange_mode main,aggressive;
      lifetime time 24 hour;
      script "/etc/racoon/scripts/pat_up.sh" phase1_up;
      script "/etc/opennhrp/racoon-ph1down.sh" phase1_down;
      script "/etc/racoon/scripts/pat_up.sh" phase1_down;
      proposal {
         encryption_algorithm #specify your encryption algorithm#;
         hash_algorithm #specify your hash function#;
         authentication_method pre_shared_key;
         dh_group #specify you diffie-hellman group#;
      }
}

sainfo anonymous
{
	lifetime time 1 hour ; #lifetime for the security association
	encryption_algorithm #specify your encryption algorithm for the sa phase# ;
	authentication_algorithm #specify your authentication algorithm# ;
	compression_algorithm deflate ;
}

```
Remember to replace:

- #specify your encryption algorithm#
- #specify your hash function#
- #specify you diffie-hellman group#
- #specify your encryption algorithm for the sa phase#
- #specify your authentication algorithm#

with your values.
Note that the phase1 up and down call the script pat\_up.sh. Try to guess why it's called pat\_up **:P**

**/etc/racoon/scripts/pat_up.sh**
```
shopt -s nocasematch
umask 0022

PATH=/bin:/sbin:/usr/bin:/usr/sbin
TMPDIR="/var/racoon"

phase1_up() {
  setkey -f /etc/racoon/setkey.conf
}

# bring down phase1
phase1_down() {
  setkey -F
  setkey -P -F
}


case "$1" in
  phase1_up)
    phase1_up
    ;;
  phase1_down)
    phase1_down
    ;;
  *)
    echo "p1_up_down: error: must be called by racoon w. arg=phase1_[up|down]"
    exit 3
    ;;
esac

echo "p1_up_down: $1 completed successfully."
exit 0
```
This script sets the security policies every time an IPSec tunnel it's brought up, by grabbing them from the */etc/racoon/setkey.conf*.
Remember to make it executable or it won't be executed:
```
chmod u+x /etc/racoon/scripts/pat_up.sh
```

**/etc/racoon/setkey.conf**
```
spdflush;

spdadd 0.0.0.0/0 0.0.0.0/0 gre -P out ipsec esp/transport//require;
spdadd 0.0.0.0/0 0.0.0.0/0 gre -P in ipsec esp/transport//require;
```

The last thing to do it's to configure Quagga to allow routing.
First of all enable IP forwarding on your network interface, if you haven't already done it.

```
# sysctl -w net.ipv4.ip_forward=1
```

Then enable *ripd* in */etc/quagga/daemons*

```
# echo ripd=1 >> /etc/quagga/daemons
```

My ripd configuration looks like this:

**/etc/quagga/ripd.conf**

```
!
hostname ripd
password mypassword
log stdout
!
interface dmvpn
 no ip rip split-horizon
!
router rip
\#specify here all the network and neighbors you want to announce through rip.
!
line vty
!
```

And your Linux based DMVPN Hub is ready to run! **:)**

This is the systemd unit file I used to launch OpenNHRP every time my server boots:

**/etc/systemd/system/opennhrp.conf**
```
[Unit]
Description=OpenNHRP
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/usr/sbin/opennhrp -d -a /var/run/opennhrp/ctrl -c /etc/opennhrp/opennhrp.conf -s /etc/opennhrp/opennhrp-script -p /var/run/opennhrp/pid
ExecReload=/usr/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```
Racoon and Quagga already provide their unit file.

## Spokes

*with the cooperation of Emanuele Bosetti*

The spokes are all Cisco 8x7 routers, let's configure them.
First of all let's declare an ISAKMP policy for Phase 1 negotiations.
```
crypto isakmp policy 1
 encr #specify encryption algorithm#
 hash #specify your hash function#
 authentication pre-share
 group #specify your diffie-hellman group# 
```
Remember to replace the values between the hash characters with your own settings (matching hub's values).

Let's specify the encryption key:
```
crypto isakmp key #yourpsk# address x.x.x.x
```
and replace **x.x.x.x** with hub's IP address.

Now define the transform set for phase2 data encryption and put the tunnel in transport mode:
```
crypto ipsec transform-set TS #specify your encryption algorithm here# #specify your hashing function here#
  mode transport
```

Let's define an IPSec profile to be applied to dynamically built GRE tunnels:
```
crypto ipsec profile GRE
 set transform-set TS 
```

Finally define the GRE Tunnel
```
interface Tunnel0
 ip address 10.254.254.2 255.255.255.0
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 ip mtu 1400
 ip flow ingress
 ip pim sparse-dense-mode
 ip nhrp map 10.254.254.1 #HUB IP#
 ip nhrp map multicast #HUB IP#
 ip nhrp network-id 1
 ip nhrp holdtime 3600
 ip nhrp nhs #HUB IP#
 ip route-cache same-interface
 ip tcp adjust-mss 1300
 no ip split-horizon
 load-interval 30
 tunnel source Dialer0
 tunnel mode gre multipoint

 tunnel key #GRE KEY#
 tunnel protection ipsec profile GRE
```
And turn on RIP:
```
router rip
 version 2
 network #YOURNETWORK#
```

And your spoke is up and running!
