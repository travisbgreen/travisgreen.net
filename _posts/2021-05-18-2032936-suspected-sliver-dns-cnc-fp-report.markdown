---
layout: post
title: 2032936 - Suspected Sliver DNS CnC FP Report
date: '2021-05-18 13:32'
tags:
  - fp
  - emergingthreats
---
## Background

It appears that SMTP MTAs or SMTP spam gateways utilizing [DKIM](https://support.dnsimple.com/articles/dkim-record/), make many `_domainkey` TXT requests and occasionally generates a false positive alert (FP) on rule 2032936. The source of these alerts is this bit of inexact rule logic, which sometimes matches legitimate requests `content:"_"; depth:1; content:"_domainkey"; distance:8;`  

For example, this non-malicious request generates a FP:  
`_conversica._domainkey.servicios.redactado.com`  

Also, this example request would also generate a FP since distance:8; is not restricted to any bytes by the "within" keyword:  
`_asdfasdfasdfasdfasdfasdfasdfasdf._domain.example.com`  

To contrast, this malicious silver CnC traffic (TP) looks similar and is the target of the logic in our original rule:
![\`\_f7jxhi.unhappy_bronco.\_domainkey.1.kaotix.xyz\`](/images/2021/05/20210518.1.png)

## Analysis

We can refer to the silver source code (how convenient!) [at line 66](https://github.com/BishopFox/sliver/blob/672c0e29d07313fcc3d093d2c6b742659e574e07/implant/sliver/transports/udp-dns.go#L66) to see that this part of the DNS TXT request is a nonce, starting with '\_' and containing 6 random characters from the set of characters stored in the variable named [dnsCharSet](https://github.com/BishopFox/sliver/blob/672c0e29d07313fcc3d093d2c6b742659e574e07/implant/sliver/transports/udp-dns.go#L75). 
![silver source code snippet](/images/2021/05/20210518.4.png)  

Armed with this information it seems we can update the rule logic to be more specific to this pattern and eliminate the FP:  
`content:"_"; depth:1; content:"_domainkey"; distance:8;`  

would become:  
`content:"_"; depth:1; pcre:"/^[a-z0-9_]{6}[^a-z0-9_]/R"; content:"_domainkey"; distance:8;`  

We've added a PCRE with a relative ("/R") modifier to ensure the next 6 characters are the ones from variable dnsCharSet. With these changes we should dramatically reduce the number of false positives (but not entirely eliminate them, for example: `_123456._domain.example.com`)

## Modification

original rule:

>```alert dns any any -> any any (msg:"ET TROJAN Suspected Sliver DNS CnC (original)"; content:"|00 10 00 01|"; isdataat:!1,relative; content:"|00 10 00 01|"; isdataat:!1,relative; dns_query; content:"_"; depth:1; content:"_domainkey"; distance:8; fast_pattern; reference:url,github.com/BishopFox/sliver; classtype:trojan-activity; sid:2032936; rev:1;)```

which becomes:

>```alert dns any any -> any any (msg:"ET TROJAN Suspected Sliver DNS CnC (modified)"; content:"|00 10 00 01|"; isdataat:!1,relative; content:"|00 10 00 01|"; isdataat:!1,relative; dns_query; content:"_"; depth:1; pcre:"/^[a-z0-9_]{6}[^a-z0-9_]/R"; content:"_domainkey"; distance:8; fast_pattern; reference:url,github.com/BishopFox/sliver; classtype:trojan-activity; sid:9999999; rev:2;)```

## Verify

So, now we verify that our fix is good (if we have the luxury of pcap). Truth be told, I had a few small mistakes to fix and that is quite common for me.
![fast log snippet](/images/2021/05/20210518.3.png)

## Conclusion

The last thing to do is to submit to the mailing list and enjoy that sweet sweet open source community karma. 

Please [DM me](https://twitter.com/travisbgreen/) with any rule analysis or FP/TP analysis requests.