+++
title = "Booting a GSI on the Pixel 1/1 XL"
date = "2020-07-24"
author = "Asineth"
cover = "phone.jpg"
description = "1st-gen Pixels seem not to be completely Treble-compliant."
+++

# Requirements

* Google Pixel 1/1 XL (sailfish/marlin)
* A GSI system.img to flash
* [ADB & Fastboot](https://developer.android.com/studio/releases/platform-tools) on your PC
* [TWRP](https://twrp.me/Devices/) for your device

# Getting a GSI

There are tons of different GSIs to try out. I used the Google one.

* [Google's Android 10 GSI](https://ci.android.com/builds/submitted/6704824/aosp_arm64_ab-userdebug/latest/aosp_arm64_ab-img-6704824.zip)
* [Google's Android 11 GSI](https://dl.google.com/developers/android/rvc/images/gsi/aosp_arm64-exp-RPB2.200611.012-6677315.zip)
* [phhTreble Android 10](https://github.com/phhusson/treble_experimentations/releases/download/v221/system-quack-arm64-ab-vanilla.img.xz)
* [phhTreble Android 10 + GApps](https://github.com/phhusson/treble_experimentations/releases/download/v221/system-quack-arm64-ab-gapps.img.xz)
* [phhTreble Android 10 + "Go" Gapps](https://github.com/phhusson/treble_experimentations/releases/download/v221/system-quack-arm64-ab-go.img.xz)

# Preparing the device

The ``system_a`` and ``system_b`` partitions on the Pixel are too small for the
GSI. Therefore, we must resize them. 

1. Put the device into bootloader mode.

This is done with power + volume down.

2. Boot TWRP on the device.

Open up a terminal and use ``fastboot`` to boot the TWRP image on the phone.

```
fastboot boot twrp-3.4.0-0-marlin.img
```

3. Backup the GPT

The partition table on the phone is called the GPT and it is a good idea to
back this up in case you mess anything up later on.

```
adb shell sgdisk --backup=/data/gpt.bin /dev/block/sda
adb pull /data/gpt.bin .
```

4. Resize the partitions

Now we delete the systen & userdata partitions and resize them. 

## 2.5 GB System A & B (recommended)

This is the configuration I used.

```
adb shell sgdisk --delete=33 /dev/block/sda
adb shell sgdisk --delete=34 /dev/block/sda
adb shell sgdisk --delete=35 /dev/block/sda
adb shell sgdisk --new=33::+2560M --change-name=33:system_a /dev/block/sda
adb shell sgdisk --new=34::+2560M --change-name=34:system_b /dev/block/sda
adb shell sgdisk --new=35:: --change-name=35:userdata /dev/block/sda
```

## 2.5 GB System A & empty B partition

If you insist on saving space and don't want to use OTA updates, you can just
resize the ``system_b`` partition to 1KB. I've tested this but it's a little
risky.

```
adb shell sgdisk --delete=33 /dev/block/sda
adb shell sgdisk --delete=34 /dev/block/sda
adb shell sgdisk --delete=35 /dev/block/sda
adb shell sgdisk --new=33::+2560M --change-name=33:system_a /dev/block/sda
adb shell sgdisk --new=34::+1K --change-name=34:system_b /dev/block/sda
adb shell sgdisk --new=35:: --change-name=35:userdata /dev/block/sda
```

## 2 GB System A & B (stock)

If you ever want to revert back to stock, this is what you'd do.

```
adb shell sgdisk --delete=33 /dev/block/sda
adb shell sgdisk --delete=34 /dev/block/sda
adb shell sgdisk --delete=35 /dev/block/sda
adb shell sgdisk --new=33::+2G --change-name=33:system_a /dev/block/sda
adb shell sgdisk --new=34::+2G --change-name=34:system_b /dev/block/sda
adb shell sgdisk --new=35:: --change-name=35:userdata /dev/block/sda
```

5. Format the /data partition

It's a good idea to format the ``userdata`` partition now.

```
# sometimes TWRP mounts these
adb shell umount /data
adb shell umount /sdcard
adb shell mke2fs /dev/block/sda35
adb shell mount -t ext4 /dev/block/sda35 /data
```

# Flashing the GSI

1. Flash the system partition

Now, we push the system.img to the device and write it to the ``system_a`` 
partition. Even if you opted to keep the ``system_b`` partition, do not write
it to the other partition yet.

```
adb push system.img /data/system.img
adb shell dd if=/data/system.img of=/dev/block/sda33
adb shell rm -f /data/system.img
```

2. Resize filesystem to partition

To finish up, we run a filesystem check and then resize the image to the full
size of the partition.

```
adb shell e2fsck /dev/block/sda33 -fy
adb shell resize2fs /dev/block/sda33
```

# Finishing up

1. Mount the system partition

Now we mount the ``system_a`` partition.

```
adb shell mount /dev/block/sda33 /system_root
```

2. Extract libraries from factory image

The GSI will fail to boot if we don't manually add some libraries to the
``system_a`` partition. We can do this by extracting them from the factory
image from Google for the phone. You can get that image 
[here](https://developers.google.com/android/images).

Once you have the image downloaded, extract the zip. Inside there will be
another zip called ``image-XXXXXX-qp1a.191005.007.a3.zip`` or similar. Extract
that as well. Now you should have a system.img file. Now we must extract that.

I opted to use p7zip, although you could simply mount it or use something else,
I was having issues mounting it.

```
7z x system.img -o out
```

3. Transfer libraries to device

Now we transfer the libraries we extracted, to the device

```
cd out/system
adb push lib/android.hardware.audio.common-util.so /system/lib/vndk-29
adb push lib/android.hardware.audio.common@5.0-util.so /system/lib/vndk-29
adb push lib/libeffectsconfig.so /system/lib/vndk-29
adb push lib64/vendor.qti.qcril.am@1.0.so /system/lib64/vndk-29
```

Once the files are copied, we must fix their permissions.

```
for i in \
    lib/vndk-29/android.hardware.audio.common-util.so \
    lib/vndk-29/android.hardware.audio.common@5.0-util.so \
    lib/vndk-29/libeffectsconfig.so \
    lib64/vndk-29/vendor.qti.qcril.am@1.0.so; do
    adb shell chcon u:object_r:system_lib_file:s0 "/system/$i"
done
```

If you opted to keep the ``system_b`` partition, now you copy the contents
of the ``system_a`` partition over to the other.

```
adb shell umount /system_root
adb shell dd if=/dev/block/sda33 of=/dev/block/sda34
```

4. Copy ADB key to device

Now, we copy the ADB public key for the PC to the device so that we can get
output from the device while it's booting for debugging, as well as getting a
shell if we need to.

```
adb shell mkdir -p /data/misc/adb
adb push ~/.android/adbkey.pub /data/misc/adb/adb_keys
```

5. Reboot into Android

Now we just tell the device to reboot through ADB.

```
adb reboot
```

# Credits

* [WaseemAlkurdi](https://github.com/phhusson/treble_experimentations/issues/1196)
  GitHub issue on the missing libraries, general help

* [Wonderlooo](https://forum.xda-developers.com/pixel-xl/how-to/guide-expand-partition-pixel-xl-pixel-t4097839)
  XDA post on resizing the system partitions