---
layout: post
title:  "Source MAC based routing"
date:   2018-06-04 10:54:25
categories: F5 LTM AFM
---

Several network/firwall vendors offer a feature that overrides the routing table to return traffic to the MAC address that initiated the connection.  The idea here is to store the MAC address that the SYN packet came from in the connection table, then when sending return traffic for that flow, to send it to the MAC address that the SYN came from.  This overrides the destination based routing table and helps prevent asymmetrical routing.  Depending upon your point of view, this is either very helpful or very confusing.

This post simply catalogs the names various vendors call this feature.  There is no standard name, as such it can be very confusing when talking with different vendors.



| Vendor      | Feature Name           | Documentation Link |
| ----------- | ---------------------- | ------------------ | 
| F5 Networks | Auto Last Hop (ALH)    | [K13876: Overview of the Auto Last Hop setting](https://support.f5.com/csp/article/K13876) |
| Palo Alto   | Symmetric Return       | [How to Configure Symmetric Return](https://live.paloaltonetworks.com/t5/Configuration-Articles/How-to-Configure-Symmetric-Return/ta-p/59374) |
| BlueCoat    | Return-To-Sender (RTS) | [RTS CLI Reference](https://origin-symwisedownload.symantec.com/resources/webguides/cacheflow/3x/3_4/webguide/Content/CLI/config_return-to-send.htm) |
| Citrix NetScaler | MAC-based forwarding (MBF) | [Configuring MAC-Based Forwarding](https://docs.citrix.com/ja-jp/netscaler/11/networking/interfaces/configuring-mac-based-forwarding.html) |
| Juniper ScreenOS | Flow reverse-route MAC Cache | [Behavior of 'set flow' commands in asymmetric routing scenario](https://kb.juniper.net/InfoCenter/index?page=content&id=KB19924) |
| Check Point | unsure                 | I'm pretty sure this feature exists for checkpoint, however I can't find docs/name. |
