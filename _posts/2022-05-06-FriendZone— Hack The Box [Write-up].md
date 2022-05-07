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

