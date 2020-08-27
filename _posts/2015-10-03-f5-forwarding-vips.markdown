---
layout: post
title:  "Forwarding VIPs"
date:   2015-07-09 10:54:25
published: false
categories: F5 LTM AFM
---

The F5 is a default deny platform, out of the box it's not going to move any packets from one network to another.  In order to configure it for life as a firewall or router, that's to say moving non-load balanced packets from one network to anotehr, we need to configure forwarding virtual servers.  There are other ways such global NATs and SNATs however, it's best to avoid these as they don't give you very good control or insight into the traffic.


## Virtual Server Types

* FastL4

* Forwarding IP

* Standard


## FastL4 profile tweaks

* Reset on Timeout: 

* Idle Timeout: 


## Helpful F5 Solution Articles

[Overview of BIG-IP virtual server types (11.x)](https://support.f5.com/kb/en-us/solutions/public/14000/100/sol14163.html)

[Overview of TCP connection setup for BIG-IP LTM virtual server types](https://support.f5.com/kb/en-us/solutions/public/8000/000/sol8082.html)


## Configuration example

Here is an example static status page I've used previously.  There is nothing complex, it includes some details so operators can see their various status options.

Here is an example HTTP status monitor on the F5 that works with the above status page.
{% highlight text %}
ltm monitor http /Common/status-http-monitor {
    adaptive disabled
    defaults-from /Common/http
    destination *:*
    interval 5
    ip-dscp 0
    recv "status=(up|online)"
    recv-disable "status=(quiesce|disabled|drain)"
    send "GET /status.html HTTP/1.1\\r\\nHost: status.com\\r\\nConnection: close\\r\\n\\r\\n"
    time-until-up 0
    timeout 16
}
{% endhighlight %}


The same monitor as above, except this one is for HTTPS.
{% highlight text %}
ltm monitor https /Common/status-https-monitor {
    adaptive disabled
    cipherlist DEFAULT:+SHA:+3DES:+kEDH
    compatibility enabled
    defaults-from /Common/https
    destination *:*
    interval 5
    ip-dscp 0
    recv "status=(up|online)"
    recv-disable "status=(quiesce|disabled|drain)"
    send "GET /status.html HTTP/1.1\\\\r\\\\nHost: status.com\\\\r\\\\nConnection: close\\\\r\\\\n\\\\r\\\\n"
    time-until-up 0
    timeout 16
}
{% endhighlight %}
