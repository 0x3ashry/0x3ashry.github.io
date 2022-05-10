---
title: "Sense — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Sense Hackthebox Machine"
date: 2021-02-02 08:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---

![](/img/HTB/Sense/Sense.png)

Sense, while not requiring many steps to complete, can be challenging for some as it is all about enumeration which takes long time for me then every thing goes easy as we will see.


#### Used Tools:
- Nmap
- Gobuster or Dirsearch
- Metasploit


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Sense/img (1).png)

We found an HTTP on port `80` which automatically redirect us to HTTPS on port `443`, After going through the website we found nothing but a login form for the pfsense which is an open source `firewall/router` computer software distribution based on **FreeBSD**.

we tried the default credentials (admin:pfsense) but it didn’t work so let’s enumerate the directories…

hold on, I usually use *"/usr/share/wordlists/dirb/common.txt"* which is short and contain the most common directory list but when i used it here it gave me an incomplete results made me go in a very long and wrong way and finally hit a block so when i used *"/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt"* it gave me extra files that put me on the right way but it took a hell of long time about an hour and half cause it contains about 661K directory.

here is the results of the common.txt gobuster enumeration:

![](/img/HTB/Sense/img (2).png)

Note: you must use the --insecuressl option to tell gobuster to ignore the unknown SSL certificate of the website.

But here i used **dirsearch** cause i think it is a bit faster and it’s output shape is better for me.

```bash
sudo python3 dirsearch.py -u https://10.10.10.60 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -f -e txt
```

![](/img/HTB/Sense/img (3).png)

We fount `changelog.txt` and `system-users.txt` which are missing from the previous enumeration, changelog.txt contain the security changelog, Let’s check the system-users.txt...

![](/img/HTB/Sense/img (4).png)

we fount an support ticket for a user creation called rohit with an default company password (we already have it previously)


## 2. EXPLOITATION:

During the first directories enumeration i found a metasploit module that exploits `pfsense's` RCE vulnerability.

Module: exploit/unix/http/pfsense_graph_injection_exec

But the problem was it requires a username and password and i didn’t have it previously but now after having these credentials let’s try it again...

![](/img/HTB/Sense/img (5).png)

And the exploit succeeded, Now we have a meterpreter session let’s checl our privileges...

![](/img/HTB/Sense/img (6).png)

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D

You’ll find user.txt and root.txt flag at:
user flag → /home/rohit/user.txt
root flag → /root/root.txt

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***