---
layout: post
title: 2032936 - Suspected Sliver DNS CnC FP Report
date: '2021-05-18 13:32'
tags:
  - fp
  - emergingthreats
---
## Background

It appears that SMTP MTAs and SMTP spam gateways utilizing [DKIM](https://support.dnsimple.com/articles/dkim-record/) sometimes make many `_domainkey` DNS TXT requests, which occasionally generate a false positive alert (FP) for rule 2032936. The cause of these alerts is a bit of inexact rule logic, which sometimes matches legitimate requests greater than a certain length that start with the underscore character: `content:"_"; depth:1; content:"_domainkey"; distance:8;`  

For example, this non-malicious request generates a FP:  
`_conversica._domainkey.servicios.redactado.com`  

This example request would also generate a FP since distance:8; is not restricted to any bytes by the "within" keyword:  
`_asdfasdfasdfasdfasdfasdfasdfasdfasdf._domainkey.example.com`  

Here is an example of the malicious sliver CnC traffic which is the target of the original detection logic:
![malicious DNS TXT request](/images/2021/05/20210518.1.png)

## Analysis

We can refer to the sliver source code (convenient!) [on github](https://github.com/BishopFox/sliver/) and search for the string "\_domainkey" to find out the purpose and details of this traffic. Doing this reveals some lovely documentation on [line 254](https://github.com/BishopFox/sliver/blob/672c0e29d07313fcc3d093d2c6b742659e574e07/server/c2/udp-dns.go#L254) which explains the first part of the domain name is a nonce. The purpose of this nonce is to generate a unique name for the DNS request so if a DNS server has been configured to ignore a record's TTL and always cache the name, we still have reason to perform recursion because it has never seen that unique domain name before. 

![sliver source code snippet](/images/2021/05/20210518.5.png)

The format of the nonce is an underscore character followed by a string of the length defined in [nonceStdSize](https://github.com/BishopFox/sliver/blob/672c0e29d07313fcc3d093d2c6b742659e574e07/implant/sliver/transports/udp-dns.go#L66) at line 66, containing random characters from the variable named [dnsCharSet](https://github.com/BishopFox/sliver/blob/672c0e29d07313fcc3d093d2c6b742659e574e07/implant/sliver/transports/udp-dns.go#L75) at line 77. 
![sliver source code snippet](/images/2021/05/20210518.4.png)  

## Modification

With this information we can update the rule logic to be more specific to this pattern and eliminate most FP.  

`content:"_"; depth:1; content:"_domainkey"; distance:8;`  
becomes  
`content:"_"; depth:1; pcre:"/^[a-z0-9_]{6}[^a-z0-9_]/R"; content:"_domainkey"; distance:8;`   

We've added a PCRE with a relative ("/R") modifier to ensure the next 6 characters after the underscore are from variable dnsCharSet. With these changes we should dramatically reduce the number of false positives (but not entirely eliminate them, for example in the case of: `_123456._domain.example.com`)

original rule:

>```alert dns any any -> any any (msg:"ET TROJAN Suspected Sliver DNS CnC (original)"; content:"|00 10 00 01|"; isdataat:!1,relative; content:"|00 10 00 01|"; isdataat:!1,relative; dns_query; content:"_"; depth:1; content:"_domainkey"; distance:8; fast_pattern; reference:url,github.com/BishopFox/sliver; classtype:trojan-activity; sid:2032936; rev:1;)```

becomes:

>```alert dns any any -> any any (msg:"ET TROJAN Suspected Sliver DNS CnC (modified)"; content:"|00 10 00 01|"; isdataat:!1,relative; content:"|00 10 00 01|"; isdataat:!1,relative; dns_query; content:"_"; depth:1; pcre:"/^[a-z0-9_]{6}[^a-z0-9_]/R"; content:"_domainkey"; distance:8; fast_pattern; reference:url,github.com/BishopFox/sliver; classtype:trojan-activity; sid:9999999; rev:2;)```

## Verify

Next we verify that our fix is good (if we have the luxury of pcap). Truth be told, I had a few small mistakes to fix and that is quite common for me.  

![fast log snippet](/images/2021/05/20210518.3.png)

## Conclusion

The last thing to do is to submit to the mailing list and enjoy that sweet sweet open source community karma. Could we make this detection logic more precise? Probably, but we might risk creating a false negative by using too exact logic or including incorrect assumptions in our logic. Therefore we seek balance. Shout out to the BishopFox team for creating such a masterfully executed tool.

Please [DM me](https://twitter.com/travisbgreen/) with any rule analysis or FP/TP analysis requests.