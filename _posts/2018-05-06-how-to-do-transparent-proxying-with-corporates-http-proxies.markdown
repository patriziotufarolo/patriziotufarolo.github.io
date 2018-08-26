---
layout: post
title: "#FuckYouHTTPProxy | How to use an HTTP proxy system-wide on Linux | How to
  do transparent proxying with corporates' HTTP proxies"
date: '2018-05-06 10:15:18'
tags:
- linux
- docker
- networking
- proxy
- http_proxy
- transparent-proxy
- iptables
- redsocks
- cntlm
- ntlm
- microsoft
- cloudflare
- cloudflared
- libvirt
---

## Preamble:
* Corporates use HTTP proxies.
* HTTP proxies suck.
* HTTP proxies implementation on Linux sucks than everything.
* I have no control of the shitty proxy I'm talking about

## Problem:
HTTP proxies suck and corporates use HTTP proxies.
Ok, I already said that.
In the company I'm working in as security consultant, all machines connected to the intranet can't reach the internet. The only way to do so, is to use an internal HTTP proxy that does deep inspection of the internet traffic in order to block "malicious" sites, control downlodaded contents and so on. It's a mess, because also security related websites are blocked too.
The big problem comes when you are also involved in system operations, particullary with Linux systems: handling the proxy in Unix-like systems is a nightmare.
Yes, there is the `http_proxy` environment variable, you set it and almost every software will use its content to proxy network requests.
Almost.
And what if you need to do some routing, for example to run Docker containers or virtual machines, or proxy-unaware applications?
It happens that, if you don't specify the `http_proxy` variable inside every single container or vm, they won't reach the internet.
Now, from a security perspective, blocking internet access to containers is good. But what about vms? Also, often we use orchestration tools like Ansible, Puppet and so on, you will forget to specify that hell's environment variable, and the whole deployment will crash.

So, how to handle this situation with a definitive solution?
Here is what I did, playing a bit with some useful Linux tools.

## Solution
The solution I've adopted chains together CNTLM, REDSOCKS and IPTABLES. Then uses cloudflare for external name resolution.
### CNTLM
First thing to consider: Everything is joined to a central Microsoft domain here, so the proxy I'm talking about uses NTLM authentication.
To address this problem I've used **CNTLM** (http://cntlm.sourceforge.net).
CNTLM is a very small daemon that binds to `127.0.0.1` and exposes a local http proxy that forwards requests to the corporate's one, authenticating them at the same time.
The configuration I've used is very simple:
```
Username        PROXY_USERNAME
Domain          PROXY_DOMAIN

Auth            NTLM
PassNT          FIRST_PART_OF_HASH
PassLM          SECOND_PART_OF_HASH

Proxy           PROXY_ADDRESS:PORT
NoProxy         localhost, 127.0.0.*, 10.*, 192.168.*

Listen          3128
```

### REDSOCKS
The exposed proxy in the step above uses plain HTTP.
There is also a directive that enables a SOCKS5 proxy, but I preferred to use **Redsocks** instead with this configuration:
```
base {
        log_debug = off;
        log_info = on;
        log = "syslog:daemon";
        daemon = on
        user = redsocks;
        group = redsocks;
        redirector = iptables;
}
redsocks {
        local_ip = 127.0.0.1;
        local_port = 12345;
        ip = 127.0.0.1;
        port = 3128;
        type = http-connect;
}
```
You can find Redsocks in the repositories of most known Linux distributions.
Redsocks connects to the local CNTLM instance at http://127.0.0.1:3128 and exposes a SOCKS5 proxy on port 12345.

### IPTABLES
Then I used `iptables` to redirect all requests to the SOCKS proxy, through DNAT.

First I created a new chain in the NAT table:
```
iptables -t nat -N TRANSPARENT_PROXY
```
then I populated that chain with some default rules to exclude private addresses provided from RFC5735(http://tools.ietf.org/html/rfc5735):
```
iptables -t nat -A TRANSPARENT_PROXY -d 0.0.0.0/8 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 100.64.0.0/10 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 169.254.0.0/16 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 198.18.0.0/15 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A TRANSPARENT_PROXY -d 240.0.0.0/4 -j RETURN
```
then I handled name resolution redirecting all TCP and UDP traffic to port 53 to a local resolver.
```
iptables -t nat -A TRANSPARENT_PROXY -p udp --dport 53 -j DNAT --to 127.0.0.1:53
iptables -t nat -A TRANSPARENT_PROXY -p tcp --dport 53 -j DNAT --to 127.0.0.1:53
```

At last, I redirected the remaining TCP traffic to the port TCP/12345.
```
iptables -t nat -A TRANSPARENT_PROXY -p tcp -j DNAT--to 127.0.0.1:12345
```

I then inserted two rules to jump to the TRANSPARENT_PROXY chain from the OUTPUT and PREROUTING ones, to redirect both the generated and routed traffic through the chain:
```
iptables -t nat -A OUTPUT -p tcp -j TRANSPARENT_PROXY
iptables -t nat -A PREROUTING -p tcp -j TRANSPARENT_PROXY
```
To make this configuration work, the last step is to set the `route_localnet` flag to 1 on sysctl with:
```
sysctl -w net.ipv4.conf.all.route_localnet=1
```
Now the TCP over IP traffic to any address in the internet works fine.


### One last problem: name resolution
The only thing that still doesn't work is the DNS traffic: DNS uses port UDP/53 and all the shit I've set up, doesn't allow UDP traffic to work.
With the standard `http_proxy` method is the proxy that does name resolution; in this way, we have no method to resolve names.
If you have an internal DNS server that indexes the internet, use that.
This wasn't my case so I used the new Cloudflare DNS over HTTPS service.

In fact, Cloudflare has set up a new secure DNS service on 1.1.1.1 and 1.0.0.1, that you can use instead of the Google spying servers at 8.8.8.8 and 8.8.4.4 to which we have been accustomed during the last decade.
This server listens also on **https**, and provides an endpoint (https://cloudflare-dns.com/dns-query) to do name resolution over a REST interface.
Cloudflare provides also a ready-to-use daemon (named *cloudflared*) that binds on port `127.0.0.1:53` and redirects all requests to that endpoint.
You can find it here:
https://developers.cloudflare.com/1.1.1.1/dns-over-https/cloudflared-proxy/
and launch it with:
```
sudo cloudflared proxy-dns
```
then you can override DNS configuration of your system by editing NetworkManager's configuration or by typing:
```
echo nameserver 127.0.0.1 | tee /etc/resolv.conf
```

It's all :) 
Have fun.

PS:
My deployment activity failed again, because the company's proxy is blocking the SaltStack repository because it is considered **MALICIOUS**.
I fucking hate this fucking proxy.