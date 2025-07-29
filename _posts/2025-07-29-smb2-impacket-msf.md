---
layout: post
title:  "Setting impacket & Metasploit to use SMB2"
categories:
tags: smb impacket metasploit
published: true
---

When testing detection capabilities in my Active Directory lab, I ran into a common issue: impacket tools default to SMB3, but I needed to generate SMB2 traffic for detection rule development. Here's how to force both impacket and Metasploit to use SMB2 instead.

## The Problem

Modern security tools like impacket automatically negotiate the highest SMB protocol version (usually SMB3), which makes it difficult to test detection rules specifically designed for SMB2 traffic patterns.

## Solution 1: Impacket Configuration

For impacket tools, you need to modify the SMB dialect preference in the source code:

**File:** `impacket/smbconnection.py` (around line 79)

**Change the preferredDialect parameter to force SMB2:**

```python
# Find this section in smbconnection.py
elif preferredDialect in [SMB2_DIALECT_002, SMB2_DIALECT_21, SMB2_DIALECT_30, SMB2_DIALECT_311]:
    self._SMBConnection = smb3.SMB3(self._remoteName, self._remoteHost, self._myName, hostType,
                                    self._sess_port, self._timeout, preferredDialect=SMB2_DIALECT_21)
    #                                                                 ^^^^^^^^^^^^^^^^
    #                                                                 Force SMB2 here
```

**Available SMB2 constants:**
- `SMB2_DIALECT_002` - SMB 2.0.2
- `SMB2_DIALECT_21` - SMB 2.1 (recommended)

## Solution 2: Metasploit Configuration

For Metasploit modules, disable SMB encryption to force protocol downgrade:

```msf
msf6 exploit(windows/smb/psexec) > set SMB::AlwaysEncrypt false
SMB::AlwaysEncrypt => false
```

## Verification

After making these changes, you can verify the SMB version using network capture tools like Wireshark or tcpdump to confirm SMB2 negotiation packets.

These modifications ensure your red team tools generate the specific SMB2 traffic needed for detection rule testing. Just remember to document these changes for your team - and hope attackers ~~don't~~ do read your blog! 😉
