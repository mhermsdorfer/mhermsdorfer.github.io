---
layout: post
title:  "Server status pages for F5 monitoring"
date:   2015-07-09 10:54:25
categories: F5 LTM
---

Automation is critical in our industry, however many companies opt for a less technically advanced self service play.  One trivially easy way to get some self-service with the F5 platform is though the use of status pages for pool member monitoring.

In this scenario the application or server administrators setup a status page, that status page can be as complex as the application or server administrators like.  On the higher end it could be a jsp page that checks the status of various other services and databases reporting overall application health.  On the lower end it can be a simple static html page.

By modifying the output of the status page the application administrators can indicate to the F5 if traffic should be sent to the server.  This allows server & application team members to perform maintenance on their servers without involvement from the F5 operators to disable pool member.

For more details on monitor regex strings: [Using regular expressions in a health monitor Receive String](https://support.f5.com/kb/en-us/solutions/public/5000/900/sol5917.html)

For more details on monitor disable strings: [Using the Receive Disable String advanced configuration setting](https://support.f5.com/kb/en-us/solutions/public/12000/800/sol12818.html)


## Configuration example

Here is an example static status page I've used previously.  There is nothing complex, it includes some details so operators can see their various status options.
{% highlight html %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8">
    <title> Application Status Page </title>
  </head>
  <body>
    <h1> Application Status Page </h1>
    <h1> status=up </h1>

    <p> Modify the current server status on header above, leaving no space between status= and expected status.  The server status can be set to various options as follows:
    <ul>
        <li> up or online:     Server is normal and eligable for all traffic. </li>
        <li> quiesce or drain: Server is going offline and eligable only for existing sessions. </li>
        <li> down or offline:  Server is down and not eligable any traffic. </li>
    </ul>
    </p>
  </body>
</html>
{% endhighlight %}

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
