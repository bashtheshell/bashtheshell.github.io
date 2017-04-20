---
layout:  
title: "Setting up PXE boot server on Raspberry Pi"
excerpt:
categories: posts
tags: [apache2, automated install, CentOS 7, Debian 8, Debian Jessie, isc-dhcp-server, Jessie Lite, PXE boot server, raspberry pi, syslinux, tftpd-hpa]
share: true
modified:
comments: true
related: true
---

I finally managed to find the time to implement a PXE boot server on my Raspberry Pi with CentOS 7 as my PXE boot image.

What I used:

+ [Raspberry Pi Model B](http://www.amazon.com/Raspberry-Pi-756-8308-Motherboard-RASPBRRYPCBA512/dp/B009SQQF9C) (512 MB with only 2 USB ports)
+ Raspian: Jessie Lite ([2016-02-09-raspbian-jessie-lite.img](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2016-02-09/2016-02-09-raspbian-jessie-lite.zip))
+ CentOS 7 DVD image ([CentOS-7-x86_64-DVD-1511.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-1511.iso))
+ [8 GB SanDisk Ultra](http://www.amazon.com/SDSDU-008G-U46-SanDisk-Ultra-Class-Memory/dp/B00812K4V4) SD memory card
+ SD memory card reader
+ home router (AirPort Extreme 2nd Generation)
+ Windows 7 64-bit on my my computer*
+ [PuTTY](https://the.earth.li/~sgtatham/putty/latest/x86/putty.exe) SSH client*
+ [VirtualBox 5.0](http://download.virtualbox.org/virtualbox/5.0.16/VirtualBox-5.0.16-105871-Win.exe)*
+ 8 GB USB flash drive
+ Ethernet cable (for Raspberry Pi)

Variables I used:
```text
Raspberry Pi's IP address:		10.0.1.25
DHCP server gateway:			10.0.1.1
subnet:							10.0.1.0/24
domain name:					example.net
hostname:						pxeboot
```

For the above variables, you would need to make changes according to your environment. E.g. Your home router may have a DHCP pool for 192.168.0.0/24 network.

###### ***While they are not required, I was using my main physical computer to remote in my Raspberry Pi using PuTTY client on Windows. This is also the same physical computer I used to install CentOS on from the PXE boot server. I tested the PXE boot server using VirtualBox prior to rebooting the physical computer.**

### **Prelude:**

Here in this prelude, I’ll elaborate on how I configured my Raspberry Pi from a vanilla install, so that newcomers can figure out how to prep their Pi for the next step.

1. Assuming you are using an existing home network, connect the Raspberry Pi to the available LAN port of the home router using Ethernet cable. Do not power on the Pi yet.


2. Download *Raspian: Jessie Lite* image file to the computer with an SD memory card reader connected.


3. Insert the SD memory card in the reader. Burn the Raspian image to the SD card. If you need further instruction on this, please refer to the documentation at [raspberrypi.org](https://www.raspberrypi.org/documentation/installation/installing-images/README.md).


4. Remove the SD memory card from the reader and insert it in the Raspberry Pi. Power on the Raspberry Pi.


5. Access the home router configuration page and view the list of connected devices on the router. You may find that it has a hostname, `raspberrypi`.


6. Perform DHCP reservation for the IP address of your Raspberry Pi. It’s important that you take note of the IP address and MAC address as we’ll use the IP address several times in later steps. In my setup, the Raspberry Pi IP address is `10.0.1.25`. I suggested DHCP reservation here for the sake of convenience in case you have to reformat and start all over.


7. Open up the SSH client. Use the Pi’s IP address to remote in. e.g. `pi@10.0.1.25`


8. You should be logged in with the username _pi_ and password _raspberrypi_. Run the command: `sudo raspi-config`


9. Select the 9th option, ‘Advanced Options’ to enter the advanced settings.


10. After entering the ‘Advanced Options’, scroll down to select the ‘A0 Update’ setting as this should be the first thing before doing the rest of the steps. This will update the 'raspi-config' tool to the latest version.


11. Now you should be back on the configuration screen. Select the first option ‘Expand Filesystem’ to use the remaining space on SD card.


12. Select the 2nd option, 'Change User Password' to set a new password for user, _Pi_.


13. Change the 5th option, 'Internationalisation Options' to change the language and locale settings.


14. Select 'I1 Change Locale' to change the locale setting. ‘en_GB.UTF-8 UTF-8' is used by default. Please deselect the locale by pressing SPACE. The asterisk should disappear. If you are from the States, you’d want to use the ‘en_US.UTF-8 UTF-8’ setting. Press SPACE to select it then press ENTER to generate the new locale. Next, you will set the default locale: For ‘Default locale for the system environment:', select ‘en_US.UTF-8’ then press ENTER.


15. Go back to the ‘Internationalisation Options’ to select the ‘I2’ option. Since I’m from the eastern part of the U.S., I selected the ‘US’ geographic area and ‘Eastern’ time zone. We’re not going to revisit the ‘Internationalisation Options’ to select the last option, ‘I3 Change Keyboard Layout’ since we are using SSH to remote in headless mode.


16. Back at the 'raspi-config' screen, enter ‘Advanced Options’ so that we can modify a few things there. First, select the ‘A2 Hostname’ sub-option to change the hostname. By default the hostname is `raspberrypi`. I changed mine to `pxeboot`. Please do not use the FQDN here or else you’ll run into problems in later steps.


17. Now back at the main screen. You can press TAB until you highlighted  `<FINISH>` then press ENTER. You will be asked to reboot your raspberry pi. Select `<YES>` and press ENTER.


18. Run the command: `sudo apt-get install screen` so that you can let the long running commands run in the background. You can use `nohup` or `disown`, but `screen` command might be easier to work with.


19. Next, run the command: `screen -R` so you can reattach to the same screen using the aforementioned command and resume the work in an event you get disconnected. You can also do so much more with [screen](http://aperiodic.net/screen/quick_reference).


20. Make sure your system’s up to date: `sudo apt-get update`


### **Installing and Configuring HTTP server:**

We’re going to configure the HTTP server to export the installation tree from CentOS image.

1. Edit the `/etc/hosts` file. Append the following line:

   `10.0.1.25   pxeboot.example.net pxeboot`

2. Install the http server and text-based web browser (to test the http server):

   `sudo apt-get install apache2 elinks`

3. By default, http server should be running immediately after the install. You can confirm by running the command:

   `elinks http://localhost/`
   
   You should see a webpage with a title, “It works!”. Press Q then answer ‘Yes’ to quit elinks.

4. Now we’re going to make a virtual host. To create the document root directory for our virtual host:

   `sudo mkdir -p /var/www/pxeboot.example.net/install`

5. Create a simple test page for our virtual host:

   `echo "PXEBOOT TEST PAGE PASSED" | sudo tee /var/www/pxeboot.example.net/install/index.html`

6. Apply appropriate permissions to the subdirectories:

   `sudo chmod -R 755 /var/www`

7. To create the configuration file for our new virtual host, let’s just copy the uncommented lines from the default website config file, `000-default.conf` to `pxeboot.example.net.conf` file:

   `grep -v '#' /etc/apache2/sites-available/000-default.conf | sudo tee /etc/apache2/sites-available/pxeboot.example.net.conf`

8. In the `pxeboot.example.net.conf` file, remove the following line:

   `ServerAdmin webmaster@localhost`

   Append the following line:

   `ServerName pxeboot.example.net`

   Modify the ‘DocumentRoot’ line to point to the new directory:

   `DocumentRoot /var/www/pxeboot.example.net/install`

9. Restart the http server:

   `sudo systemctl restart apache2.service`

10. Test the virtual host:

    `elinks http://pxeboot.example.net/`

    You should see the test page you created in step 5. After you’re done, remove the test page from the `/var/www/pxeboot.example.net/install` directory.

11. You’d need to have the image of CentOS DVD file mounted on the Raspberry Pi somehow. The best way to do this is to simply write the ISO file directly to the USB flash drive. Please do not confuse this with burning the image to the flash drive as they are not the same thing.

    Depending on the file system of your USB flash drive, you may need to install additional packages to be able to mount the device to the `/mnt` directory. Since I wrote the ISO file to my USB flash drive using Windows, my USB was formatted in EXFAT file system. I had to install the `exfat-fuse` package on the Raspberry Pi. After doing so, I was able to mount my USB flash drive on `/dev/sda1` to the `/mnt` directory by executing the following command:

    `sudo mount /dev/sda1 /mnt`

    You may need to run the `lsblk` command to see where your USB flash drive is in the `/dev` directory.

12. Mount the CentOS image from the `/mnt` directory to the `/media` directory to access the installation tree:

    `sudo mount -o loop /mnt/CentOS-7-x86_64-DVD-1511.iso /media/`

13. Now switch to the `/media` directory. Recursively copy all the contents in the directory to the `/var/www/pxeboot.example.net/install` directory. You should use the verbose option to watch the progress in real time as this can take a while.

    ```bash
    cd /media
    sudo cp -Rv * /var/www/pxeboot.example.net/install/
    ```

14. Test the website again using `elinks` as mentioned in step 10. You should now see the CentOS installation tree.

15. Before moving on, you’d have to disable the default website due to lack of name resolver in our environment. This can be done with the command:

    `sudo a2dissite 000-default.conf`

    Be sure to restart the HTTP server afterward.

### **Installing and Configuring DHCP server:**

Unlike the previous HTTP steps, this setup will be very brief.

1. Install the DHCP server:

   `sudo apt-get install isc-dhcp-server`

   You may notice that the service failed to start. That’s okay as we’ll go over this briefly. Run the following command:

   `sudo systemctl status isc-dhcp-server.service`

   We get a helpful message here that tells us that we haven’t set up a subnet declaration.
   
2. Edit the `/etc/dhcp/dhcpc.conf` file. Append the following line:

   ```text
   subnet 10.0.1.0 netmask 255.255.255.0 {
        option routers 10.0.1.1;
        range 10.0.1.200 10.0.1.250;
        next-server 10.0.1.25;
        filename "pxelinux/pxelinux.0";
   }
   ```

3. Copy the `syslinux-version-arch.rpm` package from the document root directory to the ‘/tmp directory’. We’re going to get the files needed for successful PXE boot:

   `sudo cp /var/www/pxeboot.example.net/install/Packages/syslinux-4.05-12.el7.x86_64.rpm /tmp`

4. You’d need to install the `rpm2cpio` package to extract the `syslinux` package:

   `sudo apt-get install rpm2cpio`

5. Now change to `/tmp` directory and extract the `syslinux` package:

   ```bash
   cd /tmp
   sudo rpm2cpio syslinux-4.05-12.el7.x86_64.rpm | cpio -idmv
   ```
### **Installing and Configuring TFTP server:**

This is the last server we’d need to configure to complete the PXE boot server.

1. Install the TFTP server:

   `sudo apt-get install tftpd-hpa`

   _Please note that you may receive a warning that you cannot view the initial dialog at the end of the installation. This could be due to the screen size as the screen must be at least 13 lines tall and 31 columns wide._

2. In case you did not get the dialog in the previous step, you’d need to readjust your terminal size and then run the following command to get the TUI (text-based user interface) dialog again:

   `sudo dpkg-reconfigure -plow tftpd-hpa`

   If you still can’t see the dialog, then you’d be dropped to the command-line version. Please exit by pressing ‘CTRL’ and ‘C’ simultaneously and try again.You should leave other options as is and only change the TFTP root directory. By default, the directory is set at /srv/tftp. It should be set at /var/lib/tftpboot. After finishing, you can view the setting in /etc/default/tftpd-hpa. It should look like the one below:
      
   ```text
   # /etc/default/tftpd-hpa
   TFTP_USERNAME="tftp"
   TFTP_DIRECTORY="/var/lib/tftpboot"
   TFTP_ADDRESS="0.0.0.0:69"
   TFTP_OPTIONS="--secure"
   ```
3. Create the `/var/lib/tftpboot` subdirectories:

      `sudo mkdir -p /var/lib/tftpboot/pxelinux/pxelinux.cfg`

4. Now we can copy the syslinux files to `/var/lib/tftpboot/pxelinux` directory:

   `sudo cp /tmp/usr/share/syslinux/{menu.c32,pxelinux.0} /var/lib/tftpboot/pxelinux/`

5. Create the default config file in `/var/lib/tftpboot/pxelinux/pxelinux.cfg/` directory:

   `sudo vi /var/lib/tftpboot/pxelinux/pxelinux.cfg/default`

   Append the following content:

   ```text
   DEFAULT menu.c32
   TIMEOUT 100
   TOTALTIMEOUT 300
   MENU TITLE ############# PXE Boot Menu #############
   LABEL 1
        MENU LABEL ^1) Install CentOS 7
        KERNEL vmlinuz
        APPEND initrd=initrd.img ip=dhcp inst.repo=http://10.0.1.25/
        
    LABEL 2
        MENU LABEL ^2) Boot from local drive
        LOCALBOOT 0
    LABEL 3
        MENU LABEL ^3) Rescue mode
        KERNEL vmlinuz
        APPEND initrd=initrd.img ip=dhcp inst.repo=http://10.0.1.25/ inst.rescue lang=en_us keymap=us
   ```
6. Lastly, copy the `vmlinuz` and `initrd.img` files from the document root directory to the `/var/lib/tftpboot/pxelinux` directory: 

   `sudo cp -v /var/www/pxeboot.example.net/install/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/`

7. Now restart the DHCP and TFTP servers and ensure that they’re running:  

   ```text
   sudo systemctl restart isc-dhcp-server.service tftpd-hpa.service
   sudo systemctl status isc-dhcp-server.service tftpd-hpa.service
   ```

### **Conclusion:**

Hopefully the steps above work flawlessly for you. This guide should also work for  Debian 8 (namely Jessie). 

The easiest way to test this is to use VirtualBox running from the main physical computer. Make sure the VirtualBox's networking mode is set to 'bridged' to allow it to communicate with the PXE boot server directly. You would also need to create an empty VM and change the boot order to have it boot from the network card each time. You can actually install a guest this way with VirtualBox if you wish.



