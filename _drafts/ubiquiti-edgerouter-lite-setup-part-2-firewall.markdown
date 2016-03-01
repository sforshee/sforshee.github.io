---
layout: post
title: 'Ubiquiti EdgeRouter Lite Setup Part 2: Firewall'
date: '2016-03-01T10:48:15.688-06:00'
author: Seth Forshee
tags: 
modified_time: '2016-03-01T10:48:15.688-06:00'
---

One of the most important functions of an edge router is to protect the networks behind it from bad actors on the outside. Thus this was the first thing I wanted to tackle after getting the basic setup nailed down.

# ACL vs. Zone Based Firewall

The default firewall setup on the ERL (and the only one supported via the web client) defines rules as sets of ACLs on a per-interface basis. The ERL supports another method of defining rules, using zones to create a zone-based firewall. For a pretty thorough comparison of ACL versus zone-based firewall, I suggest going here. The basic idea behind a zone-based firewall is as follows:
You define zones for your network. A common set of zones might be WAN, LAN, and DMZ.
You assign one or more interfaces to each zone.
You set up rules which match based on source and destination zones.
While an ACL firewall can be easier to set up for simple networks such as mine, a zone-based firewall is conceptually simpler (in my opinion at least) and less susceptible to the sorts of mistakes that can open up your network to the outside. Thus I opted for a zone-based firewall for my network.

# Setting Up the Zone Based Firewall

I won't go into a lot of detail here, so I suggest going [here](https://help.ubnt.com/hc/en-us/articles/204952154-EdgeMAX-Zone-Policy-CLI-Example) for a more detailed explanation. The link to the example configuration file in that article is broken however, luckily someone was kind enough to post a copy [here](https://gist.github.com/cimnine/9b9dc854a43702f953ea).

My network currently has three zones, WAN, home LAN, and office LAN, which I've creatively named _wan_, _homelan_, and _officelan_. There's also a fourth zone which is always present named _local_, which refers to frames origination from or directed towards the ERL itself.

For each pair of zones, four sets of rules are required: two each for IPv4 and IPv6, with one of these sets being for traffic from Zone A to Zone B and the other for traffic from Zone B to Zone A. In my case there are four zones, which translates to six pairs for a total of twenty-four rules. This seems like a lot, but you're likely to end up with several sets which are identical or nearly so, making it possible to speed up configuration through liberal use of the copy command.

All of my rule sets start with the same two rules. The first is a rule to accept packets for established connections, and the second is a rule to drop and log invalid state packets. Since the majority of traffic on a network is for established connections, placing this rule first avoids the need to process any other rules for most packets and thus reduces CPU load.

Each ruleset has a default action to perform if no rule in the set matches. In a zone-based firewall the default action must be `drop` or `reject`. The `enable-default-log` option enables logging for traffic hitting the default action.
