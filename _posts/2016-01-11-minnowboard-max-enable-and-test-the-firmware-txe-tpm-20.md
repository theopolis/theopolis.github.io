---
title: "Minnowboard Max: Enable the firmware (TXE) TPM 2.0"
created: 2016-01-11
tags: 
  - uefi
  - tpm

image: /assets/images/img.jpg
---

In the last two posts we walked through a 'Secured' Secure Boot of a Linux 4.1 kernel on the Intel Minnowboard MAX development board. If you have been following along, you have a device semi-resistant to boot and pre-boot tamperment. There is still a long road ahead, with a potential dead end for raising the Minnow's boot security to known-perfect, but between then and now is a lot of firmware fun!

This next step will detour a bit and provide a walkthrough of UEFI platform code modifications. Pre-built firmware updates for the Minnow, in binary form, can be downloaded on it's [firmware page](https://firmware.intel.com/projects/minnowboard-max)\-- as of January 10th 2016 the latest is version 0.84. The 64-bit image does **not enable the TPM 2.0**, whereas the 32-bit image will. I remember reading a mention in the 0.82 release notes that the 64bit disablement was related to either export controls or a licensing conflict.

<!--more-->

Perfect! Let's use this limitation as an opprotunity to build a UEFI firmware volumn to flash onto the Minnow's SPI chip. In our volumn we will use the lastest-supported 64bit TianoCore [EDK2 code](https://github.com/tianocore/edk2), the (also provided) Minnow FSF package, and enable the TPM 2.0. Unfortunately the Minnow's firmware is not 100% open but the process for slip-streaming the required binary objects is well-documented. One of the Minnow's purposes is to be a development and test platform for the UEFI EDK2 so we should assume as much TianoCore code as possible will be used in our source-built firmware.

### Minnow Hardware Security

There are a confusing number of hardware security features offered for Intel processors and compatible boards. The Minnowboard MAX is build using the [Bay Trail](http://ark.intel.com/products/codename/55844/Bay-Trail#@All) family of SoCs. My Max has a [E3825 Atom](http://ark.intel.com/products/78474/Intel-Atom-Processor-E3825-1M-Cache-1_33-GHz), so let's use that to discover the advertized security features. That ARK definition says:

- vPro Techology (dedicated environment that includes Intel AMT): NO
- VT-x (virtualization extensions): YES
- VT-d (IOMMU, device I/O isolation): NO
- TXT (Trusted Execution Technology): NO
- Execute Disable Bit: YES

This is very much standard for the lower-end models, but in the Bay Trail, Valley View case, slightly misleading. The _vPro_ set to NO technically does not mean the [Intel ME](https://en.wikipedia.org/wiki/Intel_Active_Management_Technology) (or SEC/TXE) is absent, but the Valley View uses a firmware TPM implemented within this environment. It would be nice to see this feature highlighted on the ARK documentation.

As a very very breif aside, environments like the ME can be a double-edged sword. If a 'secured' management environment provides blackbox remote administration capabilities it will be the target of attack, escalation, and privilege. At 44CON in 2013 Patrick Stewin and Iurii Bystrov [demonstrated](assets/44con_2013-dedicated_hw_malware-stewin_bystrov.pdf) a vPRO/AMT take-over by injecing user-controller code. Unrelated to the Minnow, but the [ITL researchers](http://invisiblethingslab.com/itl/Resources.html) have contributed foundational work on assesing TXT, IOMMU; The [LegbaCore](http://www.legbacore.com/Research.html) team and Intel's [ATR](http://www.intelsecurity.com/advanced-threat-research/) team have similarly been working on UEFI update mechanics, SMM, and measurement/authentication assumptions.

To summarize, this Bay Trail package does not contain a discrete TPM, but rather emulates a hardware TPM through a dedicated and isolated environment esoterically referred to as the SEC/TXE (trusted execution environment), the 3rd version of the Intel ME. This emulation-style TPM is called a fTPM for firmware-based TPM. The Bay Trail firmware TPM is a TrEE (also, trusted execution environment) ACPI TPM2.0 device. Support for the 32bit firmware has existed [since 0.79](https://firmware.intel.com/blog/security-technologies-and-minnowboard-max). Please see the following articles and specifications for the gory details:

1. Jiewen Yao and Vincent Zimmer's tour of [UEFI TPM2 support](assets/A_Tour_Beyond_BIOS_Implementing_TPM2_Support_in_EDKII.pdf) in EDKII
2. Microsoft's overview of [TrEE ACPI protocol](https://msdn.microsoft.com/en-us/library/windows/hardware/jj923067(v=vs.85).aspx) and [EFI protocols](https://msdn.microsoft.com/en-us/library/windows/hardware/jj923068(v=vs.85).aspx)
3. The Trusted Computing Group's set of [TPM 2.0 library](http://www.trustedcomputinggroup.org/resources/tpm_library_specification) resources including specifications
4. Igor Skochinsky's most recent [overview of Intel ME and SEC/TXE](assets/Recon%202014%20Skochinsky.pdf) including Bay Trail's SPARC implementation

### Build a Minnowboard UEFI update

By default there are no protections for updating the UEFI platform code on the Minnow. But there are also no runtime facilities for automatically updating the platform code. In the earlier articles we updated firmware using the provided "flash utilities" to write the contents of a file to SPI flash from within the EFI shell. You can do the same from the OS pretty simply with the CHIPSEC platform too!

When we previously enabled Secure Boot and restricted/edited the `db`, `KEK`, and `PK` credential stores we also 'locked down' EFI applications from running without a valid signature/hash. So the `MinnowBoard.MAX.FirmwareUpdateX64` application(s) will NOT run and thus will not be able to write SPI. Either have some fun with CHIPSEC's SPI writing features, sign your flash EFI application, or disable Secure Boot within the UEFI Setup. _NOTE_ this method of writing SPI with the contents of a flash descriptor will erase your NVRAM so disabling Secure Boot for the interrum makes the most sense. Once the firmware update is applied, we will need to set up Secure Boot against from the beginning.

Let's begin! This outline will very-closely follow the notes in the [0.84 release notes](https://firmware.intel.com/sites/default/files/MinnowBoard.MAX_.0.84.BIN-ReleaseNotes.txt). This assumes a Ubuntu 14.04.3 updated development environment.

```bash
# Install some dependent packages
$ sudo apt-get install build-essential bison flex iasl uuid-dev libc6:i386 libncurses5:i386 libstdc++6:i386
$ git clone https://github.com/tianocore/edk2
[...]
$ cd edk2
# git checkout svn/branches/UDK2014.SP1
# git log | grep 19122 -B 20
$ git checkout -b minnow-development 3325a69d44c2fcfbeefce344ed6b020b0840f2e2
$ git clone https://github.com/tianocore/edk2-FatPkg FatPkg
[...]
```

Now download the binary components of the Valley View setup. Unfortunately for us, almost all of the TPM2 device initialization code exists as SEC DXE drivers, use of the Intel ME HECI API, (and some seemingly-related SMM code). And those are only available as compiled UEFI files.

```bash
$ mkdir /tmp/downloads
$ pushd /tmp/downloads
$ wget https://firmware.intel.com/sites/default/files/MinnowBoard_MAX-0.84-Binary.Objects.zip
# sudo apt-get install unzip
$ unzip MinnowBoard_MAX-0.84-Binary.Objects.zip
$ cp -R ./Minnow*/* $OLDPWD/
$ popd
```

Just to make sure, your working directory should look like:

```bash
$ git status
On branch minnow-development
Untracked files:
  (use "git add <file>..." to include in what will be committed)

        FatPkg/
        IA32FamilyCpuPkg/
        Vlv2BinaryPkg/
        Vlv2MiscBinariesPkg/
```

We have to do some EDKII hand-to-hand combat and upgrade our checkout's expected OpenSSL version from 1.0.2d to 1.0.2e.

```bash
$ pushd ./CryptoPkg/Library/OpensslLib
$ wget http://www.openssl.org/source/openssl-1.0.2e.tar.gz
$ tar xzf openssl-1.0.2e.tar.gz
$ rm openssl-1.0.2e.tar.gz
$ pushd openssl-1.0.2e
$ patch -p0 -i ../EDKII_openssl-1.0.2d.patch
$ popd
$ vi Install.sh # (change d to e)
$ ./Install.sh
$ popd
# Verify EDKII sanity
$ bash edksetup.sh
```

Now we're nearly ready to build the needed components from EDKII, OpenSSL, the open source Valley View components, and binary objects. Ubuntu 14.04.3's **build-essential** package (or an **apt-get upgrade**) should pull in a GCC4.8 compiler. The build scripts should be updated to prefer this 4.8 version, we'll need to edit that as well as edit a final hard-coded binding to OpenSSL's version:

```diff
diff --git a/Vlv2TbltDevicePkg/bld_vlv.sh b/Vlv2TbltDevicePkg/bld_vlv.sh
index 569865f..33284b9 100755
--- a/Vlv2TbltDevicePkg/bld_vlv.sh
+++ b/Vlv2TbltDevicePkg/bld_vlv.sh
@@ -178,7 +178,7 @@ sed -i '/^TOOL_CHAIN_TAG/d' Conf/target.txt
 sed -i '/^MAX_CONCURRENT_THREAD_NUMBER/d' Conf/target.txt

 ACTIVE_PLATFORM=$PLATFORM_PACKAGE/PlatformPkgGcc"$Arch".dsc
-TOOL_CHAIN_TAG=GCC46
+TOOL_CHAIN_TAG=GCC48
```

```diff
diff --git a/CryptoPkg/Library/OpensslLib/OpensslLib.inf b/CryptoPkg/Library/OpensslLib/OpensslLib.inf
index 28d3aec..0da954b 100644
--- a/CryptoPkg/Library/OpensslLib/OpensslLib.inf
+++ b/CryptoPkg/Library/OpensslLib/OpensslLib.inf
@@ -20,7 +20,7 @@
   MODULE_TYPE                    = BASE
   VERSION_STRING                 = 1.0
   LIBRARY_CLASS                  = OpensslLib
- DEFINE OPENSSL_PATH            = openssl-1.0.2d
+  DEFINE OPENSSL_PATH            = openssl-1.0.2e
```

If all goes according to plan, and it seldomly does, the Valley View build script will handle compiling and firmware descriptor creation. If you were paying close attention at the begining of the build adventure, you'll have noticed the installation of a 32bit C/C++ runtimes. The firmware package here ships will 32bit compiled binaries that WILL EXECUTE during build.

```bash
$ pushd Vlv2TbltDevicePkg
# This will execute dependent 32bit ELF binary objects
$ ./Build_IFWI.sh MNW2 Release
[...]
./Vlv2TbltDevicePkg/Stitch/IFWIHeader/IFWI_HEADER.bin
Skip Running fce...
Skip Running KeyEnroll...
Skip Running BIOS_Signing ...

Build location: Build/Vlv2TbltDevicePkg/RELEASE_GCC48
BIOS ROM Created: MNW2MAX_X64_R_0084_01.ROM
$ popd
```

### Enable the fTPM in UEFI

The above build did not change any options or configuration. Aside from changing the OpenSSL version, the output should be as close to the Intel-shipped firmware image as possible. This means the output 64bit firmware platform code does NOT enable the fTPM.

The Minnowboard/Valley View firmware developers are amazing and have reduced this enablement into 2 variables. The first will enable the SEC/TXE by including the initialization drivers and [HECI API](https://en.wikipedia.org/wiki/Host_Embedded_Controller_Interface) interfaces; the second will add the similar proprietary fTPM driver code AND configure the platform to include the TrEE ACPI/PPI and measurement stack. Now let's flip those variables, correct some path delimiters, to rebuild with an fTPM.

```diff
diff --git a/Vlv2TbltDevicePkg/PlatformPkgGccX64.dsc b/Vlv2TbltDevicePkg/PlatformPkgGccX64.dsc
index 680dd5a..cee9b55 100644
--- a/Vlv2TbltDevicePkg/PlatformPkgGccX64.dsc
+++ b/Vlv2TbltDevicePkg/PlatformPkgGccX64.dsc
@@ -77,9 +77,9 @@

   DEFINE   PLATFORM_PCIEXPRESS_BASE   = 0E0000000

- DEFINE SEC_ENABLE = FALSE
- DEFINE SEC_DEBUG_INFO_ENABLE = FALSE
- DEFINE FTPM_ENABLE = FALSE
+  DEFINE SEC_ENABLE = TRUE
+  DEFINE SEC_DEBUG_INFO_ENABLE = TRUE
+  DEFINE FTPM_ENABLE = TRUE

 ################################################################################
 #
@@ -1024,7 +1024,7 @@ $(PLATFORM_BINARY_PACKAGE)/$(DXE_ARCHITECTURE)$(TARGET)/IA32/fTPMInitPeim.inf
       gEfiMdePkgTokenSpaceGuid.PcdDebugPrintErrorLevel|0x80000046
     <LibraryClasses>
       DebugLib|MdePkg/Library/BaseDebugLibSerialPort/BaseDebugLibSerialPort.inf
- NULL|SecurityPkg/Library\HashInstanceLibSha1/HashInstanceLibSha1.inf
+      NULL|SecurityPkg/Library/HashInstanceLibSha1/HashInstanceLibSha1.inf
       NULL|SecurityPkg/Library/HashInstanceLibSha256/HashInstanceLibSha256.inf
       PcdLib|MdePkg/Library/PeiPcdLib/PeiPcdLib.inf
   }
@@ -1269,7 +1269,7 @@ $(PLATFORM_BINARY_PACKAGE)/$(DXE_ARCHITECTURE)$(TARGET)/IA32/fTPMInitPeim.inf
     <LibraryClasses>
       NULL|SecurityPkg/Library/HashInstanceLibSha1/HashInstanceLibSha1.inf
       NULL|SecurityPkg/Library/HashInstanceLibSha256/HashInstanceLibSha256.inf
- PcdLib|MdePkg/Library\DxePcdLib/DxePcdLib.inf
+      PcdLib|MdePkg/Library/DxePcdLib/DxePcdLib.inf
       Tpm2DeviceLib|Vlv2TbltDevicePkg/Library/Tpm2DeviceLibSeCDxe/Tpm2DeviceLibSeC.inf
   }
   $(PLATFORM_BINARY_PACKAGE)/$(DXE_ARCHITECTURE)$(TARGET)/$(DXE_ARCHITECTURE)/FtpmSmm.inf
```

I recommend editing the build revision now such that you end up with the existing `MNW2MAX_X64_R_0084_01.ROM` and a second `MNW2MAX_X64_R_0084_02.ROM`. If you want to later diff the ouput with the [uefi\_firmware](https://pypi.python.org/pypi/uefi_firmware) python module you can verify the added UEFI files.

```bash
$ pushd Vlv2TbltDevicePkg
$ vi BiosIdx64R.env # change VERSION_MINOR = 02
$ ./Build_IFWI.sh MNW2 Release
[...]
FV Space Information
FVMAIN [99%Full] 8163328 total, 8161936 used, 1392 free
MICROCODE_FV [60%Full] 262144 total, 157792 used, 104352 free
FVRECOVERY [72%Full] 253952 total, 183824 used, 70128 free
FVMAIN_COMPACT [88%Full] 1662976 total, 1478136 used, 184840 free
FVRECOVERY2 [88%Full] 180224 total, 159144 used, 21080 free
UPDATE_DATA [99%Full] 4198400 total, 4194400 used, 4000 free

- Done -
Build end time: 21:50:34, Jan.02 2016
Build total time: 00:02:38

./Vlv2TbltDevicePkg/Stitch/IFWIHeader/IFWI_HEADER.bin
Skip Running fce...
Skip Running KeyEnroll...
Skip Running BIOS_Signing ...

Build location: Build/Vlv2TbltDevicePkg/RELEASE_GCC48
BIOS ROM Created: MNW2MAX_X64_R_0084_02.ROM
$ popd
```

### Flashing SPI and restoring UEFI Setup

I recommend moving the `MNW2MAX_X64_R_0084_02.ROM` file to the EFI System Partition (ESP), the first partition on the Minnow's MMC. You can place the required `MinnowBoard.MAX.FirmwareUpdateX64.efi` alongside. Then boot into the UEFI Shell by interrupting the boot with F2.

Shell> fs0:\\EFI\\MinnowBoard.MAX.FirmwareUpdateX64.efi fs0:\\EFI\\MNW2MAX\_X64\_R\_0084\_02.ROM
# If you see 'Security Violation\` you need to disable Secure Boot

Now, let's reboot while feindishly mashing F2 while we hope the firmware boots the platform. It may be a tad late to mention, but you can recover from a failed SPI flash with several tools such as a Bus Pirate or [Dediprog SF100](assets/Flashing_MinnowBoard_MAX_with_Dediprog_SF100_in_Linux.pdf).

If every thing has gone according to plan then navigate to: `Device Manager - System Setup - Security Configuration - PTT`

![Screenshot of the Security Configuration SETUP with PTT and Measured Boot Enabled.](/assets/images/max-enable-tpm-20-1.jpeg)

Screenshot of the Security Configuration SETUP with PTT and Measured Boot Enabled.

All of the NVRAM variables will be erased, so Secure Boot cert stores and enablement will need to be restored as well as the boot entry for vmlinuz. Return to the `Boot Maintainence` to add and reorder the boot options.

### Verifing fTPM in Linux

The updated firmware version string will be reflected in the SMBIOS, but a helpful verification is also in `dmesg`:

```bash
[    0.000000] efi:  ACPI=0x78dbd000  ACPI 2.0=0x78dbd014  SMBIOS=0x78474000
[    0.000000] DMI: Circuitco Minnowboard Max D0 PLATFORM/MinnowBoard MAX, BIOS MNW2MAX1.X64.0084.R02.1601022230 01/02/2016
```

Whereas discrete TPMs are available on a system bus like LPC or I2C, the fTPM utilizes an ACPI-based API. Dumping/listing the ACPI tables should include the fTPM. Newer Linux kernels use this ACPI entry and a replacement to the historic TIS driver to expose a `tpm0` device node.

```bash
$ sudo acpidump -s
ACPI: RSDP 0x0000000078DBD014 000024 (v02 INTEL )
ACPI: RSDT 0x0000000078DBC074 000060 (v01 INTEL  EDK2     00000003      01000013)
ACPI: XSDT 0x0000000078DBC0E8 00009C (v01 INTEL  EDK2     00000003      01000013)
ACPI: DSDT 0x0000000078DAE000 007AA6 (v02 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: FACS 0x0000000078D02000 000040
ACPI: FACP 0x0000000078DBA000 00010C (v05 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: TCPA 0x0000000078DBB000 000032 (v02 INTEL  EDK2     00000002      01000013)
ACPI: UEFI 0x0000000078D05000 000042 (v01 INTEL  EDK2     00000002      01000013)
ACPI: HPET 0x0000000078DB9000 000038 (v01 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: LPIT 0x0000000078DB8000 000104 (v01 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: APIC 0x0000000078DB7000 000084 (v03 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: MCFG 0x0000000078DB6000 00003C (v01 INTEL  EDK2     00000003 VLV2 0100000D)
ACPI: SSDT 0x0000000078DAD000 0004AC (v01 INTEL  RHPROXY  00000003 VLV2 0100000D)
ACPI: SSDT 0x0000000078DAC000 00043A (v01 Intel_ Tpm2Tabl 00001000 INTL 20120518)
ACPI: TPM2 0x0000000078DAB000 000034 (v03                 00000000      00000000)
ACPI: SSDT 0x0000000078DAA000 000763 (v01 PmRef  CpuPm    00003000 INTL 20140214)
ACPI: SSDT 0x0000000078DA9000 000261 (v01 PmRef  Cpu0Tst  00003000 INTL 20140214)
ACPI: SSDT 0x0000000078DA8000 00017A (v01 PmRef  ApTst    00003000 INTL 20140214)
ACPI: CSRT 0x0000000078DA7000 00014C (v00 INTEL  EDK2     00000005 INTL 20120624)
ACPI: FPDT 0x0000000078DA6000 000044 (v01 INTEL  EDK2     00000002      01000013)
ACPI: SSDT 0x0000000000000000 000311 (v01 PmRef  Cpu0Ist  00003000 INTL 20140214)
ACPI: SSDT 0x0000000000000000 000233 (v01 PmRef  Cpu0Cst  00003001 INTL 20140214)
ACPI: SSDT 0x0000000000000000 00015F (v01 PmRef  ApIst    00003000 INTL 20140214)
ACPI: SSDT 0x0000000000000000 00008D (v01 PmRef  ApCst    00003000 INTL 20140214)
$ stat /dev/tpm0
  File: ‘/dev/tpm0’
  Size: 0               Blocks: 0          IO Block: 4096   character special file
Device: 6h/6d   Inode: 361         Links: 1     Device type: a,e0
Access: (0600/crw-------)  Uid: (  117/     tss)   Gid: (  126/     tss)
Access: 2016-01-01 16:04:47.804106974 -0800
Modify: 2016-01-01 16:04:47.804106974 -0800
Change: 2016-01-01 16:04:47.804106974 -0800
 Birth: -
```

If your kernel version (4.2 or earlier) does not detect the ACPI entry and likewise create the device node, update to the 4.4 kernel. Aside from enabling TPM support, the only option needed is `TPM_CRB`, the Command Response Buffer-style transport.

### Develop with a TPM2 TSS stack

Unlike TPM1.2 devices, our 2.0 TPM suffers from a lack of supporting software stacks. There are two in-development implementations of the TPM2.0-TSS by Intel and IBM. I recommend the [Intel TPM2.0-TSS](https://github.com/01org/TPM2.0-TSS) as the development takes place on Github and includes local TCP implementation of the specification's Resource Manager concept. This is the 2.0 version of [TrouSerS](http://trousers.sourceforge.net/)' `tcsd` daemon. TrouSerS, [Trusted Grub2](https://github.com/Sirrix-AG/TrustedGRUB2/), tpm-tools, and [Open Attestation](https://github.com/OpenAttestation/OpenAttestation) are all limited to 1.2 TPMs.

Let's install TPM2.0-TSS and run through an example on how to use beta library to list the TPM PCRs. If our UEFI platform code enabled the measurement log we'll have some deterministic values in the PCRs, predictable across boots, ready to be attested.

```bash
$ sudo apt-get install autoconf autoconf-archive libtool
$ git clone https://github.com/01org/TPM2.0-TSS
Cloning into 'TPM2.0-TSS'...
remote: Counting objects: 2664, done.
remote: Total 2664 (delta 0), reused 0 (delta 0), pack-reused 2664
Receiving objects: 100% (2664/2664), 12.55 MiB | 5.29 MiB/s, done.
Resolving deltas: 100% (2140/2140), done.
Checking connectivity... done.
$ pushd TPM2.0-TSS
$ ./bootstrap
[...]
$ mkdir build
$ pushd build
$ ../configure
[...]
$ make -j2
[...]
$ sudo make install
$ popd
$ popd
```

Now we'll execute the installed resource manager. This will start a service in the foreground and wait for connections from TPM tools/projects. The resource manager's function and API is documented in the TPM2.0 specifications.

```bash
$ sudo ./src/resourcemgr
Initializing local TPM Interface
Initializing Resource Manager
maxActiveSessions = 64
gapMaxValue = 65535
socket created:  0x4
bind to IP address:port:  127.0.0.1:2324
Other CMD server listening to socket:  0x4
socket created:  0x5
bind to IP address:port:  127.0.0.1:2323
TPM CMD server listening to socket:  0x5
Starting SockServer (TPM CMD), socket: 0x5.
Starting SockServer (Other CMD), socket: 0x4.
[...]
```

The TPM2.0-TSS project is under heavy development and filling up with unit tests. Over the next few months expect a more-robust testing and reporting framework. The companion project [tpm2.0-tools](https://github.com/01org/tpm2.0-tools) is the downstream tested consumer of the TSS. Explore that project for example implementations of most of the TPM TCTI.

Also feel free to check out a small [tpm2-examples](https://github.com/theopolis/tpm2-examples) project created for this tutorial. The project demonstrates linking against the installed TSS libraries and simply reports the TPM manufacturer and lists PCRs. These are only two of the vast number of TCTI commands.

```bash
$ git clone https://github.com/theopolis/tpm2-examples
Cloning into 'tpm2-examples'...
remote: Counting objects: 26, done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 26 (delta 4), reused 4 (delta 4), pack-reused 20
Unpacking objects: 100% (26/26), done.
Checking connectivity... done.
$ pushd tpm2-examples/
$ make
clang++ -Wall -g -std=c++11 -c tpm2.cpp
clang++ -Wall -g -std=c++11 -c tpm2_pcrs.cpp
clang++ -Wall -g -std=c++11 -L /usr/local/lib/ -ltpm2sapi -ltpm2tctisock tpm2.o tpm2_pcrs.o -o tpm2_pcrs
clang++ -Wall -g -std=c++11 -c tpm2_info.cpp
clang++ -Wall -g -std=c++11 -L /usr/local/lib/ -ltpm2sapi -ltpm2tctisock tpm2.o tpm2_info.o -o tpm2_info
$ ./tpm2_info
TPM SPEC Version: 0x005d
TPM Manufacturer: 0x494e5443
TPM Manufacturer Name: INTC
$ ./tpm2_pcrs
PCR 0: 0xe1883362094cd66c63380c1ea51beb29f9ff1ef8
PCR 1: 0x0916e8d1be7270ff126db49a14f3c19b77ccf3a4
PCR 2: 0xb2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
PCR 3: 0xb2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
PCR 4: 0x2a7672933eaa892196281e660fd4aa84df973921
PCR 5: 0x2077a48d1b606b12b8374b2c5f16fdf1d888e6cd
PCR 6: 0xb2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
PCR 7: 0x518bd167271fbb64589c61e43d8c0165861431d8
PCR 8: 0x0000000000000000000000000000000000000000
PCR 9: 0x0000000000000000000000000000000000000000
PCR a: 0x0000000000000000000000000000000000000000
PCR b: 0x0000000000000000000000000000000000000000
PCR c: 0x0000000000000000000000000000000000000000
PCR d: 0x0000000000000000000000000000000000000000
PCR e: 0x0000000000000000000000000000000000000000
PCR f: 0x0000000000000000000000000000000000000000
PCR 10: 0x0000000000000000000000000000000000000000
PCR 11: 0xffffffffffffffffffffffffffffffffffffffff
$ popd
```

### TrEE Measurement Implementation within UEFI

The Microsoft [articles on TrEE](https://msdn.microsoft.com/en-us/library/windows/hardware/jj923068.aspx) referenced above does a very good job at outlining the interaction of the UEFI 2.3.1 Secure Boot policy, the asserts made by Windows Secure Boot, the TrEE EFI protocol API, and what TPM 2.0 PCRs should be used for various datums.

At a high level, stolen from the article, the UEFI platform code we just flashed is responsible for measuring:

- Platform firmware that contains or measures the UEFI Boot Services and UEFI Runtime Services
- Security relevant variables associated with platform firmware
- UEFI drivers or boot applications loaded separately
- Variables associated with separately loaded UEFI Drivers or UEFI Boot applications

Let's look into some of the TrEE code within the EDKII and pick out some of the measurement sites.

From **SecurityPkg**: **/Library/DxeTpm2MeasureBootLib/[DxeTpm2MeasureBootLib.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeTpm2MeasureBootLib/DxeTpm2MeasureBootLib.c)**

```c
/// Excerpt: For each PE UEFI file executed:
  Tcg2Event->Size = EventSize + sizeof (EFI_TCG2_EVENT) - sizeof(Tcg2Event->Event);
  Tcg2Event->Header.HeaderSize    = sizeof(EFI_TCG2_EVENT_HEADER);
  Tcg2Event->Header.HeaderVersion = EFI_TCG2_EVENT_HEADER_VERSION;
  ImageLoad           = (EFI_IMAGE_LOAD_EVENT *) Tcg2Event->Event;
  switch (ImageType) {
    case EFI_IMAGE_SUBSYSTEM_EFI_APPLICATION:
      TreeEvent->Header.EventType = EV_EFI_BOOT_SERVICES_APPLICATION;
      TreeEvent->Header.PCRIndex  = 4;
      break;
    case EFI_IMAGE_SUBSYSTEM_EFI_BOOT_SERVICE_DRIVER:
      TreeEvent->Header.EventType = EV_EFI_BOOT_SERVICES_DRIVER;
      TreeEvent->Header.PCRIndex  = 2;
      break;
    case EFI_IMAGE_SUBSYSTEM_EFI_RUNTIME_DRIVER:
      TreeEvent->Header.EventType = EV_EFI_RUNTIME_SERVICES_DRIVER;
      TreeEvent->Header.PCRIndex  = 2;
      break;
    default:
      DEBUG ((
        EFI_D_ERROR,
        "TrEEMeasurePeImage: Unknown subsystem type %d",
        ImageType
        ));
      goto Finish;
  }

  Status = TreeProtocol->HashLogExtendEvent (
             TreeProtocol,
             PE_COFF_IMAGE,
             ImageAddress,
             ImageSize,
             TreeEvent
             );

/// Excerpt: For GUID partitions tables:
  TreeEvent->Size = EventSize + sizeof (TrEE_EVENT) - sizeof(TreeEvent->Event);
  TreeEvent->Header.HeaderSize    = sizeof(TrEE_EVENT_HEADER);
  TreeEvent->Header.HeaderVersion = TREE_EVENT_HEADER_VERSION;
  TreeEvent->Header.PCRIndex      = 5;
  TreeEvent->Header.EventType     = EV_EFI_GPT_EVENT;
```

In the _Tcg_ namespace there are DXE drivers and PEI code to perform most of the measuring. Whereas the above library is provided for fetch/execute callsites most important of which is DXE driver execution and application launch.

From **SecurityPkg**: **/Tcg/TrEEDxe/[TrEEDxe.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Tcg/TrEEDxe/TrEEDxe.c)**

There are tons of call sites worthy of including but instead I'll provide a short summary. "Action strings", SMBIOS tables, processor package physical locations (the slots), the Secure Boot policy variables like `PK`, `KEK`, `SecureBoot`, `db`, `dbx`, and the boot policy are measured. The boot order and values for each `Boot%04x` options are extended into `PCR[5]`. Later we can edit the order or the value of an option and observe the PCRs change.

From **SecurityPkg**: **/Tcg/TrEEPei/[TrEEPei.c](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Tcg/TrEEPei/TrEEPei.c)**

```c
/// Excerpt: Establish a CTRM from within PEI using the firmware version string
  TcgEventHdr.PCRIndex  = 0;
  TcgEventHdr.EventType = EV_S_CRTM_VERSION;
  TcgEventHdr.EventSize = (UINT32) StrSize((CHAR16*)PcdGetPtr (PcdFirmwareVersionString));

/// Excerpt: Also measure the platform code into PCR 0.
  TcgEventHdr.PCRIndex = 0;
  TcgEventHdr.EventType = EV_EFI_PLATFORM_FIRMWARE_BLOB;
  TcgEventHdr.EventSize = sizeof (FvBlob);
```

A massive thanks to Guo Dong, Jiewin Ywo, Star Zeng, Chao Zhang, Eric Dong, and the several others for writing and maintaining this open implementation within TianoCore.

The TrEE EFI protocol is defined in **MdePkg/Include/Protocol/TrEEProtocol.h** and more-or-less consumed by the Security package. Later in the boot, after execution has passed to a bootloader (in our case the EFI-bootable kernel), but before `ExitBootServices()` is called-- a log of measurements can be obtained. This log should represent the quotable/attestable state of the respective TPM registers. If an attesting server replays the hash extensions the signed values should match. This is extremely helpful for responding and alerting (as well as debugging) failed attestations.

Trusted Grub2 records this [event log](https://github.com/Sirrix-AG/TrustedGRUB2/blob/150e0fac9ca418fbd16cf327e220b250adf5771f/grub-core/kern/i386/pc/tpm/tpm_kern.c#L139) for a TPM1.2 device. Unfortunately there is no bootloader making use of the TrEE measurement log, and the data is removed when an OS gains control. Sounds like an opprotunity! We are measuring our OS, meaning the EFI bootloader code, Linux kernel, and a slip-streamed initial ramdisk environment. It is possible to extend the early-boot EFI code in Linux to keep the measurement log data around, then perform a retreival and quote/attestation using the TPM2.0-TSS stack within the ramdisk environment before releasing control to init. A first step would be to perform the quote of the STRM without log, just on the accumulated (and separated) PCR values.

We will put the TPM and trusted computing concepts on hold for a few articles while we explore detecting compromise in these environment and work towards locking down the firmware environment and securing some of the binary code and SMM. Thanks for reading and please stay tuned!
