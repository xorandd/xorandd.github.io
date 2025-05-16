---
title: "TryHackMe Madness"
categories: [TryHackMe]
tags: [linux, web, steganography, data-encoding, suid]
media_subpath: /images/tryhackme_madness
image: 
  path: room-image.png
---
## Enumeration

Let's start from nmap scan:

Discovered 2 open ports:
  - 22 (SSH)
  - 80 (HTTP)
 
![Nmap-scan](1.png)

## Web analysis
  
Tried to run gobuster to enumerate directories but it didnt found anything, let's check the website manually. In the source code we found a comment and an image reference

  ![Website-source-code](2.png)

Opened this picture but it seems like corrupted

![THM-image-website](3.png)

Let's download image to our machine

  ![Copy-image](4.png)

## Stego

After checking the image it really seems corrupted. The signature was missing in header so I edited the first line using hexeditor like this:

```
FF D8 FF E0 00 10 4A 46 49 46 00 01
```

  ![JPEG-signature](6.png)

  ![Hexeditor](8.png)

After fixing and opening the image there is a text with directory.
Open the url with the given directory.

The source code mentions entering some "secret" between 0–99.
  ![Web-path](9.png)

  ![Hidden-path-source-code](10.png)

Suppose we can add "secret" tag to this url and this guess was right.
 
  ![Tag-secret](11.png)

  ## Getting right "secret"

Generate a list of numbers from 0 to 99

  ```bash
  seq 0 99 > numbers.txt
  ```

Wrote a python script which can automate the process:

  ![Python-script](12.png)

Script found correct secret and we got a possible password, but no username yet.

Used steghide to extract embedded file in image we recovered earlier using found password

![Correct-Secret](13.png)

![Steghide-extract1](15.png)

A hint says `There's something ROTten about this guys name!` - used ROT13 to decode a cipher and get the username

## Getting SSH access

Well, this was the longest part of this room - we cant brute the SSH (as stated in the room) and no password for ssh login. Eventually found another stego, and guess where was it? Image from task section. Download it and run steghide to extract embedded file

![Room-picture](23.png)  

![Room-picture-steg](16.png)

Now we have all credentials we need, let's log into ssh and get user flag

![user-flag](17.png)

## Privilege Escalation

After checking directories and not finding anything useful decided to run linpeas on target machine

It found some SUID binary screen 4.5.0 which is not default

![Room-picture-stego](19.png)

Found exploit to this on [exploitDB](https://www.exploit-db.com/exploits/41154)

Upload and run this exploit. Now we root and can get root flag!

![Root-access](21.png)

![Root-flag](22.png) 

