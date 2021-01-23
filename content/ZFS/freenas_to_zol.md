Title: Moving From FreeNAS to ZFS On Linux (Part 1)
Date: 2021-01-22 20:15
Category: ZFS
Summary: Setting up ZoL with an existing pool and removing GELI encryption.
Tags: ZFS, ZoL, FreeNAS

For the past few years, I've been running a FreeNAS box at home. I've always intended it to be a multi-purpose server, as I just don't have demanding enough usage for multiple servers. When I decided on FreeNAS, it seemed to perfectly fit my needs. It had "first class" Docker and VM support in the just-relesed Corral version, and as the FreeNAS forums recommended against virtualization, I figured I'd just virtualize as needed _inside_ of FreeNAS.

FreeNAS Corral ended up being a dead end. It was quickly done away with, and the reality of living with the FreeBSD ecosystem set in for me. Bhyve tended to have many issues, my VMs were forced to use Samba shares to access disks on the same host, and the stability and performance just wasn't what I expected. After limping along for a few years, I decided to redo everything over Christmas break 2020, and move to ZFS On Linux. Debian is my distro of choice, due to their philosohpy both in regards to FOSS and stabilty, as well as being a very familiar distro for me.

As far as hardware, the machine specs are:

* Motherboard: SuperMicro
* CPU: Intel Xeon E3-1230 v5
* Memory: 32GB ECC
* HDDs: 7x 4TB WD Reds
* SSD: 1x 128GB

FreeNAS was configured with RAID-Z2, meaning I could lose 2 drives without data loss, and have ~16TB of usable space. As the Xeon processor has the module needed for essentially performance-penalty-free encryption, the drives were all encrypted via a checkbox when setting up FreeNAS - this would turn out to be a mistake.

Before embarking on the migration, I had to ensure that ZoL supported all the feature flags enabled on my pool. [This handy chart](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/Feature%20Flags.html) shows what feature flags various versions of ZoL support, and `zpool get all` will show you which feature flags are enabled on your pool. Check against the version of ZoL in your distro of choice before migrating, or your pool may not import, or will be stuck in read-only mode.

The first order of business was to install the latest stable Debian version on a spare SSD, so that I could swap back and forth to keep FreeNAS working if something went wrong. The box has no CD drive and I had none in the house. Thankfully, the SuperMicro IPMI has a setting to load a virtual CD drive via Samba share, so I set up a share with the Debian ISO on another computer. After an hour or so of fiddling, I could not get it to work. Luckily, the Java interface in the SuperMicro IPMI lets you load a virtual CD drive from an ISO on the accessing computer, which _did_ work. After a few tense moments setting up the partitions while being extremely careful not to ruin the ZFS drives, Debian was mostly installed. For some reason Grub would not install, so I used the same install disk to chroot and install Grub. Once that was complete, the system booted into Debian fine.

As FreeNAS had coddled me, I did not know the ZFS commands. A quick search showed that `zpool list` should show my pool, and `zpool import` would then make it usable. However, no matter what I tried, I _could not_ get anything to show up using the ZFS tools. Even `fdisk -l` refused to report my drives as having a ZFS partition. More searching eventually revealed that FreeNAS encryption _is not_ the same as native ZFS encryption. I thought that since Debian had a ZFS version with native encryption support, I'd be fine. It turns out that FreeNAS had encrypted my drives using `GELI`, which encrypts the drives at a lower level than what ZFS operates on. As such, the ZFS tooling in Linux wasn't "seeing" my ZFS partitions because to the ZFS tooling, the drives were just full of random data.

