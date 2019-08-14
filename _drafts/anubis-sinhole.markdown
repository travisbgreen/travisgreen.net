---
layout: "post"
title: "About the Anubis Sinkhole"
date: "2019-01-07 12:54"
---

A question that pops up every once in a while is "What is the Anubis Networks Sinkhole, and what does it mean when I get IDS events for it?". 

## What is a Sinkhole?
When you type a domain name like "google.com" into your browser's address bar, your system generates a DNS request to turn that domain name into a usable IP address. The same process takes place if your computer is infected with a malicious program, and that program wants to communicate with the control server by domain name. A sinkhole is where a defender has successfully taken contol of the the malicious domain name, and redirected it to a machine that allows the connection and gathers information about the infected host, effectively disabling the malicious domain as well as providing info about where the victims are located, number of victims, rate of spread, etc.

Consider the following DNS request:
![wireshark screenshot](/assets/img/anubis1.png)

First we see our infected host attempting to resolve eksyghskgsbakrys[.]com, and then we see the response pointing at 195.22.26.248. 

## AnubisNetworks Sinkhole

A search of this IP address on AlienVault OTX, VirusTotal, or Google returns no clear result as to the identity of this sinkhole.

<img align="right" src="/assets/img/anubis2.png" />

A bit of sleuthing reveals references to the sinkhole on several mailing lists, like a [sinkhole list gist](https://github.com/stamparm/maltrail/blob/master/trails/static/malware/sinkhole_anubis.txt), a forum user sharing a [response from AnubisNetworks](https://www.alienvault.com/forums/discussion/10634/multiple-alarms-for-sinkhole-anubis-this-week), and indeed João Gouveia, CTO of AnubisNetworks [weighing in on the emerging-sigs mailing list](https://lists.emergingthreats.net/pipermail/emerging-sigs/2017-July/028240.html). In this email to the list he also reveals that not every host sinkholed at this _particular_ IP address is malicious, whereas connections to other IP addresses controlled by AnubisNetworks could be considered 100% malicious. Furthermore, he reveals that the sinkhole sets a cookie (snkz=), as well as deploys custom HTML/JS to colect browser metadata. You can find more information from João in an [interview with BitSight about the sinkhole infrastructure](https://info.bitsight.com/bitsight-risk-review-episode-4).










<br><br><br>

## Sinkhole IDS Events
The following signatures 
```
ET TROJAN Connection to AnubisNetworks Sinkhole IP (Possible Infected Host)
ET TROJAN AnubisNetworks Sinkhole HTTP Response - 195.22.26.192/26
ET TROJAN AnubisNetworks Sinkhole TCP Connection
ET TROJAN AnubisNetworks Sinkhole UDP Connection
ET TROJAN AnubisNetworks Sinkhole SSL Cert lolcat - specific IPs
ET TROJAN DNS Reply Sinkhole - Anubis - 195.22.26.192/26
ET TROJAN Possible Compromised Host AnubisNetworks Sinkhole Cookie Value Snkz
```





## notes
- check for new certs? (did that, all default junk)
- stackoverflow question
- OTX
- refs
  - https://lists.emergingthreats.net/pipermail/emerging-sigs/2017-July/028241.html
  - http://jshlbrd.github.io/blog/2015/08/18/sinkholes/
  - https://augen.emergingthreatspro.com/admin/sidclear.php?sidhist=2018141

