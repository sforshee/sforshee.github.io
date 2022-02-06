---
layout: single
title: 'Ubiquiti EdgeRouter Lite Setup Part 6: Odds and Ends'
tags:
- edgerouter
- networking
---

In this final post on configuring the ERL we'll cover some miscellaneous items
that don't warrant a full post on their own. See
[part 1]({% post_url 2016-03-01-ubiquiti-edgerouter-lite-setup-part-1-the-basics %})
for a list of all posts in this series.

## Dynamic DNS

The ERL can automatically update your IP address with a number of dynamic DNS
providers. I'm using the following configuration with DNS-O-Matic:

{% highlight console %}
$ configure
# show service dns dynamic 
 interface eth0.2 {
     service dyndns {
         host-name all.dnsomatic.com
         login <username>
         password <password>
         server updates.dnsomatic.com
     }
 }
{% endhighlight %}

## Static DHCP Mappings

Sometimes it's useful to assign known IPv4 addresses to specific machines. This
example demonstrates how to set this up.

{% highlight console %}
$ configure
# edit service dhcp-server shared-network-name <name> subnet <subnet>
# set static-mapping <name> mac-address <mac-address>
# set static-mapping <name> ip-address <ip-address>
{% endhighlight %}

## Changing the Host Name

The default host name for the ERL is `ubnt`, but this can be easily changed.

{% highlight console %}
$ configure
# set system host-name <name>
{% endhighlight %}

## Google Fiber

According to the information available on various forums and blogs, in order to
get maximum performance when using the ERL in place of the Google Fiber network
box traffic in and out of the WAN interface must be tagged for VLAN 2 and have
the VLAN priority set to 3, and hardware offloading should be enabled. I've
never tried using the ERL with any other configuration, but I can confirm that
I'm seeing [good speeds](http://speedtest.dslreports.com/speedtest/3203290)
with this configuration.

To apply these settings, delete most other settings from the WAN interface
(besides `duplex auto` and `speed auto`) and do the following:

{% highlight console %}
$ configure
# edit interfaces ethernet <iface> vif 2
# set description "Google Fiber"
# set address dhcp
# set egress-qos 0:3
# top
# set system offload ipv4 forwarding enable
# set system offload ipv4 vlan enable
# set system offload ipv6 forwarding enable
# set system offload ipv6 vlan enable
{% endhighlight %}

You'll also need to apply your IPv6 prefix delegation settings to the virtual
interface.

## Staying Up to Date

It's a good idea to keep your router firmware up to date to ensure that you
have all the latest security fixes. I haven't found any way to receive emails
or similar when new firmware is made available, but firmware announcements are
posted to the
[EdgeMax Updates Blog](http://community.ubnt.com/t5/EdgeMAX-Updates-Blog/bg-p/Blog_EdgeMAX).

## Conclusion

In my opinion Ubiquiti's EdgeRouter Lite is a very nice step up from the
typical home or small office router for those who want more control over their
networks. These posts have described configuration of some of the most common
features needed for a basic home or small office network.
