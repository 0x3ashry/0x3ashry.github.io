---
title: "Poison — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Poison Hackthebox Machine"
date: 2022-05-09 22:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---


![](/img/HTB/Poison/Poison1.png)

Poison is a Medium rated FreeBSD retired box, but an enjoyable one with easy user access and good privesc.
Let's get started. 

#### Used Tools:
- Nmap
- Cyberchef
- Unzip
- SSH
- Wget
- Linpeas.sh


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Poison/img (1).png)

We found 2 opened ports:
- 22 for an SSH
- 80 for an HTTP server

I tried anonymous login in the ssh but it didn't work so I jumbed directly to the http...

All I found is a temporary website to test local **.php** scripts

![](/img/HTB/Poison/img (2).png)

It says that the scripts or sites to be tested are `ini.php, info.php, listfiles.php, phpinfo.php`

I tried them all and but when I tried `listfiles.php` it showed me a list of the files in there...

![](/img/HTB/Poison/img (3).png)

There is an interesting file called `pwdbackup.txt`, What if we put it in the script name field in the temporary website and see if it will accept it

![](/img/HTB/Poison/img (4).png)

It said "This password is secure, it's encoded atleast 13 times.. what could go wrong really.. " and have a very long base64 password

![](/img/HTB/Poison/img (5).png)

I will try decoding it 13 times using Cyberchef

Cyberchef: https://gchq.github.io/CyberChef/

I applied *From Base64* module 12 times and it reveals a password `Charix!2#4%6&8(0`

![](/img/HTB/Poison/img (6).png)


## 2. EXPLOITATION:

I tried ssh with creds username:charix (this was a guess because it wass the prefix of the password) and password:Charix!2#4%6&8(0

![](/img/HTB/Poison/img (7).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOM ! We successfully gained a ssh shell for `charix` with the user flag in the home directory


## 3. PRIVILEGE ESCALATION:

I found beside user.txt file a filename called `secret.zip` I tried to unzip it in Poison but it required a passphrase so I transfered it to kali and tried to unzip it there...

I found python module installed in poison so I will open python http server and get the secret.zip file from there...

```bash
python -m SimpleHTTPServer 8080
```

![](/img/HTB/Poison/img (8).png)

At kali 

```bash
wget http://10.10.10.84:8080/secret.zip
```

And it is transfered successfully.

![](/img/HTB/Poison/img (9).png)

Let's Unzip it

```bash
unzip secret.zip
```

It required a password so I tried the same password as charix's ssh password:Charix!2#4%6&8(0 and it worked !!

![](/img/HTB/Poison/img (10).png)

This is a live example on the dangers of using the same password in many places...

I can't understand the file content righ now but I think it is a password for something, so let's leave it now...

Let's transfere `linpeas.sh` to Poison and run it...

![](/img/HTB/Poison/img (12).png)

I found a `VNC` Service running as a root on port `5901`

But I can not access VNC server from Poison. Let us **tunnel** the Poison’s port 5901 to my local box.

```bash
ssh -L 5901:127.0.0.1:5901 charix@10.10.10.84
```

![](/img/HTB/Poison/img (13).png)

Now that SSH tunneling is done, let us try to access the server via VNC client.

I tried:

```bash
vnc 127.0.0.1:5901
```

but it required a password and we had previously unzipped a secret file so let's pass it...

```bash
vnc -passwd secret 127.0.0.1:5901
```

![](/img/HTB/Poison/img (14).png)

![](/img/HTB/Poison/img (15).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***