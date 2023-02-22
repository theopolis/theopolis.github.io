---
title: "Embedded Trust (P2): U-Boot Secured Boot"
created: 2013-02-12
tags: 
  - diy
  - u-boot
  - embedded
  - kernel
  - sboot
  - tpm
  - trust
  - vip

image: /assets/images/img.png
---

This post will function as a short walk through for installing and using a TPM on a BeagleBone to implement a Secured Boot (wooo...). I will use an example Secure Boot implementation called libsboot for U-Boot. Let's jump right in with a schematic for the (mostly) required additions to the BeagleBone.

<!--more-->

![embedded-trust-1](/assets/images/embedded-trust-1.png)

At the time of my original TPM-tomfoolery (and writing this post), the only "compatible" TPM I could find for the BeagleBone was Atmel's [AT97SC3204T](http://www.digikey.com/product-detail/en/AT97SC3204T-X2A17-00/AT97SC3204T-X2A17-00CT-ND/3440836). By compatible I mean, purchasable, and the TPM used a bus readily available on the BeagleBone. It retails for around $6.50, is a rather easy to solder SSOP package and requires very few additional components to get up and running. The hardest requirement for me was an external 33MHz clock generator. These are easy to find as a small form factor SMD, but I originally opted to purchase the Atmel-recommended 636L3C033M00000, and wired it up in a "dead-bug" super-professional manner (as seen below). I later purchased a programmable oscilator (Maxim's DS1077LZ-66+), to use with other components and found it very convientent (a SOIC package is easier on the eyes than my dead-bug). I'll recommend that to anyone like me who's new to prototyping/hardware design. 

What do you need:

1. Atmel I2C TPM ([AT97SC3204T](http://www.atmel.com/devices/at97sc3204t.aspx))
2. Maxim Programmable Oscilator ([DS1077LZ-66+](http://www.maximintegrated.com/datasheet/index.mvp/id/3143))
3. (Optional) SD/MMC Enclosure or Breakout
4. 1.5k Resistor, 800 Resistor
5. 4x 2200pf CAP, 3x 0.1uf CAP, 0.01uf CAP

In the schematic above R1 is a 1.5k Resistor, R2 is an 800 Registor. C1, C4, C5, and C6 are 2200pf caps, C2, C3, C9 are 0.1uf caps, and C10 is a 0.01uf capacitor. 

![tpm-on-ssop-breakout.gif](/assets/images/tpm-on-ssop-breakout.gif)

Software:

- U-Boot with [libSboot](https://github.com/theopolis/u-boot-sboot), AT97SC3204T driver, and libTLCL ([standalone](https://github.com/theopolis/sboot))
- [DS1077LZ-66+](https://github.com/theopolis/DS1077L-linux) programmer (written for DS1077LZ-40)
- Linux [Kernel driver](https://github.com/theopolis/tpm-i2c-atmel) for AT97SC3204T
- Linux Kernel [source](https://github.com/beagleboard/kernel) for BeagleBone

### Use the Atmel TPM in the Linux Kernel

First grab a copy of a compatible Linux kernel source for the BeagleBone. I recommend something 3.7+ so you can take advantage of the newer TPM-related extensions to Linux IMA (Integrity Measurement Architecture). The link above is to BeagleBoard's github repository which includes a 3.7 branch. This inluces the mainline Linux kernel source with a script to apply the BeagleBone (AM335x) patches, along with a working config.

```
 [~git$] git clone https://github.com/beagleboard/kernel linux-bb
 [~git$] cd linux-bb [linux-bb$] git checkout -b 3.7 origin/3.7
 [linux-bb$] ./patch.sh
```

Now to add support for the Atmel I2C TPM:

```
 [~git$] git clone https://github.com/theopolis/tpm-i2c-atmel
 [~git$] cd tpm-i2c-atmel
 [tpm-i2c-atmel$] ./patch_kernel.sh ~/git/linux-bb/kernel
 [tpm-i2c-atmel$] cd ~/git/linux-bb/kernel
 [kernel$] patch -p0 < ~/git/tpm-i2c-atmel/patches/bb-ra5-add-tpm-dts-3.7.patch
```

The last 2 lines may change (they enable the I2C-1 bus on the BeagleBone, and add an entry for the AT97SC3204T TPM), for this set of steps I'm assuming you're using a device tree blob. In the patches folder there are examples for enabling I2C-1 and adding the AT97SC3204T in code. Finally build the source, after enabling the TPM (and optionally LINUX\_IMA):

```
 [~git$] cd linux-bb/kernel
 [kernel$] cp ../configs/beaglebone .config
 [kernel$] make ARCH=arm menuconfig
 # Enable devices/char/tpm/tpm_i2c_atmel (CONFIG_TCG_TIS_I2C_ATMEL)
 [kernel$] make ARCH=arm CROSS_COMPILE=/path/to/your/compiler- uImage dtbs
```

This will make an arch/arm/boot/uImage and am335x-bone.dtb 

### **Use the Atmel TPM in U-Boot and SPL**

This is a bit easier as the TPM drivers are already bundled into a U-Boot source. The problem with this is you wont receive mainline U-Boot updates until I decide to merge them into the repo :(. I'm trying to provide a method similar to the above instructions which places the driver code, TPM interface library, and libSboot external from U-Boot. The main issue this THIS method is how tightly libSboot is intergreated into U-Boot code, meaning a whole lot of patches. Either way, it's not easy. Since this is just an example, I'm happy with the first option of providing an all-in-one.

```
 [~git$] git clone https://github.com/theopolis/u-boot-sboot
 [~git$] cd u-boot-sboot
 [u-boot-sboot$] make CROSS_COMPILE=/path/to/your/compiler- am335x_evm_tpm_config
```

This works because I've added an entry into U-Boot's boards.cfg to compile the am335x\_evm board with TPM-related functions enabled. The more-important CONFIG options will be explained below. Documentation for every CONFIG option can be found in the [README](https://github.com/theopolis/sboot/blob/master/README) for the standalone libSboot. I'll also **note** that this U-Boot includes an added CONFIG\_SPL\_MMC\_SD\_FAT\_BOOT\_DEVICE.

```
 [u-boot-sboot$] make CROSS_COMPILE=/path/to/your/compiler-
```

This will make a MLO and u-boot.img

### **Secure Boot for U-Boot (libSboot)**

This description is also in the README for libSboot :).

libSboot provides an example 'Secured Boot' for U-Boot and a U-Boot Second Phase Loader (SPL). libSboot attempts to define an example of how a platform can measure a pre-OS boot environment, thus providing a capability to ensure that a libSboot-enforced OS is only loaded in an owner-authorized fashion. A 'Secure Boot' concept is a common means to ensure platform security and integrity; understand that there are many implementations of a 'Secure Boot'.

The pre-boot environment is defined as:

- The U-Boot binary loaded by a SPL
- EEPROM defining platform identification and configuration
- Environment data read from an initial external source
- Environment variables set via the U-Boot console
- Commands interpreted via the U-Boot console
- Flat Device Tree files
- Initial Ram Disks and Ram Disks
- An OS Kernel

Currently libSboot does not require augmentation (signatures or keys) to data or configuration options for boot. It only requires patching U-Boot and SPL boot routines to measure and check platform state. This does not provide the user with much robustness. A change to the pre-boot environment will require interaction on the U-Boot console to 'reseal' the configuration. A more robust implementation would apply signature checking to data and options to provide flexible updates to the pre-boot environment. A signature-based libSboot exists and requires the user to enable the corresponding CONFIG options.

Understanding the implementation of libSboot: libSboot uses a TPM v1.2 to implement a secure boot using a static root of trust measurement (SRTM). The static adjective implies a 'read-only' attribute, meaning libSboot expects its initialization to occur from ROM code. During this initialization libSboot performs a TPM\_Start, TPM\_SelfTest and checks that the TPM is neither deactivated nor disabled. The TPM must have its NVRAM locked, meaning access control is enforced. Initialization then checks each PCR used to measure the pre-boot environment and verifies they are reset. Finally Physical Presence is asserted to satisfy NVRAM read/write permissions. The sealed data for a securely measured pre-boot environment is stored in TPM NVRAM with a Physical Presence requirement for read and write. Note: the sealed data is an encrypted blob, thus a Physical Presence requirement for reading is not required. Though the Physical Presence requirement for writing is very important! If arbitrary sealed data can be written, then an attacker can measure and store from a compromised OS state. Because of this, libSboot must de-assert Physical Presence and extend the PCRs with random data when libSboot finishes measuring or encounters an error.

libSboot uses two sealed blobs stored in TPM NVRAM, one measured for the pre-execution of U-Boot, the other for the OS. This enables flexibility within U-Boot to seal modifications to the pre-boot environment for the U-Boot environment, U-Boot console usage, OS kernel, etc. Modifications to U-Boot are more difficult, U-Boot can issue a re-seal of a new U-Boot binary, but first the PCR which measured the running U-Boot must be reset. This requires an authenticated TPM\_Reset command. libSboot will report to the console if an unseal fails, if libSboot is in 'enforce' (see below) mode then a failed unseal will halt execution. This implementation does not depend on the sealed and unsealed data (meaning we can seal well-known data), it only depends on the TPM response (success/failure) of an unseal. Since libSboot does not require authentication during initialization, subsequent initializations will normally fail. There are several ways to assure successful subsequent initializations: (1) build a method for authenticating a TPM owner within the SRTM; (2) require hardware Physical Presence; (3) issue a TPM Reset before the OS reboots.

![tpm-kit.jpg](/assets/images/tpm-kit.jpg)

### **Exmaples**

When libSboot is enabled in the SPL, the output would be similar to the following. Here I've turned on timing measurements, which are tick counts, the seconds is a gross estimation of 1000Hz clock.

```
U-Boot SPL 2012.10-gc3e21ab-dirty (Feb 08 2013 - 22:17:23)
OMAP SD/MMC: 0 
reading u-boot.img 
reading u-boot.img 
sboot: initializing SRTM 
1.2 TPM (atmel) 
sboot init time taken: 0 minutes, 1.506 seconds, 1506 ticks sboot 
srtm_init time taken: 0 minutes, 3.347 seconds, 3347 ticks 
sboot extend time taken: 0 minutes, 0.789 seconds, 789 ticks 
SPL: (Sboot) measuring U-Boot ... 
sboot check time taken: 0 minutes, 1.923 seconds, 1923 ticks 
Success
```

At this point U-Boot will be executed, I will interrrupt the auto-boot to drop us to a U-Boot command shell:

```
U-Boot 2012.10-gc3e21ab-dirty (Feb 08 2013 - 22:17:23)

I2C:   ready
DRAM:  256 MiB
WARNING: Caches not enabled
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
Using default environment

Net:   cpsw
U-Boot#
```

Now if we were to issue commands to U-Boot at the prompt we would see:

```
U-Boot# fake_command
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
1.2 TPM (atmel)
sboot init time taken:  0 minutes, 0.567 seconds, 567 ticks
Unknown command 'fake_command' - try 'help'
U-Boot# another_command
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
Unknown command 'another_command' - try 'help'
U-Boot#
```

The first 'fake\_command' initialized sboot within U-Boot (as it is the first time U-Boot needs to measure data). Issuing the boot command tells U-Boot to resume the auto-boot:

```
U-Boot# boot
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
SD/MMC found on device 0
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
reading uEnv.txt

211 bytes read
Loaded environment from uEnv.txt
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
Importing environment from mmc ...
Running uenvcmd ...
sboot extend_console time taken:  0 minutes, 0.001 seconds, 1 ticks
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
sboot extend_console time taken:  0 minutes, 0.000 seconds, 0 ticks
reading uImage

3896328 bytes read
reading am335x-bone.dtb

20290 bytes read
## Booting kernel from Legacy Image at 80200000
   Image Name:   Linux-3.7.4-00641-ge961db3-dirty
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    3896264 Bytes = 3.7 MiB
   Load Address: 80008000
   Entry Point:  80008000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 80f80000
   Booting using the fdt blob at 0x80f80000
Sboot measuring ... sboot extend_os time taken:  0 minutes, 5.286 seconds, 5286 ticks
sboot extend_os time taken:  0 minutes, 0.028 seconds, 28 ticks
sboot check time taken:  0 minutes, 1.199 seconds, 1199 ticks
Failed
sboot total time taken:  0 minutes, 7.081 seconds, 7081 ticks
   Loading Kernel Image ... OK
OK
   Loading Device Tree to 8fe58000, end 8fe5ff41 ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0
[    0.000000] Initializing cgroup subsys cpu
...
```

Some of the text is overlapping so it may be a bit hard to read, but during the boot, U-Boot ran a few commands stored in CONFIG\_BOOTCOMMAND. One of these commands read and added the contents of uEnv.txt to the environment. Finally am335x-bone.dtb and uImage were read, measured and executed. The sboot "total time taken" lists the total amount of ticks spents in sboot-related code. On average a STRM init takes **0.991s**, an extend of U-Boot takes **0.788s**, a measure takes **0.138s**, and a policy check (init in U-Boot) takes **0.508s**

All of the implementation code is called out nicely in a set of patches found in the standalone libSboot. These patches are the code that bind libSboot's initialize, extend, and check methods to U-Boot's execution, interpreter, and OS loader.

### **Shmoocon IX 2013 and more!**

[Here](assets/DIY-Secure-Embedded-Trust.pdf) are the slides presented at Shmoocon IX about using TPMs and projects like libSboot to secure embedded devices. I'd like to give a final thanks to the Shmoo group and all those who helped run the event! And of course to all those who attended the talk, you raised ~$245 in donations to Hackers for Charity, you folks rock!
