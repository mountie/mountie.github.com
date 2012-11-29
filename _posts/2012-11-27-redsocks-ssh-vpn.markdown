---
layout: post
title: VPN through an SSH server
categories:
  - blog
  - linux
  - network
  - VPN
---

I'm at a company now where I work from work, and from home.  And much of the work involves
access to work servers in the lab.   We have access through a single SSH server, running
on a "bastion host" on a non-standard port...

So, when I'm at home, I want quick access to everything on the work lan.  HTTP servers
(running gitlab, jenkins, bugzilla, wiki, etc), GIT servers (over ssh and git), VMWare
servers, RDP servers, etc...

My solution to this involves the SSH server we have access to, SSH's built-in SOCKS5 proxy,
and redsocks, a proxy server that makes use of "REDIRECT" firewall rules.

One potential problem is if both the "work lan" and my "home lan" are using the same address
range.  That would cause IP address conflicts between home devices, and work devices.  But
thankfully, I don't have such a conflict.  When I numbered my home LAN, I purposely avoided
`192.168.1.0/24`, which it seems most other network use "by default".

So, when I'm at home, I have transparent access to any services in my work lan.


Step 1:  Set up a persistant SSH connection
-----------------------------------------------------------------------------------------------
Since the SSH SOCKS proxy is my access to the work LAN, I have a persitant SSH connection
running on my laptop.  It's this simple:

	while date; do ssh -S none -d 4444 $USER@$SSH_SERVER -t watch date; sleep 1; done

This is just a simple loop that retries the SSH connection if it ever dies.  The "watch date"
part just makes sure there is somethign in it, so I can easily see if it's "stuck".

You should verify now that your SOCKS5 proxy works to get you stuff in the work lan now:

	echo GET / | socat - SOCKS:localhost:192.168.1.XXX:80,socksport=4444

Pick an IP address where your work lan has a HTTP server listening...

Step 2:  Remove LVM
-----------------------------------------------------------------------------------------------

Step 3: Clean up old mirrors
-----------------------------------------------------------------------------------------------

