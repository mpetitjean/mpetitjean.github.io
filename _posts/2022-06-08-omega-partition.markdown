---
layout: post
title:  "Extend memory and swap with an SD card on an Omega2S"
date:   2022-05-30 12:17:02 +0200
categories: omega
---

### TL;DR

This post describes how to use an SD card to extend the storage space and create swap for an Omega2S/Omega2S+.

### Context

# Storage 

The filesystem of the Omega has two main parts: `/rom` contains the base firmware and is read-only and `/overlay` contains the changes on top of the base firmware, typically the packages installed via `opkg`. At boot, both are used to create the entire filesystem.

The `/overlay` and `/root` can be mounted to dedicated partitions of an SD card to circumvent the harsh storage limitations of the Omega.

# Swap

Yet another partition of the SD Card can be used to virtually extend the RAM of the Omega.

### Setup

# Installation

Install the required packages via `opkg`

```
opkg update
opkg install fdisk kmod-fs-ext4 e2fsprogs swap-utils block-mount
```

# Preparation

Insert your SD Card and look for it via 

```
fdisk -l
```

One of the entries should look like the following:

```
Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

The SD card should appear under `/dev/mmcblk0`. Mine is a 16 GB SD card, detected as 14.9 GiB. If your SD card is already partitioned, the partitions will show up as `/dev/mmcblk0p1`, `/dev/mmcblk0p2`, etc. These partitions should be deleted before continuing the process.

Make sure the SD card is not currently mounted by running

```
umount /dev/mmcblk0
```
If it was not mounted, the following error will be thrown:

```
umount: can't unmount /dev/mmcblk0: Invalid argument
```

You can confirm that no swap is available by running

```
free
```
and observing in the output that all swap entries are 0.

```
             total       used       free     shared    buffers     cached
Mem:        124792      43848      80944         96       5784      15928
-/+ buffers/cache:      22136     102656
Swap:            0          0          0
```

# Create partitions

Use `fdisk` to create the partitions.

```
fdisk /dev/mmcblk0
```

A prompt will open. We will create 3 partitions: a first of 2GB for the swap, a second of 6GB for the `/overlay` and a third of roughly 8GB for `/root`.

Enter the following commands to create the 3 partitions:
- type `n` to create the swap partition
- type `p` to select the primary type
- type `1` for the partition number
- type `Enter` to accept the proposed first sector
- type `+2G` to select a size of 2GB for the partition
- type `n` to create the `/overlay` partition
- type `p` to select the primary type
- type `2` for the partition number
- type `Enter` to accept the proposed first sector
- type `+6G` to select a size of 6GB for the partition
- type `n` to create the `/root` partition
- type `p` to select the primary type
- type `3` for the partition number
- type `Enter` to accept the proposed first sector
- type `Enter` to select the last sector as to use the remaining space (should be around 8GB for a 16GB SD Card)

The full output of the `fdisk` prompt should look like this:

```
Welcome to fdisk (util-linux 2.32).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-31116287, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-31116287, default 31116287): +2G

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 2
First sector (4196352-31116287, default 4196352):
Last sector, +sectors or +size{K,M,G,T,P} (4196352-31116287, default 31116287): +6G

Created a new partition 2 of type 'Linux' and of size 6 GiB.

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (3,4, default 3): 3
First sector (16779264-31116287, default 16779264):
Last sector, +sectors or +size{K,M,G,T,P} (16779264-31116287, default 31116287):

Created a new partition 3 of type 'Linux' and of size 6.9 GiB.
```

Switch the partition type of the swap partition by pressing `t`, `1`, then `82`:

```
Command (m for help): t
Partition number (1,2,3, default 3): 1
Partition type (type L to list all types): 82

Changed type of partition 'Linux' to 'Linux swap / Solaris'.
```

You can now print the partition table that you created with the `p` command and verify that everything looks right.

```
Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3d19fbba

Device         Boot    Start      End  Sectors  Size Id Type
/dev/mmcblk0p1          2048  4196351  4194304    2G 82 Linux swap / Solaris
/dev/mmcblk0p2       4196352 16779263 12582912    6G 83 Linux
/dev/mmcblk0p3      16779264 31116287 14337024  6.9G 83 Linux
```

Write the partition with the `w` command.

```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

You can now run `fdisk -l` again and observe that your partitions have been changed.

