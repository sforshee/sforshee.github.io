---
layout: single
title: 'EdgeRouter: Accept OpenVPN Connections on Multiple Ports'
tags:
---

Since different networks frequently have different restrictions on what ports
are allowed to access the internet, it's useful to have OpenVPN listening on 
multiple ports for incoming connections. However OpenVPN does not support 
listening on multiple ports from a single instance of the server. The usual 
advice is to run separate instances of OpenVPN for each port.

This has some obvious disadvantages - more configurations to manage, more 
subnets, more firewall rules, etc. Fortunately there's another way we can do 
this, by having the router translate the destination port for incoming packets
to a different port, commonly called _port forwarding_.

The Ubiquiti EdgeRouter products offer two different configuration interfaces 
for setting this up, _destination NAT_ (_DNAT_) and _port forwarding_. I've 
chosen to use DNAT, simply because I wanted to play around with the NAT 
functionality.

## Which Ports?

Before starting we need to know which ports we want OpenVPN to listen on. This 
will vary depending on the situation. I'm going to assume a typical home user 
who wants to be able to access their home network from various locations and who 
does not run a public facing web server from their network.

Any network which allows general web access must allow clients to establish 
connections over TCP ports 80 and 443, which respectively are the standard ports 
for HTTP and HTTPS. Therefore setting up OpenVPN to listen on these ports will 
allow access from practically any network. In this example I'll assume that 
OpenVPN is listening on TCP port 443 and we want to also forward connections on 
port 80.

## Setting Up DNAT

The instructions here assume that you already have OpenVPN up and running on the 
EdgeRouter. Please see
[here]({% post_url 2016-03-16-ubiquiti-edgerouter-lite-setup-part-5-openvpn-setup %})
for more information about OpenVPN setup. That post also notes a pitfall of 
running OpenVPN on port 443 and how to handle it.

The following commands set up a DNAT rule to forward incoming connections on 
port 80 to port 443.

{% highlight console %}
$ configure
# edit service nat rule <rule-number>
# set description "Port forward for OpenVPN"
# set type destination
# set inbound-interface <wan-iface>
# set destination port 80
# set inside-address port 443
# set protocol tcp
{% endhighlight %}

Fill in the rule number and the WAN interface. The rule number is only going to 
be significant if you have other DNAT rules, as rules will be executed 
sequentially.  I found
[this article](https://help.ubnt.com/hc/en-us/articles/204976494-EdgeMAX-Add-source-NAT-rules) 
from Ubiquiti which states that source/masquerade NAT rules must be numbered 
5000 or higher, but I didn't find anything stating a similar limitation on the 
number of DNAT rules. It's probably a good idea to leave some space in case you 
later need to insert rules before this one, so something like 100 should be 
fine. You can always change the rule number later by copying the rule to a new 
one and deleting the original.

After committing these changes it will be possible to connect to OpenVPN on both 
TCP ports 80 and 443. You might think that the firewall rules need updates to 
allow incoming TCP connections on port 80, but this isn't the case. DNAT happens 
before the firewall, so a rule is only needed for the translated address.

## Limitations

There is one significant limitation to this approach - it's not possible to 
forward TCP connections to a UDP port, or vice versa, since they are different 
protocols. If you wish to accept both TCP and UDP connections it is necessary to 
run two OpenVPN instances. In that case the rule above can be modified to 
specify `tcp_udp` for the protocol so that connections using either protocol 
will be forwarded.
