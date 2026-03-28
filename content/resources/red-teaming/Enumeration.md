---
title: "Enumeration"
date: 2026-04-01
weight: 4
---

## Introduction
- Enumeration is the process of gathering information on a target system to identify potential attack vectors and vulnerabilities. It is a crucial step in the penetration testing process, as it helps to identify the target's weaknesses and vulnerabilities.
- More info on Privilege Escalation can be found in the [Windows Privilege Escalation](https://tryhackme.com/r/room/windowsprivilegeescalation) room and the [Linux PrivEsc](https://tryhackme.com/r/room/linuxprivesc) room. Moreover, there are two handy scripts, [WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) and [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) for MS Windows and Linux privilege escalation respectively.
- We aim, to collect the information that would allow us to pivot to other systems on the network or to loot the current system. Some of the interested in gathering include:
  - Users and groups
  - Hostnames
  - Routing tables
  - Network shares
  - network services
  - Applications and banners
  - Firewall configurations
  - service settings and audit configurations
  - SNMP and DNS details
  - Hunting for credentials (saved on browsers or client applications)


## Linux Enumeration
```bash
# release info
ls /etc/*-release

# system info
hostame
cat /etc/passwd
sudo cat /etc/shadow
ls -lh /var/mail
uname -a

# installed apps
ls -la /usr/bin/
ls -la /usr/local/bin/
rpm -qa # rpm based systems
dpkg -l # deb based systems

#Users
who # who logged in
w # who logged in and whats he doing..
whoami
id -a
groups
last # want to who has been using the system recently
sudo -l # want to know what commands the current user can run with sudo

# Networking
ip addr show
netstat -tuln
ss -tuln
cat /etc/resolv.conf # dns servers
sudo lsof -i # list open network connections
sudo lsof -i :22 # list open network connections on port 22

# running service
ps axf
```


## Windows enumerations
```powershell
# system info
systeminfo 
wmic qfe get Caption,Description,HotFixID,InstalledOn # installed updates
net start # installed and started windows services
wmic product get  name,version,vendor # installed applications

# Users
whoami
whoami /priv # privileges
whoami /groups # groups
whoami /all # all info
net config # config
net user # all running users
net group # all groups
net localgroup # all local groups
net localgroup administrators # all administrators
net accounts # local account settings
net accounts /domain # domain account settings


# Networking
ipconfig /all
netstat -abno # list open network connections, -b shows the executable, -o shows the process ID, -a for listening ports
arp -a # arp table
```


## DNS, SMB, and SNMP
- **DNS**:
  - We are all familiar with DNS queries where we can look up A, AAAA, MX, NS, and other record types.
  - One easy way to try DNS zone transfer is via the `dig` command: depending on dns server configuration, dns zone might be restricted, it should be achievable using `dig -t AXFR DOMAIN_NAME @DNS_SERVER`. The `-t AXFR` indicates that we are requesting a zone transfer, while `@DNS_SERVER` specifies the DNS server to query regarding the records related to `DOMAIN_NAME`.

- **SMB**:
  - Server Message Block (SMB) is communication protocol that provides shared access to files and printers. We can check shared folders using `net share` command.
- **SNMP**:
  - Simple Network Management Protocol (SNMP) is a protocol used for network management. It was designed to help collect information about different devices on the network.
  - It lets us know about various network events, from a server with a faulty disk to pointer out of ink. Consequently, SNMP can hold a trove of information for the attacker. One simple tool to query server related to SNMP is `snmpcheck`.
  - Syntax as follows - `snmpcheck.rb 10.49.173.77 -c COMMUNITY_STRING`. Remember to replace `COMMUNITY_STRING` with the actual community name.
    ```powershell
    git clone https://gitlab.com/kalilinux/packages/snmpcheck.git
    cd snmpcheck
    gem install snmp
    chmod +x snmpcheck.rb
    ```



## More windows tools
- **Sysinternals Suite**:
    |Utility Name|Description|
    |---|---|
    |Process Explorer|Shows the processes along with the open files and registry keys|
    |Process Monitor|Monitor the file system, processes, and Registry|
    |PsList|Provides information about processes|
    |PsLoggedOn|Shows the logged-in users|

- **Process Hacker:** Another efficient and reliable MS Windows GUI tool that lets you gather information about running processes is Process Hacker. Process Hacker gives you detailed information regarding running processes and related active network connections; moreover, it gives you deep insight into system resource utilization from CPU and memory to disk and network.

- **GhostPack Seatbelt:** Seatbelt, part of the GhostPack collection, is a tool written in C#. It is not officially released in binary form; therefore, you are expected to compile it yourself using MS Visual Studio.





