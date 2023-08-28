---
title: "[Writeup] Hacking Contest 2019 (Qualification Round)"
date: 2019-03-03T22:36:00+07:00
categories: [writeup]
tags: [ctf, writeup, competition]
---

Yesterday, I got a chance to participate in the qualification round of competition called "Hacking Contest 2019" held by SOSECURE and MFEC. It is claimed to be a hacking competition with the largest prize pool in Thailand.

{{< img src="/images/writeup/hackingcontest2019/hc2019_poster.jpg" alt="Poster1">}}

{{< img src="/images/writeup/hackingcontest2019/hc2019_poster2.jpg" alt="Poster2">}}
*Posters*

Once again, I was participating with my team "!IsCaptured" (Dawit Chusetthagarn, Patipon Suwanbon, and I), the same one from the previous competition. ([Financial Cybersecurity Bootcamp 2018](/writeup/fincybersec2018))

# Qualification Round

The qualification round is an 18-hour long competition, from 18.00 of 1 March 2019 until 12.00 of 2 March 2019. The challenge is given as a VM image and the goal is getting the root access. The answer we needed to submit is a report describing the steps we have done in order to gain the root access.

## Port Scanning

After setting up the VM and the network, we started by scanning the ports for services with Nmap.
{{< img src="/images/writeup/hackingcontest2019/hc2019_nmap.png" alt="Nmap">}}
*Nmap Scanning Result*

The result is pretty interesting, as we found multiple services running, FTP on 21, SSH on 22, Samba, Apache web server on 27777 (and another one possibly on 80). And so we started looking at each service.

## FTP

With further investigation, we found out that the FTP server allows Anonymous login, and there are 4 subdirectories inside: dressrosa, enieslobby, marineford, and punk-hazard (The challenge creator must be a One Piece fan for sure).

{{< img src="/images/writeup/hackingcontest2019/hc2019_ftp.png" alt="FTP">}}
*FTP Scanning*

We got inside by logging in as Anonymous, searched through the directories, and found a file called ".onepiece.zip" hidden inside "enieslobby".

{{< img src="/images/writeup/hackingcontest2019/hc2019_ftp2.png" alt="FTP">}}
{{< img src="/images/writeup/hackingcontest2019/hc2019_ftp3.png" alt="FTP">}}
{{< img src="/images/writeup/hackingcontest2019/hc2019_ftp4.png" alt="FTP">}}
*Digging Through FTP Server*

We downloaded it; however, it is a password protected zip, so we started the password cracker to brute force it and left it running. Dictionary brute force yielded no result.

## Samba

For the Samba services, we listed the shares using smbclient and found an available share called "sambashare". We connected to it and found 4 subdirectories: y, 2, 3, and d. (Must be the ~~3d~~2y reference from One Piece)

{{< img src="/images/writeup/hackingcontest2019/hc2019_samba.png" alt="FTP">}}
*Searching the Samba Share*

We found a file named "qr code naja" inside "2" directory. It is an image file of a QR code.

{{< img src="/images/writeup/hackingcontest2019/hc2019_qr.png" alt="QR">}}
*qr code naja*

When scanned, we got the text "KEY3 = 478"

## SSH

Using enum4linux we tried to enumerate the users of the system and found 4 users, tester, john, loki, and franky.

{{< img src="/images/writeup/hackingcontest2019/hc2019_enum4linux.jpg" alt="enum4linux">}}
*List of Users*

We tried using hydra to bruteforce the password for all 4 users found, but none was found.

## Web Server

Initially, we couldn't access the web server on port 80, so we started by going into the web server on port 27777.

{{< img src="/images/writeup/hackingcontest2019/hc2019_webhome.png" alt="Homepage">}}
*Homepage on Port 27777*

We used gobuster to list the possible directories and pages, and found the following:

{{< img src="/images/writeup/hackingcontest2019/hc2019_gobuster.png" alt="gobuster">}}
*gobuster Result*

Nothing interesting in /images or robots.txt. ziggy.php however, is a directory with a file named "decode.txt" inside.

{{< img src="/images/writeup/hackingcontest2019/hc2019_ziggy.png" alt="ziggy">}}
*ziggy.php*

The "decode.txt file contains Brainfuck code which can be decoded as “grandline.htb”.

{{< img src="/images/writeup/hackingcontest2019/hc2019_decode.png" alt="decode.txt">}}
*decode.txt*

