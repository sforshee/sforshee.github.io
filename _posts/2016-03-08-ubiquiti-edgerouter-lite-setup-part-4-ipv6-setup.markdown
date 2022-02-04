---
layout: post
title: 'Ubiquiti EdgeRouter Lite Setup Part 4: IPv6 Setup'
author: Seth Forshee
tags: 
---

At this point we've done
[basic setup]({% post_url 2016-03-01-ubiquiti-edgerouter-lite-setup-part-1-the-basics %})
of the ERL, configured a
[zone-based firewall]({% post_url 2016-03-02-ubiquiti-edgerouter-lite-setup-part-2-firewall-setup %})
and set up flexible network partitioning using
[VLANs]({% post_url 2016-03-04-ubiquiti-edgerouter-lite-setup-part-3-vlan-setup %}).
But we're still only supplying an IPv4 network to clients. In this post we're
going to learn about deploying IPv6.

To my knowledge DHCPv6 prefix delegation is widely supported among ISPs that
provide native IPv6. That's what I describe here; check with your ISP for
specifics about using IPv6 on their network.

If your ISP doesn't support IPv6 at all a good option is a 6to4 service like
[tunnelbroker.net](https://tunnelbroker.net). I won't describe setting that up
here, but instructions can be found
[elsewhere](https://help.ubnt.com/hc/en-us/articles/204976104-EdgeMAX-IPv6-Tunnel).

# DHCPv6 and Prefix Delegation

[RFC 3633](https://tools.ietf.org/html/rfc3633) defines a mechanism by which
[DHCPv6](https://en.wikipedia.org/wiki/DHCPv6) can be used to delegate a
network address prefix to a network. This is known as
[prefix delegation](https://en.wikipedia.org/wiki/Prefix_delegation). The
router can then assign addresses to clients within the network using either
DHCPv6 or
[stateless address autoconfiguraton](https://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration)
(SLAAC). With SLAAC the router advertises a prefix to clients, and clients pick
their own address within that network. This example will use SLAAC.

# WAN Setup

The first step in configuring the ERL is to set up the WAN interface to request
a prefix via DHCPv6. Our network is divided into multiple LANs, so we'll divide
up the assigned prefix into smaller networks that can be advertised on each
lan, assigning the ::1 address in each network to the virtual interface in the
ERL.

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

The prefix length is determined by your ISP, so you will need to check with
them to determine the correct value.

# LAN Setup

_Update_: There is a much simpler way to get router advertisements on your LANs 
than what I described in my original instructions. Simply do this:

{% highlight console %}
# edit interfaces ethernet eth2 vif 1
# set ipv6 address autoconf
# set ipv6 dup-addr-detect-transmits 1
{% endhighlight %}

I can't recall now why I didn't do it this way originally. Possibly it's a 
legacy of having previously used a 6to4 tunnel. Anyway, I'd suggest you try this 
first, and refer to the original instructions (below) only if you need to 
customize the router advertisements.

---

**Original Instructions**:

Now each vif must be configured to advertise its assigned IPv6 prefix to
clients.

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

Again here the size of your prefix may depend on the size of the prefix
assigned by your ISP. Since my ISP assigns a /56 prefix I have 256 /64 networks
that I can assign to my subnets.

Repeat these steps for the other virtual interfaces on eth2.

# Potential Problems

When I was originally playing with this I found that either the ERL or 
my ISP or both were a bit finicky when it came to making IPV6 changes on 
the ERL. Here are a few tips for some scenaios I encountered:

- If you're replacing a router supplied by your ISP it may have already 
  requested a prefix, and your ISP may not allow the ERL to request a 
  prefix until that lease has expired. If you're having trouble getting 
  a prefix and nothing seems to be working, you might just need to wait.
- In some cases the ERL seemed to get a bit confused after I changed 
  some IPv6-related settings. Rebooting the ERL always cleared this up.
- I ended up with cases where, after changing the prefixes I was assigning to
  my LANs, clients ended up with IPv6 addresses in both the old and new
  prefixes. Disconnecting then reconnecting clients to the network generally
  straightened things out. If all else fails you can always reboot the
  client.

# Conclusion

Hopefully you find these instructions helpful, but if your situation 
turns out to be different from mine there's a lot of information to be 
found in forums. Just fire up your favorite search engine and you're 
likely to find others who have already solved your problems.

At this point we've covered everything for a reasonably complete network 
setup. From here we'll start exploring some more specialized topics, 
starting with setting up an
[OpenVPN server]({% post_url 2016-03-16-ubiquiti-edgerouter-lite-setup-part-5-openvpn-setup %})
on the ERL.
