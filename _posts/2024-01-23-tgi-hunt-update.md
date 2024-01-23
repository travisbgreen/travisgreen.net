---
layout: post
title:  "TGI HUNT Ruleset Update"
categories: suricata 
tags: suricata hunting ruleset
published: false
permalink: /updates/20240123
---

# TGI HUNT - Update January 2024
Hey all, I've decided to start being more verbose about this project's activity in the hope of generating more feedback from people checking it out.

### Documentation
Documentation will slowly appear in the ruleset as reference links to this site. If there is a particular rule you'd like to know more about, simply open a github issue [here](https://github.com/travisbgreen/hunting-rules/issues/new/choose) and I will bump it to the top of the doumentation queue.


### Ruleset Housekeeping
- cleaned up spacing
- removed some obsolete encoding rules
- renamed & simplified `2610338`
- removed previously disabled JA3 rules

### New Rules

##### xmrigCC Donation Mining Pool Domain
I found this domain while examining torminer. It is a hardcoded domain found in [xmrigCC source code](https://github.com/Bendr0id/xmrigCC/blob/e3ea3139ac57c546a312ce5b699fa7caf2edf6f6/src/net/strategies/DonateStrategy.cpp#L53)

##### Suspicious String Inbound (b64 DownloadString)
This was found analyzing malicious powershell downloading an exe file per CrackMapExec web delivery of implant

##### Powershell.exe Inbound to SQL (UTF-16LE)
This was found surveying MSSQL attack techniques in Metasploit and CrackMapExec

##### MSSQL Antivirus Error
This error was observed during adversary simulation against MSSQL, when the MSSQL antivirus found something naughty and refused the query/command

##### Malicious Shell Script Artifact Inbound
These artifacts were observed when a compromised system reached out for a script containing these lines, which are meant to disable command logging at the bash terminal. For example PEASS-ng, a local privesc utility, [uses this technique](https://github.com/carlospolop/PEASS-ng/blob/46612a23aad2e0039a731a8826a0654ac6b48655/linPEAS/builder/linpeas_parts/linpeas_base.sh#L991).

##### MSSQL Configuration Changed Message
This is the output of `sp_configure` MSSQL stored procedure, which occurs when using any of the Metasploit techniques to execute on MSSQL.

##### MSSQL Blocked Stored Procedure Message
Observed in MSSQL when using Metasploit
> Server blocked access to procedure 'sys.xp_cmdshell' of component 'xp_cmdshell' because this component is turned off as part of the security configuration for this server. A system administrator can enable the use of 'xp_cmdshell' by using sp_configure. For more information about enabling 'xp_cmdshell', search for 'xp_cmdshell' in SQL Server Books Online.

##### MSSQL Generic xp_cmdshell
This is a generic `xp_cmdshell` string observed across many red team [github repos](https://github.com/search?q=%22exec+master..xp_cmdshell%22&type=code)

##### Base64 Encoded EXE File in DNS
This came up in my daily reading, and I found an example from the venerable [Didier Stevens](https://blog.didierstevens.com/2019/08/07/downloading-executables-over-dns-capture-files/). Note that I didn't specify TXT record type here.
