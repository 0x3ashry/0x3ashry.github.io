---
title: "Irked — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Irked Hackthebox Machine"
date: 2022-05-05 22:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---


## Introduction

![](/img/HTB/Irked/Irked.png)

#### Used Tools:
- nmap
- Metasploit


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Irked/img (1).png)

Nmap is done with port 22 (FTP), port 80 (http) and port 111 (rpcbind) open.

Let's check the web application first on port 80...

![](/img/HTB/Irked/img (2).png)

Nothing Except the sentence below "**IRC** is almost working!" which made me suspect for having and IRC service working

IRC: https://en.wikipedia.org/wiki/Internet_Relay_Chat

Let's test its existance through this nmap script

```bash
nmap -sV --script irc-unrealircd-backdoor -p 194,6660-7000 10.10.10.117
````
![](/img/HTB/Irked/img (3).png)

We successfully found open IRC service on port **6697**


## 2. EXPLOITATION

Searching for exploit online I found most of the exploits on version 3.2.8.1 and found an exploit for it on metasploit, So let's try it...

![](/img/HTB/Irked/img (4).png)

We successfully gained a shell and upgrade it to interactive shell using:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

But our user can't do anything and doesn't even have privalages to read user.txt, So let's try to find some way to upgrade it...


## 3. PRIVILEGE ESCALATION:

After some enumeration let's try finding if there is a SUID files we could execute as a root
SUID: https://pentestlab.blog/2017/09/25/suid-executables/

```bash
find / -perm -u=s 2>/dev/null
```

![](/img/HTB/Irked/img (5).png)

We found **/usr/bin/viewusr** and we could run it with our user but it will be executed with root premissions, Let's try to run it...

![](/img/HTB/Irked/img (6).png)

He prompted that there is a file called /tmp/listusers that isn't found, and it is located in the tmp directory and any user can write on it, So we could create it in the tmp directory and write an exploit in it which will be executed as root when we run /usr/bin/viewusr again.

![](/img/HTB/Irked/img (7).png)

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***