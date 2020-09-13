---
title: "Gentoo VFIO for high performance Windows VM's"
date: 2018-07-21T11:16:19-04:00
draft: false
---

I've always had a love-hate relationship with Linux.  As a developer, its CLI tools are exactly what I need, it offers so many desktop environments that can be as minimal or bloated as you want, and installing a library you need is one command away, but Windows has all the mainstream applications.  For so many years, I've dual-booted between Windows for games and Linux for work.  It was good enough for what I wanted, but as I get older, my tolerance for inconvenient workarounds gets thinner.  

I've tried running Windows fulltime while using Cygwin for applications, but it makes your environment messy.  You essentially have two user directories and will frequently use one or the other depending on whether you're calling it from DOS or Cygwin, or flat-out won't let you call it from Cygwin.  So you'll frequently have one Cygwin window open for regular work and a DOS window open for working around applications that don't like Cygwin (hi, Go).

I also dabbled with WSL.  It seems to work nicely for the minimal amount of time I spent on it.  However, my immediate first thought was whether I could switch the distro to Gentoo.  I'm sure someone, somewhere, figured out how to do it, but it wasn't obvious to me.  Also, [some benchmarks](https://www.phoronix.com/scan.php?page=article&item=wsl-february-2018&num=2) shows WSL's I/O performance dealing with the disk is laughable.  So using Gentoo will be a no-go anyway.

So, I decided to go the crazy route.  The route that's not for the faint-of-heart.  The route that alienates you at parties when you try to explain it to people.  VGA passthrough.  Specifically, using the Linux kernel's VFIO features to expose physical hardware to your VM's.

This requires specific hardware, tons of research, and hours of tweaking and fine-tuning through trial and error.  That should be enough to scare anyone off, but apparently not Linux users.  This is a rough guide for setting up a VFIO machine.  I'm writing it mainly as notes for myself in case I do a reinstall, but should be applicable for anyone.

The first thing is picking out the hardware.  You should get the hard part of the way first and find a motherboard that is known to work.  Manufacturers are really spotty when it comes to supporting different features that's required to make this work.  For Intel, you need VT-d.  For AMD, IOMMU.  I would suggest (at the time of this writing), to stick with AMD Ryzen, since it has better performance for the price point.

For the motherboard, I bought an [ASRock X370](https://www.newegg.com/Product/Product.aspx?Item=N82E16813157757).  Any Ryzen CPU should do, but I bought an [1800X](https://www.newegg.com/Product/Product.aspx?Item=N82E16819113430).  You'll want enough CPU's for the host, as well as the VM.  You'll also need two GPU's: one for the host and one for the VM.  Get two nVidia GPU's.  AMD's GPU's still suck on Linux and nVidia has the least amount of hassle when passing it through to the VM.  This setup also relies on two monitors and [Synergy](https://symless.com/synergy).

Also, special consideration is needed for the audio.  It is a GIANT pain in the ass to set up if you want the VM's audio to be redirected to the host.  The first part is figuring out the right parameters to pass to the QEMU VM to get the audio working correctly.  The second part is trying to tweak and fine-tune the host and VM to get rid of the crackling and latency problems.  I wouldn't recommend this route.  Instead, pass through a GPU with HDMI audio to the VM and hook your speakers up to that.  If you want audio on both the host and the VM, get a hardware audio mixer or a second set of speakers.

Now set up the host by installing your distribution of choice.  In Gentoo's case, make sure you read through their [wiki page](https://wiki.gentoo.org/wiki/Ryzen) for Ryzen.  I've seen random segfaults when compiling even when not setting up any Ryzen-specific GCC options, but haven't bothered to investigate since it hasn't occured since lowering the number of jobs to `-j3`.  At the time of this writiing, GCC 6 seems to work with what I've thrown at it when setting `-march=znver1`.  I even set the kernel Processor family to `AMD Zen` and haven't seen any other odd behavior.

Speaking of the kernel, as a side-note, I don't bother going through the Kernel Seeds manual and whatnot.  I use genkernel to generate an initial configuration.  After booting into it, I then reconfigure the kernel with `make localyesconfig` to remove whatever modules that aren't loaded.  Also, be sure to set up the kernel for [bridging](https://wiki.gentoo.org/wiki/Network_bridge), and [VFIO](https://wiki.installgentoo.com/index.php/PCI_passthrough#Step_0:_Compile_IOMMU_support_if_you_use_Gentoo).

Also note that you'll need Xorg.  Wayland is not going to work with Synergy.  Another thing to consider is if you have a programmable keyboard and mouse.  If you do, it's not going to work with Linux in all liklyhood.  In that case, you'll need to pass through a USB hub to the VM, as well as the GPU.  If you plan on gaming in the VM, you'll want to pass through a USB hub anyway because USB redirection will cause latency issues anyway.

So, to find the device you want to pass through, run `lspci -nn` to find the devices you want exposed to the VM.  The catch, however, is you have to pass through an entire IOMMU group to the VM, not just the one device.  Run [this script](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid) to list all the available groups.  When passing through a USB hub, find a hub that has the minimal amount of devices in its IOMMU group.

For example, on my system:
```bash
$ lspci -nn
...
0f:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
0f:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
...
11:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Device [1022:145a]
11:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-0fh) Platform Security Processor [1022:1456]
11:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-0fh) USB 3.0 Host Controller [1022:145c]
...

$ shopt -s nullglob
$ for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;
...
IOMMU Group 13 0f:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
IOMMU Group 13 0f:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
...
IOMMU Group 7 11:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Device [1022:145a]
IOMMU Group 7 11:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-0fh) Platform Security Processor [1022:1456]
IOMMU Group 7 11:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 00h-0fh) USB 3.0 Host Controller [1022:145c]
...
```

I want to pass through my my GTX 1060 and the USB hub on 11:00.*.  So I'll add the following to the kernel arguments:

```
iommu=1 amd_iommu=on ignore_msrs=1 pci-stub.ids=10de:1c03,10de:10f1,1022:145a,1022:1456,1022:145c
```

`ignore_msrs=1` is necessary for fixing a BSOD problem with Windows.

Now install libvirtd with the OVMF firmware.  If you don't use OVMF, PCI passthrough will not work.  Also, [download the ISO](https://docs.fedoraproject.org/quick-docs/en-US/creating-windows-virtual-machines-using-virtio-drivers.html#virtio-win-direct-downloads) containing the VFIO drivers for Windows.  When running the initial installation process, attach the ISO to the VM and install the disk VFIO drivers.

Now, instead of going through the entire installation process with virt-manager, I'll post my configuration with comments.  It should be sane enough for most environments and can be tweaked to your needs.

```xml
<domain type='kvm'>
  <name>win10</name>
  <uuid>...</uuid>
  <memory unit='KiB'>8388608</memory> <!-- Be sure to allocate at least 8G of RAM -->
  <currentMemory unit='KiB'>8388608</currentMemory>
  <vcpu placement='static'>8</vcpu> <!-- My CPU has 16 logical cores.  A sufficient amount will be needed for the host so it can run emerge without disrupting the VM.  Plus, 8 cores will probably be enough for a gaming VM -->
  <iothreads>4</iothreads> <!-- 4 iothreads seems to be a good number. -->
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/> <!-- This pins the VM's execution to a fixed set of physical CPU's.  This should give better throughput in the VM since the host won't be context switching the VM's execution around to different CPU's. -->
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='5'/>
    <vcpupin vcpu='6' cpuset='6'/>
    <vcpupin vcpu='7' cpuset='7'/>
    <iothreadpin iothread='1' cpuset='0-1'/>  <!-- This pins I/O operations to the host CPU's, not the VM's CPU's.  This should give better throughput for CPU-intensive applications on the VM since it won't be spending its own CPU cycles on disk/network operations. -->
    <iothreadpin iothread='2' cpuset='2-3'/>
    <iothreadpin iothread='3' cpuset='4-5'/>
    <iothreadpin iothread='4' cpuset='6-7'/>
  </cputune>
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.12'>hvm</type> <!-- Again, you need OVMF, or VGA passthrough will not work -->
    <loader readonly='yes' type='pflash'>/usr/share/edk2-ovmf/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <synic state='on'/>
      <stimer state='on'>
        <direct state='on'/>
      </stimer>
      <frequencies state='on'/>
      <vendor_id state='on' value='whatever'/> <!-- nVidia GPU's don't like KVM, so we'll hide the fact that we're running under KVM by giving it a different vendor ID... -->
    </hyperv>
    <kvm>
      <hidden state='on'/> <!-- ... and hide the fact that we're running on KVM.  Thanks nVidia. -->
    </kvm>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='none'> <!-- host-passthrough exposes the CPU's features to the VM.  This will be especially necessary if you want to run more VM's inside the VM. -->
    <topology sockets='1' dies='1' cores='4' threads='2'/> <!-- This needs to match your host topology.  If you have hyperthreading, divide the CPU's your exposing in half and use two threads. -->
    <feature policy='require' name='topoext'/>
  </cpu>
  <clock offset='localtime'> <!-- We don't want our VM's time to be set halfway across the world, now do we? -->
    <timer name='hpet' present='yes'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
  <on_poweroff>destroy</on_poweroff> <!-- Don't suspend the VM.  It will cause problems with the GPU -->
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='block' device='disk'> <!-- We want to use virtio for the disk.  Be sure to give the VM the VFIO ISO when first installing.  -->
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/path/to/device'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='bridge'> <!-- Again, we want to use virtio for the network.  After installing Windows, install the network VFIO drivers from the ISO.  -->
      <mac address='FF:FF:FF:FF:FF:FF'/>
      <source bridge='br0'/>
      <model type='virtio'/>
    </interface>
    <memballoon model='none'/> <!-- memballoon seems to hurt performance -->
    <!-- The rest is PCI subsystem configurtions.  Be sure to use vfio for the driver for each section. e.g.
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x11' slot='0x00' function='0x2'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
    </hostdev>
    -->
  </devices>
</domain>
```

That's about it.  After installing the VM, you'll have to install Synergy on the host and client.  The easiest way to do this is to give your host and VM static IP addresses.  Have fun plugging the keyboard/mouse into different ports when setting this up.

Some TODO's involve investigating isolcpu's and irqbalance, though my initial tests with them seemed to hurt performance.

Here's some general notes that I'll keep appending to as time goes on.

* Configure libvirtd to shutdown your VM's when rebooting or shutting down the host.  It defaults to hibernate.
* Set the number of make jobs on the host to `-j3`.  Set it any higher and it will cause a noticable performance impact on the VM.
* For some reason, Synergy goes haywire if you move the mouse cursor out of the Windows VM when it has the picture dropdown on the lock screen.
* If you're using Gnome, [this extension](https://extensions.gnome.org/extension/963/disable-barrier-support/) fixing the hot corner when using Synergy.
* Consider enabling [MSI Interrupts](http://lime-technology.com/wiki/index.php/UnRAID_6/VM_Guest_Support#Enable_MSI_for_Interrupts_to_Fix_HDMI_Audio_Support).  It can [help with latency problems](https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts.378044/).

