---
title: "DIY Root of Trust using ARM Trusted Firmware on the 96Boards Hikey"
created: 2018-07-08
tags: 
  - secure-boot
  - firmware
  - arm
  - hikey

image: /assets/images/hikey_rotpk_hash-1.png
---

This is a series of notes designed to be a walkthrough on how to configure the HiKey Kirin 620 to boot securely with ARM Trusted Firmware's Trusted Board Boot. This does not use any proprietary settings or vendor-specific details about the SoC. Instead, the secure boot path relies on the SoC's `BOOT_SEL` configured to boot solely from the eMMC. With this configuration there **should** be no way to interrupt or bypass the root of trust via runtime changes.

Pay special attention to the **should** as this is not speaking from authority but rather from suspicion and research.

The Root of Trust (ROT) begins in the `BL2` programmed to the eMMC's `boot0` partition. The bootrom must execute the HiKey's `l-loader.bin` and [ARM-Trusted-Firmware's](https://github.com/ARM-software/arm-trusted-firmware) (ATF) `bl2.bin` written to this alternate boot partition. The eMMC's extended CSD register 173 (`BOOT_WP`) is written to permanent write-protect this content. This is a 1-time program operation that has the potential to brick the device.

<!--more-->

As a quick preview, here are the functions and features:

- `BL2` (not `BL1` because HiKey skips `BL1`, [see the ATF notes](https://github.com/ARM-software/arm-trusted-firmware/blob/master/plat/hisilicon/hikey/platform.mk#L11)) implementing a ROT.
- eMMC `boot0` and `boot1` partitions permanently hardware write-protected.
- ROT implemented as a SHA256 of an RSA2048 public key you control, written to hardware write-protected region.
- Chain of Trust implemented with ATF's development [Trusted Board Boot (TBB) implementation](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/trusted-board-boot.rst).
- Configurable and verified Secure OS, and Non-Trusted World Firmware, essentially `BL31`, `BL32`, `SCP_BL2`, and `BL3`.
- Non-Trusted World implemented with U-Boot, for now the last thing verified.

Let's start bottom up.

## Dependencies

I recommend going through a few [HiKey tutorials](https://www.96boards.org/documentation/consumer/hikey/getting-started/) to become familiar with the flashing, recovery, and boot flows. This walkthrough will bootstrap many things, but if this is your entrypoint into HiKey development you may be lost by some references.

All of the following experimentation happens on a Ubuntu 16.04 host machine using an AARCH32 toolchain from apt and an AARCH64 toolchain from Linaro: [https://releases.linaro.org/components/toolchain/binaries/7.2-2017.11/aarch64-linux-gnu/gcc-linaro-7.2.1-2017.11-i686\_aarch64-linux-gnu.tar.xz](https://releases.linaro.org/components/toolchain/binaries/7.2-2017.11/aarch64-linux-gnu/gcc-linaro-7.2.1-2017.11-i686_aarch64-linux-gnu.tar.xz)

## Build Linux

Use the recent build of Linux maintained by 96Boards for the HiKey SoC: [https://github.com/96boards-hikey/linux](https://github.com/96boards-hikey/linux) and the branch `working-android-hikey-linaro-4.4`.

I found the following changes needed to boot from U-Boot.

```diff
157,164c157
< CONFIG_BLK_DEV_INITRD=y
< CONFIG_INITRAMFS_SOURCE=""
< CONFIG_RD_GZIP=y
< CONFIG_RD_BZIP2=y
< CONFIG_RD_LZMA=y
< CONFIG_RD_XZ=y
< CONFIG_RD_LZO=y
< CONFIG_RD_LZ4=y
---
> # CONFIG_BLK_DEV_INITRD is not set
464,465c457,458
< CONFIG_CMDLINE="console=ttyAMA0"
< CONFIG_CMDLINE_FROM_BOOTLOADER=y
---
> CONFIG_CMDLINE="console=ttyAMA3,115200n8 root=/dev/mmcblk0p9 rw"
> # CONFIG_CMDLINE_FROM_BOOTLOADER is not set
467,470c460,461
< # CONFIG_CMDLINE_FORCE is not set
< CONFIG_EFI_STUB=y
< CONFIG_EFI=y
< CONFIG_DMI=y
---
> CONFIG_CMDLINE_FORCE=y
> # CONFIG_EFI is not set
1063c1054,1055
< # CONFIG_DEVTMPFS is not set
---
> CONFIG_DEVTMPFS=y
> CONFIG_DEVTMPFS_MOUNT=y
1163c1155,1156
< # CONFIG_TIFM_CORE is not set
---
> CONFIG_TIFM_CORE=y
> CONFIG_TIFM_7XX1=y
1198c1191,1193
< # CONFIG_CB710_CORE is not set
---
> CONFIG_CB710_CORE=y
> # CONFIG_CB710_DEBUG is not set
> CONFIG_CB710_DEBUG_ASSUMPTIONS=y
3479,3480c3474,3477
< # CONFIG_MMC_SDHCI_PCI is not set
< # CONFIG_MMC_SDHCI_ACPI is not set
---
> CONFIG_MMC_SDHCI_IO_ACCESSORS=y
> CONFIG_MMC_SDHCI_PCI=y
> CONFIG_MMC_RICOH_MMC=y
> CONFIG_MMC_SDHCI_ACPI=y
3482,3485c3479,3482
< # CONFIG_MMC_SDHCI_OF_ARASAN is not set
< # CONFIG_MMC_SDHCI_OF_AT91 is not set
< # CONFIG_MMC_SDHCI_F_SDH30 is not set
< # CONFIG_MMC_TIFM_SD is not set
---
> CONFIG_MMC_SDHCI_OF_ARASAN=y
> CONFIG_MMC_SDHCI_OF_AT91=y
> CONFIG_MMC_SDHCI_F_SDH30=y
> CONFIG_MMC_TIFM_SD=y
3487,3488c3484,3485
< # CONFIG_MMC_CB710 is not set
< # CONFIG_MMC_VIA_SDMMC is not set
---
> CONFIG_MMC_CB710=y
> CONFIG_MMC_VIA_SDMMC=y
3493,3498c3490,3495
< # CONFIG_MMC_DW_PCI is not set
< # CONFIG_MMC_VUB300 is not set
< # CONFIG_MMC_USHC is not set
< # CONFIG_MMC_USDHI6ROL0 is not set
< # CONFIG_MMC_TOSHIBA_PCI is not set
< # CONFIG_MMC_MTK is not set
---
> CONFIG_MMC_DW_PCI=y
> CONFIG_MMC_VUB300=y
> CONFIG_MMC_USHC=y
> CONFIG_MMC_USDHI6ROL0=y
> CONFIG_MMC_TOSHIBA_PCI=y
> CONFIG_MMC_MTK=y
3524d3520
< # CONFIG_LEDS_INTEL_SS4200 is not set
3631d3626
< CONFIG_RTC_DRV_EFI=y
3882,3883d3876
< CONFIG_DMIID=y
< # CONFIG_DMI_SYSFS is not set
3886,3896d3878
< 
< #
< # EFI (Extensible Firmware Interface) Support
< #
< CONFIG_EFI_VARS=y
< CONFIG_EFI_ESRT=y
< CONFIG_EFI_VARS_PSTORE=y
< CONFIG_EFI_VARS_PSTORE_DEFAULT_DISABLE=y
< CONFIG_EFI_PARAMS_FROM_FDT=y
< CONFIG_EFI_RUNTIME_WRAPPERS=y
< CONFIG_EFI_ARMSTUB=y
4005d3986
< CONFIG_EFIVAR_FS=y
4557,4562d4537
< CONFIG_DECOMPRESS_GZIP=y
< CONFIG_DECOMPRESS_BZIP2=y
< CONFIG_DECOMPRESS_LZMA=y
< CONFIG_DECOMPRESS_XZ=y
< CONFIG_DECOMPRESS_LZO=y
< CONFIG_DECOMPRESS_LZ4=y
4585d4559
< CONFIG_UCS2_STRING=y
```

The goal of this configuration is to boot as simple as possible. No initial ramdisk is used and the arguments are set statically.

The commands used to build are as follows:

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- hikey_defconfig
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j8 Image  hisilicon/hi6220-hikey.dtb
```

For these tests I was using the image built for the HiKey. If it is helpful, the kernel can be replaced using the following flow, assuming you are working in the clone of Linux.

```bash
$ simg2img ./jessie.updated.img jessie.updated.fimg
$ sudo mount -o loop ./jessie.updated.fimg ./jessie
$ sudo cp ./arch/arm64/boot/dts/hisilicon/hi6220-hikey.dtb ../jessie/boot
$ sudo cp ./arch/arm64/boot/Image ../jessie/boot
$ sudo umount ./jessie
$ img2simg ./jessie.updated.fimg jessie.updated.simg
```

If you have a HiKey working, simply move the `hi6220-hikey.dtb` and `Image` to `/boot` on the board's filesystem.

## Build U-Boot as the non-trusted world firmware

The use of U-Boot over UEFI is not required. I am more familiar with U-Boot and it can bootstrap Linux and a device tree easily. Later on it may be replaced with booting straight to Linux from ATF.

```bash
$ git clone https://github.com/trini/u-boot
$ cd u-boot

$ make O=build_hikey hikey_defconfig
```

I simplified the `bootcommand` for easier debugging. This removes the default, flexible, U-Boot `bootcommand`:

```bash
$ cat ./build_hikey/.config| grep BOOTCOMMAND
CONFIG_USE_BOOTCOMMAND=y
CONFIG_BOOTCOMMAND="ext2load mmc 0:9 0x00080000 boot/Image; ext2load mmc 0:9 0x02000000 boot/hi6220-hikey.dtb; booti 0x00080000 - 0x02000000"
```

The goal is to load only Linux and the device tree.

The following change is needed because Linux is uncompressed:

```diff
index e789f68..bba2837 100644
--- a/common/bootm.c
+++ b/common/bootm.c
@@ -31,7 +31,7 @@
 
 #ifndef CONFIG_SYS_BOOTM_LEN
 /* use 8MByte as default max gunzip size */
-#define CONFIG_SYS_BOOTM_LEN   0x800000
+#define CONFIG_SYS_BOOTM_LEN   0x8000000
 #endif
 
 #define IH_INITRD_ARCH IH_ARCH_DEFAULT
```

And to build U-Boot:

```bash
$ make O=build_hikey -j6
[...]
  LD      u-boot
  OBJCOPY u-boot.srec
  OBJCOPY u-boot-nodtb.bin
  SYM     u-boot.sym
  DTC     arch/arm/dts/hi6220-hikey.dtb
make[3]: 'arch/arm/dts/hi6220-hikey.dtb' is up to date.
  SHIPPED dts/dt.dtb
  FDTGREP dts/dt-spl.dtb
  COPY    u-boot.dtb
  CAT     u-boot-dtb.bin
  COPY    u-boot.bin
  LD      u-boot.elf
  CFGCHK  u-boot.cfg
make[1]: Leaving directory './build_hikey'
```

## Build OP-TEE OS

OP-TEE has plenty of documentation. We will rush through this and focus on building a `BL32` example only.

```bash
$ mkdir optee
$ cd optee

$ git clone https://github.com/OP-TEE/optee_os
$ git clone https://github.com/OP-TEE/build

$ cd build
$ make -f hikey.mk \
  DEBUG=1 \
  AARCH64_CROSS_COMPILE=$AARCH64_PATH/bin/aarch64-linux-gnu- \
  AARCH32_CROSS_COMPILE=arm-linux-gnueabihf- \
  optee-os
```

## Build ATF without Trusted Board Boot

This is a stepping stone to booting securely. It will provide a `BL1` to use in the recovery image as well as verify the ATF code can boot our non-trusted world firmware, U-Boot, and our Linux. We will build ATF again with TBB enabled soon.

```bash
$ git clone https://github.com/ARM-software/arm-trusted-firmware
$ cd arm-trusted-firmware

$ make \
  PLAT=hikey DEBUG=1 \
  SCP_BL2=../mcuimage.bin \
  BL32=../optee/optee_os/out/arm/core/tee-header_v2.bin \
  BL32_EXTRA1=../optee/optee_os/out/arm/core/tee-pager_v2.bin \
  BL32_EXTRA2=../optee/optee_os/out/arm/core/tee-pageable_v2.bin \
  SPD=opteed \
  BL33=../u-boot/build_hikey/u-boot.bin \
  all fip
[...]
Building hikey
mkdir -p  "./build/hikey/debug/bl1"
mkdir -p  "./build/hikey/debug/bl2"
mkdir -p  "./build/hikey/debug/bl31"
make CPPFLAGS="-DVERSION='\"v1.5(debug):v1.5-285-g498161a\"'" --no-print-directory -C tools/fiptool

[...]
Built build/hikey/debug/fip.bin successfully
```

## Building `l-loader` and recovery mode

Recovery mode is booted by the HiKey when `Boot Select` is enabled, pin 3-4 are connected on `J15`. There is plenty of documentation for this: [https://github.com/96boards/documentation/wiki/HiKeyUEFI](https://github.com/96boards/documentation/wiki/HiKeyUEFI) but I would like to include the specifics of how I build a recovery mode image for completeness.

This will allow us to flash images during this walkthrough using `fastboot`.

To accomplish this we will use another version of ARM-Trusted-Firmware maintained by 96Boards. This will build a `bl1.bin` used in the `l-loader` build. We will only use this once and it will not be part of our boot flow, just the recovery flow. The recovery image is loaded into device memory over the USB OTG so we can fastboot flash the eMMC. This code will not persist on the device.

```bash
$ export CROSS_COMPILE=$AARCH64_PATH/bin/aarch64-linux-gnu-

$ git clone https://github.com/96boards-hikey/atf-fastboot
$ cd atf-fastboot

$ make DEBUG=1
[...]
  LD      build/hikey/debug/bl1/bl1.elf
  BIN     build/hikey/debug/bl1.bin

Built build/hikey/debug/bl1.bin successfully
```

> NB: We are not building for HiKey960, but we will use a checkout referencing the 960 board, sorry for this confusion.

Checkout the `l-loader` repository, which has Makefiles for building the recovery image.

```bash
$ git clone https://github.com/96boards-hikey/l-loader -b testing/hikey960_v1.2
$ cd l-loader

$ ln -s ../arm-trusted-firmware/build/hikey/debug/bl1.bin bl1.bin
$ ln -s ../arm-trusted-firmware/build/hikey/debug/fip.bin fip.bin
$ ln -s ../atf-fastboot/build/hikey/debug/bl1.bin fastboot.bin
```

Confusing right?

Make the recovery image using:

```bash
$ make -f hikey.mk PTABLE_LST=linux-8g recovery.bin
```

Make the `l-loader.bin` and `bl2.bin` image using:

```bash
$ make -f hikey.mk PTABLE_LST=linux-8g l-loader.bin
```

## Assembling the non-Secured boot flow

When CPU0 comes out of reset the bootrom will read the eMMC's `boot0` partition. This is memory mapped to address `0xF980:0000` and the first address executed is at offset `0x800`. This contains the `l-loader` code that switches execution from aarch32 to aarch64 and jumps to offset `0x1000`. `0xF980:1000` is the entrypoint of ARM-Trusted-Firmware's `BL2`.

You can dive into the build logic in the ARM-Trusted-Firmware code to understand why `BL1` is skipped as an optimization. In this case `BL2` is executed in `EL3`.

The ARM-Trusted-Boot flow takes over and `BL2` is our loader, executing the proprietary `mcuimage.bin` from HiKey, our OP-TEE secure OS as `BL32`, and eventually our `BL33` non-trusted firmware, U-Boot. U-Boot will auto-boot in 3 seconds unless interrupted on the command line.

> NB: U-Boot is configured to load an environment from the uSD card. In this example no uSD is needed or used. The U-Boot default environment, compiled into the image, is used.

U-Boot loads our Linux and device tree.

The `recovery.img` is used to program our content using fastboot. It uses the normal `l-loader` bootstrap code, the normal ATF `bl1.bin`, and the `atf-fastboot`'s `bl1.bin` as `NS_BL1U`. Using this recovery mode is optional. If you have a working board then you can flash the eMMC partitions using `dd`.

Here is an example:

```bash
$ sudo python ~/git/hikey/hisi-idt.py --img1=recovery.bin
$ sudo fastboot flash loader l-loader.bin
$ sudo fastboot flash fastboot fip.bin
```

If you have an already running HiKey you can use the following:

```bash
$ sudo dd if=fip.bin of=/dev/mmcblk0p4
$ echo 0 | sudo tee /sys/block/mmcblk0boot0/force_ro
$ sudo dd if=l-loader.bin of=/dev/mmcblk0boot0
```

## Build ATF with Trusted Board Boot for HiKey

The first thing we should do is create an RSA2048 keypair. ATF is limited to 2048bit keys at the moment.

```bash
$ mkdir keys
$ openssl genrsa 2048 > ./keys/rot.key 2>/dev/null
```

Patch your ATF to build Trusted Board Boot for the HiKey using: [https://patch-diff.githubusercontent.com/raw/ARM-software/arm-trusted-firmware/pull/1449.diff](https://patch-diff.githubusercontent.com/raw/ARM-software/arm-trusted-firmware/pull/1449.diff) be careful as this is still undergoing code review and has not been heavily tested.

This will build a `BL2` with the ROT key's public key SHA256 hash. The key will be placed into RW storage within the `fip.bin` later.

Checkout the mbedtls code, version 2.10.0:

```bash
$ git clone https://github.com/ARMmbed/mbedtls -b mbedtls-2.10.0
```

And finally glue it all together to create a `bl1.bin` and `fip.bin`:

```bash
$ cd arm-trusted-firmware

$ make \
  PLAT=hikey \
  DEBUG=1 \
  BL33=../u-boot/build_hikey/u-boot.bin \
  SCP_BL2=../mcuimage.bin \
  BL32=../optee/optee_os/out/arm/core/tee-header_v2.bin \
  BL32_EXTRA1=../optee/optee_os/out/arm/core/tee-pager_v2.bin \
  BL32_EXTRA2=../optee/optee_os/out/arm/core/tee-pageable_v2.bin \
  SPD=opteed \
  TRUSTED_BOARD_BOOT=1 \
  MBEDTLS_DIR=../../mbedtls \
  GENERATE_COT=1 \
  SAVE_KEYS=1 \
  TRUSTED_WORLD_KEY=../keys/trusted_world.key \
  NON_TRUSTED_WORLD_KEY=../keys/nt_worlded.key \
  ROT_KEY=../keys/rot.key \
  SCP_BL2_KEY=../keys/scp_bl2_content.key \
  BL31_KEY=../keys/soc_content.key \
  BL32_KEY=../keys/tos_content.key \
  BL33_KEY=../keys/nt_fw_content.key \
  all fip
```

The first time you run this command it should finish with some stdout:

```bash
NOTICE:  Creating new key for 'Trusted World key'
NOTICE:  Creating new key for 'Non Trusted World key'
NOTICE:  Creating new key for 'SCP Firmware Content Certificate key'
NOTICE:  Creating new key for 'SoC Firmware Content Certificate key'
NOTICE:  Creating new key for 'Trusted OS Firmware Content Certificate key'
NOTICE:  Creating new key for 'Non Trusted Firmware Content Certificate key'
```

Subsequent runs should be using your existing keys. Everything but the `rot.key` can be updated later since only the hash of the `rot.key` will be write-protected.

Note that the `_KEY` variables are optional. Keys will be generated during the build automatically for you.

## Basic Trusted Board Boot failure testing

As a very-quick smoke test for TBB I will edit a byte in U-Boot.

```bash
$ dd if=/dev/mmcblk0p4 of=fip.bin
$ grep -ab 2018.07-rc1-00132-g606fddd-dirty fip.bin 
530221:U-Boot 2018.07-rc1-00132-g606fddd-dirty (Jun 18 2018 - 22:17:00 -0400)hikey_SM__DMI_ERROR : memory not allocated
```

I will switch the 2018 to be 3018.

```bash
$ dd of=/dev/mmcblk0p4 if=fip.bin
16384+0 records in
16384+0 records out
8388608 bytes (8.4 MB) copied, 1.52253 s, 5.5 MB/s
```

And sure enough, after a reset I see on the debug UART:

```bash
[...]
INFO:    Loading image id=9 at address 0xf9858000
INFO:    Image id=9 loaded: 0xf9858000 - 0xf98584da
INFO:    Loading image id=13 at address 0xf9858000
INFO:    Image id=13 loaded: 0xf9858000 - 0xf9858430
INFO:    Loading image id=3 at address 0xf9858000
INFO:    Image id=3 loaded: 0xf9858000 - 0xf9861058
INFO:    BL2: Loading image id 5
INFO:    Loading image id=11 at address 0x35000000
INFO:    Image id=11 loaded: 0x35000000 - 0x350004ea
INFO:    Loading image id=15 at address 0x35000000
INFO:    Image id=15 loaded: 0x35000000 - 0x35000440
INFO:    Loading image id=5 at address 0x35000000
INFO:    Image id=5 loaded: 0x35000000 - 0x35061d9a

Platform exception reporting:
ESR_EL3: 0000000096000061
ELR_EL3: 00000000f9801574
```

Success! A more complete test involves generating a fake signature for U-Boot and writing a correct new SHA256 hash. But at least we know the auth-driver code is running.

## Danger Zone! Permanent Write Protect eMMC `boot0`

Once you are happy with the `BL2` build and **DO NOT PLAN ON UPDATING THE ROT.KEY OR BL2 CODE** we need to create our root of trust.

There is a feature of MMC similar to the hardware block write protection on SPI flash that allows you to write-protect the alternate boot partitions. **Be very very careful** as this is a 1-time operation and since this is a BGA eMMC, recovery is difficult. Remember we are only concerned with write-protecting the boot partitions. [Yifan](https://yifan.lu/2015/06/05/secure-your-emmc-devices/) has a quick read on the dangers of this write-protection being in an non-configured tri-state. While you are here you might consider locking the GP and User sections non-write-protect.

The `/dev/mmcblk0boot{0,1}` partitions hold the `l-loader.bin` code, this is `l-loader` and `BL2`. Everything else is within the `fip.bin` on the fourth partition. Only this `l-loader.bin` code will become permanent read-only/write-protected.

This uses the `eCSD[173]` register on the MMC, named `EXT_CSD_BOOT_WP`. We will 1-time program this with a value of `0x04` or `EXT_CSD_BOOT_WP_B_PERM_WP_EN`. Our Linux kernel allows programming via an `ioctl`.

On the HiKey:

```bash
(hikey)$ git clone https://github.com/chiel99/mmc-utils
(hikey)$ cd mmc-utils;
```

Apply this diff to change the `mmc` tool's write-protection logic from Power-On (write-protect until next power cycle) to permanent write-protection. Permanent write-protection is not available by default because of the danger mentioned above.

```diff
diff --git a/mmc_cmds.c b/mmc_cmds.c
index d7215bb..a32d317 100644
--- a/mmc_cmds.c
+++ b/mmc_cmds.c
@@ -279,7 +279,7 @@ int do_writeprotect_boot_set(int nargs, char **argv)
        }
 
        value = ext_csd[EXT_CSD_BOOT_WP] |
- EXT_CSD_BOOT_WP_B_PWR_WP_EN;
+               EXT_CSD_BOOT_WP_B_PERM_WP_EN;
        ret = write_extcsd_value(fd, EXT_CSD_BOOT_WP, value);
        if (ret) {
                fprintf(stderr, "Could not write 0x%02x to "
```

Then build and set the write-protection:

```bash
$ make
(DANGER)$ sudo ./mmc writeprotect boot set /dev/mmcblk0boot0
```

Now we can check the status:

```bash
$ sudo ./mmc writeprotect boot get /dev/mmcblk0boot0
Boot write protection status registers [BOOT_WP_STATUS]: 0x0a
Boot Area Write protection [BOOT_WP]: 0x04
 Power ro locking: possible
 Permanent ro locking: possible
 ro lock status: locked permanently

$ cat /sys/block/mmcblk0boot{0,1}/ro_lock_until_next_power_on 
2
2
```

Now the `boot0` partition is the only entrypoint from the bootrom, unless you modify physical properties like the J15 jumpers. This partition's content cannot be modified without an external programmer on the eMMC. In this configuration, from software perspective, the BL2 code is a ROM.

If you try to use `fastboot` in recovery mode, flashing the `l-loader` partition will not produce an error, but will not write the content, it silently fails.

The BL2 contains a permanent hash of the `rot.key`'s public key content and forces signature verification of all additional content down to the non-trusted firmware code. U-Boot is the last part of code verified, in this form it will happily run any Linux and device tree configuration. It is possible to later extend this chain of trust to the Kernel using U-Boot's verified boot implementation.

Again **be careful** and please **challenge my assertions** because I am not an authority and this DIY secured boot should not be used where high-security controls are needed. You should engage device manufactures and implement their ODM-preferred secured boot features. If you find any issues with the assumptions used here, please reach out!

## Limitations

- There is no secure counter implemented, this is dangerous as there is no way to revoke old builds and the keys used to sign.
- The `BOOT_WP` permanent feature write-protect of eMMC is not fully investigated as a secure method for starting a Root of Trust.
- The HiKey's bootrom is not confirmed to only boot from the eMMC's alternate boot partitions. There may be a software/eFUSE that can toggle this to boot another user-controlled location, such as a general purpose partition on the eMMC.
- The RSA key sizes used in ATF's Trusted Board Boot only support RSA2048 (SHA256 is used for hashing). From what I can see the ECDSA curves available in the ATF build provide an equivalent level of security. Increasing the relative bits available in the build is absolutely a follow up.
- The `certtool` in ATF cannot use a smartcard/HSM/PKCS11 to sign content.
- There is no runtime recovery option. If verification fails the board will hang.