```
Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3d19fbba

Device         Boot    Start      End  Sectors  Size Id Type
/dev/mmcblk0p1          2048  4196351  4194304    2G 82 Linux swap / Solaris
/dev/mmcblk0p2       4196352 16779263 12582912    6G 83 Linux
/dev/mmcblk0p3      16779264 31116287 14337024  6.9G 83 Linux
```

# Format the SD Card

Format both Linux partitions as ext4.

```
mkfs.ext4 /dev/mmcblk0p2
mkfs.ext4 /dev/mmcblk0p3
```

# Setup swap

Format the Linux swap partition and enable the swap.

```
mkswap /dev/mmcblk0p1
swapon /dev/mmcblk0p1
```

You can run again the `free -k` command, which should now display the new swap space:
```
             total       used       free     shared    buffers     cached
Mem:        124792      35748      89044         96       5876       7160
-/+ buffers/cache:      22712     102080
Swap:      2097148          0    2097148

```

# Mount the `/root` partition

In case you have files in your `/root` directory, move it temporarily to `/tmp`.

```
mkdir /tmp/root_temp
mv /root/* /tmp/root_temp
ls -lah /root
```

Mount the drive and write to it to confirm everything works as intended.

```
mount /dev/mmcblk0p3 /root
cd /root
echo "test" > test.txt
cat test.txt
```

You can now move all your files back to `/root`.

```
mv /tmp/root_temp/* /root
rmdir /tmp/root_temp/
```

# Mount the `/overlay` partition

Move the `/overlay` directory to the new partition with the following commands :

```
mount /dev/mmcblk0p2 /mnt/
tar -C /overlay -cvf - . | tar -C /mnt/ -xf -
umount /mnt/
```

# Automatically mount the SD Card at boot

Enable the fstab and detect the current mounts:

```
/etc/init.d/fstab enable
block detect > /etc/config/fstab
```

Edit the fstab.

```
vi /etc/config/fstab
```

Make the following changes:
- enable all mounts by switching `enabled` from `0` to `1`
- delete the `uuid` field of the swap
- change the target `/mnt/mmcblk0p2` to `/overlay`
- change the target `/mnt/mmcblk0p3` to `/root`

The resulting file should look like this:

```
config 'global'
    option  anon_swap   '0'
    option  anon_mount  '0'
    option  auto_swap   '1'
    option  auto_mount  '1'
    option  delay_root  '5'
    option  check_fs    '0'

config 'swap'
    option  device  '/dev/mmcblk0p1'
    option  enabled '1'

config 'mount'
    option  target  '/overlay'
    option  uuid    'YOUR-UUID-HERE'
    option  enabled '1'

config 'mount'
    option  target  '/root'
    option  uuid    'YOUR-UUID-HERE'
    option  enabled '1'
```
# Finish and validate

Reboot and exit the shell.

```
reboot && exit
```

SSH back into the Omega and run `df -h` to confirm that `/root` and `/overlay` have the expected available space.

```
Filesystem                Size      Used Available Use% Mounted on
/dev/root                 7.8M      7.8M         0 100% /rom
tmpfs                    60.9M     96.0K     60.8M   0% /tmp
/dev/mmcblk0p2            5.8G     24.2M      5.5G   0% /overlay
overlayfs:/overlay        5.8G     24.2M      5.5G   0% /
tmpfs                   512.0K         0    512.0K   0% /dev
/dev/mmcblk0p3            6.7G     30.8M      6.3G   0% /mnt/mmcblk0p3
/dev/mtdblock6           22.1M    700.0K     21.4M   3% /mnt/mtdblock6
/dev/mtdblock7          512.0K    244.0K    268.0K  48% /mnt/mtdblock7
```

Then check the swap with `free`. (TODO update)

```
             total       used       free     shared    buffers     cached
Mem:        125764      35868      89896        200       5256      12732
-/+ buffers/cache:      17880     107884
Swap:      2097148          0    2097148
```

You are done !

# Links

This tutorial is largely based on [this guide][guide] by GitHub user [pjobson][user] and [this guide][guide-omega] by Onion.

[guide]: https://github.com/pjobson/onion_omega2p_experiments/blob/master/docs/setting_up_sdcard_for_root_and_swap.md
[user]: https://github.com/pjobson
[guide-omega]: https://docs.onion.io/omega2-docs/boot-from-external-storage.html