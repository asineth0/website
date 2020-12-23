+++
title = "Debian 10 Root on Encrypted ZFS"
date = "2020-08-13"
author = "Asineth"
cover = "linux.jpg"
description = "Close to the perfect filesystem."
+++

# Motivation

Why would you want to use ZFS as your root filesystem?

* Snapshots and rollbacks
* Send incremental snapshots over the network
* Datasets (i.e. /home, /var, /tmp)
* Copy-on-write architecture
* Compression to save space
* Checksums for data integrity, auto-repair
* Built-in native encryption
* Flexible configuration per dataset

# Context

ZFS is awesome, why isn't it the default filesystem? Well, it mainly stems from
one thing, and that's licensing. ZFS is licensed under the CDDL, and the Linux
kenrel is licensed under the GPL. They're incompatible licenses, so generally
distributions don't ship them together. This is why ZFS doesn't have mainstream
adoption across Linux.

Debian ships DKMS packages of the ZFS kernel modules, meaning that they're
dyanmically compiled for your kernel at install time. This avoids the licensing
issue that came with ZFS.

Some distributions, such as Ubuntu are going as far as to ship ZFS without DKMS,
even enabling it by default. They believe it is 
[fine to ship](https://ubuntu.com/blog/zfs-licensing-and-linux). This is good,
and the future for ZFS is looking good. ZFS should become more common over the
coming years and hopefully more people try it out.

# Preparation

You'll need a system to install from. Any Linux distribution that supports ZFS
will work for this. I suggest using [Alpine](https://alpinelinux.org) because
of it's small size and simplicity. Make sure to download the "extended" image
under the downloads page, you'll need it for ZFS.

# Installation

1. Login as ``root``

2. Setup network connection

```
ip link set eth0 up
udhcpc -i eth0
```

3. Setup package manager

```
setup-apkrepos -1
apk upgrade
```

4. Install tools and load the ZFS kernel module

```
apk add zfs util-linux dosfstools debootstrap
modprobe zfs
```

5. Identify the target disk to install to

```
ls -l /dev/disk/by-id
```

You'll see something like this:

```
ata-WDC_WD10EZEX-00RKKA0_WD-WMC1S6721017 -> ../../sda
nvme-eui.0000000001000000e4d25c5582715101 -> ../../nvme0n1
nvme-eui.0000000001000000e4d25c5582715101-part1 -> ../../nvme0n1p1
nvme-eui.0000000001000000e4d25c5582715101-part2 -> ../../nvme0n1p2
nvme-INTEL_SSDPEKNW010T8_BTNH93852L1J1P0B -> ../../nvme0n1
nvme-INTEL_SSDPEKNW010T8_BTNH93852L1J1P0B-part1 -> ../../nvme0n1p1
nvme-INTEL_SSDPEKNW010T8_BTNH93852L1J1P0B-part2 -> ../../nvme0n1p2
wwn-0x50014ee0ae51230a -> ../../sda
```

To look at the partitions and size of a drive, we can use ``fdisk``. Replace
``nvme0n1`` with the name that the symlink above points to.

```
fdisk -l /dev/sda
```

6. Partition the drive

We'll be using ``fdisk`` but feel free to use whatever you like.

```
fdisk /dev/sda
```

First, we create a new GPT partition table.

```
Command (m for help): g
Created a new GPT disklabel (GUID: C9D0492C-81C5-844B-8D73-FA530B1F66DE).
```

For systems without UEFI, we create a partition for the bootloader.

```
Command (m for help): n 
Partition number (1-128, default 1): 
First sector (2048-1953516877, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953516877, default 1953516877): +1000K

Created a new partition 1 of type 'Linux filesystem' and of size 1 MiB.

Command (m for help): t

Selected partition 1
Partition type or alias (type L to list all): 4
Changed type of partition 'EFI System' to 'BIOS boot'.
```

For systems with UEFI, we create an EFI partition for the bootloader.

```
Command (m for help): n
Partition number (1-128, default 1): 
First sector (2048-1953516877, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953516877, default 1953516877): +100M

Created a new partition 1 of type 'Linux filesystem' and of size 100 MiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): 1
Changed type of partition 'Linux filesystem' to 'EFI System'.
```

Now, we create the ``/boot`` partition that will store our kernel, initrd,
and bootloader configuration.

```
Command (m for help): n
Partition number (2-128, default 2): 
First sector (206848-1953516877, default 206848): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-1953516877, default 1953516877): +512M

Created a new partition 2 of type 'Linux filesystem' and of size 512 MiB.
```

Finally, we create our partition for ZFS.

```
Command (m for help): n
Partition number (3-128, default 3): 
First sector (1255424-1953516877, default 1255424): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1255424-1953516877, default 1953516877): 

Created a new partition 3 of type 'Linux filesystem' and of size 930.9 GiB.
```

To finish it off, we write our changes to the disk.

```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

7. Create the ZFS pool

```
zpool create \
    -R /mnt \
    -O canmount=off \
    -O mountpoint=none \
    -O encryption=aes-256-gcm \
    -O keyformat=passphrase \
    -O compression=lz4 \
    -O acltype=posixacl \
    -O atime=off \
    tank /dev/disk/by-id/<your disk here>-part3
```

8. Create the ZFS datasets

My configuration is just a matter of personal preference, you can/should change 
this to fit your needs. I create datasets for /, and then more datasets for
things where I want to tweak ZFS's behavior.

```
zfs create -o mountpoint=/ tank/root
zfs create -o mountpoint=/home tank/home
zfs create -o mountpoint=/usr tank/root/usr
zfs create -o mountpoint=/opt tank/root/opt
zfs create -o mountpoint=/srv tank/root/srv
zfs create -o mountpoint=/var tank/root/var
zfs create -o mountpoint=/var/lib tank/root/var/lib
zfs create -o compression=zle -o com.sun:auto-snapshot=off -V 8G tank/swap
zfs create -o mountpoint=/root -o com.sun:auto-snapshot=off tank/root/root
zfs create -o mountpoint=/usr/local -o com.sun:auto-snapshot=off tank/root/usr/local
zfs create -o mountpoint=/var/log -o com.sun:auto-snapshot=off tank/root/var/log
zfs create -o mountpoint=/var/tmp -o com.sun:auto-snapshot=off tank/root/var/tmp
zfs create -o mountpoint=/var/mail -o com.sun:auto-snapshot=off tank/root/var/mail
zfs create -o mountpoint=/var/spool -o com.sun:auto-snapshot=off tank/root/var/spool
zfs create -o mountpoint=/var/cache -o com.sun:auto-snapshot=off tank/root/var/cache
zfs create -o mountpoint=/tmp -o com.sun:auto-snapshot=off tank/root/tmp
```

If you opted for a swap partition, set it up.

```
mkswap /dev/zvol/tank/swap && swapon $_
```

Here's some other common datasets you may want to create.

```
zfs create -o mountpoint=/var/lib/docker -o com.sun:auto-snapshot=off tank/docker
zfs create -o mountpoint=/var/lib/libvirt -o com.sun:auto-snapshot=off tank/libvirt
```

9. Set up /boot partition

```
mkfs.ext4 /dev/sda2
mkdir -p /mnt/boot
mount -t ext4 /dev/sda2 /mnt/boot
```

10. Install packages

I opted to install Debian unstable (``sid``), but you can swap that our for
``buster`` which is the current stable branch, or ``bullseye`` for the testing
branch.

```
debootstrap --arch=amd64 sid /mnt http://deb.debian.org/debian
```

Because of ZFS's license, Debian only includes it in the ``non-free`` package
repositories. So, we must edit APT's sources file to use that repository.

```
vi /mnt/etc/apt/sources.list
```

Make sure the line that ends with ``main`` now has the repositories 
``main contrib non-free`` at the end of it.

11. Configure the system

First we must mount /dev, /sys, and /proc, then copy /etc/resolv.conf so our
DNS will resolve properly. We also add our /boot partition to our fstab. Then 
we use ``chroot`` to get a shell in the new system.

```
for i in dev sys proc; do mount /$i /mnt/$i; done
cp /etc/resolv.conf /mnt/etc
echo "/dev/sda2 /boot ext4 rw,noatime 0 1" >> /mnt/etc/fstab
chroot /mnt /bin/bash
```

Once we have a shell, we still have some packages to install and some
configuration to do before we can successfully boot for the first time.

First, we install some packages. We install the kernel, kernel headers, a
compiler for the ZFS DKMS module, ZFS utilities for our initramfs, the GRUB 2
bootloader for booting our system, and some other important stuff.

```
apt update
apt install -yq --no-install-recommends \
    linux-image-amd64 \
    linux-headers-amd64 \
    build-essential \
    zfs-dkms \
    zfs-initramfs \
    grub2-common \
    grub-pc-bin \
    grub-efi-amd64-bin \
    locales \
    tzdata \
    network-manager \
    sudo
```

For legacy/BIOS boot systems, configure the bootloader as such:

```
grub-install --target=i386-pc --force /dev/sda1
```

For UEFI-based systems, configure the bootloader as such:

```
mkdir -p /boot/efi
mkfs.vfat -F 32 /dev/sda1
mount -t vfat /dev/sda1 /boot/efi
echo "/dev/sda1 /boot/efi vfat rw,noatime,umask=077 0 1" >> /etc/fstab
grub-install --target=x86_64-pc --force --efi-directory=/boot/efi
```

Now, update the initramfs and bootloader configuration.

```
update-initramfs -u -k all
update-grub
```

To finish up, set up the system's locale, timezone, and create a user account 
that will be able to use ``sudo``.

```
dpkg-reconfigure locales
dpkg-reconfigure tzdata
useradd -m -G audio,video,input,sudo <username>
passwd <username>
```

Now, exit the chroot.

```
exit
```

12. Cleaning up

Now that we have finished our install, it is time to unmount everything and
reboot the system.

```
for i in dev sys proc boot/efi boot; do umount /mnt/$i; done
zpool export tank
sync
reboot -f
```

12. First boot

The first time the system boots, it may say something along the lines of "error
importing ZFS pool, import manually". This is normal on the first boot. Just
import the pool manually. This issue arises because of machine IDs, notice the
``-f`` bypasses this issue and resets the ID.

```
zpool import -f tank
```

Once done, press Control+D or type ``exit`` to continue boot as usual.

# Resources

If you want to get the most out of ZFS, take the time to read the documentation
and learn how to manage and administer your new filesystem. There's some quite
powerful & useful things you can do.

* [OpenZFS Docs](https://openzfs.github.io/openzfs-docs/)
* [Debian's ZFS Docs](https://wiki.debian.org/ZFS)