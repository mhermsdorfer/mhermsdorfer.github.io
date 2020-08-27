---
layout: post
title:  "Force F5 to send Gratuitous ARP"
date:   2020-08-27 10:54:25
categories: F5 LTM
---

To force the F5 BIG-IP to send Gratuitous ARP (GARP) you can of course failover, this will issue a GARP frame for all floating addresses.  However, in some cases you may want to tell the BIG-IP to send GARPs for non-floating self-ip addresses as well and/or force a Gratuitous ARP without needing to perform a failover.

The following set of commands are useful to force a Gratuitous ARP/GARP for all IPs:
{% highlight text %}
tmsh create net vlan temp_garp_vlan
tmsh create net self temp_garp_self address 192.0.2.254/32 vlan temp_garp_vlan
tmsh delete net self temp_garp_self
tmsh delete net vlan temp_garp_vlan
{% endhighlight %}

Interestingly, It’s the act of deleting any Self IP that causes the BIG-IP to GARP out all other addresses.

The vlan & address itself are irrelevant, of course. Since the VLAN’s not tied to an interface, or on the hypervisor, no GARP for that specific IP should leave the box. I’d still pick something unrelated to the real networks though, so as not to cause a hiccup on traffic currently traversing the box.

Note, that this example uses 192.0.0.254/32, this is a reserved for documentation per [RFC5737](https://tools.ietf.org/html/rfc5737#section-3) so you can be reasonably sure it's not in use.
