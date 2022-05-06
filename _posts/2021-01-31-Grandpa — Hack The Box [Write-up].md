---
title: "Grandpa — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Blunder Hackthebox Machine"
date: 2021-01-31 07:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---

## Introduction

![](/img/HTB/Grandpa/Grandpa.png)

Grandpa is one of the simpler machines on Hack The Box, however it covers the widely-exploited CVE-2017–7269. This vulnerability is trivial to exploit and granted immediate access to thousands of IIS servers around the globe when it became public knowledge.

Grandpa and Granny are exploited nearly the same way through the exploit [CVE-2017–7269] but Granny could be exploited in other way through uploading and executing files to the server as Grandpa doesn’t allow.

Granny write-up: https://0x3ashry.medium.com/granny-hack-the-box-write-up-37b2799e460e


#### Skills Required:
- Basic knowledge of Windows
- Enumerating ports and services
- Basic knowledge of Metasploit usage

#### Skills Learned:
- Identifying known vulnerabilities
- Basic Windows privilege escalation techniques

#### Used Tools:
- nmap
- Searchsploit
- Metasploit


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Grandpa/img (1).png)

Nmap is done with only port 80 open, I also noticed webdav-scan which is particularly interesting…

After checking the website i found nothing, it is still under development… So let’s enumerate the directories:

![](/img/HTB/Grandpa/img (2).png)

I tried to navigate through them but i hit a block wall so i had to think of other way…

in the enumeration phase we found that webdav is enabled and we are working on IIS v6.0 so let’s look for a public exploit for it…


## 2. EXPLOITATION

    searchsploit webdav

![](/img/HTB/Grandpa/img (3).png)

We found alot of compatible exploits but the first one “Microsoft IIS 6.0 — WebDAV ‘ScStoragePathFromUrl’ Remote Buffer Overflow” seems to be different than the others let’s look if metasploit have similar one…

![](/img/HTB/Grandpa/img (4).png)

Awesome !!! we found a similar exploit available on metasploit so let’s use it and specify it’s options…

![](/img/HTB/Grandpa/img (5).png)

Failed !!? and after long time trying to find other way i tried to restart the lab and try the same exploit again and it worked perfectly, so if it failed with you restart the lab and continue.

![](/img/HTB/Grandpa/img (6).png)

The exploit succeeded and we successfully had a meterpreter session on the server :D

![](/img/HTB/Grandpa/img (7).png)

but when we try to access Lakis to get the user flag it gives Access is denied

Let’s try to elevate our privileges…


## 3. PRIVILEGE ESCALATION

Let’s checkout local exploits and we won’t get easier or better option that Metasploit’s Module (multi/recon/local_exploit_suggester)

So first background you session and get it’s session id

```bash
use multi/recon/local_exploit_suggester
```

![](/img/HTB/Grandpa/img (8).png)

It suggests a lot of local exploits, Ignore the first one cause it is not validated… Let’s try ms14_058_track_popup_menu

```Bash
use exploit/windows/local/ms14_058_track_popup_menu
```

![](/img/HTB/Grandpa/img (9).png)

The exploit succeeded, Let’s check our privilege level…

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D

You’ll find the two flags at:

user → C:\Documents and Settings\Harry\Desktop\user.txt

root→ C:\Documents and Settings\Administrator\Desktop\root.txt

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***