This was not good news, as I had no spare drives in which to put my data so I could reformat them. A 2013 forum thread [on the TrueNAS forums](https://www.truenas.com/community/threads/how-to-remove-encryption-from-a-zfs-volume-while-keeping-the-data.16467/) came to the rescue. It allows you to remove GELI in place. Ideally you'd have a backup in case something went wrong, but with an extra drive worth of redundancy via RAID-Z2 I went forward, ensuring only one drive was ever offline at a time. All credit to Patrick Hausen for the post, but I'll list the steps I took below in case the link dies.

```bash
1. Scrub your pool to make sure all disks are in good condition.

2. Get the IDs of your zpool's devices:

[root@freenas-pmh] ~# zpool status
pool: zfs
state: ONLINE
config:

NAME STATE READ WRITE CKSUM
zfs ONLINE 0 0 0
raidz2-0 ONLINE 0 0 0
gptid/b4a21304-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
gptid/b4f40dbd-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
gptid/b5527faa-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0

errors: No known data errors

3. Start with the first disk - take offline and remove geli:

[root@freenas-pmh] ~# zpool offline zfs gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32.eli
[root@freenas-pmh] ~# geli detach gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32.eli
[root@freenas-pmh] ~# zpool status
pool: zfs
state: DEGRADED
status: One or more devices has been taken offline by the administrator.
Sufficient replicas exist for the pool to continue functioning in a
degraded state.
action: Online the device using 'zpool online' or replace the device with
'zpool replace'.
config:
NAME STATE READ WRITE CKSUM
zfs DEGRADED 0 0 0
raidz2-0 DEGRADED 0 0 0
gptid/b4a21304-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
gptid/b4f40dbd-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
gptid/b5527faa-8ec4-11e2-a224-28924a2bff32.eli ONLINE 0 0 0
5939868321408276145 OFFLINE 0 0 0 was /dev/gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32.eli
errors: No known data errors

4. Replace formerly encrypted device with unencrypted one:

[root@freenas-pmh] ~# zpool replace zfs 5939868321408276145 gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32
[root@freenas-pmh] ~# zpool status
...
replacing-3 OFFLINE 0 0 0
5939868321408276145 OFFLINE 0 0 0 was /dev/gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32.eli
gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32 ONLINE 0 0 0 (resilvering)

5. Remove information about encryption from FreeNAS' config database

I'm not quite sure if this is strictly necessary. I did it, and it definitely did not hurt. FreeNAS keeps track of which devices are encrypted. So I wanted to make sure it treats the disks correctly.

[root@freenas-pmh] ~# /usr/local/bin/sqlite3 /data/freenas-v1.db "delete from storage_encrypteddisk where encrypted_provider = 'gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32';"

6. Wait for the resilvering to finish

7. Repeat 2. - 6. for each remaining disk. In the end your pool should look like this:

[root@freenas-pmh] ~# zpool status
pool: zfs
state: ONLINE
...
config:
NAME STATE READ WRITE CKSUM
zfs ONLINE 0 0 0
raidz2-0 ONLINE 0 0 0
gptid/b4a21304-8ec4-11e2-a224-28924a2bff32 ONLINE 0 0 0
gptid/b4f40dbd-8ec4-11e2-a224-28924a2bff32 ONLINE 0 0 0
gptid/b5527faa-8ec4-11e2-a224-28924a2bff32 ONLINE 0 0 0
gptid/b5ae2aed-8ec4-11e2-a224-28924a2bff32 ONLINE 0 0 0
errors: No known data errors

Note the absence of ".eli" from the device IDs. Check, if there are any entries left in the config database for encrypted disks:

[root@freenas-pmh] ~# /usr/local/bin/sqlite3 /data/freenas-v1.db "select * from storage_encrypteddisk;"

8. Stop all sharing services depending on the ZFS volume

9. Detach volume from the GUI - double check not to mark disks as new (i.e. erase them)
```

This procedure still works in 2021. It essentially just drops a drive from the pool, removes the underlying GELI encryption, and then lets ZFS resilver it which adds the data back to the now unencrypted drive. This works perfectly, as ZFS doesn't "know" about the GELI encryption anyway. On my 4TB drives, this took around 8-10 hours per drive, meaning what was supposed to be a one day task became a week long task consisting of starting the next drive every morning or night.

Once all drives were done, I exported the pool in FreeNAS, rebooted into Debian, and the drives were immediately avaiable. I exported them one more time in Debian as they were imported as adaX instead of by ID, which can cause issues when it comes time to replace a drive. The command is `zpool import -d /dev/disk/by-id <pool_name`. Unfortunately the drives now have no encryption, but as ZFS native encryption cannot yet be done in place as far as I know, that will have to be tackled another time.

I now had my ZFS pool working in Debian, and could read/write to it. Reboots showed it was auto-mounted out of the box. Yet, there's more to do. Next is ensuring scrubs happen, configuring automatic snapshots, and setting up Samba so the server is usable as a NAS box. These will be covered in the next post.
