---
layout: posts
title: "ExfilNode CTF Writeup"
date: 2026-01-09
categories: writeups
tags: Linux Forensics Exfiltration
exerpt: Continue hunting for the exfiltration footprints in the ex-employee's personal workstation.
difficulty: Medium
---

# TryHackMe: ExfilNode Writeup

**Difficulty:** {{page.difficulty}} | **Focus:** {{ page.tags | join: ', ' }}.

> **Room link:** [https://tryhackme.com/room/exfilnode](https://tryhackme.com/room/exfilnode)

Hey everyone! Today we are diving into a digital forensics challenge on TryHackMe called **ExfilNode**. This one is a classic "insider threat" scenario where we have to piece together the trail left by an ex-employee named Liam who was up to some shady business. 

I’ll be walking you through my thought process as we hunt for footprints in his workstation. Let's get to work!

---

**Q1: When did Liam last log into the system?**

I started by digging through the authentication logs. On Debian-based systems, `var/log/auth.log` is the place to be. I filtered for Liam’s session openings, specifically focusing on the Gnome Display Manager (gdm) to find his GUI login.
* **Command:** `cat var/log/auth.log | grep -a "user liam" | grep -a "session opened" | grep -a gdm`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/1.png"></div>

**Q2: What was the timezone of Liam’s device?**

This is a crucial step in forensics so we can sync our timeline with the real world. This was the easiest part of the recon.
* **Command:** `cat etc/timezone`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/2.png"></div>

**Q3:What is the serial number of the USB that was inserted by Liam?** 

I looked into `var/log/syslog` for any USB-related events immediately following Liam's login. 

* **Command:** `cat var/log/syslog | grep -a "usb"`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/3.png"></div>

**Q4: When was the USB connected to the system? (Format: YYYY-MM-DD HH:MM:SS)?**

Once you find the registration entry, the serial number and the exact timestamp (YYYY-MM-DD HH:MM:SS) are right there in the same block of text.


**Q5: What command was executed when Liam ran 'transferfiles'?**

In Linux, users can define shortcuts in their `.bashrc` file. I checked Liam's home directory to see what that alias was actually doing.
* **Command:** `cat home/liam/.bashrc | grep -a transferfiles`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/4.png"></div>

**Q6: What command did Liam execute to transfer the exfiltrated files to an external server?**

The `.bash_history` file is a goldmine for forensic investigators. It records the literal commands a user typed in their terminal.
* **Command:** `cat home/liam/.bash_history`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/5.png"></div>


**Q7: What is the IP address of the domain to which Liam transferred the files to?**

The history showed a domain name, but not an IP. By checking the `etc/hosts` file, we can see if Liam manually mapped that domain to a specific IP address to bypass DNS.
* **Command:** `cat etc/hosts`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/6.png"></div>

**Q8: Which directory was the user in when they created the file 'mth'?**

Whenever someone uses `sudo`, the event is logged in `auth.log`, including the Working Directory (PWD). This is a great way to find where a user was when they ran privileged commands.
* **Command:** `cat var/log/auth.log | grep -a mth`


<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/7.png"></div>

**Q9: Remember Henry, the external entity helping Liam during the exfiltration? What was the amount in USD that Henry had to give Liam for this exfiltration task?**

We found the file `mth` (likely standing for "Message To Henry"). Reading its contents reveals the "contract" between them.

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/8.png"></div>

**Q10: When was the USB disconnected by Liam? (Format: YYYY-MM-DD HH:MM:SS)**

We head back to the syslog and look for the "disconnect" event for the specific USB bus Liam was using (usb 1-1).
* **Command:** `cat var/log/syslog | grep -a "usb 1-1" | grep -a "disconnect"`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/9.png"></div>

**Q11: There is a .hidden/ folder that Liam listed the contents of in his commands. What is the full path of this directory?**

Liam interacted with a hidden folder. Using `find`, we located two possibilities. One was empty, while the other sat in the Public folder and was packed with files—the clear winner for an exfiltration staging area.
* **Command:** `find /mnt/liam_disk -type d -name ".hidden" 2>/dev/null`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/10.png"></div>

**Q12: Which files are likely timstomped in this .hidden/ directory (answer in alphabetical order, ascending, separated by a comma. e.g example1.txt,example2.txt)**

Timestomping is an anti-forensic technique where an attacker changes file modified dates to make them look old or "legitimate." In a directory where most files are from 2025, the ones with wildly different dates stick out like a sore thumb.
* **Command:** `ls -la /mnt/liam_disk/home/liam/Public/.hidden`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/11.png"></div>

**Q13: Liam thought the work was done, but the external entity had other plans. Which IP address was connected via SSH to Liam's machine a few hours after the exfiltration?**

I checked for incoming SSH connections after the exfiltration window.
* **Command:** `cat var/log/auth.log | grep -a "sshd"`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/12.png"></div>


**Q14: Which cronjob did the external entity set up inside Liam’s machine?**

The external entity wanted persistence. They set up a scheduled task (cronjob). While system-wide cronjobs are common, this one was hidden in Liam's user-specific crontab.
* **Command:** `sudo cat var/spool/cron/crontabs/liam | grep -v "#"`

<div class="post-img"><img src="{{ site.baseurl }}/assets/img/exfilnode/13.png"></div>