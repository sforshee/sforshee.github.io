---
layout: single
title: 'Ubiquiti EdgeRouter Lite Setup Part 2: Firewall Setup'
tags:
- edgerouter
- networking
---

In [part 1]({% post_url 2016-03-01-ubiquiti-edgerouter-lite-setup-part-1-the-basics %})
we covered the basics of setting up the ERL for one WAN interface and one LAN 
interface with a basic firewall on the WAN interface. But isolating our internal 
networks against bad actors on the outside is one of the most important 
functions of a router, so let's explore a more robust firewall configuration.

## ACL vs. Zone Based Firewall

The default firewall setup on the ERL (and the only one supported via the web 
client) allows defining firewalls as sets of ACL rules on a per-interface and 
per-direction basis. But the ERL also supports zone-based firewalls, which work 
by dividing your network into zones and matching rules based on source and 
destination zones. For a pretty thorough comparison of ACL versus zone-based 
firewall, I suggest going 
[here](https://www.nnbfn.net/2011/06/per-interface-vs-zone-based-firewall/). The 
basic idea behind a zone-based firewall is as follows:

- You define zones for your network. A common set of zones might be WAN, LAN, 
  and DMZ.
- You assign one or more interfaces to each zone.
- You set up rules which match based on source and destination zones.

While an ACL firewall can be easier to set up for simple networks such as the 
one in this example, a zone-based firewall is conceptually simpler (in my 
opinion at least) and less susceptible to the sorts of mistakes that can open up 
your network to the outside.

## Setting Up a Zone Based Firewall

The approach I've taken is based on
[this article](https://help.ubnt.com/hc/en-us/articles/204952154-EdgeMAX-Zone-Policy-CLI-Example), 
and I recommend reading it before proceeding. The link to the example 
configuration file in that article is broken however, luckily someone was kind 
enough to post a copy 
[here](https://gist.github.com/cimnine/9b9dc854a43702f953ea).

Let's convert the firewall we created in
[part 1]({% post_url 2016-03-01-ubiquiti-edgerouter-lite-setup-part-1-the-basics %})
to a roughly equivalent zone-based firewall. In the end the result will in fact 
be much more robust than the ACL firewall.

#### Define Zones and Allowed Connections

The first step is to determine what our zones are and what connections will be 
permited for each pair of source and destination zones.

In this simple setup we have a _WAN_ zone for the connection to the internet and 
a _LAN_ zone for our internal LAN. We also need to define one more zone, named 
_local_, for connections to the router itself (DHCP, DNS, ssh, etc.).

Three zones gives us six `<source>,<destination>` zone pairs. A reasonable 
initial set of rules for traffic to allow between the zones is:

- **WAN to LAN**: Allow only traffic for established connections.
- **WAN to local**: Allow only traffic for established connections.
- **LAN to WAN**: Drop invalid state packets, allow all other traffic.
- **LAN to local**: Allow traffic for established connections. Also allow new 
  ICMP, DHCP, DNS, ssh, and HTTP/HTTPS connections.
- **local to WAN**: Drop invalid state packets, allow all other traffic.
- **local to LAN**: Drop invalid state packets, allow all other traffic.

#### Create Firewall Rulesets

Now we need to translate the list of permissible traffic into firewall rules.

The article linked to above suggests defining two sets of rules for every 
`<source>,<destination>` pair, using the naming convention 
`<source>-<destination>` for IPv4 and `<source>-<destination>-6` for IPv6. I 
generally follow this suggestion, but it results in quite a few identical 
rulesets, as you can see from the list above. Therefore I define a few 
"standard" rulesets for these rather than having redundant rules. Let's write 
these rulesets first.

The most basic of these is what I call the _allow established, drop invalid_ 
ruleset. For performance reasons these rules form the basis of all rulesets, but 
often they are the only rules needed. The following commands in the CLI will 
create this ruleset for IPv4.

{% highlight console %}
$ configure
# edit firewall name allow-est-drop-inv
# set default-action drop
# set enable-default-log
# set rule 1 action accept
# set rule 1 state established enable
# set rule 1 state related enable
# set rule 2 action drop
# set rule 2 log enable
# set rule 2 state invalid enable
# top
{% endhighlight %}

We need an equivalent rule for IPv6, but here we need to additionally allow ICMP 
connections.

{% highlight console %}
# edit firewall ipv6-name allow-est-drop-inv-6
# set default-action drop
# set enable-default-log
# set rule 1 action accept
# set rule 1 state established enable
# set rule 1 state related enable
# set rule 2 action drop
# set rule 2 log enable
# set rule 2 state invalid enable
# set rule 100 action accept
# set rule 100 protocol ipv6-icmp
# top
{% endhighlight %}

The other repeated case we have is the _allow all connections_ ruleset. To save 
some typing we can start off by making a copy of the `allow-est-drop-invalid` 
rulesets, then simply change the default action to `accept` and disable logging 
for the default rule.

{% highlight console %}
# edit firewall
# copy name allow-est-drop-inv to name allow-all
# set name allow-all default-action accept
# delete name allow-all enable-default-log
# top
{% endhighlight %}

Repeat these steps to create a `allow-all-6` ruleset.

We have only one ruleset left to create now, for connections from the LAN to the 
router. For IPv4 this looks like:

{% highlight console %}
# edit firewall
# copy name allow-est-drop-inv to name lan-local
# edit name lan-local
# set rule 100 action accept
# set rule 100 protocol icmp
# set rule 200 description "Allow HTTP/HTTPS"
# set rule 200 action accept
# set rule 200 destination port 80,443
# set rule 200 protocol tcp
# set rule 600 description "Allow DNS"
# set rule 600 action accept
# set rule 600 destination port 53
# set rule 600 protocol tcp_udp
# set rule 700 description "Allow DHCP"
# set rule 700 action accept
# set rule 700 destination port 67,68
# set rule 700 protocol udp
# set rule 800 description "Allow SSH"
# set rule 800 action accept
# set rule 800 destination port 22
# set rule 800 protocol tcp
# top
{% endhighlight %}

This should be done for IPv6 as well. Keep in mind that `allow-est-drop-inv-6` 
already includes a rule for ICMP.

#### Set Up Zones

Now that we have our rulesets, we need to tell the router about our zones, which 
interfaces belong to each zone, and which rulesets to apply for traffic 
originating from other zones. This information goes in the `zone-policy` stanza 
of the configuration, with one `zone` stanza for each zone of our network.

Let's start by creating a local zone.

{% highlight console %}
# edit zone-policy zone local
{% endhighlight %}

Each zone has a default action, which must be either _drop_ or _reject_.

{% highlight console %}
# set default-action drop
{% endhighlight %}

Next specify which interface(s) are in this zone. Normally this would be a `set 
interface <iface>` command, but the local zone is a bit different:

{% highlight console %}
# set local-zone
{% endhighlight %}

Now we must create `from <zone>` stanzas to specify which rulesets to apply for 
traffic from the specified zone to the local zone.

{% highlight console %}
# set from WAN firewall name allow-est-drop-inv
# set from WAN firewall ipv6-name allow-est-drop-inv-6
# set from LAN firewall name lan-local
# set from LAN firewall ipv6-name lan-local-6
# top
{% endhighlight %}

Repeat this procedure for the LAN and WAN zones.

#### Delete Existing WAN Rules

If you've been following along you will already have some ACL rules applied to 
the WAN interface. It's time to delete those.

{% highlight console %}
# delete interfaces ethernet eth0 firewall
# delete firewall name WAN_IN
# delete firewall name WAN_LOCAL
{% endhighlight %}

#### Apply Changes

Now it's time to cross your fingers and commit the load of changes we just made. 
If you've made any mistakes the CLI will let you know, and you can correct them 
and commit again. Don't forget to save your changes and back them up once 
everything is working!

## Conclusion

Setting up a zone-based firewall on the EdgeRouter is a bit of work, but for me 
the conceptual simplicity and inherent protection against mistakes make it 
worthwhile.

In
[part 3]({% post_url 2016-03-04-ubiquiti-edgerouter-lite-setup-part-3-vlan-setup %})
we'll talk about setting up VLANs.