{{< img src="/images/writeup/hackingcontest2019/hc2019_brainfuck.png" alt="brainfuck">}}
*Decoding Brainfuck code*

After investigating for a while, we found out that "grandline.htb" is the domain name that we can use to access the web server on port 80, so we put it in the "X-Forwarded-Host" header using Burp Suite and accessed the site on 80.

{{< img src="/images/writeup/hackingcontest2019/hc2019_burp.png" alt="Burp">}}
*Editing the HTTP Header*

With that, we found a jQuery File Upload Demo page, which we can easily upload a web shell and access the machine.

{{< img src="/images/writeup/hackingcontest2019/hc2019_upload.png" alt="Upload">}}
*Uploading the Web Shell*

{{< img src="/images/writeup/hackingcontest2019/hc2019_shell.png" alt="Web Shell">}}
*Accessing the Web Shell*

We used [p0wny-shell](https://github.com/flozz/p0wny-shell) since it is interactive and easy to use (It can upload and download file through the browser dialog too!).

We dug through the directories and found a hidden directory on the web server on port 27777 with another webpage.

*There should be another method to find this page, but we got the shell, so we could access and see everything.*

{{< img src="/images/writeup/hackingcontest2019/hc2019_directory.png" alt="Directory">}}
*Finding the Hidden Directory*

{{< img src="/images/writeup/hackingcontest2019/hc2019_page2.png" alt="Webpage">}}
*Webpage in the Directory*

With the shell, we could see the source, so we know that the page would like us to type in the creator's favorite One Piece character, which is "Foxy", then it would tell us that there's the file called "bookie1223sa.txt" inside the directory (which we have already known).

{{< img src="/images/writeup/hackingcontest2019/hc2019_bookie.png" alt="bookie">}}
*bookie1223sa.txt*

There are 0s and 1s in “bookie1223sa.txt”, so we changed them to black and white characters and resize the window to show 48 characters per line and got the key "KEY IS 193".

{{< img src="/images/writeup/hackingcontest2019/hc2019_bookie2.png" alt="bookie">}}
*Replacing the Characters and Resizing the Window*

After digging a bit further into /home/franky, we found the .bash_history with the password "3d2yhijinbe" of the user "franky".
{{< img src="/images/writeup/hackingcontest2019/hc2019_franky.png" alt="franky">}}
*franky's Password*

## Privilege Escalation

Our goal is to get the root access, so we tried to get more information of the system.

{{< img src="/images/writeup/hackingcontest2019/hc2019_uname.png" alt="System Information">}}
*Linux Version*

It's Linux Ubuntu 4.4.0-31, and there are LOTS of exploits available. We downloaded the C source of one, uploaded it, and tried to compile it. Unfortunately, gcc is not installed on the system, so we couldn't get root the easy way. We thought we could download the same version of the OS and compile the exploit, so that we could send the precompiled binary to the system and run, but we found an **EASIER WAY**!

{{< img src="/images/writeup/hackingcontest2019/hc2019_passwd.png" alt="passwd">}}
*Permission Setting on /etc/passwd*

Yes, the passwd is **WRITABLE BY ANYONE!**

We used the following command to add a new suid user with the password “foo”:
```
echo "IsCaptured:aaKNIEDOaueR6:0:0:IsCaptured:/root:/bin/bash" >> /etc/passwd
```

{{< img src="/images/writeup/hackingcontest2019/hc2019_root.png" alt="root">}}
*Rooted*

And thus, we got the root access and the flag "KEY:alabasta" inside the /root directory.

Going back to the zip we got from the FTP server, we used "alabasta" and the password successfully extracted the files.

Inside, we found 5 images of Ace (One Piece character). And that's all we have done and submitted in the report.

{{< img src="/images/writeup/hackingcontest2019/hc2019_ace.png" alt="Ace">}}
*Pictures of Ace*

*We didn't know that there's another key in one of the images and only found that out later from the [writeup by STH](https://sth.sh/assets/files/STH_SOSECURE_CTF_Writeup_v1.0.pdf) after the competition.*

And guess what, we got through to the final round!
{{< img src="/images/writeup/hackingcontest2019/hc2019_final.png" alt="Finalists">}}
*Finalists*

The final round is going to be held on 16 March 2019. Wish us luck!