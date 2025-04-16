---
layout: post
title:  "Hunting for browser extension abuse"
categories:
tags: malware javascript
published: true
---

I came across a funny thing while digging into discord stealers. It seems the world of discord stealers is very much in the business of cryptocurrency theft, and as a result of many crypto wallets being browser extensions, we see these class of attacks frequently looking for these browser extensions to inject malicious javascript. I've introduced a new set to the TGI HUNT rules to detect these browser extension ID strings in HTTP. 

For example, here is [1336 stealer v3](https://github.com/Yxxtsuu/1336-V3/blob/c32a0f68b502cf17aa7b6e63ec9ff919593dfa5c/utils/crypto.js#L98):

![Browser Extension Abuse](/assets/img/20250416.1.png)


To enumerate any new javascript inbound or identifying browser extension information outbound, I've introduce `browser-extensions.rules` available on the TGI HUNT git repo: https://github.com/travisbgreen/hunting-rules/