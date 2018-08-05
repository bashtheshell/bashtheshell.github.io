---
title: "SSD TRIM on CentOS 7"
categories:
  - Guide
tags:
  - CentOS 7
  - discard
  - fstrim
  - SSD
  - TRIM
//last_modified_at: 2017-04-26T00:00:01-04:00
comments: true
---

If you have SSD in your Linux server, and you have no idea what SSD TRIM is, then chances are that you are probably using your SSD the wrong way all along. You should probably pause and read below.

In this excerpt, we’ll be covering very briefly how SSD TRIM works. By any mean, I’m not an expert on this subject, let alone my storage knowledge is very limited at this point. I’ll refer to other published documentations online for further read.

### **Prelude:**

You may already know that SSD and traditional hard disk are managed differently. This may be an obvious statement if you are solely perceiving this from a physical aspect, but SSD doesn’t even write, read, and erase data the same way as hard disk from a software aspect at the operating system level. SSD has a minor drawback when “overwriting” data. Rather than writing over old data in place, the old data must be deleted first before the new data can be written in its place. Hence, this causes a delay in write performance, which is why it’s important to have TRIM configured for your SSD. For further overview on how SSD works, I cannot recommend this blog site enough: [http://blog.neutrino.es/2013/howto-properly-activate-trim-for-your-ssd-on-linux-fstrim-lvm-and-dmcrypt/](http://blog.neutrino.es/2013/howto-properly-activate-trim-for-your-ssd-on-linux-fstrim-lvm-and-dmcrypt/). While the author may not be an authority on the subject, he shared sufficient details on how TRIM works on Linux. However, the article is bit dated since it didn’t cover the systemd approach, which I’ll cover in a moment.

Basically, all you need to know for CentOS 7 is that fstrim is the recommended way of configuring TRIM as opposed to using the `discard` mount option in the `/etc/fstab` file. The latter method would enable continuous TRIM, which would always erase the underlying data as soon as files are deleted on the filesystem level, constantly keeping the SSD busy. However, not all SSD hardware support the `discard` option. Most early generation SSDs lack this. The `discard` method may be ideal for your server if you are still noticing SSD write performance issue after using the fstrim, which by default trim all mounted supported filesystems on a weekly basis. You’d need to carefully evaluate which option suits you the best as fstrim would work best for most personal servers. At the time of this writing, the supported filesystems are btrfs, ext4, xfs, and jfs. By any means, this is not a complete list of supported filesystems, and I only named a few as they are widely used.

For the `discard` method, according to `mount` man page, `noatime` mount option is not recommended as it can prevent certain application such as `mutt` from functioning properly despite `noatime` option being recommended by several sources for SSD optimization. The man page suggested to use `relatime` mount option instead, but I’ve yet to find documentations backing this option for SSD optimization.

### **Enable TRIM:**

To start using TRIM immediately, you can just enable the service by issuing the command, `sudo systemctl enable fstrim.timer`. The `fstrim.timer` unit file works in tandem with `fstrim.service` unit file. Those unit files can be found in the `/usr/lib/systemd/system/` directory. I’ll not elaborate on how systemd’s unit files work here but please refer to the permalink to ArchWiki article for more information: [https://wiki.archlinux.org/index.php/Systemd/Timers#Service_unit](https://wiki.archlinux.org/index.php/Systemd/Timers#Service_unit). You can also manually trim by issuing the `fstrim` command itself. As you can see in `fstrim.timer` unit file, the `OnCalendar` definition under `[Timer]` section is set to weekly by default. Editing this file directly is strongly discouraged. ArchWiki has good guide on how to make the appropriate modification to the unit files. 

This is yet another great read I highly recommend, which covers everything discussed about SSD and TRIM here: [https://wiki.archlinux.org/index.php/Solid_State_Drives](https://wiki.archlinux.org/index.php/Solid_State_Drives)


Last but not least, I should also include the TRIM instruction for LVM, which is fairly simple too. To enable TRIM for LVM, change the value of `issue_discards` option from `0` to `1` in `/etc/lvm/lvm.conf` then you are done. You may wonder why fstrim cannot support LVM. The reason is simple as LVM is not a filesystem but operates at block level. The filesystems mounted on top of the LVM stack would be able to pass the `discard` command to the underlying supported block devices with the `issue_discards` option enabled.


### **THIS ARTICLE WAS MIGRATED FROM WORDPRESS.COM**
