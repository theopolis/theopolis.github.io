---
title: "Embedded Trust (P1): Beginning to trust my BeagleBone"
created: 2012-07-06
tags: 
  - hardware
  - diy
  - embedded
  - trust

---

I plan to have a series of posts outlining my curiosity with embedded development and trust. Let's start with poking around where my (our) trust lies when deciding on a SoC for embedded development, using the [SRM](assets/BONE_SRM.pdf)\] as an example. In this post we'll move trust from CircuitCO's (the Bone manufacture) included bootloaders, [Angstrom](http://www.angstrom-distribution.org/) Linux kernel, and Angstrom development environment to your own compiled bootloaders, kernel, and OS.

<!--more-->

I purchased my Bone through AVNET, why? Because they were the only distributor that had Bones in stock last week. Do I trust them? Not really, but I'll have to for this example since I cannot wait for a "more"-trusted distributor's lead time. CircuitCO manufactures the Bone, which contains a [TI AM3359](http://www.ti.com/product/am3359) ARM Cortex-A8 processor, a Kingston microSD card, and a ton of "little" things \[[bill of materials](http://beagleboard.org/static/beaglebone/a3/Docs/Hardware/BONE_BOM.xls)\]. Here's a simple chain of trust.

![](/assets/images/embedded-trust-1.png)

Blue means we "need" to trust the entity.

I dug through the "little" things and pulled out the manufactures of the IC EEPROMs on the SoC, we'll need to trust them too. Let's take out the distributor since it'll vary between user and purchase. Here's the modified trust model I'd like.

![Embedded-Trust-m2.png](/assets/images/Embedded-Trust-m2.png)

But this model isn't true. That Bone comes complete with software to get you up and running with Bone development. ...there's not much you can do about adjusting the hardware trust, but we can do a decent job on the software. I'll redraw the trust model with software, making most dependent on CircuitCO. We cannot modify the TI OMAP3 code running on the AM335x or the BootROM code it runs \[[I think?](http://processors.wiki.ti.com/index.php/AM335X_StarterWare_Booting_And_Flashing)\]. I highlighted the entities we can control in orange.

![Embedded-Trust-m3.png](/assets/images/Embedded-Trust-m3.png)

To trust these entities let's propose a complete code review. (Like that's really going to happen...) There are a lot of Linux distros you can run on the Bone, and most will provide binaries for the boot loaders and kernels. I propose compiling all the code yourself on whatever system you like best, using CrossTool-NG. Then using a stage3 Gentoo OS and recompiling the works.

I followed the instructions here: ([http://randomsplat.com/id192-building-a-hard-float-arm-toolchain.html](http://randomsplat.com/id192-building-a-hard-float-arm-toolchain.html)) to build an ARMv7l hardfloat toolchain. Using a hardfloat-enabled toolchain might get you [better performance](https://groups.google.com/forum/?fromgroups#!topic/beagleboard/igI4h3Kh0Xc) for subsequent binaries you decide to compile, but will not improve your performance when compiling just the stage 1/2 bootloaders and Linux kernel.

My toolchain was built at: _**TC=/home/teddy/crosstool-ng-1.15.2/armv7hf**_

And I used the following versions:

- CrossTool-NG: 1.15.2
- Linux Kernel: 3.3.4
- binutils: 2.21.1a
- gcc: 4.6.3
- glibc: 2.14.1

Clone the following repo from Angstrom, which supports the TI AM335x EVM chip. Remember to perform the code-review.

```
$ git clone git://arago-project.org/git/projects/u-boot-am33x.git
$ cd u-boot-am33x
$ make ARCH=arm CROSS_COMPILE=${TC}/bin/arm-unknown-linux-gnueabi- \
SYSROOT=${TC}/arm-unknown-linux-gnueabi/sysroot/ am335x_evm_config
$ make -j4 ARCH=arm CROSS_COMPILE=${TC}/bin/arm-unknown-linux-gnueabi- \
SYSROOT=${TC}/arm-unknown-linux-gnueabi/sysroot/
[...]
tools/mkimage -A arm -T firmware -C none \
 -O u-boot -a 0x80100000 -e 0 \
 -n  "U-Boot 2011.09-00039-g2e37929 for am335x board" \
 -d u-boot.bin u-boot.img
Image Name:   U-Boot 2011.09-00039-g2e37929 fo
Created:      Wed Jul  4 18:16:32 2012
Image Type:   ARM U-Boot Firmware (uncompressed)
Data Size:    232120 Bytes = 226.68 kB = 0.22 MB
Load Address: 80100000
Entry Point:  00000000
```

This will generate {MLO,u-boot.img} files in the same directory. Now clone the kernel source with support for the TI AM335x EVM chip. Review, repeat.

```
$ git clone git://arago-project.org/git/projects/linux-am33x.git
$ cd linux-am33x
$ make ARCH=arm CROSS_COMPILE=${TC}/bin/arm-unknown-linux-gnueabi- \
SYSROOT=${TC}/arm-unknown-linux-gnueabi/sysroot/ am335x_evm_defconfig
$ make -j4 ARCH=arm CROSS_COMPILE=${TC}/bin/arm-unknown-linux-gnueabi- \
SYSROOT=${TC}/arm-unknown-linux-gnueabi/sysroot/ uImage
[...]
Image Name:   Linux-3.1.0-00010-g66bfbd2
Created:      Wed Jul  4 18:12:25 2012
Image Type:   ARM Linux Kernel Image (uncompressed)
Data Size:    2872648 Bytes = 2805.32 kB = 2.74 MB
Load Address: 80008000
Entry Point:  80008000
  Image arch/arm/boot/uImage is ready
```

Then follow the Gentoo-on-Bone install guide starting here: ([SD card setup](http://dev.gentoo.org/~armin76/arm/beaglebone/install.xml#doc_chap4)). And remember to grab a stage3 armv7 with hardfloat. This setup gave me the following openssl speed results:

```
type          16 bytes   64 bytes   256 bytes  1024 bytes  8192 bytes
sha512        1150.26k   4598.66k   7553.19k   10904.23k   12533.76k
              sign       verify     sign/s     verify/s
rsa 2048 bits 0.089464s  0.002728s  11.2       366.6
```

Which are perfect for the tests and exploration to come. This also changes the trust model to highlight a cool security concept "trusting trust" made famous during Ken Thompson's [Turing award lecture](http://cm.bell-labs.com/who/ken/trust.html). Here I highlight in green: what you now trust after your meticulous code review; in orange: your compiler; and in red: the security you implemented when downloading the source and protecting the host you compiled on. I'd also recommend David A. Wheeler's [Diverse Double-Compiling](http://www.dwheeler.com/trusting-trust/) approach to solving the trusted compiler concept.

![Embedded-Trust-m4.png](/assets/images/Embedded-Trust-m4.png)

Finally, we can trust the software we've installed on the SD card used on the BeagleBone. But what trust can we guarantee once the Bone turnes on, connects to a network, or begins to install software? Modifying either bootloader, or kernel, is easy as mounting the first SD card partition and overwriting a file. How do we continue to trust? The next post will detail how to secure these files, and the final post will implement a trusted (and secured) boot of the Bone.
