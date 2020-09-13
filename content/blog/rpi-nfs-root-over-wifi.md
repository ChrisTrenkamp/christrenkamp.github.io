---
title: "Raspberry Pi: Running on an NFS root over Wifi"
date: 2018-09-22T18:30:23-04:00
draft: false
---

It seems a lot of people work directly or compile software directly on their Raspberry Pi.  I prefer to cross-compile on my desktop because it's way faster.  However, this also requires setting up a network share; transferring the SD card between the desktop and the RPI will get annoying very quickly.  While we're at it, we'd might as well use an NFS root so we can make sure the init process only starts what we need.

Gentoo's crossdev tool allows you to create your own distribution that contains only what we need.  We will be running that over an NFS root.  This should work fine for most purposes, but I was getting annoyed with being restricted to where I can plug in an ethernet cable.

This post will show you how to set up a base system for your RPI and how to boot into it as an NFS root using a Wifi connection.  This will require a Gentoo system.  You can find the reference scripts [here](https://github.com/ChrisTrenkamp/rpi-nfs-root-over-wifi).

First install crossdev...

```bash
$ emerge crossdev
```

... then create the base system by creating a cross-compile toolchain.  Check the [here](https://wiki.gentoo.org/wiki/Raspberry_Pi#Crossdev) to see which toolchain to create.  Through this post, we'll be using `armv6j-hardfloat-linux-gnueabi` because I have RPI v1 B+.

```bash
$ crossdev -S -t armv6j-hardfloat-linux-gnueabi
```

When the toolchain is done compiling, we'll need to install some basic utilities.

```bash
$ cat >> /usr/armv6j-hardfloat-linux-gnueabi/etc/portage/package.use << EOF
net-wireless/wpa_supplicant ssl
net-misc/dhcpcd -udev
EOF

$ armv6j-hardfloat-linux-gnueabi-emerge -av busybox iw linux-firmware wpa_supplicant dhcpcd
```

We need `busybox` for general usage, `iw` and `wpa_supplicant` for connecting to the Wifi network, `linux-firmware` because your Wifi adapter will undoubtly require some firmware to run, and `dhcpcd` because the default DHCP client supplied by `busybox` didn't work with the Wifi adapter.  If you know you don't need any firmware or know which firmware you need, `linux-firmware` and be removed or replaced with your adapter's specific ebuild.

Now we need to build the kernel and firmware.  Pull the Raspberry Pi fork and build it with the cross compiler.  Check [the wiki](https://wiki.gentoo.org/wiki/Raspberry_Pi#Manual_compilation) to find out which default configuration to use.

```bash
$ git clone https://github.com/raspberrypi/linux rpi
$ git clone --depth 1 https://github.com/raspberrypi/firmware -b 1.20180817

$ cd rpi
$ git checkout rpi-4.9.y

$ export \
	ARCH=arm \
	CROSS_COMPILE=armv6j-hardfloat-linux-gnueabi- \
	INSTALL_PATH=/usr/armv6j-hardfloat-linux-gnueabi/boot \
	INSTALL_MOD_PATH=/usr/armv6j-hardfloat-linux-gnueabi/

$ make bcmrpi_defconfig
$ make -j3
$ make modules_install
$ cp arch/arm/boot/Image /usr/armv6j-hardfloat-linux-gnueabi/boot/rpi-kernel
```

We need to configure the Raspberry Pi's bootloader so it will load the initramfs on boot.  So create these [boot configuration files](https://github.com/ChrisTrenkamp/rpi-nfs-root-over-wifi/tree/master/boot).

```bash
$ cat config.txt
kernel=rpi-kernel
initramfs init.cpio.gz followkernel

$ cat cmdline.txt
root=/dev/ram0
```

Now we have to create the initramfs.  First, we'll need to create the wpa_supplicant configuration for connecting to the Wifi network.

```bash
# Strip out the plain-text password
$ wpa_passphrase "wifissid" "wifipass" | grep -v '#' > ramfs/wpa_supplicant.conf
```

Set up the base directories.
```bash
$ mkdir -p /tmp/initramfs/
$ mkdir -p /tmp/initramfs/{bin,etc,dev,lib,proc,sbin,sys,usr/lib,new_root}
```

Copy the wpa_supplicant config and [init file](https://github.com/ChrisTrenkamp/rpi-nfs-root-over-wifi/blob/master/ramfs/init).  Replace 192.168.1.2 with the IP address of the machine hosting the NFS share.
```bash
$ cp ramfs/wpa_supplicant.conf /tmp/initramfs/etc
$ cp ramfs/init /tmp/initramfs/
$ chmod +x /tmp/initramfs/init
```

Copy the required initramfs binaries.
```bash
$ cp /usr/armv6j-hardfloat-linux-gnueabi/{bin/busybox,usr/sbin/{iw,wpa_supplicant},sbin/dhcpcd} /tmp/initramfs/bin/
$ ln -s busybox /tmp/initramfs/bin/sh
```

Copy the required shared libraries.
```bash
$ cp /usr/armv6j-hardfloat-linux-gnueabi/lib/{libresolv*,libnss_{dns,files}*,ld-*,libc-*,libc.so*,libpthread*,librt*,libdl*,libz.so*} /tmp/initramfs/lib
$ cp /usr/armv6j-hardfloat-linux-gnueabi/usr/lib/{libnl-*,libtommath*,libssl*,libcrypto.so*} /tmp/initramfs/usr/lib/
```

Gather a list of required kernel modules using [this script](https://github.com/ChrisTrenkamp/rpi-nfs-root-over-wifi/blob/master/modules.sh).  Replace `rtl8192cu` with the adapter you use.
```bash
$ mods="$(./modules.sh rtl8192cu ipv6 ccm ctr)"
$ cd /usr/armv6j-hardfloat-linux-gnueabi/
$ for i in $mods; do
	find lib/modules/4.9.80+/ -type f -name "${i}.ko" -exec cp --parents \{\} /tmp/initramfs/ \;
done
```

Create the script for modprobing the modules in the init script.  We need them loaded so we can use our Wifi adapter.
```bash
$ echo "$mods" | sed 's/^/modprobe /' | sed '1i#!/bin/sh' > /tmp/initramfs/etc/modules.sh
$ chmod +x /tmp/initramfs/etc/modules.sh
```

Pull in the required firmware.  Replace 8192cu with the firmware your adapter uses.
```bash
$ find lib/firmware -type f -name "*8192cu*" -exec cp --parents \{\} /tmp/initramfs/ \;
```

Pull in the `dhcpcd` configuration files.
```bash
$ find usr/share/dhcpcd/ lib/dhcpcd/ -type f -exec cp --parents \{\} /tmp/initramfs/ \;
$ cp etc/dhcpcd.conf /tmp/initramfs/etc
$ mkdir /tmp/initramfs/var/lib/dhcpcd -p
```

And finally pull it all into a CPIO archive.
```bash
$ cd /tmp/initramfs
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > /usr/armv6j-hardfloat-linux-gnueabi/boot/init.cpio.gz
```

The root filesystem is missing some directories and an init system, so copy [these files](https://github.com/ChrisTrenkamp/rpi-nfs-root-over-wifi/tree/master/root_files) to the NFS root.
```bash
$ ln -s /proc/mounts /usr/armv6j-hardfloat-linux-gnueabi/etc/mtab
$ cp -r root_files/* /usr/armv6j-hardfloat-linux-gnueabi/
```

Finally, we need to make sure the embedded system is being shared.  
```bash
$ cat /etc/exports
/usr/armv6j-hardfloat-linux-gnueabi/ *(ro,sync,fsid=0)
$
```

Now the base system and initramfs is set up.  The last step is to copy the files over to the SD card.  [Partition the SD card](https://wiki.gentoo.org/wiki/Raspberry_Pi#Partitioning) and copy the files over.
```bash
mkfs.vfat /dev/sdc1
mount /dev/sdc1 /mnt
cp -r kernel/firmware/boot/* boot/* /usr/armv6j-hardfloat-linux-gnueabi/boot/{rpi-kernel,init.cpio.gz} /mnt
umount /dev/sdc1
```

If all goes well, your PI should set up its Wifi on boot, mount the NFS share, and `switch_root` into it.  Now you can tweak the `rc` file to start the services you need.

<img src="/rpi-nfs-root-over-wifi/boot.jpg" style="width: 100%"/>
