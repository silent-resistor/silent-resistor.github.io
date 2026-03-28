+++
title = "WiFi Hack Setup Guide - Atheros AR9271"
date = "2026-03-29"
author = "Silent Resistor"
+++

## Overview
This guide walks you through the painful process of weaponizing an Atheros AR9271 WiFi adapter with your Linux machine and VM running inside it on KVM/QEMU. 

## Prerequisites
- A Linux host machine with KVM/QEMU
- VM on KVM/QEMU
- An Atheros AR9271 WiFi adapter


## Chapter 1: Host Machine Reconnaissance
### 1.1 Identifying the WiFi Adapter
- Recently I purchased a WiFi adapter for my pentesting lab on Amazon. I verified clearly in the product description that it supports monitor mode and packet injection: **"2.4GHz 802.11n 150Mbps USB Wireless WiFi Adapter with 3dBi External Antenna for Atheros AR9271 Kali Linux Ubuntu Centos Desktop Windows, Black"** 
- After receiving it, I noticed there's no proper white label or branding on the product body indicating it belongs to a particular company. I'm a bit tense because I thought it might be a duplicate product. But after a little research, I found that this is quite common for these generic adapters.
- So, without wasting more time, I plugged the device into my Linux host machine and checked if it was detected.
    ```bash
    # Initially I checked it with lspci, and it turns out it only lists devices
    # connected to the PCI bus like graphics cards, internal WiFi cards, etc.
    lspci | grep Network
    # 04:00.0 Network controller: MEDIATEK Corp. Device 9999

    # Since the inserted device is a USB wireless adapter, it connects via USB bus
    # Its vendor_id: 0xfw, product_id: 0x9213
    lsusb | grep -i AR9271
    # Bus 001 Device 010: ID 0xfw:9213 Qualcomm Atheros Communications AR9271 802.11n

    # It's good to know that the device is recognized, now let's check the connections
    ip link show
    # 20: wlxbc307e231dg8: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DORMANT group default qlen 1000
    #     link/ether 16:16:36:71:f7:d8 brd ff:ff:ff:ff:ff:ff permaddr bc:33:7e:29:2c:f1
    ```
  - It seems all good, please make note of the interface name `wlxbc307e231dg8` and the MAC address `bc:33:7e:29:2c:f1`.
  - I understand the weird name of the interface, and we can try to rename it to something shorter in the next steps.

### 1.2 Testing Monitor Mode and Packet Injection in Linux Host
- Without thinking much, I started playing with the Aircrack-ng suite.
    ```bash
    # Killing the processes that might try to take control of the device
    sudo airmon-ng check kill

    # Let's put the device in monitor mode
    sudo airmon-ng start wlxbc307e231dg8

    # Now check if the device is in monitor mode, and
    # Note that its interface name is changed or appended with "mon", in my case it's wlan1
    ip link show

    # Test packet injection - sending spoofed packets, observe if it says injection is working.
    # Let's connect an antenna to the device if available to increase the range.
    sudo aireplay-ng --9 wlan1

    # To verify the antenna is picking up traffic, run airodump-ng
    sudo airodump-ng wlan1

    ```
- All looks good, except for the fact that I didn't notice something burning on the other side. I lost the internet connection, and that's when reality hit me hard. 
- When we run `airmon-ng check kill`, we are essentially killing the `NetworkManager` and `wpa_supplicant` services. Because the internal MediaTek card relies on those exact same processes to route your internet traffic, the entire host machine goes offline when you run it.
- But anyway, we need to kill these processes somehow or tell those services to stop interfering with the device's monitor mode to use it for wireless packet capture. 
- A simple solution is to tell the Network Manager to not touch the interface we're using for monitor mode. This way, we can have our cake and eat it too - keeping internet while doing wireless reconnaissance.
    ```bash
    # Please also note that Network Manager won't crash even if it's not able to find the mentioned interfaces - it will simply ignore them.
    sudo vi /etc/NetworkManager/NetworkManager.conf
    # Add the below lines at the end of the file
    [keyfile]
    unmanaged-devices=interface-name:wlxbc307e231dg8

    # Now restart the Network Manager service to apply the changes.
    sudo systemctl restart NetworkManager

    # Check now - the device connection should be changed to unmanaged.
    nmcli device
    ```
- With this change, we can now put the device in monitor mode without losing our internet connection, but only under these conditions:
  - Do NOT run `airmon-ng check kill`.
  - Just ignore that warning about NetworkManager interfering. Since we set the Atheros card to unmanaged in the config file earlier, NetworkManager shouldn't actually interfere with it, even if the service is still running for your internal card.

- Now let's test the monitor mode and packet injection again and see if it's working. This is the moment of truth.


