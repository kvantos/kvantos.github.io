---
title: "Udev Device Mapping for Daily Tasks"
date: 2020-12-06T22:36:36+02:00
description: "UDEV is tool to set persistent device names, desired access rights and much more for Linux devices."
draft: false
featured_image: "/img/good_bytes.jpg"
tags: [linux, udev, system]
categories: Linux
---

Does USB device should always have specific access rights? Or network cards must have specific names? For such questions, and even more, Linux has a standard tool in its arsenal -- UDEV. UDEV is a userspace system that allows you to identify devices based on their properties, and register userspace handlers.

## Set persistent name for device.
The task is to make persistent drive name. To do this, you need to create a new UDEV rule. It is better to create your own separate config for custom rules:
```
[root@amigo rules.d]# cd /etc/udev/rules.d
[root@amigo rules.d]# touch 30-local.rules
```

The rule itself may look like this:

```
## Persistent name for removable USB hdd used for storage data
SUBSYSTEMS=="usb", SUBSYSTEM=="block", KERNEL=="sd?1", ATTRS{serial}=="124689F86033", NAME="%k", SYMLINK+="usbhdd1"
```

UDEV allows you to choose the required device in a rather delicate way. You can get all parameters of UDEV rules from the documentation. Now we will focus on the `ATTRS {serial}` parameter, which allows us to isolate the required drive on the USB bus by the serial number `SUBSYSTEMS ==" usb ", SUBSYSTEM ==" block "`. The permanent name is set with the command `SYMLINK + =" usbhdd1 "`. To find the serial number, execute the following command, substituting the drive path with your:

``` sh
[root@amigo rules.d]# udevadm info -a -p /sys/block/sda | grep serial
    ATTRS{serial}=="124689F86033"
    ATTRS{serial}=="0000:06:00.2"
```

## Automatically set access for device.

By default, only the **root** user has access to the scanner. For simple users to be able to use the scanner, you need to set access rights. You can set access rights to the device each time the scanner is connected, or you can use UDEV to assign the necessary rights automatically. On my system, after installing SANE, also configuration file `/etc/udev/rules.d/60-libsane.rules` installed.

I quickly found my scanner:

```
# Mustek BearPaw 2448TA Pro 
SYSFS{idVendor}=="055f", SYSFS{idProduct}=="0409", SYMLINK+="scanner-%k"
```

Instead of granting rights to an individual user, it is better to give access to a group. The modified rule now looks like this:

```
SYSFS{idVendor}=="055f", SYSFS{idProduct}=="0409", GROUP:="scanner", MODE:="0660"
```

There was no `scanner` group on my system, so I added a group and included my user in it:

``` sh
[root@amigo rules.d]$ groupadd -r scanner
[root@amigo rules.d]$ usermod -G scanner USERNAME
```

This concludes the small journey into the world of UDEV automation. For those who interested in knowing more about UDEV, here are the links:

* [https://wiki.archlinux.org/index.php/Udev](https://wiki.archlinux.org/index.php/Udev)
* [https://wiki.debian.org/udev](https://wiki.debian.org/udev)



