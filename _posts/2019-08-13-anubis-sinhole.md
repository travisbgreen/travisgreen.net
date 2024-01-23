---
layout: post
title: "About the Anubis Sinkhole"
date: "2019-08-13 12:54"
---

A question that frequently comes up is: "What is the Anubis Networks Sinkhole, and what does it mean when I see IDS alerts for it?". 

## What is a Sinkhole?
When you type a domain name like "google.com" into your browser's address bar, your computer generates a DNS request to turn that name into an IP address. The same process happens when your computer is infected with a malicious program, and that program wants to communicate with the control server by domain name. A sinkhole is where a defender has successfully taken contol of the the malicious domain name, and redirected it to a benign server that gathers information about the infected system. The Anubis Networks Sinkhole is one such server. 

Consider the following DNS request:
![wireshark screenshot](/assets/img/anubis1.png)

First we see our infected host attempting to resolve eksyghskgsbakrys[.]com, and then we see the response points at 195.22.26.248. 

## AnubisNetworks Sinkhole

A search of this IP address on AlienVault OTX, VirusTotal, or Google returns no clear result as to the identity of this sinkhole.

<img align="right" src="/assets/img/anubis2.png" />

A bit of sleuthing reveals references to the sinkhole, like this [known sinkhole list gist](https://github.com/stamparm/maltrail/blob/master/trails/static/malware/sinkhole_anubis.txt), or a forum user sharing a [response from AnubisNetworks](https://www.alienvault.com/forums/discussion/10634/multiple-alarms-for-sinkhole-anubis-this-week), and indeed João Gouveia, CTO of AnubisNetworks [weighing in on the emerging-sigs mailing list](https://lists.emergingthreats.net/pipermail/emerging-sigs/2017-July/028240.html). In this email to the list he also reveals that not every host sinkholed at this _particular_ IP address is malicious, whereas connections to other IP addresses owned by Anubis Networks sinkhole could be considered 100% malicious. Furthermore, he reveals that the sinkhole sets a cookie (snkz=), as well as deploys custom HTML/JS to colect browser metadata. You can find more information from João in an [interview with BitSight about the sinkhole infrastructure](https://info.bitsight.com/bitsight-risk-review-episode-4).

<br>

## Sinkhole IDS Events
The following alerts can be triggered when a system attempts to interact with a sinkholed system or domain:
```
ET TROJAN Connection to AnubisNetworks Sinkhole IP (Possible Infected Host)
ET TROJAN AnubisNetworks Sinkhole HTTP Response - 195.22.26.192/26
ET TROJAN AnubisNetworks Sinkhole TCP Connection
ET TROJAN AnubisNetworks Sinkhole UDP Connection
ET TROJAN AnubisNetworks Sinkhole SSL Cert lolcat - specific IPs
ET TROJAN DNS Reply Sinkhole - Anubis - 195.22.26.192/26
ET TROJAN Possible Compromised Host AnubisNetworks Sinkhole Cookie Value Snkz
```

These alerts can be tricky to investigate for a couple of reasons. Firstly, the system that generates the DNS request can be the infected system, but it can also be a DNS server making the request on behalf of the infected system. Secondly, some rules fire on the query response, so an alert may have 8.8.8.8, 8.8.4.4, or another popular public resolver as the src IP. You may also find the src/dst IP is your DNS server or perhaps even a domain controller, so you'll need to investigate the systems that use your DNS server.

## In closing
Anubis Networks have been a security vendor for a long time, and experience has shown that the domains sinkholed by them are malicious, and always worthy of further investigation. There are other sinkholes in operation, and overall they provide an excellent defense and high fidelity alerts to blue teams.

If you think these alerts could be better placed or there is a less complicated way to detect these infections, please drop me a line [@travisbgreen](https://twitter.com/travisbgreen).















