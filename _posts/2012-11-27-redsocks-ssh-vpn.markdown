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

So, when I'm at home (or in a cafe with wifi), I have transparent access to any services in my work lan.


Step 1:  Set up a persistant SSH connection
-----------------------------------------------------------------------------------------------
Since the SSH SOCKS proxy is my access to the work LAN, I have a persitant SSH connection
running on my laptop.  It's this simple:

	while date; do ssh -S none -d 4444 $USER@$SSH_SERVER -t watch date; sleep 1; done

This is just a simple loop that retries the SSH connection if it ever dies.  The "watch date"
part just makes sure there is somethign in it, so I can easily see if it's "stuck".

You should verify now that your SOCKS5 proxy works to get you stuff in the work lan now:

	echo GET / | socat - SOCKS:localhost:192.168.1.XXX:80,socksport=4444

Pick an IP address where your work lan has a HTTP server listening, and verify that you get
the response you expect.

Step 2: Install redsocks transparent proxy
-----------------------------------------------------------------------------------------------

Install redsocks:

	apt-get install redsocks

Make sure it's enabled in /etc/default/redsocks.

Configure redsocks by editing /etc/redsocks.conf.  I have it set to send all
redirected TCP connections (port 12345) through my SSH SOCKS proxy (port 4444),
and to "stub" any DNS/UDP packet on port 12353 as "truncated answer" to redirect it to
TCP (which is then handled by the TCP redirection proxy).

Here are my 2 redsock stanza:

	redsocks {
		/* `local_ip' defaults to 127.0.0.1 for security reasons,
		 * use 0.0.0.0 if you want to listen on every interface.
		 * `local_*' are used as port to redirect to.
		 */
		local_ip = 127.0.0.1;
		local_port = 12345;

		// `ip' and `port' are IP and tcp-port of proxy-server
		// You can also use hostname instead of IP, only one (random)
		// address of multihomed host will be used.
		ip = 127.0.0.1;
		port = 4444;


		// known types: socks4, socks5, http-connect, http-relay
		type = socks5;

		// login = "foobar";
		// password = "baz";
	}


	dnstc {
		// fake and really dumb DNS server that returns "truncated answer" to
		// every query via UDP, RFC-compliant resolver should repeat same query
		// via TCP in this case.
		local_ip = 127.0.0.1;
		local_port = 12353;
	}



Step 3: Setup dnsmasq
-----------------------------------------------------------------------------------------------

In my setup, I use dnsmasq on my laptop as a "local resolver".  Dnsmasq works really well with resolvconf
on debian, so any time I get DNS servers from DHCP, or from static interfaces, dnsmasq automatically
forwards requests to them, unless a local config overrides it.

In my case, I have this in my dnsmasq.conf:

	server=/example.com/192.168.1.1
	server=/1.168.192.in-addr.arpa/192.168.1.1

This tells my dnsmasq that for resolving my work addresses, always use the work lan DNS server.

Step 3: Setup iptables
-----------------------------------------------------------------------------------------------

To make traffic go through the redsocks transparent proxy, just make use of the iptables
REDIRECT nat target to point traffic to it.  I have a REDSOCKS chain I send traffic into,
and it looks like:

	Chain REDSOCKS (2 references)
	    pkts      bytes target     prot opt in     out     source               destination         
	    2495   159894 RETURN     all  --  *      *       0.0.0.0/0           !192.168.1.0/24      
	       0        0 RETURN     all  --  *      *       192.168.1.0/24       0.0.0.0/0           
	       0        0 REDIRECT   udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53 redir ports 12353
	      20     1200 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 12345

The 1st rule is just a shortcut, if traffic get's in there that's not destined
for my work network, just return.  The 2nd rule checks that I'm on on the work lan,
if it is, just returns it.

The 3rd rule redirects the "DNS UDP" traffic to my redsocks DNS stub.  That will
just make every UDP DNS query to my work lan DNS servers be instantly with a "truncated answer"
and the DNS query switches to TCP.

The 4th rule catches all TCP connections, and redirects them to redsocks.

