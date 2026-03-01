+++
title = "Unlock Bootloader in Android, Root it, and Install Kali Net Hunter"
date = "2026-02-28"
author = "Silent Resistor"
+++

## 1. Overview:
This miserable guide will drag you through the painful process of unlocking your Android device's bootloader, rooting it, and installing Kali Linux NetHunter. Follow these steps desperately if you want to succeed, or don't, whatever.
### Table of contents
- [1. Overview:](#1-overview)
  - [Table of contents](#table-of-contents)
- [2. Prerequisites:](#2-prerequisites)
  - [2.1 Hardware Requirements](#21-hardware-requirements)
  - [2.2 Software Requirements](#22-software-requirements)
  - [2.3 Setting Up USB Debugging](#23-setting-up-usb-debugging)
  - [2.4 Understand some theory behind bootloader unlocking](#24-understand-some-theory-behind-bootloader-unlocking)
- [3. An Exploitation](#3-an-exploitation)
  - [3.1 Setting up the environment](#31-setting-up-the-environment)
  - [3.2 The "Button" Method to get BROM access](#32-the-button-method-to-get-brom-access)
  - [3.3 The Hardware unlock](#33-the-hardware-unlock)
  - [3.4 Erase the corrupted Data states](#34-erase-the-corrupted-data-states)
  - [3.5 Verification of unlocked bootloader](#35-verification-of-unlocked-bootloader)
- [4. The Rooting Process](#4-the-rooting-process)
  - [4.1 Some Basics](#41-some-basics)
  - [4.2 Get the Phone a Brain (Install Fresh OS)](#42-get-the-phone-a-brain-install-fresh-os)
  - [4.3 Rooting for Nethunter](#43-rooting-for-nethunter)
  - [4.4 Installing Kali NetHunter](#44-installing-kali-nethunter)
- [5. A Surgical Strike on all trackers and De-Googling](#5-a-surgical-strike-on-all-trackers-and-de-googling)
- [6. How do we use Kali NetHunter (Phone's) on our Laptop?](#6-how-do-we-use-kali-nethunter-phones-on-our-laptop)


## 2. Prerequisites:

### 2.1 Hardware Requirements
- Some Android device that hopefully works with NetHunter 
- A USB cable that hopefully isn't broken
- Linux computer (Ubuntu/Debian, because we're all suffering anyway)

### 2.2 Software Requirements
- ADB (Android Debug Bridge) and Fastboot tools
- Android Studio (optional, comes with those platform tools we need)
- Let's figure out how to install these damn tools with one of these two desperate options:
```bash
# Option:1
# Install ADB and Fastboot using package manager
sudo apt update
sudo apt install android-tools-adb android-tools-fastboot
```
```bash
# Option:2
# If you have Android Studio installed, the tools are located at:
ls -l $HOME/Android/Sdk/platform-tools/
# Add to your PATH for easier access:
echo 'export PATH=$PATH:/path/to/Android/Sdk/platform-tools' >> ~/.bashrc
source ~/.bashrc
```
### 2.3 Setting Up USB Debugging
- Enable Developer Options on Android
  - Go to **Settings** → **About Phone**
  - Tap **Build Number** 7 times to enable Developer Options
  - Go back to **Settings** → **Developer Options**
  - Enable **USB Debugging** and **OEM Unlocking**

- Connect the Android device via USB and let's do a pathetic test run on the Linux side to see if this piece of crap gets detected:
  ```bash
  # Check If device is detected via ADB
  adb devices
  # Expected output:
  # List of devices attached
  # <device_id>    device

  # If device is not listed, check USB debugging and try again
  # You may need to authorize the computer on the device
  ```

- Let's also check if the device is being detected through fastboot. Get the Android device into fastboot mode.
  - Power off the device
  - Hold **Volume Down + Power** buttons simultaneously until you feel the vibration and see the bootloader menu.
  - Now, crawl back to the linux terminal and see if the device is detected through fastboot using these damn commands.
    ```bash
    # Check if device is detected via fastboot
    fastboot devices
    # Expected output:
    # <device_id>    fastboot

    # If device is not listed, check USB debugging and try again
    # You may need to authorize the computer on the device
    # If still not working, follow instruction explained below
    ```

- If your device isn't being detected through fastboot, it's probably because the udev rules are messed up. Let's identify this mysterious device and create the udev rules.
  ```bash
  # identify the android device that's supposedly connected via usb
  lsusb
  # Example output:
  # Bus 001 Device 035: ID 18d1:d00d Google Inc. Xiaomi Mi/Redmi 2 (fastboot)

  # And, make note of vendor ID and product id of this android thing
  # Here, in this pathetic case 
  # Vendor ID: 18d1
  # Product ID: d00d
  ```
- Creating a udev rule file because fastboot is too dumb to recognize our device
  ```bash
  # Create the udev rule file, with vim (or nano if you're weak)
  sudo vim /etc/udev/rules.d/51-android.rules

  # Paste below content in the file
  # Add the following line (replace `18d1` with your device's vendor ID):
  SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", MODE="0666", GROUP="plugdev"
  
  # Reload UDEV Rules
  sudo udevadm control --reload-rules
  sudo udevadm trigger
  ```

- Let's try to get familiar with these adb and fastboot commands (not like we have a choice)
```bash
# check if device is detected via adb when it's actually powered on, you should see the device ID (fingers crossed)
adb devices

# Reboot to bootloader, to get the device into fastboot mode (here we go again)
adb reboot bootloader

# check if device is detected via fastboot, You should see your device listed with "device" status (if the gods are smiling today).
fastboot devices

# get the device information (let's see what secrets it holds)
fastboot getvar all

# get the device specific information (because we're nosy like that)
# oem command might not exist on mediatek devices, but let's try it anyway (what have we got to lose?)
fastboot oem device-info

# Unlock the bootloader, 100% this won't work, as most devices require specific unlock keys or codes
# and mediatek devices are special snowflakes that make our lives miserable
fastboot flashing unlock

# if it's a Qualcomm device, you might get lucky with this command
# fastboot oem unlock

# if your luck is absolute garbage, you may end up seeing this device corruption message, and your device will be bricked (congratulations, you played yourself)
# "dm-verify corruption your device is corrupted, it cant be trusted and may not work properly"
# "Press power button to continue or device will power off in 58 sec" (the countdown to your misery)
```

### 2.4 Understand some theory behind bootloader unlocking
- Now, it seems like we've fallen down this rabbit hole. Let's try to understand the actual theory behind bootloader unlocking (as if that'll help us sleep at night).
- The majority of Android manufacturers lock their bootloaders primarily to maintain a "Chain of Trust" that protects the integrity of the software running on the phone. This is done for various reasons like security (Verified Boot), data protection, ecosystem and services, and DRM (Digital Rights Management) - basically to control us like puppets.
- But, usually in my case (Poco M3 Pro), Xiaomi devices are notorious for their strict security measures. They claim we can unlock the bootloader with their official tool called **Mi Flash Unlock Tool**. But honestly, this tool is completely useless, requires your MI account to be registered in the phone beforehand, and only works on Windows (the most frustrating thing ever). So if you somehow pass all those hurdles, the app will ask you to login, send an OTP (99% chance you won't receive it), and make you wait for 10 days to get an unlock code from their support team. There's more suspense in this method than a cheap thriller movie.
- After all this pathetic research, I also discovered we can unlock the bootloader by messing with the hardware inside the phone, which is obviously extremely risky and can brick your device yet again. Qualcomm calls this **EDL (Emergency Download Mode)**, and entering EDL mode typically requires opening the phone and using tweezers to create a "Test Point Short" (usually on gold-plated dots) on the motherboard while connecting the USB cable. This can bypass the locked bootloader. It puts the chip in a "raw" state where it actually listens to commands from your Linux laptop. The same process is called **BROM mode** by MediaTek devices (because why not make it more confusing?).
- Let's try to think about this differently - do we really need to unlock the bootloader using the predefined method suggested by vendors?? Maybe there's already an existing vulnerability on these chips that we can exploit to unlock the bootloader and bypass Xiaomi's (in my case) strict security measures? We can use this brilliant open-source reverse engineering tool called **mtkclient** (a Python-based tool that exploits vulnerabilities in MediaTek's Boot ROM, usually called the `BROM Exploit`). This tool will allow us to read and write partitions directly, force-unlock the bootloader, or flash stock firmware to fix any issues we inevitably cause.
- Before grabbing your tweezers (Hardware way), MediaTek devices often allow BROM entry via a button combination when the device is completely powered off. Let's see this process in detail (because we love complicated procedures):
  - You can run `mtkclient` on the Linux terminal so it's waiting and listening for BROM connection from the phone (because we love staring at terminals waiting for something to happen).
  - Ensure the phone is completely off (Force turn off by holding power button for 10-15 seconds, because sometimes these phones just don't want to die)
  - Hold Volume Down + Volume Up buttons simultaneously
  - While holding buttons, plug the USB cable into your laptop and observe its detection on the terminal (if you're lucky, you might see something happen).
- If this doesn't trigger BROM mode (sometimes the preloader blocks it if it's not totally hard-bricked), we need to grab the tweezers (here comes the fun part): 
  - Same thing again. You can run `mtkclient` on the Linux terminal so it's waiting and listening for BROM connection from the phone (because we're masochists who love repeating steps).
  - Remove the back cover and plastic shielding over the motherboard (say goodbye to your warranty)
  - Locate the specific BROM/Preloader test point on the motherboard (usually a small gold dot near CPU shielding, good luck finding it)
  - Disconnect the battery ribbon
  - Use your tweezers to short the test point pad (usually a small gold dot near the CPU shielding) directly to ground (don't slip, it's easy to break the tiny pad if you're not careful)
  - While holding the short, plug the USB cable into your laptop.
  - Once the device enters BROM mode (observe the terminal screen), you can release the tweezers (congratulations, you didn't break it... yet)

- Once the `mtkclient` sees the BROM connection, it injects the exploit to bypass the Xiaomi locks, and open raw connection to the phone's memory.
- OK, that's enough theory for now, I guess we can move on to actual action in the next sections (not like it's going to work anyway).




## 3. An Exploitation

### 3.1 Setting up the environment
- Since we have Linux, this is the perfect environment for `mtkclient` (at least something went right in our miserable lives).
- Let's follow these command line instructions (because what else are we going to do with our time?):
```bash
# Install dependencies (because nothing ever works out of the box)
sudo apt update
sudo apt install python3 python3-pip git libusb-1.0-0 python3-tk -y

# Clone the mtkclient repository
cd /opt
git clone https://github.com/bkerler/mtkclient
cd mtkclient
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# install udev rules (Crucial)
sudo cp Setup/Linux/50-android.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 3.2 The "Button" Method to get BROM access
- OK, before we proceed with tweezers and getting our hands dirty with the motherboard, let's try the easier option with holding buttons first (because who doesn't love avoiding hardware destruction?).
- Here we can often force BROM mode using the physical volume buttons, but only if `mtkclient` is actively waiting to catch it. Let's try this safer route first (because we're not completely suicidal yet).
  - Ensure the phone is off (yeah, hope you actually read how to turn it off in the section above, unlike me who always skips instructions)
  - Let's start `mtkclient` in the Linux terminal.
    ```bash
    cd /opt/mtkclient
    source .venv/bin/activate
    # starting mtkclient, which is listening for BROM connection from the phone
    python3 mtk.py printgpt
    # terminal should pause and say something like "Waiting for device..." (or just mock us with a blank screen)
    ```
  - On the phone, press and hold Volume Up and Volume Down buttons at the same time, and DO NOT press the power button (this is important).
  - While holding both volume buttons, plug the USB cable from your laptop into the phone (here goes nothing...).

- Watch your Linux terminal closely. If `mtkclient` catches the BROM connection, the terminal will instantly light up with text, identifying the chipset and listing out the **partition table** (or it'll just sit there mocking our existence).
- If you are successful here in this step, yellow text screams that it's detected BROM, bypassing the auth set by MediaTek by exploiting the BROM vulnerability and granting your Linux machine with full VIP access (finally, something went our way).

### 3.3 The Hardware unlock
- Now, we're going to use our newly gained BROM access to directly rewrite the `seccfg` partition. This is the secure partition that MediaTek uses to handle the bootloader lock status (time to mess with the core). 
- Disconnect and Power off (you know how we do that right? or did you skip that section too like I always do?)
- In Linux terminal, run the following command to rewrite the `seccfg` partition and unlock the bootloader at hardware level (here goes nothing... again):
```bash
# hope you're still in the same directory and with activated environment
python3 mtk.py da seccfg unlock
# terminal will pause and wait for BROM connection from the phone (stare at it intensely, maybe it'll work faster)
```
- Trigger BROM mode, same thing - while holding both volume buttons, plug the USB cable from laptop to phone (third time's the charm, right?).
- Watch the terminal. Once it lights up with yellow text and processes the unlock command, release the volume buttons.


### 3.4 Erase the corrupted Data states
- Because the bootloader security state is changing, Android's encrypted data partition will now panic and bootloop if we don't clear it. Now we need to erase it (because nothing can ever be simple).
- Power off the device 
- Run the command below (because we love typing commands that might brick our device):
```bash
# include additional data if you're concerned about missing something (paranoia is your friend here)
python3 mtk.py e metadata,userdata,frp
```
- Reconnect the phone using the volume button combo one last time to execute the wipe (final boss battle, let's hope we don't die).
- Once the terminal finishes, unplug the USB cable and hold the power button for about 10-15 seconds to force the phone to reboot out of BROM mode and start up normally (if it even works, that is).
- You may notice a strange symbol like an "unlocked padlock" on the screen. This means you have successfully unlocked the bootloader without ever touching the hardware and without using their official tools (you know, the ones where you'd still be waiting for an OTP after 5 years...).

### 3.5 Verification of unlocked bootloader
- It's sometimes good to have a mindset like "something is really odd, things never get this easy for me, I have to double check if this isn't just a dream" (because let's be honest, we probably messed up somewhere and this is all imaginary).
- They say it's being cautious, but only I know this is self-doubt.
- OK, let's verify this thing by getting the phone into fastboot (holding together power button + volume down button):
```bash
# Hope you are in fastboot now
# verify the fastboot connection
fastboot devices

# check unlock status, if you see "yes", you are good to go. The exploit worked!
fastboot getvar unlocked

# Detailed OEM info (you may not see it on MediaTek, anyway give it a try)
fastboot oem device-info

# check the anti-rollback index, make sure it usually has (1,2,3,4)
fastboot getvar anti
```
- Yeah, that's it. Exploit successful (surprisingly). 




## 4. The Rooting Process

### 4.1 Some Basics
- Let's try to understand some basic misconceptions. Unlocking the bootloader doesn't mean the phone is rooted.
- Think of it like a house:
  - **Unlocking bootloader (what we just did):** This is like unlocking the front door. We are allowing stuff to be brought inside.
  - **Rooting:** This is like giving yourself the master key to every locked room inside the house.
  - **The current situation:** Right now, the house is completely empty, and set on fire. We wiped data and the operating system inside is still a corrupted mess from previous attempts.
- And Kali NetHunter does not replace Android entirely, it runs on top of working Android OS, and requires root access to interact with it.
- Here is our ultimate goal:
  - Give the phone a brain (get a fresh OS inside)
  - Actually root the phone.
  - Install Kali NetHunter on top of it.



### 4.2 Get the Phone a Brain (Install Fresh OS)
- We need to grab the official Xiaomi factory image.
- Open browser in your laptop, go to a trusted firmware data base like [xiaomifirmwareupdater.com](https://xiaomifirmwareupdater.com/) or [mifirmware.com](https://mifirmware.com/).
- In the search bar, type in your phone model (e.g. Poco M3 pro 5g), select ROM, and download the latest stable version.
- Look at type column, you must download the **Fastboot** version (file extends with .tgz, around 5gb to 7gb size), not the recovery version.
- Extract the downloaded file, into `/opt` directory.
```bash
mv ~/Downlaod/camellia_in_global_images_V14.0.6.0.TKSINXM_20230922.0000.00_13.0_in_940393ef3c.tgz /opt/
cd /opt/
tar -xvf camellia_in_global_images_V14.0.6.0.TKSINXM_20230922.0000.00_13.0_in_940393ef3c.tgz
cd camellia_in_global_images_V14.0.6.0.TKSINXM_20230922.0000.00_13.0_in_940393ef3c/
# run `ls`, and check if you see the flash_all.sh
```
- Before we flash, copy the `images/boot.img` file somewhere safe on your laptop, let's say `/opt/boot.img`
- Now we're supposed to run the `flash_all.sh` script, but it might fail due to some initial prechecks in it, which might sound so dumb. Let's remove them:
```bash
# open file
vi flash_all.sh

# comment out initial checks like below
#fastboot $* getvar product 2>&1 | grep -E "^product: *camellia"
#if [ $? -ne 0 ] ; then echo "error : Missmatching image and device"; exit 1; fi
#CURRENT_ANTI_VER=1
#version=`fastboot getvar anti 2>&1 | grep "anti:" | awk -F ": " '{print $2}'`
#if [  "${version}"x == ""x ] ; then version=0 ; fi
#if [ ${version} -gt ${CURRENT_ANTI_VER} ] ; then  echo "error : current device antirollback #version is greater than this package" ; exit 1 ; fi

# And now add the PATH env, to get the fastboot bin to other users when we run the script
export PATH=$PATH:"/home/<username>/Android/Sdk/platform-tools/"
```
- Get the phone in fastboot mode and connect it to your laptop via USB cable.
- Let's follow the instructions here:
```bash
# verify the fastboot connection
fastboot devices

# let's run the flashing script
sudo sh flash_all.sh

# what do we expect?
# terminal is going to go crazy, start sending and writing dozens of partition files. Do not touch the phone and do not bump the cable.
# let it do its job. It usually takes about 5 to 10 minutes.
```
- When the script finishes, it will automatically send a `fastboot reboot` command. The phone will restart, show the POCO logo, and begin its first restart. This boot can take up to 10 minutes while rebuilding the Android file system.
- After the first boot, the phone will boot into MIUI and ask you to set up the phone. Like asking for SIM setup, Location, Google account, WiFi connection, etc. Set up the WiFi connection and then you're good to go. Enter the old password that you set earlier if it asks.
- That's it. You have given a new brain (fresh OS) to the phone.



### 4.3 Rooting for Nethunter
- Re-enable USB debugging, because we wiped out the phone completely, we need to turn the developer tools back on (hope you already know how to do this by now).
- Connect the phone to your Linux laptop and select "File transfer (MTP)" on the phone screen.
- Copy that `/opt/boot.img` file you saved earlier to the phone's `Download` directory.
- Download and install `Magisk.apk` from the internet.
   - Visit https://github.com/topjohnwu/Magisk/releases and download the latest version of Magisk. At the time of writing it's `v30.7`
   - Install it on the phone by allowing installation from unknown sources.
- Open the Magisk app and you will see a section at the top that says "Magisk" with an "Install" button next to it. Tap on install.
- Choose the option to `Select and patch a file`
- A file explorer will open, navigate to the `Download` directory and select the `boot.img` file you copied earlier. Then tap on `Let's go`.
- Magisk will now run a script on the phone. When it finishes, it will say "All done!" and output a brand new file (a master key) into your `Downloads` folder, something like `magisk_patched_27000_xxxx.img`.
- Copy the Master key (`magisk_patched_27000_xxxx.img`) to your Linux laptop, usually in the `/opt/` directory.
- Enter Fastboot mode on the phone and connect it to your Linux laptop via USB cable.
- Run the following command to flash the master key:
```bash
# verify the connection
fastboot devices

# flash the patched boot image
fastboot flash boot magisk_patched_27000_xxxx.img

# reboot the phone
fastboot reboot
```
- Once the phone has rebooted, open the Magisk app and check if it says `Installed 30.7`. 
- If yes, you have officially rooted the device (oh, great, another thing that could have gone wrong but somehow didn't).

### 4.4 Installing Kali NetHunter
- Because we are running a Xiaomi kernel, we are going to use the NetHunter App/Chroot method. This allows Kali to run seamlessly on top of a rooted Android system.
- Get the Kali NetHunter store app by visiting https://store.nethunter.com/ and downloading the app. Install the app by allowing installation from unknown sources.
- Open the Kali NetHunter store app and install NetHunter, NetHunter Terminal, NetHunter KeX apps.
- Open the installed NetHunter app, and a Magisk key will pop up and ask you to grant superuser access. Tap Grant.
- The NetHunter app will ask for a bunch of permissions. Allow all of them.
- Inside the NetHunter app, tap the menu icon in the top left, select `Kali Chroot Manager`.
- Tap `Install kali chroot` and then select download latest.
- You will be asked to choose an architecture, choose full chroot.
- Let it download and extract. The extraction process takes 15 to 30 minutes, and it might look like the app is frozen, just leave the phone screen and let it unpack.
- Once the extraction is done, you will see a button that says `Start kali chroot`, tap it.
- That's it. Kali NetHunter is officially in your phone now (surprisingly, nothing exploded).
## 5. A Surgical Strike on all trackers and De-Googling
- Since our device is rooted now, we have much more power than a standard ADB user. We can use ADB to reach "Superuser" status and actually disable or remove these trackers, bloatware, and all other Google's invasive software.
- Now, let's connect the phone back to Linux if you have removed it in case. Follow terminal instructions below:
- 
```bash
# get the shell from the phone (using adb)
adb shell

# once you get the shell, try to get root.
# if it's saying permission denied, Grant superuser access to the Android shell in Magisk
su -

# Let's remove the stuff that we don't need
# Here in Android Package manager (pm), --user 0 refers to Primary Human User profile, unlike root user in Linux.
# And note that, we're just telling Android here, that "Put this app in the basement, lock the door, don't let the primary user interface see it or run it". 
# So, it stops the app from using your RAM, battery, CPU, but base APK file is technically still sitting in the /system partitions.
pm uninstall --user 0 com.android.chrome
pm uninstall --user 0 com.android.deskclock
pm uninstall --user 0 com.android.hotwordenrollment.okgoogle
pm uninstall --user 0 com.android.mms
pm uninstall --user 0 com.android.providers.downloads.ui
pm uninstall --user 0 com.android.soundrecorder
pm uninstall --user 0 com.android.updater
pm uninstall --user 0 pm uninstall --user 0 com.android.vending
pm uninstall --user 0 com.google.android.adservices.api
pm uninstall --user 0 com.google.android.apps.googleassistant
pm uninstall --user 0 com.google.android.apps.maps
pm uninstall --user 0 com.google.android.apps.messaging
pm uninstall --user 0 com.google.android.apps.nbu.paisa.user
pm uninstall --user 0 com.google.android.apps.safetyhub
pm uninstall --user 0 com.google.android.apps.subscriptions.red
pm uninstall --user 0 com.google.android.apps.wellbeing
pm uninstall --user 0 com.google.android.configupdater
pm uninstall --user 0 com.google.android.contacts
pm uninstall --user 0 com.google.android.feedback
pm uninstall --user 0 com.google.android.gm
pm uninstall --user 0 com.google.android.gms
pm uninstall --user 0 com.google.android.gms.location.history
pm uninstall --user 0 com.google.android.googlequicksearchbox
pm uninstall --user 0 com.google.android.gsf
pm uninstall --user 0 com.google.android.marvin.talkback
pm uninstall --user 0 com.google.android.overlay.gmsconfig.personalsafety
pm uninstall --user 0 com.google.android.setupwizard
pm uninstall --user 0 com.google.android.syncadapters.calendar
pm uninstall --user 0 com.google.android.youtube
pm uninstall --user 0 com.google.mainline.adservices
pm uninstall --user 0 com.google.mainline.telemetry
pm uninstall --user 0 com.lbe.security.miui
pm uninstall --user 0 com.mi.android.globalFileexplorer
pm uninstall --user 0 com.miui.analytics
pm uninstall --user 0 com.miui.bugreport
pm uninstall --user 0 com.miui.calculator
pm uninstall --user 0 com.miui.cleaner
pm uninstall --user 0 com.miui.cloudbackup
pm uninstall --user 0 com.miui.cloudservice
pm uninstall --user 0 com.miui.compass
pm uninstall --user 0 com.miui.daemon
pm uninstall --user 0 com.miui.face
pm uninstall --user 0 com.miui.fm
pm uninstall --user 0 com.miui.fmservice
pm uninstall --user 0 com.miui.freeform
pm uninstall --user 0 com.miui.gallery
pm uninstall --user 0 com.miui.micloudsync
pm uninstall --user 0 com.miui.msa.global
pm uninstall --user 0 com.miui.notes
pm uninstall --user 0 com.miui.player
pm uninstall --user 0 com.miui.qr
pm uninstall --user 0 com.miui.screenrecorder
pm uninstall --user 0 com.miui.wallpaper.overlay
pm uninstall --user 0 com.miui.weather2
pm uninstall --user 0 com.miui.yellowpage
pm uninstall --user 0 com.miuix.editor
pm uninstall --user 0 com.xiaomi.account
pm uninstall --user 0 com.xiaomi.calendar
pm uninstall --user 0 com.xiaomi.cameratools
pm uninstall --user 0 com.xiaomi.discover
pm uninstall --user 0 com.xiaomi.glgm
pm uninstall --user 0 com.xiaomi.joyose
pm uninstall --user 0 com.xiaomi.micloud.sdk
pm uninstall --user 0 com.xiaomi.midrop
pm uninstall --user 0 com.xiaomi.mipicks
pm uninstall --user 0 com.xiaomi.payment
pm uninstall --user 0 com.xiaomi.scanner
pm uninstall --user 0 com.xiaomi.xmsf
pm uninstall --user 0 org.mipay.android.manager

# if you messed with the phone, somehow you want a specific application back on track
cmd package install-existing <package_name>

# If you wanted to see the currently installed packages, and filter them
pm list packages | grep <package_name>

# If you wanted to see all the packages, including the ones you uninstalled
pm list packages -u

# get a list of packages you uninstalled
pm list packages -u | sort | tee all_packages.txt
pm list packages | sort | tee installed_packages.txt
# get diff and list the second file packages
comm -23  all_packages.txt installed_packages.txt | cut -d ":" -F 2 > uninstalled_packages.txt 
```
- We can talk later on how to remove these crap entirely from superuser, in future. 
- But for the time being, this is it.




## 6. How do we use Kali NetHunter (Phone's) on our Laptop?
- Yeah, when I see the `adb shell` file system and `kali nethunter terminal` in the phone, it seems they look like they're from both separate worlds, that don't match each other. I mean, can't they just get along? 
- Yes, "Two worlds" Architecture
  - **World1 - Android Base (adb shell):** When we type `adb shell`, you are talking directly to the Android OS, which is built on a linux kernal, but it is heavily stripped down. It uses a very basic shell `mksh` or `sh`, and it does not have any of out pentesting tools installed in in bin folder `/system/bin/`.
  - **World2 - The Kali Chroot (NetHunter Terminal):**  When we open Nethunter app, and extract that massive "Chroot" file, you essentially created a giant folder on phone's internal storage (usually `/data/local/nhsystem/kali-armhf`). The Nethunter app uses your root access to trick the kali system into thinking that this folder is the entire universe. That's why file systems don't match with Android base.

- How to cross the bridge?
  - If you wanted to use beautiful zsh on kali terminal from your laptop via ADB, you just have to tell ADB shell to open the door and step into the kali container.
  - Follow the terminal instructions below
  ```bash
  # get the adb shell
  adb shell
  
  # now you are in the android base world, Get the root now to access nethunter kali container
  su -

  # Now, just execute the nethunter wrapper to bridge the gap, gives you a magic transition to kali terminal
  bootkali
  ```
- Yeah that's it, to get out of Kali shell, use `exit` command.






















