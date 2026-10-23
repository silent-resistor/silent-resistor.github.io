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
- For instance, if web browsing is allowed, it usually means that protected hosts can connect to ports 80 and 443 unless a local proxy is used. In one case, the client relied on google services for their bussiness, so the attacker used 






























