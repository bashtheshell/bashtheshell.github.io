---
title: "Setting up LVM Cache on CentOS 7"
categories:
  - guide
tags:
  - cache
  - CentOS 7
  - LVM
  - lvmcache
  - SSD
last_modified_at: 2017-04-26T00:00:01-04:00       
comments: true
---

You may have heard or read about _bcache_, and _LVM cache_ is not that much different from _bcache_ as they both help enhance read/write performance. I’ll not elaborate on _bcache_ here. Instead, I’ll refer to a good source that explains both technology. For this tutorial, I assume you understand the rudimentary concept of caching, and a solid understanding of LVM.

By default, `lvmcache` is installed with CentOS 7. You can view the `lvmcache` man page, and there you can find a clear summary of how LVM cache works in the **DESCRIPTION** section. Essentially, if you are using LVM on a traditional hard disk or multiple hard disks using software RAID configuration, then you’d probably benefit from LVM cache if you have a spare SSD to use.

While retaining your current LVM set up on the hard disk(s), you can conveniently set up LVM cache on SSD by simply extending your existing volume group containing the logical volumes you want to cache, to the physical volumes on the SSD. On the extended volume group you just added to the SSD, you’d need to create two logical volumes for caching. One would be for the cache pool, and another would be for the cache metadata, which is at most 1/1000 the size of the cache pool if greater than 8192 KiB or 8 MiB.

As I promised, here’s a good article that explains the difference between bcache and LVM cache, and you’ll find a good walkthrough for existing LVM set up: [ http://blog-vpodzime.rhcloud.com/?p=45]( http://blog-vpodzime.rhcloud.com/?p=45). Like I previously mentioned, you can follow the `lvmcache` man page for guidance. However, the man page doesn’t assume you have an existing logical volume you’d like to cache. With multiple practices, you’ll be able to pick up the instruction in the man page rather quickly.

There are few more things I should add that the above resources did not address. You may find that you can’t boot up your server after setting up LVM. I’ve personally ran into this issue, and it turned out I need to rebuild the initial ram disk. You can do so by running the `dracut ––regenerate-all ––force`. This appears to be a common knowledge among experienced Linux admins as any changes made to the LVM configuration would require a rebuild of initial ram disk. If you have a modern hardware that supports GPT, then I recommend you to take full advantage of GPT for your SSD as you can add at least 128 partitions. Even if your current configuration boots from a primary disk using MBR partition table, you can still take full advantage of GPT as this adds more flexibility to your storage customization. When using GPT, be sure to leave some room at the beginning of the sector as creating physical volume can wipe off partition table data with its signature. I’d suggest to have the first partition created 512 MiB after the first sector.


It’s not unheard of that thing can go awry with LVM for some reasons. I’ll provide few tips here that may help troubleshoot LVM issue. By any mean, this is not an exhaustive list of troubleshooting methods, and there’s no guarantee you can recover your lost data. With the tips below, you may gain better understanding how LVM works.

You may have accidentally format a partition containing an active PV (physical volume), VG (volume group), and/or LV (logical volume) before properly removing the volumes safely in order. When performing the `pvs`, `vgs`, or `lvs` command, you may get an warning about a missing volume. However, if you are certain that no data loss has occurred as you merely extended an existing volume group and created logical volume(s), with no critical data on it, then you may can prevent the warning message by running the command: `vgreduce ––removemissing <volume-group>` with <volume-group> being the volume group that couldn't get removed correctly before a partition reformat occurred. To force it, append the **––force** option at the end of the command. You’d need to run `vgscan` followed by `vgchange -ay`.

If the warning is still persistent, then you can try restarting the `lvm2-lvmetad.service` followed by running the `pvscan –cache` command as instructed in the `lvmetad` man page.

This is where the above troubleshooting commands get dangerous as I was unable to “reset” the PV metadata when creating PV again on the same partition number after deliberately reformatting the partition using different size and even resetting the partition table. The new PV I created would still show the old size in `pvscan` and `pvs` command outputs. However, I can see the new size in `lsblk` command. I had to resort to `pvresize <physical volume>` command to synchronize with the actual PV and set the new size. At this time, I’m still looking for efficient way to get the actual representation in real time rather than viewing the cached data after making inadvertent changes to the PV, VG, and/or LV. `dmsetup` command may be the next step as it works at a low level.

