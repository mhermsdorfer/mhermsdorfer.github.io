---
layout: post
title:  "GTM Translation addresses for Generic Host objects"
date:   2014-12-23 17:54:25
categories: F5 GTM
---

A reading of the [F5's Configuring BIG-IP GTM server objects for BIG-IP devices that reside behind a firewall NAT](https://support.f5.com/kb/en-us/solutions/public/14000/700/sol14707.html) might lead one to believe that translation addresses for non BIG-IP server objects would have a similar effect as they do for BIG-IP objects.  However, translation addresses are ONLY supported by BIG-IP objects when using iQuery or the "bigip" health monitor in GTM.  The translation-address and translation-port configuration options have no impact on generic host or any other non-iquery monitored server or virtual server object.

This can be frustrating, the user experience for this configuration is honestly pretty awful in the F5 web interface.  One would think that if translation addresses did nothing for non-iquery monitored server objects then the GUI & configuration objects would not allow their configuration which only causes further confusion.

For an overview of split DNS configuration on the GTM see: [Spit DNS with GTM]({% post_url 2014-12-23-split-dns-with-gtm %})

## Broken Configuration

The configuration below does not work as one might expect.  

One might expect the GTM to issue the following health monitors:

* gateway\_icmp to 172.20.1.31 the server object address.

* tcp\_half\_open to 172.20.1.31:80 for the "GenericServer01-internal" virtual server object.

* tcp\_half\_open to 172.20.1.31:80 for the "GenericServer01-external" virtual server object, due to the inclusion of the translation-address & translation-port.

{% highlight text %}
gtm server /Common/GenericServer01 {
    addresses {
        172.20.1.31 {
            device-name GenericServer01
        }
    }
    datacenter /Common/DataCenter01
    monitor /Common/gateway_icmp
    virtual-servers {
        GenericServer01-external {
            destination 192.0.2.31:80
            monitor /Common/tcp_half_open
            translation-address 172.20.1.31
        }
        GenericServer01-internal {
            destination 172.20.1.31:80
            monitor /Common/tcp_half_open
        }
    }
}
{% endhighlight %}

However what actually happens is as follows:

* gateway\_icmp to 172.20.1.31 the server object address.

* tcp\_half\_open to 172.20.1.31:80 for the "GenericServer01-internal" virtual server object.

* tcp\_half\_open to 192.0.2.31:80 for the "GenericServer01-external" virtual server object, because translation addresses are ignored for non-iquery or bigip monitored objects.

This often leaves the public or external virtual server object marked down as many organizations do not configure their firewalls for hairpin NAT.


## Working Configuration

First, a few considerations, I'd argue the best possible fix is to setup public IP addresses on your LTMs and let them act as your primary edge device.  This puts all the management and configuration for your public presence on a highly secure, highly scalable, best of breed system.

Failing using F5 LTM for all external IPs the next best fix is to setup or enable hairpin NAT on the firewall or router performing the external NAT.  In an ideal world the F5 would monitor via the public IP address so that if the device performing NAT goes down the F5 correctly marks the system as down.

If all that is not possible and you must monitor the private address while handing out the public address then continue reading.  The way to configure a generic host server/virtual server object for NAT translation is to use alias addresses a health monitor assigned to the public virtual server object.  This should sound familiar from LTM, an alias address on a health monitor allows us to apply that health check to any arbitrary virtual server or server object in the GTM, while actually having the health check go to a different address/port.

The disadvantage of this configuration is that you'll end up creating and maintaining a lot of very specific GTM monitors with address aliases for all of your NATed generic objects.

{% highlight text %}
gtm monitor tcp-half-open /Common/GenericServer01-tcp-half-open {
    defaults-from /Common/tcp_half_open
    destination 172.20.1.31:80
    interval 30
    probe-attempts 3
    probe-interval 1
    probe-timeout 5
    timeout 120
}
gtm server /Common/GenericServer01 {
    addresses {
        172.20.1.31 {
            device-name GenericServer01
        }
    }
    datacenter /Common/DataCenter01
    monitor /Common/gateway_icmp
    virtual-servers {
        GenericServer01-external {
            destination 192.0.2.31:80
            monitor /Common/GenericServer01-tcp-half-open
        }
        GenericServer01-internal {
            destination 172.20.1.31:80
            monitor /Common/tcp_half_open
        }
    }
}
{% endhighlight %}

* Note: Unfortunately, GTM monitors require a destination IP and port if a specific IP is used.  Ideally we could configure a monitor with a specific destination address and a wildcard port.

