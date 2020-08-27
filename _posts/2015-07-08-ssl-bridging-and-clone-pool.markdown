---
layout: post
title:  "LTM clone pools & SSL Bridging"
date:   2015-07-08 10:54:25
categories: F5 LTM
---

Security teams place intrusion detection devices (IDS) on the network in order to monitor network traffic for signs of intrusion.  Often this is accomplished with network taps and/or switch monitor ports, such setups provide the security team all packets on the network.  As companies migrate all sites to use SSL payloads are encrypted, and attack traffic looks the same as legitimate traffic.  What the security teams really need are the unencrypted traffic flows.

F5 devices are high performance SSL platforms and often act as a central decryption/encryption point for applications.  As such, it makes for a great place to strip off ssl and send the decrypted flow to an IDS appliance.  Fortunately F5 makes such configurations easy with the clone pool feature.  Clone pools can be configured on a per virtual server basis, they copy the traffic from the proxies client-side or server-side.  This works great when you're not running any SSL or the F5 is performing SSL offload.  In the of ssl offload you simply assign a clone-pool to the server-side of the proxy, and clear-text traffic is sent to the IDS systems.

However, if the F5 is performing SSL bridging things get slightly more complex.  This is due to how the F5's proxy chain or HUD chain works.  The clone pool action is performed very early on the client-side and very late on the server-side, in-fact it happens prior to ssl decryption on the client-side of the proxy and after ssl encryption on the server-side.  Because of this, we need to setup two VIPs, the first VIP will terminate ssl from the client, then send the cleartext http to another vip.  The second vip will then re-encrypt ssl and send the http request to servers.  We can then place a clone-pool on the server-side of the first vip and mirror unencrypted to our IDS system.

For more details on clone pools see: [F5's Configuring Configuring the BIG-IP system to send traffic to an intrusion detection system ](https://support.f5.com/kb/en-us/solutions/public/13000/300/sol13392.html#clone)


## Configuration example

Let's walk though how to create a configuration using VIP targeting VIP for cloning cleartext traffic while the F5 performs SSL bridging.


First create pools, one for the server-side traffic, the other for our IDS system that we'll clone traffic to:
{% highlight text %}
ltm pool /Common/sitefoo.bar.com-clone-pool {
    members {
        /Common/ids-system:12345 {
            address 10.2.200.200
        }
    }
    monitor /Common/tcp_half_open
}
ltm pool /Common/sitefoo.bar.com-server-pool {
    members {
        /Common/venkman:443 {
            address 10.2.1.51
        }
    }
    monitor /Common/tcp_half_open
}
{% endhighlight %}

Next let's create a server-side VIP.  This VIP will use a pool with the back-end web servers, and will have server-ssl but no client-ssl.  It will accept clear text http on it's client-side and send ssl encrypted http to the servers on it's server-side.

NOTE: You should secure this virtual server such that it can-not be accessed by end-users, you can do this with AFM rules, or by using a non-routable destination address.
{% highlight text %}
ltm virtual /Common/sitefoo.bar.com-server-side-vip {
    destination /Common/10.1.50.160:8443
    ip-protocol tcp
    mask 255.255.255.255
    pool /Common/sitefoo.bar.com-server-pool
    profiles {
        /Common/http { }
        /Common/serverssl-insecure-compatible {
            context serverside
        }
        /Common/tcp { }
    }
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}
{% endhighlight %}


Next let's create an irule that targets the server-side vip from the client-side vip:
{% highlight text %}
ltm rule /Common/sitefoo.bar.com-vip-target-vip-ir {
    when HTTP_REQUEST {
    virtual sitefoo.bar.com-server-side-vip
}
}
{% endhighlight %}

Finally, create a client-side VIP.  This VIP has a clientssl on the client-side of the proxy that terminates ssl, the clone-pool on the server-side of the proxy and finally the iRule that tells heh F5 to send traffic to the other virtual server.  Notably it does not have a default pool.
{% highlight text %}
ltm virtual /Common/sitefoo.bar.com-client-side-vip {
    clone-pools {
        /Common/sitefoo.bar.com-clone-pool {
            context serverside
        }
    }
    destination /Common/10.1.50.160:443
    ip-protocol tcp
    mask 255.255.255.255
    profiles {
        /Common/clientssl {
            context clientside
        }
        /Common/http { }
        /Common/tcp { }
    }
    rules {
        /Common/sitefoo.bar.com-vip-target-vip-ir
    }
    source 0.0.0.0/0
    translate-address enabled
    translate-port enabled
}
{% endhighlight %}


