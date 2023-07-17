---
title: "[Writeup] Hack The Box: Magic"
date: 2020-05-06T13:40:00+07:00
categories: [writeup]
tags: [writeup, walkthrough, htb, hackthebox]
---

Let me show you a Magic! This is a Medium difficulty Linux box that employs old but still relevant tricks. You need to know the ***Magic*** and how Linux operates with files to clear this box.

![](/images/htb_magic/infocard.png)

<!--more-->

**OS:** Linux  
**Difficulty:** Medium  
**Points:** 30  
**Release:** 18 Apr 2020  
**IP:** 10.10.10.185  
**Box Creator:** TRX  
**Date Cleared:** 23 Apr 2020

## TL;DR

- Use SQL injection to bypass the login page
- Upload a PHP reverse shell with the `png` magic bytes and `.php;.png` extension
- Get a user credential from the database
- Use path hijacking to get a reverse shell with suid binary

## Information Gathering

As always, an `nmap` scan first.

```
Nmap scan report for 10.10.10.185
Host is up (0.22s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Apr 24 09:15:30 2020 -- 1 IP address (1 host up) scanned in 11.58 seconds
```

There are only 2 ports open, `ssh` on `22` and `http` on `80`, so let's start with the web.

On the home page, there's a gallery of images and a Login button at the bottom left, telling us that new images could be uploaded.

![](/images/htb_magic/80_home.png)

From the source, we can see some images are from the upload path at `images/uploads/*`

![](/images/htb_magic/80_home_source.png)

The login page is just a simple username and password form.

![](/images/htb_magic/80_login.png)

## Gaining Access

We can get through that super easily with a common SQL injection payload `' OR '1'='1` as our username and password.

![](/images/htb_magic/80_sql.png)

![](/images/htb_magic/80_upload.png)

Now that we can upload files to the server, I tried to upload a PHP reverse shell script to the server.

![](/images/htb_magic/nano_phpreverseshell.png)

However, the error message shows that only image file extensions are allowed.

![](/images/htb_magic/upload_php.png)

I renamed the reverse shell file to end with the `.php;.png` extension to bypass the restriction. This way, the upload checker would see that the file is a `png` file, but the web server could process the semicolon `;` and terminates the string, so it sees the file as `php-reverse-shell.php` and execute the PHP script inside.

![](/images/htb_magic/phppng.png)

Unfortunately, I got a new error message when I tried to upload that. This could mean that the upload function checks further than just the filename, and very likely checks the file content.

![](/images/htb_magic/upload_phppng.png)

One of the easiest ways to determine the file type without the extension is using the file `magic bytes` which is the file signature in the beginning bytes of the file (Read more at <https://en.wikipedia.org/wiki/List_of_file_signatures>).

So, to make the system think that my PHP reverse shell file is really an image file, I copied the `magic bytes` from a real png file and prepend it to the `.php;.png` file.

![](/images/htb_magic/revshellpng.png)

I could successfully upload this file with the `magic bytes` added.

![](/images/htb_magic/upload_phppng2.png)

I then accessed `http://10.10.10.185/images/uploads/reverse.php;.png` on my web browser to execute the reverse shell script and get a connection back to my listener.

![](/images/htb_magic/wwwshell.png)

With access to the files, I looked at the website files and found a database credential `theseus:iamkingtheseus` in the database config file of the website.


![](/images/htb_magic/dbphp5.png)

I tried to log in as `theseus` with the credential found but failed.

![](/images/htb_magic/sshfailed.png)

![](/images/htb_magic/sufailed.png)

On the `login.php` file, there is a part of the code that log in to the database and fetches the user from the `login` table.

![](/images/htb_magic/loginphp.png)

I copied the file, modified it to print out the whole table, and uploaded it to the web path.

```php
<?php
require 'db.php5';

try {
    $pdo = Database::connect();
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_OBJ);
    $stmt = $pdo->query("SELECT * FROM login");
    $user = $stmt->fetch();
    $count = 0;
    foreach ($user as $value) {
        echo $value->username. $value->password
        $count += 1;
    }
    Database::disconnect();

} catch (PDOException $e) {
    //echo "Error: " . $e->getMessage();
    //echo "An SQL Error occurred!";
}

?>
```

![](/images/htb_magic/uploaddbread.png)

I then accessed the page with my web browser and got another set of credential `theseus:Th3s3usW4sK1ng`.

![](/images/htb_magic/dbreadpassword.png)

I was able to log in as `thesues` with the new credential.

![](/images/htb_magic/theseusshell.png)

## Privilege Escalation

The user `thesues` is in a suspicious group named `users`, so I searched for the files owned by the `users` group and found `/bin/sysinfo`.

![](/images/htb_magic/theseusgroup.png)

The file is a suid binary, so it would run with `root` privilege.

![](/images/htb_magic/sysinfosuid.png)

I run the binary and it prints out information about the system.

![](/images/htb_magic/sysinforun.png)

With `strings /bin/sysinfo`, I could see that it runs multiple binaries with relative paths rather than absolute paths, such as `lshw`, `fdisk`, `cat`, and `free`.

![](/images/htb_magic/sysinfostrings.png)

With that known, we can easily hijack the `PATH` variable to use `sysinfo` to run our desired binary as root. I uploaded a reverse shell binary created with `msfvenom`, renamed it to one of the binaries found previously, `lshw`, and added the current path to the beginning of the `PATH` variable.

![](/images/htb_magic/msfvenom.png)

![](/images/htb_magic/revupload.png)

![](/images/htb_magic/revprepare.png)

When the `/bin/sysinfo` is executed and reaches the path that calls `lshw`, the system would search for the binary by looking for it in the directories specified in the `PATH` variable one by one.

As we added our path to the beginning, the system will see our version of `lshw` first, and execute it. With the listener ready, I run `/bin/sysinfo` and got a reverse shell connection back as `root`.

![](/images/htb_magic/root.png)