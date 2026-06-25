---
title: "Data Exfiltration"
date: 2026-06-25
weight: 11
---

## 1. Introduction
- Someone may ask: how does a threat actor transfer stolen data from a company's network to the outside, also known as a **data breach**, without being detected? The answer varies. There are many techniques that a threat actor can perform, including data exfiltration. 
- **Data exfiltration** is a non-traditional approach for copying and transferring data from a compromised to an attacker's machine. The data exfiltration technique is used to emulate the normal network activities, and It relies on network protocols such as DNS, HTTP, SSH, etc. Data Exfiltration over common protocols is challenging to detect and distinguish between legitimate and malicious traffic.
- Some protocols are not designed to carry data over them. However, threat actors find ways to abuse these protocols to bypass network-based security products such as a firewall.


## 2. Network Infrastructure
- For this room, we have built a network to simulate practical scenarios where we can perform data exfiltration and tunneling using various network protocols. 
- The provided VM contains two separated networks with multiple clients. We also have a "JumpBox" machine that accesses both networks. 
- The following diagram shows more information about the network environment used in this room.
    ![Network Diagram](/resources_redteam/data_exfil_setup.png)


## 3. Data Exfiltration
- It is important to note that Data Exfiltration is a post-compromised process where a threat actor has already gained access to a network and performed various activities to get hands on sensitive data. Data Exfiltration often happens at the last stage of the Cyber Kill Chain model, Actions on Objectives.
  ![kill chain](/resources_redteam/kill_chain.png)
- Data exfiltration is also used to hide an adversary's malicious activities and bypass security products. For example, the DNS exfiltration technique can evade security products, such as a firewall.

### 3.1 How to use Data Exfiltration
- There are three primary test case scenarios including
  - Exifilterate data
  - Command and Control communications
  - Tunnelling

### 3.2 Tradional Data Exfiltraton
- Its scenario is moving sensitive data out of the organization's network. 
- An attacker can make one or more network requests to transfer the data, depending on the data size and the protocol used. 
- Note that a threat actor does not care about the reply or response to his request. Thus, all traffic will be in one direction, from inside the network to outside. 
- Once the data is stored on the attacker's server, he logs into it and grabs the data.
![Tradional Data Exifiltraton](/resources_redteam/tradional_data_exfil.png)

### 3.3 C2 Communications
- Many C2 frameworks provide options to establish a communication channel, including standard and non-traditional protocols to send commands and receive responses from a victim machine. 
- In C2 communications a limited number of requests where an attacker sends a request to execute a command in the victim's machine. Then, the agent's client executes the command and sends a reply with the result over a non-traditional protocol. The communications will go in two directions: into and out of the network.
![C2 Data Exfiltration](/resources_redteam/c2_data_exfil.png)

### 3.4 Tunneling
- In the Tunneling scenario, an attacker uses this data exfiltration technique to establish a communication channel between a victim and an attacker's machine. 
- The communication channel acts as a bridge to let the attacker machine access the entire internal network. 
- There will be continuous traffic sent and received while establishing the connection.
![tunnelling data exfiltration](/resources_redteam/tunnelling_data_exifl.png)


## 4. Exfiltrating using TCP Socket
- Here we discuss how to exfiltrate data over TCP using data encoding. 
- Using the TCP socket is one of the data exfiltration techniques that an attacker may use in a non-secured environment where they know there are no network-based security products. 
- If we are in a well-secured environment, then this kind of exfiltration is not recommended. This exfiltration type is easy to detect because we rely on non-standard protocols.
- Besides the TCP socket, we will also use various other techniques, including data encoding and archiving. One of the benefits of this technique is that it encodes the data during transmission and makes it harder to examine.
- The following diagram explains how traditional communications over TCP work. If two machines want to communicate, then one of them has to listen and wait for the incoming traffic. It is similar to how two people talk and communicate, where one of them is listening, and the other person is speaking.
  ![Data Exfil over tcp socket](/resources_redteam/data_exfil_over_tcp_socket.png)







