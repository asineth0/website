+++
title = "Unofficial ROMs for Moto G8/G8 Power"
date = "2020-09-14"
author = "Asineth"
cover = "android.jpg"
description = "LineageOS 17.1 & Pixel Experience 10 for the SD 635 Motos."
+++

# Status

Both ROMs are currently very stable and I've used them for multiple hours with
minimal or no issues. I may port Bliss or some other ROMs later.

# Downloads

Last updated: 2020.09.17 @ 21:55 EDT

* lineage-17.1-20200918-UNOFFICIAL-sofiar.zip
[MEGA](https://mega.nz/file/bAYlHQ5J#CsB51snT_u3rb5FyyPaKiublgO1xW5LfeUj9_xNGHbw)
[pCloud](https://u.pcloud.link/publink/show?code=XZVJB5XZDaPlYbJPQLzmF0wvvgCo10DhMh77)

SHA256: ba7445fbadada6edbe12b55ecfce0d4bd2a7ca1b407a5f0a44a42b8e2eb27e99

* PixelExperience_Plus_sofiar-10.0-20200918-0058-UNOFFICIAL.zip
[MEGA](https://mega.nz/file/SdJ2XA6S#Ambrl7vXE2g4WmuMATztqoetatjZ5WjRItc9YvIWuWw)
[pCloud](https://u.pcloud.link/publink/show?code=XZf6f5XZU9UqeNInhMkCX8K08ixHbFgT0PDk)

SHA256: 6e6a3f050e3bf8227abf5d46a616d772515884d1c2b6256777f51518158877cb

# Flashing

1. Unlock bootloader

You can unlock your bootloader at 
[Motorola's website](https://motorola-global-portal.custhelp.com/app/standalone/bootloader/unlock-your-device-a).

2. Boot into ``fastbootd``

Reboot your device into the bootloader by holding power + volume down. Once it's
detected, reboot it into ``fastbootd`` mode.

```
fastboot devices
fastboot reboot fastboot
```

If this fails, it's likely you flashed TWRP or replaced your recovery partition,
if so, you can download the latest stock firmware for your device from
[Lolinet](https://mirrors.lolinet.com/firmware/moto/). Flash the recovery.img
like so.

```
fastboot flash recovery recovery.img
```

Once flashed, boot into ``fastbootd``.

3. Flash images

Download and extract the desired zip above. Change to the directory it's 
extracted to and flash the images to the device as so. Once done, we wipe the 
phone's ``userdata`` partition.

```
fastboot set_active a
fastboot flash boot boot.img
fastboot flash system system.img
fastboot flash product product.img
fastboot flash vbmeta vbmeta.img
fastboot -w
```

4. Flash Magisk & GApps (optional)

**Pixel Experience has GApps built in.**

If at this stage, you may want to flash Magisk for root, or GApps to access the 
Play Store. To do so, download 
[TWRP](https://mega.nz/file/jVZjCaKK#OdCGje8Qs2z49tmpyqxsx17iOQigDxPXg1Ssa9-z84k)
and temporarily boot your device with it. Do not flash it to the recovery
partition, as you will lose ``fastbootd``.

```
fastboot boot twrp-3.4.0-0-rav-test6.img
```

Once TWRP is booted, push the ZIPs to /sdcard.

```
adb push Magisk-v20.4.zip /sdcard
adb push open_gapps-arm64-10.0-pico-20200912.zip /sdcard
```

After that, proceed to flash them as you usually would in TWRP.

5. Reboot device into LineageOS

```
fastboot reboot
```

6. Enable Rav overlay (Moto G8/G Fast only)

Enable USB debugging in developer options first. This is needed to have the
notch behave properly. This will be fixed in a future update.

```
adb shell cmd overlay enable org.omnirom.overlay.moto.rav
```

# Working

Tested on a Moto G Fast from Amazon US.

* Display
* Touchscreen
* Wi-Fi
* Bluetooth (audio as well)
* Headphones via 3.5mm
* Internal speaker
* Moto Actions
* Calls/SMS/LTE on Verizon
* VoLTE on Verizon

# Issues

* Vibration motor is broken
* Stock wallpaper animation (fixed by changing it)
* Rav (G8/G Fast) users must enable their overlay manually

# Sources

* [Device](https://git.asineth.gq/android_device_motorola_sofiar)
* [Vendor](https://git.asineth.gq/android_vendor_motorola_sofiar)
* [Kernel](https://git.asineth.gq/android_kernel_motorola_trinket)

# Telegram

https://t.me/MotoG8Official

# Credits

* @vachounet - helped a lot, original device/vendor tree, TWRP build
* @kjjjnob - some LineageOS changes to @vachounet's tree