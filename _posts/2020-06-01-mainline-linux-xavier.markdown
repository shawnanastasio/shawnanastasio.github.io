---
layout: post
title:  "Running a mainline linux kernel on the NVIDIA Jetson Xavier AGX"
date:   2020-06-01 12:00:00 -0500
categories: embedded linux
---
The [Jetson Xavier AGX](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-agx-xavier/)
is an embedded ARM64 linux platform from NVIDIA with pretty impressive specifications and a decent software ecosystem.
Most users will probably be using the NVIDIA-provided [Linux4Tegra](https://developer.nvidia.com/embedded/linux-tegra) (L4T)
distribution on their Xaviers, which comes with a laughably ancient 4.9 kernel build and quite a few out-of-tree drivers.
This means users will miss out on new features, upstream driver additions, and security fixes from the upstream Linux
kernel project.

Thankfully though, support for the Xavier SoC and a lot of its peripherals are present in recent Linux releases, so running
a mainline kernel is possible (though NVIDIA's documentation certainly won't tell you how). Below we'll walk through
the process of compiling a kernel from mainline sources and booting it on the Xavier.

## Prerequisites
To perform the whole operation, you'll need a few prerequisites.

# Kernel Sources
We'll start off by downloading the mainline kernel sources we wish to compile, as well as some build prerequisites.

On Ubuntu 18.04, you can download the build requisites with the following:
```
$ sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev gcc-aarch64-linux-gnu
```

Next, you'll need to grab a copy of the kernel sources you want to compile. For this tutorial we'll download straight
from git.
```
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

# Nvidia Jetpack SDK
[Jetpack](https://developer.nvidia.com/embedded/jetpack) is NVIDIA's SDK that comes with everything you'll need to
flash the Xavier. Unfortunately, many of the tools it ships seem to be proprietary binaries, so you'll need to
do this on an x86_64 machine.

The easiest way to get Jetpack set up is with the [NVIDIA SDK Manager](https://developer.nvidia.com/nvsdk-manager).
You'll need to make an NVIDIA Developer account to download it unfortunately.

Once you've got the SDK Manager, click through and download the latest Jetpack release (4.4 at the time of writing)
for the Xavier AGX. I'd also recommend un-checking the Host Machine software selector which will install a bunch
of unnecessary NVIDIA stuff to your host machine.

Finally, follow the directions to flash the included L4T release to your Xavier. We'll use L4T as the base distro which
we'll install our mainline kernel build over. See [here](https://docs.nvidia.com/sdk-manager/install-with-sdkm-jetson/index.html)
for more information on using the SDK Manager.

## Building the kernel
With the prerequisites out of the way, we can begin building the kernel. Change to the directory where you downloaded
the kernel sources and begin the configuration process:

```
$ cd linux
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```
In the menuconfig interface, enable the ethernet driver under:
```
Device Drivers -> Network device support -> Ethernet driver support
    -> STMicroelectronics devices
        -> STMicroelectronics Multi-Gigabit Ethernet driver
            -> STMMac Platform bus support
                -> Support for snps,dwc-qos-ethernet.txt DT binding
```
Be sure to enable support for any other peripherals or features you'll use.

After the configuration's done, you can launch the build:
```
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j`nproc`
```
This will launch one build for every logical core on your machine. Change the `-j` parameter if you want something else.

If all goes well, the build will complete successfully and you can move on to the next step.

# Installing the kernel
With the kernel built, the next step is to install it to your Xavier's root filesystem, along with an initramfs
and bootloader configuration. The easiest way to accomplish this is to simply copy over the kernel build directory
to your Xavier and run the make install targets there.

It's worth nothing that there are other ways of accomplishing this, like using qemu-aarch64 with binfmt-misc
to chroot into the L4T rootfs and installing there. Advanced techniques like that are out of the scope of this
tutorial though, so we'll continue on by copying the built source to an up-and-running Xavier.

Copying the build directory can be easily accomplished `rsync`:
```
$ rsync -azP ../linux <xavier_user>@<xavier_ip>:~/
```
*Substitute your Xavier's username and IP address.*

Next, log in to the Xavier and execute the following commands to install the kernel
```
$ cd linux
$ sudo make modules_install
$ sudo make install
```

Finally, add an entry for the kernel in `/boot/extlinux/extlinux.conf`, substituting kernel and initramfs names:
```
LABEL mainline
      MENU LABEL mainline kernel
      LINUX /boot/vmlinuz-5.7.0-rc7-vanilla-00212-gbdc48fa11e46
      INITRD /boot/initrd.img-5.7.0-rc7-vanilla-00212-gbdc48fa11e46
      APPEND ${cbootargs}
```

With the kernel installed, all that's left is to flash the mainline device tree before we can boot it.

# Flashing the device tree
One major way difference between L4T's kernel 4.9 fork and mainline is the layout of the Xavier's device tree.
This has the unfortunate effect of making mainline kernels unbootable with L4T device trees and vice versa.

On many embedded linux systems, the device tree is simply stored as a normal file that can be read by
the bootloader, just like the kernel. Unfortunately for us, the Xavier does things a bit differently.

With the default bootloader provided with L4T, the Xavier loads its device tree from a special partition
on internal flash memory. This means that replacing the device tree requires using Jetpack's flashing tools
from an x86_64 machine.

To get started, head back to your x86_64 machine, and navigate to the Jetpack installation directory.
If you used the NVIDIA SDK Manager, that path will probably be something like:
```
~/nvidia/nvidia_sdk/JetPack_4.4_DP_Linux_DP_JETSON_AGX_XAVIER/Linux_for_Tegra
```

From here, we're going to copy the jetson-xavier configuration and modify it to point to the mainline
kernel's device tree, instead of the L4T one.

```
$ cp jetson-xavier.conf jetson-xavier-mainline.conf
$ vi jetson-xavier-mainline.conf
```
Append the following line to the end of the file:
```
DTB_FILE=tegra194-p2972-0000.dtb;
```
Then, save and exit your editor.

Now we need to copy the device tree produced by the linux build into the L4T directory. Assuming your
linux tree is at `~/linux`, execute the following:
```
cp ~/linux/arch/arm64/boot/dts/nvidia/tegra194-p2972-0000.dtb kernel/dtb/
```

Now we're ready to flash the device tree. Put your Xavier into RCM mode by holding down the middle
button while turning it on (left button) or resetting (right button), and enter the following command:
```
sudo ./flash.sh -r -k kernel-dtb jetson-xavier-mainline mmcblk0p1
```

This will read in the new device tree and flash it to the device.

If all goes well, the device should reboot and you will be able to select your mainline kernel
build from the extlinux boot menu.
