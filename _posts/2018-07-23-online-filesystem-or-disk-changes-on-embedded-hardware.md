---
title: "Online filesystem or disk changes on embedded hardware"
created: 2018-07-24
tags: 
  - embedded
  - filesystem
---

This is a tiny log of my experience expanding an `ext4` filesystem on an embedded system that had no alternate boot option. This situation is very nuanced. If you are here because you need to make filesystem or disk changes your _best course of action is to boot from a LiveUSB/CD_.

I moved very fast when making theses changes. There may be an easier and more efficent method, if you know one, please reach out to me!

<!--more-->

**Goal: Unmount the root filesystem and make structural changes without rebooting.**

```
root@linaro-alip:~# resize2fs /dev/mmcblk0p9 
resize2fs 1.42.12 (29-Aug-2014)
Filesystem at /dev/mmcblk0p9 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
[   61.307874] EXT4-fs (mmcblk0p9): resizing filesystem from 819200 to 1757691 blocks
[   61.384709] EXT4-fs warning (device mmcblk0p9): reserve_backup_gdb:968: reserved block 192 not at offset 191
[   61.394945] EXT4-fs (mmcblk0p9): resized filesystem to 1757691
Performing an on-line resize of /dev/mmcblk0p9 to 1757691 (4k) blocks.
[   61.507789] EXT4-fs warning (device mmcblk0p9): ext4_group_add:1605: No reserved GDT blocks, can't resize
resize2fs: Operation not permitted While trying to add group #25
```

Here you can see my kernel messages and the `stderr` from `resize2fs`. This error may be common if you are building squashed images for flashing to SD/MMC media.

For context my system has 1G of DRAM and was running Debian, to pull this off about 400MB of unused DRAM was needed.

**Note: I was running as root on the serial console (e.g., `ttyAMA0`) for the embedded device.**

**Step 1**: Create a DRAM-based basic root filesystem.

```bash
mkdir /tmp/tmproot
mount -t tmpfs none /tmp/tmproot
```

Now run `mount` and look for any non-root `/` mounts using your disk (if making disk changes). On my system `/boot/efi` was mounted, a `umount /boot/efi` did the trick.

**Step 2:** Move needed tools into the temporary root.

Note that later on, several commands run in the temporary root complained about not having some shared libraries. The only application I needed was `resize2fs` and `fsck` so I sort of moved fast and broke some things (temporarily).

```bash
mkdir /tmp/tmproot/{proc,sys,dev,run,usr,var,tmp,oldroot}
cp -ax /{bin,etc,sbin,lib} /tmp/tmproot/
cp -ax /usr/{bin,sbin} /tmp/tmproot/usr/
```

**Step 3:** Mount pseudo-filesystems in the temporary root.

```bash
mount --rbind /proc /tmp/tmproot/proc
mount --rbind /dev /tmp/tmproot/dev
mount --rbind /sys /tmp/tmproot/sys
```

**Step 4:** Remount the root filesystem readonly

The general guidance is to STOP as many `systemctl` (systemd) services as possible before this step. Close out remote sessions, stop as many things. In my experience after trying several times (n=4) this step always completed without complaining.

```bash
mount -r -o remount /
```

**Step 4:** Pivot your root into the temporary root filesystem.

I encounted an error here (seen below) and the 'fix' was using the `--make-private` mount command.

```
root@linaro-alip:/tmp# pivot_root /tmp/tmproot/ /tmp/tmproot/oldroot/
pivot_root: failed to change root from `/tmp/tmproot/' to `/tmp/tmproot/oldroot/': Invalid argument
```

```bash
mount --make-rprivate /
pivot_root /tmp/tmproot/ /tmp/tmproot/oldroot/
```

**Step 5:** Unmount previously mounted pseudo-filesystems

```bash
for i in dev proc sys run; do mount --move /oldroot/$i /$i; done
```

**Step 6:** Stop services using the old readonly-mounted filesystem

```bash
telinit u
systemctl isolate default.target
```

**Step 7:** Aggresively stop remaining process

I messed up a few times here. Mistakes resulted in losing my console/shell/etc and forced me to reboot. The goal is to run `fuser -vm /oldroot`, note the pids, then `kill PID` each of the PIDs.

Here is an example of the output in my situation and how I responded.

```
root@linaro-alip:/# fuser -vm /oldroot
                     USER        PID ACCESS COMMAND
/oldroot:            root     kernel mount /oldroot
                     systemd-timesync   2370 ....m systemd-timesyn
                     messagebus   2433 ....m dbus-daemon
                     root       2507 ....m agetty
```

```bash
kill 2507
kill 2433
kill 2370
```

**Step 8:** Finally unmount the old readonly-mounted root

The last command unmounts the root filesystem. You are then free to run whatever modifications needed. In this log I needed to filesystem-check and expand the filesystem to cover the entire partition.

```bash
umount /oldroot/
resize2fs /dev/mmcblk0p9
e2fsck -f /dev/mmcblk0p9
```

## References

Here is the list of documents I reviewed. Thank you to all of those who took the time to document their experience and explainations (they are more detailed than this log!)

- [https://devops.profitbricks.com/tutorials/increase-the-size-of-a-linux-root-partition-without-rebooting/](https://devops.profitbricks.com/tutorials/increase-the-size-of-a-linux-root-partition-without-rebooting/)
- [https://unix.stackexchange.com/questions/226872/how-to-shrink-root-filesystem-without-booting-a-livecd](https://unix.stackexchange.com/questions/226872/how-to-shrink-root-filesystem-without-booting-a-livecd)
- [https://www.ivarch.com/blogs/oss/2007/01/resize-a-live-root-fs-a-howto.shtml](https://www.ivarch.com/blogs/oss/2007/01/resize-a-live-root-fs-a-howto.shtml)
- [https://www.openattic.org/posts/how-to-shrink-root-filesystem-without-booting-a-livecd/](https://www.openattic.org/posts/how-to-shrink-root-filesystem-without-booting-a-livecd/) (very helpful!)
