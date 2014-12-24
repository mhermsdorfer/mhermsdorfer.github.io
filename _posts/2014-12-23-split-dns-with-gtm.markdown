---
layout: post
title:  "Split DNS with GTM"
date:   2014-12-23 15:32:34
categories: F5 GTM
---

NAT adds complexity to everything, I avoid it whenever possible, however it's a necessary evil of our IPv4 world.  A common configuration that needs to be addressed in GTM deployments is NAT and split DNS.  Often the GTM will need to respond to internal queries with a private IP address while external queries get a public IP address.  This is typically done inside GTM with topology records and possibly a combination of internal and external pools containing different virtual server objects.

# Avoid Split DNS when possible
I strongly advocate that administrators simply hand out external public IP addresses for anything that is externally accessible.  This allows you to avoid the entire problem space around split DNS and the additional configuration overhead and complexity it brings to your environment.  Many will argue about the inefficiency of configurations that use [Hairpin nat](http://tools.ietf.org/html/rfc4787#section-6), the arguments typically revolve around hairpinning putting additional load on the NAT device & causing inefficient traffic flows on the network.  However, if there are capacity concerns about the NAT device, then what's the expectation for when a syn-flood or other DOS attack hits the nat box from the Internet?  Additionally, complexity is every IT operations worst enemy, if you can reduce configuration complexity at the cost of a bigger firewall that's a huge win.  Firewalls are cheap compared to outages and additional labor overhead caused by complexity.

If however, hairpin nat is not an option for you, continue on.  [F5's Achieving split DNS behavior through BIG-IP GTM wide IPs](https://support.f5.com/kb/en-us/solutions/public/14000/700/sol14707.html) walks you though the steps to configure split dns on GTM.  However it lacks configuration examples, this post is intended to clarify configurations where needed and to provide some example configuration snippets.

# Sorry work in progress...

# Using Translation fields in virtual server objects

[F5's Configuring BIG-IP GTM server objects for BIG-IP devices that reside behind a firewall NAT](https://support.f5.com/kb/en-us/solutions/public/14000/700/sol14707.html) covers this fairly well.

# Pool Creation

# Topology Records

# Gluing it all together

