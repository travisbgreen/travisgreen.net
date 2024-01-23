---
layout: "post"
title: "Cobalt Group Report"
date: "2019-09-13 12:09"
---

# Cobalt Group Report

**Update/correction**
It has been brought to my attention that the attribution of this activity to cobalt group is not ironclad, it may change in the future.


Recently an excellent report by [Checkpoint](https://research.checkpoint.com/cobalt-group-returns-to-kazakhstan/) was published explaining recent developments of the Cobalt Group threat actor group. The Checkpoint report covers a lot of interesting TTPs and the evolution of techniques and procedures, but for now we will focus on detecting the unique C2 used by the group.

[Cobalt Group](https://attack.mitre.org/groups/G0080/) recap:
- financially motivated
- targets eastern EU and central Asian banks
- leader arrested in 2018
- RU language decoy doc downloads & executes cobalt strike beacon via squibbly2
- known to pivot from victim's email account to infect others to exploit trust


## C2 - The Interesting Bits
First we see the O365 Malleable C2 Profile as mentioned in the report, and as usual Cobalt Strike does an impressive job of masquerading its C2 as legitimate traffic.

### HTTP Request:

![](/assets/img/cg_owa.png)

First we have a long URL with legitimate looking /owa/ path and convincing parameters, but with no Referer this would have to be the first request, which seems odd. Also we have a Cookie header, again with no Referer; where did the cookie come from if this is the first request? There would be no chance for a server to respond with Set-Cookie, so this jumps out as odd.

`Hunting sig idea: detect Cookie header with no Referer`

We have a legitimate but ancient user agent string, and a dotted quad IP address. Cobalt strike is known for using a fake HTTP Host header, why do we not see that here? Looking at the source code mentioned in the article, we see that line 33 is commented out by default in this profile. 

![](/assets/img/cg_line33.png)

Further down in the file at line 93 we see what is really going on here. The real C2 data is contained within wla42= GET parameter. 

![](/assets/img/cg_line93.png)

All of the other static fields defined here would make for great detection strings, but would catch users of this exact profile only, and only in the default configuration.

### HTTP Response:
We also see a lot of nice custom X- headers in the request convincing us of its legitimacy. One thing that jumps out as odd in the server response is that it is serving no data (Content-Length: 0). Could we build a hunting signature based on this?

`Hunting sig idea 1: detect Content-Length: 0 and HTTP 200`

From this information we can build these two signatures:

```bash
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"TGI TROJAN Cobalt Strike
 Malleable C2 Request (Cobalt Group O365 Profile)"; flow:established,to_server;
 http.uri; content:"/owa/?wa="; startswith; content:"&path=/calendar";
 endswith; http.header_names; content:!"Referer"; content:"Cookie";
 http.cookie; content:"MicrosoftApplicationsTelemetryDeviceId="; startswith;
 content:"|3b|ClientId="; distance:0; content:"|3b|MSPAuth="; distance:0;
 content:"|3b|xid="; distance:0; content:"|3b|wla42="; distance:0;
 reference:md5,a26722fc7e5882b5a273239cddfe755f;
 reference:url,attack.mitre.org/groups/G0080/;  classtype:trojan-activity;
 sid:1003918; rev:1;) 

alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"TGI TROJAN Cobalt Strike
 Malleable C2 Response (Cobalt Group O365 Profile)";
 flow:established,to_client; http.header;
 content:"16723708fc9|0d 0a|X-CalculatedBETarget|3a 20|BY2PR06MB549.namprd06";
 content:"X-FEServer|3a 20|CY4PR02CA0010"; distance:0; 
 reference:md5,a26722fc7e5882b5a273239cddfe755f;
 reference:url,attack.mitre.org/groups/G0080/; classtype:trojan-activity;
 sid:1003919; rev:1;)
```


### Additional Sample
Additional samples have come to light which show another malleable C2 profile being used, this time mimicing YouTube activity:

![](/assets/img/cg_yt.png)

Here we see the fake HTTP Host common to Cobalt Strike Beacon. Also, interestingly we also have a null response from the server, but because the C2 profile pads the response, we do not see "Content-Length: 0" header in the response, instead we have "content=''" for our empty response.

For these, we can detect with:
```bash
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"TGI TROJAN Cobalt Strike
 Malleable C2 Request (Cobalt Group YouTube Profile)";
 flow:established,to_server; http.uri; content:"/watch?v="; http.header_names;
 content:!"Referer"; content:"Cookie";
 reference:md5,69c6e302cc4394cae7ed8c6f7b288e92;
 reference:url,attack.mitre.org/groups/G0080/; classtype:trojan-activity;
 sid:1003920; rev:1;)

alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"TGI TROJAN Cobalt Strike
 Malleable C2 Response (Cobalt Group YouTube Profile)";
 flow:established,to_client; http.header; content:"Frontend Proxy|0d
 0a|Set-Cookie|3a 20|YSC=LT4ZGGSgKoE|3b|"; fast_pattern; content:"X-FEServer|3a
 20|CY4PR02CA0010"; distance:0; reference:md5,69c6e302cc4394cae7ed8c6f7b288e92;
 reference:url,attack.mitre.org/groups/G0080/; classtype:trojan-activity;
 sid:1003921; rev:1;)
```


### Signatures for Threat Hunting
If we implemented either of the proposed hunting sigs above, we would probably end up wading through huge amounts of alerts with potentially bad traffic somewhere in there, i.e. the proverbial needle in a haystack. Perhaps there is a way we can combine these two together to produce a less daunting haystack?

Consider the following:

```bash
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"TGI HUNT Possible Cobalt
 Strike Malleable C2 Null Response (Flowbit Set)"; flow:established,to_server;
 http.header_names; content:!"Referer"; content:"Cookie";
 flowbits:set,hunt.cs_null_response; flowbits:noalert; threshold:type limit,
 track by_src, seconds 60, count 1; classtype:trojan-activity; sid:2610202;
 rev:1;)

alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"TGI HUNT Possible Cobalt
 Strike Malleable C2 Null Response"; flow:established,to_client;
 http.stat_code; content:"200"; bsize:3; pkt_data; 
 content:"Content-Length:|20|0|0d 0a|"; fast_pattern;
 flowbits:isset,hunt.cs_null_response; threshold:type limit, track by_src,
 seconds 60, count 1; classtype:trojan-activity; sid:2610203; rev:1;)
```

Here we are setting a flowbit on any HTTP request that has a Cookie set, but lacks the Referer header. Then we detect any HTTP 200 responses for 0 content length, then also check if the flowbit has been set by our previous sig. The end result is a strong indicator that further investigation is warranted. The caveat here is that these are signatures that will consume high amount of resources. In the first case, every request with a Cookie header will be inspected, and in the second case, any traffic with "Content-Length: 0" will be inspected. So these would not be great signatures for production use where network traffic is of any significant volume.

You can find these hunting sigs and more @ https://github.com/travisbgreen/hunting-rules

Comments? Suggestions? DMs open: [@travisbgreen](https://twitter.com/travisbgreen)
