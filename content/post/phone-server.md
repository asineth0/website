+++
title = "Using an Android as a SBC/Pi alternative"
date = "2020-08-17"
author = "Asineth"
cover = "sbc.jpg"
description = "Cheap low-powered Minecraft box?"
+++

# Motivation

I have old Android phones lying around and wanted to host a Minecraft server
for friends. I initially thought of buying an SBC such as Pi or ODROID, instead
I though to try and use an old phone instead.

# How

My setup works by configuring an SSH server, a chroot with a Linux distro that
supports aarch64, and configuring init scripts so that Android's system UI gets
stopped and doesn't take up resources.

# Setup

I expect that you have an Android device and an unlocked bootloader.

1. Flash [LineageOS](https://lineageos.org) (recommended)

2. Flash [Magisk](https://www.xda-developers.com/how-to-install-magisk/)

3. Install "SSH for Magisk" (from Magisk Manager app)

4. Enable USB debugging and get a root shell

```
adb shell
su
```

5. Add your SSH keys

```
nano /data/ssh/root/.ssh/authorized_keys
```

6. SSH into the device

```
ssh root@<device ip>
```

7. Set up Alpine rootfs

For this step, I use [Alpine](https://alpinelinux.org) but you can use any
distro's rootfs as long as it's compiled for aarch64.

```
mkdir -p alpine && cd alpine
curl -LO http://dl-cdn.alpinelinux.org/alpine/v3.12/releases/aarch64/alpine-minirootfs-3.12.0-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.12.0-aarch64.tar.gz
rm -f alpine-minirootfs-3.12.0-aarch64.tar.gz
cat << ! > etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
!
cd ..
```

8. Create chrot scripts

The chroot scripts are what allow you to use the rootfs as if it were natively
installed by mounting important filesystems.

```
nano chroot.sh
chmod +x chroot.sh
```

chroot.sh:

```
#!/bin/sh
for i in dev dev/pts sys proc; do
	mount | grep alpine/$i > /dev/null || mount -o bind /$i alpine/$i;
done
chroot alpine /bin/login -f root
```

9. Create boot scripts (optional)

These scripts will stop the SystemUI and other stuff Android normally has
running, therefore freeing up RAM.

```
nano boot.sh
chmod +x boot.sh
```

boot.sh:

```
#!/bin/sh
stop
sysctl -w vm.drop_caches=3
```

We also have to remount the system partition so we can write to it, allowing us
to modify /init.rc to have our script run at startup.

```
mount -o remount,rw /
nano /init.rc
```

Find the line that has ``sys.boot_completed=1`` on it and add this under it.

```
exec u:r:magisk:s0 root root -- /system/bin/sh /data/ssh/root/boot.sh
```

Once this is set up, reboot the device.

10. Auto-chroot on SSH login (optional)

This can be done with SSH forced commands. In your ``authorized_keys`` file,
prefix your public key with the chroot script, like this.

```
command="./chroot.sh" ssh-ed25519 redacted asineth@dtop0
```

