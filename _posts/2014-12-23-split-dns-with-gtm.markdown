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

# Create Virtual Server objects using Translation fields in virtual server objects

[F5's Configuring BIG-IP GTM server objects for BIG-IP devices that reside behind a firewall NAT](https://support.f5.com/kb/en-us/solutions/public/14000/700/sol14707.html) covers this fairly well.

The first step is to create the virtual server objects for our application using translation fields.  As you can see below we create two virtual server objects, one for internal (with no translation-address) and one for external (with a translation address).

Finally, note that once you use a translation-address on a big-ip server object, virtual-server discovery is automatically disabled and no longer functions.  iQuery is still used for virtual server monitoring, but you'll need to manually add the virtual server objects to GTM server objects from now on out.  You can read about this limitation here: [K14707: Configuring BIG-IP DNS server objects for BIG-IP devices that reside behind a firewall NAT](https://support.f5.com/csp/article/K14707)

Example configuration:
{% highlight text %}
gtm server bigip-1-dc-east {
    addresses {
        10.1.31.56 {
            device-name bigip-1.dc-east.f5lab.net
        }
    }
    datacenter dc-east
    devices {
        bigip-1.dc-east.f5lab.net {
            addresses {
                10.1.31.56 { }
            }
        }
    }
    monitor bigip 
    product bigip
    virtual-servers {
        app1-bigip1-dc-east-external {
            destination 192.0.2.101:443
            translation-address 10.1.82.101
            translation-port 443
        }
        app1-bigip1-dc-east-internal {
            destination 10.1.82.101:443
        }
    }
}
gtm server bigip-1-dc-west {
    addresses {
        10.2.31.56 {
            device-name bigip-1.dc-west .f5lab.net
        }
    }
    datacenter dc-east
    devices {
        bigip-1.dc-west.f5lab.net {
            addresses {
                10.2.31.56 { }
            }
        }
    }
    monitor bigip 
    product bigip
    virtual-servers {
        app1-bigip1-dc-west-external {
            destination 198.51.100.101:443
            translation-address 10.2.82.101
            translation-port 443
        }
        app1-bigip1-dc-est-internal {
            destination 10.2.82.101:443
        }
    }
}
{% endhighlight %}


# Pool Creation

Next we create our pools, we need an internal pool that contains all the internal virtual server objects and an external pool that contains all the external virtual server objects.

Example configuration:
{% highlight text %}
gtm pool a app1.example.com-gtmpool-external {
    load-balancing-mode least-connections
    members {
        bigip-1-dc-east:app1-bigip1-dc-east-external {
            member-order 0
        }
        bigip-1-dc-west:app1-bigip1-dc-west-external {
            member-order 1
        }
    }
}
gtm pool a app1.example.com-gtmpool-internal {
    load-balancing-mode least-connections
    members {
        bigip-1-dc-east:app1-bigip1-dc-east-internal {
            member-order 0
        }
        bigip-1-dc-west:app1-bigip1-dc-west-internal {
            member-order 1
        }
    }
}

{% endhighlight %}

# Wide IP Configuration

The next step is to create a pool that uses topology based load balancing and uses both our internal and external pools.

Example configuration:
{% highlight text %}
gtm wideip a app1.example.com {
    pool-lb-mode topology
    pools {
        app1.example.com-gtmpool-external {
            order 0
        }
        app1.example.com-gtmpool-internal {
            order 1
        }
    }
}
{% endhighlight %}


# Topology Records

The final step is to create GTM Topology Regions and Records.  This is where we tell GTM how to perform the topology based load balancing we set on our Wide IP object.  Otherwise the GTM wouldn't know which pool should be used for which clients.  The regions help us group like things together so that we can limit the number of topology records we need to create.  In this case, we'll have two regions, one external that contains all non-RFC1918 IP addresses and our external pool.  The other region will be "Internal" which will contain all of our internal RFC1918 address space plus the internal pool.

As you add more applications, you will need to remember to add the internal pool to the internal topology region and the external pool to the external topology region.

Example configuration:
{% highlight text %}
gtm region external-region {
    region-members {
        not subnet 10.0.0.0/8 { }
        not subnet 172.16.0.0/12 { }
        not subnet 192.168.0.0/16 { }
        pool /Common/app1.example.com-gtmpool-external { }
        subnet 0.0.0.0/0 { }
    }
}
gtm region internal-region {
    region-members {
        pool /Common/app1.example.com-gtmpool-internal { }
        subnet 10.0.0.0/8 { }
        subnet 172.16.0.0/12 { }
        subnet 192.168.0.0/16 { }
    }
}
gtm topology ldns: region /Common/external-region server: region /Common/external-region {
    order 1
    score 500
}
gtm topology ldns: region /Common/internal-region server: region /Common/internal-region {
    order 2
    score 500
}
{% endhighlight %}

