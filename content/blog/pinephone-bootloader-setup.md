---
title: "PinePhone p-boot Setup"
date: 2020-09-13T17:22:43-04:00
draft: false
---

Here's a general walkthrough for setting up a bootloader on the PinePhone.  We're going to use p-boot, which has nice start-times and lets you customize your boot screen with fancy graphics.  SD-card is not supplied with this tutorial, you need to provide your own.

These instructions are for Linux, but should be distribution-agnostic.  At the end, I will be using a Gentoo-specific tool, but it doesn't involve installing p-boot.

First, we need to grab some cross-compilers.  We will need ARM64 and OpenRisk.  Grab the toolchains and add the compiler's to your PATH:

```bash
$ wget https://toolchains.bootlin.com/downloads/releases/toolchains/aarch64/tarballs/aarch64--musl--bleeding-edge-2020.02-2.tar.bz2
$ wget https://toolchains.bootlin.com/downloads/releases/toolchains/openrisc/tarballs/openrisc--musl--bleeding-edge-2020.02-2.tar.bz2
$ tar xpf aarch64--musl--bleeding-edge-2020.02-2.tar.bz2
$ tar xpf openrisc--musl--bleeding-edge-2020.02-2.tar.bz2
$ export PATH="$(pwd)/aarch64--musl--bleeding-edge-2020.02-2/bin:$PATH"
$ export PATH="$(pwd)/openrisc--musl--bleeding-edge-2020.02-2/bin:$PATH"
```

Now let's retrieve p-boot:

```bash
$ git clone https://megous.com/git/p-boot/
```

First, we need to build the ARM firmware for bootstrapping the system.  Clone these repositories:

```bash
$ cd p-boot/fw
$ git clone https://github.com/ARM-software/arm-trusted-firmware.git atf
$ git clone https://github.com/crust-firmware/crust.git crust
```

The firmware needs some patches:

```bash
$ cd atf && git am ../atf.patch && cd ..
```

Now we need to edit a couple files to use the bootlin cross compilers:

* Edit `build-atf.sh`, set `CROSS_COMPILE` to `aarch64-buildroot-linux-musl-`
* Edit `crust/arch/or1k/Makefile`, set `CROSS_COMPILE` to `or1k-buildroot-linux-musl-`
* Edit `crust/Makefile`, set HOST_COMPILE to `aarch64-buildroot-linux-musl-`

Now build the firmware:


```bash
$ ./build-all.sh
```

If all goes well, you should now have a `fw.bin` file.  Now go into the build directory:

```bash
$ cd ../build
```

Edit `build.ninja`, set `aarch64_prefix` to `aarch64-buildroot-linux-musl-` and build:

```bash
$ ninja
```

You should now have the utilities for installing p-boot on your SD-card.  Before we do that, let's grab a pre-built Linux image to test it.  We'll use an image from the same maintainer as p-boot:

```bash
$ wget https://xff.cz/kernels/5.9/pp.tar.gz
$ tar xpf pp.tar.gz
$ ls pp-5.9/
board-1.0.dtb  board-1.1.dtb  board-1.2.dtb  headers  Image  linux.config  modules
```

As an optional step, you can also create your own splashscreen image.  You need `imagemagick` to do this.  Compile the `argb.c` program in the `theme` directory and run the following commands to convert your image into a format consumable for p-boot:

```bash
gcc -o argb argb.c
convert your-image.png your-image.rgba
./argb your-image.rgba pboot2.rgba
```

You will also need a `off.rgba` file for this.  You can find it in the `theme` directory or a prebuilt one in `example/files/`.  Also note that `pboot2.rgba` is the default image that is displayed when p-boot is loaded.

Now let's make the bootloader configuration:

```bash
$ mkdir config
$ cat > config/boot.conf <<EOF
no=0
  name=Test Linux
  atf=fw.bin
  dtb=board-1.2.dtb
  linux=Image
  bootargs=console=ttyS0,115200 console=tty1 root=/dev/mmcblk0p3 rootfstype=ext4 ro rootwait panic=30 init=/hello
  splash=files/pboot2.argb
EOF
```

The `root` paramter is pointing to the third partition of the SD-card.  Also notice the `init` parameter is set to `/hello`.  More on that at the end...

Copy over the `fw.bin` firmware file.  Also copy the kernel image, `Image`, and the device tree blob for your Phone (`board-1.2.dtb` in my case).

```bash
$ ls config
boot.conf  files  fw.bin  Image  board-1.2.dtb
```

Also make a `files` directory and copy your splash images over:

```bash
$ ls config/files
off.argb  pboot2.argb
```

Now lets create a partition table on your SD-card.  It must be a DOS filesystem table.  Create a dedicated partition for p-boot, set it to Linux and set the bootable flag for it.  For example:

```bash
$ fdisk -l /dev/sdc
Disk /dev/sdc: 29.81 GiB, 32010928128 bytes, 62521344 sectors
Disk model: SD  Transcend   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x317e999c

Device     Boot   Start      End  Sectors  Size Id Type
/dev/sdc1  *       2048   206847   204800  100M 83 Linux
/dev/sdc2        206848  4401151  4194304    2G 82 Linux swap / Solaris
/dev/sdc3       4401152 62521343 58120192 27.7G 83 Linux
```

`sdc1` is the p-boot partition, `sdc2` is the swap partition, and `sdc3` is the root.

Finally, write the bootloader images to your SD-card.

```bash
$ dd if=p-boot.bin of=/dev/sdX bs=1024 seek=8
$ ./p-boot-conf-native config/ /dev/sdX# # This is going to the boot partition (sdc1 in my case)
```

Now we're done with the booatloader.  For good measure, let's test PID1 (remember the `init=/hello` parameter?).  I set up a compiler with Gentoo's crossdev tool and copied it out to the SD-card.

```bash
$ crossdev -t aarch64-linux-musl
# Wait a million years
$ cat > hello.c <<EOF
#include <stdio.h>
int main() {printf("Hello world\n");}
EOF
$ aarch646-linux-musl-gcc -static hello.c
$ mkfs.ext4 /dev/sdc3
$ mount /dev/sdc3 /mnt
$ cp a.out /mnt/hello
$ umount /mnt
```

Now the moment of truth.  Stick your SD-card in and see if it works.  Here's some pictures capturing the magical p-boot and kernel initialization process on my phone:

<img src="/pinephone-bootloader-setup/p-boot.jpg" style="width: 100%"/>

<img src="/pinephone-bootloader-setup/kernel-init.jpg" style="width: 100%"/>
