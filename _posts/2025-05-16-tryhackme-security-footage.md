---
title: "TryHackMe Security Footage"
categories: [TryHackMe]
tags: [forensics, pcap, wireshark, data recovery] 
media_subpath: /images/tryhackme_security_footage
image:
  path: room-image.png
---


## Analyzing file
First, download provided `.pcap` file

![Download-file](1.png)


After downloading pcap file and opening it with **wireshark** follow tcp stream that contains encrypted traffic. Notice that stream starts with **JFIF** header - meaning that a JPEG file within that stream.

![Follow-TCP-stream](2.png)

![Contents-of-TCP-stream](3.png)


## Files recovery

To recover files use `foremost` with this command
```bash
foremost -i security-footage-1648933966395.pcap -o extracted
```

![Extracted-data](4.png)

Once images were extracted, open them to reveal the flag

![flag](5.png)

