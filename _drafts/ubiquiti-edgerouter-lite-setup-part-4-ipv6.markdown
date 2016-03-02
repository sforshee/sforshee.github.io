---
layout: post
title: 'Ubiquiti EdgeRouter Lite Setup Part 4: IPv6'
author: Seth Forshee
tags: 
---

At this point we've done [basic setup]() of the ERL, configured a [zone-based firewall]() and set up flexible network partitioning using [VLANs](). But so far we're only supplying an IPv4 network to our clients. In this post we're going to learn about deploying IPv6.

Talking about IPv6 setup is a bit tricky because the details may vary depending on how the ISP has set up their network, assuming the ISP supports IPv6 at all. Here I'll use my ISP's network as an example. As luck would have it this is probably a fairly common setup.

# DHCPv6 and Prefix Delegation

[RFC 3633](https://tools.ietf.org/html/rfc3633) defines a mechanism by which [DHCPv6](https://en.wikipedia.org/wiki/DHCPv6) can be used to delegate a network address prefix to a network. This is known as [prefix delegation](https://en.wikipedia.org/wiki/Prefix_delegation). The router can then assign addresses to clients within the network using either DHCPv6 or [stateless address autoconfiguraton](https://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration) (SLAAC). With SLAAC the router advertises a prefix to clients, and clients pick their own address within that network. This example will use SLAAC.

# WAN Setup

The first step in configuring the ERL is to set up the WAN interface to request a prefix via DHCPv6. Our network is divided into multiple LANs, so we'll divide up the assigned prefix into smaller networks that can be advertised on each lan, assigning the ::1 address in the network to the virtual interface in the ERL.

{% highlight console %}
$ configure
# edit interfaces ethernet eth0
# set dhcpv6-pd rapid-commit enable
# set dhcpv6-pd pd 1 prefix-length /56
# set dhcpv6-pd pd 1 interface eth2.1 service slaac
# set dhcpv6-pd pd 1 interface eth2.1 prefix-id 1
# set dhcpv6-pd pd 1 interface eth2.1 host-address ::1
# set dhcpv6-pd pd 1 interface eth2.2 service slaac
# set dhcpv6-pd pd 1 interface eth2.2 prefix-id 2
# set dhcpv6-pd pd 1 interface eth2.2 host-address ::1
# set dhcpv6-pd pd 1 interface eth2.3 service slaac
# set dhcpv6-pd pd 1 interface eth2.3 prefix-id 3
# set dhcpv6-pd pd 1 interface eth2.3 host-address ::1
# top
{% endhighlight %}

The prefix length is determined by your ISP, so you will need to check with them to determine the correct value.

# LAN Setup

Now each vif must be configured to advertise its assigned IPv6 prefix to clients.

{% highlight console %}
# edit interfaces ethernet eth2 vif 1
# set ipv6 dup-addr-detect-transmits 1
# set ipv6 router-advert cur-hop-limit 64
# set ipv6 router-advert link-mtu 0
# set ipv6 router-advert managed-flag false
# set ipv6 router-advert max-interval 600
# set ipv6 router-advert other-config-flag false
# set ipv6 router-advert prefix ::/64 autonomous-flag true
# set ipv6 router-advert prefix ::/64 on-link-flag true
# set ipv6 router-advert prefix ::/64 valid-lifetime 2592000
# set ipv6 router-advert reachable-time 0
# set ipv6 router-advert retrans-timer 0
# set ipv6 router-advert sent-advert true
{% endhighlight %}

Again here the size of your prefix may depend on the size of the prefix assigned by your ISP. Since my ISP assigns a /56 prefix I have 256 /64 networks that I can assign to my subnets.

Repeat these steps for the other virtual interfaces on eth2.

# When Things Go Wrong

When I was originally playing with this I found that either the ERL or my ISP or both were a bit finicky when it came to making changes wrt IPv6. Here are a few tips for some scenaios I encountered:

- If you're replacing a router supplied by your ISP it may have already requested a prefix, and your ISP may not allow the ERL to request a prefix until that lease has expired. If you're having trouble getting a prefix and nothing seems to be working, you might just need to wait.
- In some cases the ERL seemed to get a bit confused after I changed some IPv6-related settings. Rebooting the ERL always cleared this up.
- I ended up with cases where, after changing the prefixes I was assigning to my LANs, clients ended up with IPv6 addresses in both the old and new prefixes. This is probably harmless, but reconnecting to the network generally straightened things out. If all else fails you can always reboot the client.

# Conclusion

Hopefully you find these instructions helpful, but if your situation turns out to be different from mine there's a lot of helpful information to be found in forums. Just fire up your favorite search engine and you're likely to find others who have already solved your problems.

At this point we've covered everything for a reasonably complete network setup. From here we'll start exploring some more specialized topics, starting with setting up an OpenVPN server on the ERL.