### 1.3 Changing the Interface Name of the WiFi Device
- After all this shit, I still couldn't stop thinking about changing the interface name of this WiFi device. By default, its name is a nightmare - `wlxbc307e231dg8`, which is a total pain to type every time. Sometimes the Linux kernel completely fucks up the interface name when changing to monitor mode instead of just appending `mon` like it should. 
- The problem is, we can't simply use `ip link set <old_name> name <new_name>` because when you unplug this device and plug it back in, the interface name resets to that garbage default name again.
- There has to be some persistent way to change this interface name, like creating a udev rule or some other Linux magic.
- Yeah, we'll create a rule that says: "Whenever a device with this specific hardware MAC shows up, name it wlan1 so I don't lose my mind typing that long-ass name."
    ```bash
    # Get the current interface name so we know what we're dealing with
    ip link show

    # Get the original MAC address of this device - our unique identifier
    cat /sys/class/net/<current_interface_name>/address

    sudo vi /etc/udev/rules.d/70-persistent-cd.rules
    # Add the following line to the file - replace the placeholders with your actual values
    # SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="<original_mac_address>", NAME="<new_interface_name>"
    SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="bc:33:7e:29:2c:f1", NAME="wlan1"

    # Now reload the udev rules and trigger this bad boy to apply our changes
    sudo udevadm control --reload
    sudo udevadm trigger

    # Update the NetworkManager config with our new, sane interface name
    sudo vi /etc/NetworkManager/NetworkManager.conf
    [keyfile]
    unmanaged-devices=interface-name:wlan1

    # Now restart Network Manager service to make it all work together
    sudo systemctl restart NetworkManager

    # Make sure the connection is unmanaged.
    nmcli devices
    ```
- Now we can use `wlan1` instead of that nightmare `wlxbc307e231dg8`. Finally, some sanity in this setup process!
  



## Chapter 2: VM Setup and USB Passthrough
- In my case, I have a Parrot VM running on KVM/QEMU.
- **Important:** Kindly remove the udev configurations we did in the previous chapter.
  - Go to `/etc/udev/rules.d/70-persistent-cd.rules` and remove the line we added.
  - Reload and trigger the udev rules: `sudo udevadm control --reload`, `sudo udevadm trigger`
  - Update the NetworkManager config to unmanage the interface with its original name: `sudo vi /etc/NetworkManager/NetworkManager.conf`
  - Otherwise, we may experience a classic "race condition" between Host and VM over who gets to "own" the USB device when it's first plugged in.
  - As the Host touches the USB device first here for applying udev rules, the VM thinks it's already owned by someone else and never sees it.
- Before diving into the action, let's clear up the difference between **Bridge/NAT** and **USB Passthrough** in VM configuration.
- **Bridge/NAT** networking is for internet connection - the VM uses a "virtual" ethernet cable connected to your host, and it thinks it's plugged into a router, but it has no access to actual WiFi radio waves.
- **USB Passthrough** is like giving the physical USB device directly to the VM. The host machine "loses" the device, and the VM sees it as if it were plugged directly into its own virtual motherboard. 
    - So, when we perform `passthrough`, we aren't talking about IP addresses or bridges - we're passing raw hardware data to the VM.
    - **Host side:** Linux kernel sees the AR9271 device, but doesn't attach a driver to it because KVM claimed it.
    - **VM Side:** The VM sees a new hardware device on the USB bus and loads the `ath9k_htc` driver locally.
    - **Result:** The VM can have its own `wlan0` interface, which can scan and inject.

- So, once we've done the USB passthrough, can the host machine see the device and make use of it?
  - Yes, unless the VM is started, the host can see it and use it. But once the VM is started, the host can't see it anymore.
- Yeah, that's enough for the theory. Let's move on to the action.
- Add the USB device to the VM configuration.
  - Open Parrot VM settings in virt-manager.
  - Click Add hardware
  - Select USB Host Device
  - Find the entry: `Atheros Communications AR9271 802.11"
  - Finish and start the VM.
- Now, let's verify if the VM can see the device.
    ```bash
    # One thing to notice: if the VM is a guest addition, it may not contain the AR9271 driver by default.
    # In that case, we need to install the driver manually.
    sudo apt install firmware-atheros

    # OK, let's check if the VM can see the device - hope you're ready for this.
    lsusb

    # And check the connections
    ip link show


    # If the device connection is not showing up, let's debug...
    # Check if the driver is loaded properly
    lsmod | grep ath9k_htc
    # If not, load it with the below command or just restart the VM.
    # sudo modprobe ath9k_htc


    # See if there are any errors in the kernel log while loading the driver
    dmesg | grep -i ath9k
    # If it says something like "firmware: failed to load ath9k_htc/htc_9271-1.4.0.fw (-2)"
    # Then we need to remove and reinstall the firmware.
    sudo apt remove --purge firmware-atheros
    sudo apt install firmware-atheros


    # If something's still not working, let's try to upgrade the machine
    sudo apt update
    sudo apt upgrade

    ```

- Alright, if all goes well, we should be able to see the device in the VM.
- Now, let's try to play with `aireplay-ng` in the VM as we did in the previous chapter with the host machine.



