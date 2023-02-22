---
title: "Minnowboard Max: Booting Linux Securely"
created: 2015-11-01
tags: 
  - uefi
  - firmware
  - linux

image: /assets/images/img.jpg
---

Linux supports UEFI Secure Boot, and works out-of-the-box on boards that 'only' include Microsoft UEFI certificates, using a bootloader shim. The [shim](https://github.com/rhinstaller/shim) is a small UEFI bootloader distributed in binary form ([source code](https://bugs.launchpad.net/ubuntu/+source/grub2-signed/+bug/1401532)), signed by Microsoft. At the most basic level the shim allows Canonical, RHEL/Fedora, and other signed grub bootloaders to be executed when UEFI Secure Boot is enabled. The shim allows system owners to extend trust beyond Microsoft without needing to modify protected UEFI variables. This is awesome because you and I can install Linux from a USB and not worry about UEFI and Secure Boot details. We also don’t need to turn off any security features to do cool stuff like run our favorite OS.

<!--more-->

Newish desktops, laptops, and other systems, might come with Secure Boot enforcement enabled; those system owners can install Ubuntu and get 'for free' a more-or-less verified boot starting with their UEFI firmware and extended all the way to their kernel. I say 'more-or-less' because there are **tons of places where the verification can be subverted**. Unfortunately, if you start examining the implementation and configuration details of the streamlined Secure Boot support, you'll find plenty of bypasses.

Let's talk briefly about each bypass and conclude with a simple way to use Secure Boot and enforce a signed kernel execution on Ubuntu. **To be clear, there are no vulnerabilities here as there is no documented intention to boot Linux securely** (e.g., [BUG/1401532](https://bugs.launchpad.net/ubuntu/+source/grub2-signed/+bug/1401532)), only to support a Secure Boot and boot Linux. To be super clear, this is echoed in the [Secure Boot article](https://wiki.ubuntu.com/SecurityTeam/SecureBoot) on Ubuntu's documentation:

> The 2nd stage grub2 bootloader boots an Ubuntu kernel (as of 2012/11, if the kernel (linux-signed) is signed with the 'Canonical Ltd. Secure Boot Signing' key, then grub2 will boot the kernel which will in turn apply quirks and call ExitBootServices. If the kernel is unsigned, grub2 will call ExitBootServices before booting the unsigned kernel)

> IMPORTANT: Canonical's Secure Boot implementation is primarily about hardware-enablement and this page focuses on how to test Secure Boot for common hardware-enablement configurations, not for enabling Secure Boot to harden your system. If you want to use Secure Boot as a security mechanism, an appropriate solution would be to use your own keys (optionally enrolling additional keys, see above) and update the bootloader to prohibit booting an unsigned kernel. Future releases of Ubuntu may make this easier

[Matthew Garrett's](https://mjg59.dreamwidth.org/) blog has plenty of related articles, if you enjoy all things Secure Boot, measurement, trusted computing and Linux related you will love his work. Immediately following, if not during, start reading [Peter Jones'](http://blog.uncooperative.org/) articles-- these folks are very influential in bringing Linux Secure Boot to life; they do amazing work!

### Building a 4.2 kernel for the MinnowBoard Max

This is the second article of in a series focused on hardening a Minnowboard Max. Check out the [first article](http://prosauce.org/blog/2015/9/13/minnowboard-max-quickstart-uefi-secure-boot), which will guide you through a Secure Boot-supported boot on Ubuntu. Let's first get a development/testing environment setup for compiling bootloaders and kernels for a Minnowboard Max target. Assuming a default install of Ubuntu on the Minnow, your boot should be similar to mine:

```bash
$ cat cmdline 
BOOT_IMAGE=/boot/vmlinuz-3.19.0-26-generic.efi.signed root=UUID=91e78c91-db22-429e-a481-3b8213577eb0 ro quiet splash vt.handoff=7
$ ls -la /boot 
-rw-r--r-- 1 root root  1270654 Aug 12 08:26 abi-3.19.0-26-generic
-rw-r--r-- 1 root root   177632 Aug 12 08:26 config-3.19.0-26-generic
drwxr-xr-x  3 root root     4096 Dec 31  1969 efi
drwxr-xr-x  5 root root     4096 Aug 30 18:25 grub
-rw-r--r-- 1 root root 19801552 Sep  2 14:47 initrd.img-3.19.0-26-generic
-rw-r--r-- 1 root root   176500 Mar 12  2014 memtest86+.bin
-rw-r--r-- 1 root root   178176 Mar 12  2014 memtest86+.elf
-rw-r--r-- 1 root root   178680 Mar 12  2014 memtest86+_multiboot.bin
-rw------- 1 root root  3626965 Aug 12 08:26 System.map-3.19.0-26-generic
-rw------- 1 root root  6572120 Aug 30 18:19 vmlinuz-3.19.0-26-generic.efi.signed
$ ls -la /boot/efi/EFI/ubuntu 
-rwxr-xr-x 1 root root     117 Aug 31 01:24 grub.cfg
-rwxr-xr-x 1 root root  956792 Aug 31 01:24 grubx64.efi
-rwxr-xr-x 1 root root 1178240 Aug 31 01:24 MokManager.efi
-rwxr-xr-x 1 root root 1355736 Aug 31 01:24 shimx64.efi
```

Grab Ubuntu Trusty's fork of the Linux kernel and use the MinnowBoard Max's example config from [eLinux's wiki](http://elinux.org/Minnowboard:MinnowMaxLinuxKernel):

```bash
$ apt-get install libncurses-dev
$ git  git clone git://kernel.ubuntu.com/ubuntu/ubuntu-trusty.git
Cloning into 'ubuntu-trusty'...
remote: Counting objects: 4715891, done.
remote: Compressing objects: 100% (725101/725101), done.
remote: Total 4715891 (delta 3957617), reused 4714607 (delta 3956398)
Receiving objects: 100% (4715891/4715891), 948.77 MiB | 2.31 MiB/s, done.
Resolving deltas: 100% (3957617/3957617), done.
Checking connectivity... done.
Checking out files: 100% (45692/45692), done.
$ git checkout Ubuntu-lts-4.2.0-10.12_14.04.1 -b 4.2.0
Switched to a new branch '4.2.0'
$ wget http://www.elinux.org/images/e/e2/Minnowmax-3.18.txt \
  -O ./ubuntu-trusty/.config
$ make menuconfig
```

We are going to plan ahead and add [TPM 2.0 support](https://lwn.net/Articles/624241/), thanks to excellent work by Jarkko Sakkinen, to our kernel and enable [signed module loading](https://wiki.gentoo.org/wiki/Signed_kernel_module_support). _As an aside:_ The TPM 2.0 support is very new and we will most likely need to update our kernel sources to a 4.4 before the next article in this series.

```none
  Device Drivers  --->
    Character devices  --->
      <*> TPM Hardware Support  --->
        <*>   TPM 2.0 CRB Interface
```

### Enabling signed kernel modules

I highly recommend following Gentoo's guide linked above. But we are essentially turning on `MODULE_SIG` and `MODULE_SIG_ALL`:

```none
--- Enable loadable module support
[*] Module signature verification
[*]   Require modules to be validly signed
[*]   Automatically sign all modules
      Which hash algorithm should modules be signed with? (Sign modules with SHA-512) --->
```

We will use the same CA from the previous article and create a new “kernel” keypair for module signing:

```bash
$ openssl req -new -utf8 -sha512 -key kernel.key -out kernel.csr 
$ openssl ca -in kernel.csr -config /etc/ssl/openssl.cnf
$ openssl x509 -in newcerts/03.pem -outform der -out kernel.der
$ ln -s ./kernel.der /path/to/ubuntu-trusty/signing_key.x509
$ ln -s ./kernel.key /path/to/ubuntu-trusty/signing_key.priv
```

Now build with `make`. _NB:_ I added a “`ps1`” to the version string to produce a “`4.2.0-ps1+`”:

```bash
$ make -j 4
[...]
  CC      kernel/crash_dump.o
  CC      mm/mlock.o
  CERTS   kernel/x509_certificate_list
  - Including cert signing_key.x509
  AS      kernel/system_cert
[...]
  HOSTCC  arch/x86/boot/tools/build
  CPUSTR  arch/x86/boot/cpustr.h
  CC      arch/x86/boot/cpu.o
  MKPIGGY arch/x86/boot/compressed/piggy.S
  AS      arch/x86/boot/compressed/piggy.o
  LD      arch/x86/boot/compressed/vmlinux
  ZOFFSET arch/x86/boot/zoffset.h
  OBJCOPY arch/x86/boot/vmlinux.bin
  AS      arch/x86/boot/header.o
  LD      arch/x86/boot/setup.elf
  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
Setup is 15596 bytes (padded to 15872 bytes).
System is 6059 kB
CRC 16036608
Kernel: arch/x86/boot/bzImage is ready  (#4)
$ make -j 4 modules
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/bounds.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
  Building modules, stage 2.
  MODPOST 19 modules
$ sudo make modules_install
  INSTALL crypto/echainiv.ko
Enter pass phrase for ./signing_key.priv:
[...]
  DEPMOD  4.2.0ps1+
$ sudo make install
sh ./arch/x86/boot/install.sh 4.2.0ps1+ arch/x86/boot/bzImage System.map "/boot"
run-parts: executing /etc/kernel/postinst.d/apt-auto-removal 4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
run-parts: executing /etc/kernel/postinst.d/initramfs-tools 4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
update-initramfs: Generating /boot/initrd.img-4.2.0ps1+
run-parts: executing /etc/kernel/postinst.d/pm-utils 4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
run-parts: executing /etc/kernel/postinst.d/update-notifier 4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
run-parts: executing /etc/kernel/postinst.d/zz-update-grub 4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
Generating grub configuration file ...
Warning: Setting GRUB_TIMEOUT to a non-zero value when GRUB_HIDDEN_TIMEOUT is set is no longer supported.
Found linux image: /boot/vmlinuz-4.2.0ps1+
Found initrd image: /boot/initrd.img-4.2.0ps1+
Found linux image: /boot/vmlinuz-3.19.0-26-generic
Found initrd image: /boot/initrd.img-3.19.0-26-generic
Adding boot menu entry for EFI firmware configuration
done
```

Perfect! If we boot into this new kernel (addressed in the next section) we can try to load modules that are unsigned or not signed with the x509 certificate built into the kernel. The kernel module signatures are added to a specific section in the ELF binary. We can test loading insecure modules by stripping these sections.

```bash
$ cd /lib/modules/4.2.0ps1+/kernel/drivers/virtio/
$ modinfo virtio_pci         
filename:       /lib/modules/4.2.0ps1+/kernel/drivers/virtio/virtio_pci.ko
version:        1
license:        GPL
description:    virtio-pci
author:         Anthony Liguori 
srcversion:     5DF745C216A58E858C15B2C
alias:          pci:v00001AF4d*sv*sd*bc*sc*i*
depends:        virtio_ring,virtio
intree:         Y
vermagic:       4.2.0ps1+ SMP mod_unload modversions 
signer:         proSauce Kernel Signing
sig_key:        85:5B:A4:2F:5E:0D:CD:18:2C:BB:F1:01:15:E6:CB:62:F5:96:9B:BB
sig_hashalgo:   sha512
parm:           force_legacy:Force legacy mode for transitional virtio 1 devices (bool)
$ sudo modprobe virtio_pci
$ sudo modprobe -r virtio_pci
$ cp virtio_pci.ko ~
$ sudo strip virtio_pci.ko 
$ modinfo virtio_pci      
filename:       /lib/modules/4.2.0ps1+/kernel/drivers/virtio/virtio_pci.ko
version:        1
license:        GPL
description:    virtio-pci
author:         Anthony Liguori 
srcversion:     5DF745C216A58E858C15B2C
alias:          pci:v00001AF4d*sv*sd*bc*sc*i*
depends:        virtio_ring,virtio
intree:         Y
vermagic:       4.2.0ps1+ SMP mod_unload modversions 
parm:           force_legacy:Force legacy mode for transitional virtio 1 devices (bool)
$ sudo modprobe virtio_pci   
modprobe: ERROR: could not insert 'virtio_pci': Required key not available
$ sudo mv ~/virtio_pci.ko .
```

The `modinfo` binary will parse the ELF sections and report the signature hashing algorithm and the public key fingerprint used to sign. The kernel will fail a module load if this signature is missing or invalid. _An aside_: it would be worth while testing to what extent modules are blocked from loading.

### Attempt to boot and unsigned kernel: Works!

Now, if we haven't alreadly, reboot and interrupt the Grub bootloader by holding “`SHIFT`” to select the `4.2.0-ps1+` kernel. It will boot just fine, and it is **not** signed. What this means is, even though Secure Boot is enabled in your UEFI Setup:

```bash
$ sudo hexdump -C /sys/firmware/efi/efivars/SecureBoot-8be4df61-93ca-11d2-aa0d-00e098032b8c 
00000000  06 00 00 00 01                                    |.....|
00000005
```

...any arbitrary kernel can be booted by editing `/boot/efi/EFI/ubuntu/grub.cfg` or `/boot/grub/grub.cfg`, or by simply replacing the default kernel configured by grub’s configuration.

### Attempt to boot an untrusted kernel: Works!

The grubx64.efi bootloader, installed by the [grub-efi-amd64-signed](http://packages.ubuntu.com/trusty-updates/utils/grub-efi-amd64-signed) package, allows non-signed kernels to boot, how about kernels signed with non-trusted certificates:

```bash
$ sbsign --key /etc/ssl/private/ssl-cert-snakeoil.key --cert /etc/ssl/certs/ssl-cert-snakeoil.pem \
  --output /boot/vmlinuz-4.2.0ps1+ /boot/vmlinuz-4.2.0ps1+
$ sbverify --verbose /boot/vmlinuz-4.2.0ps1+
image signature issuers:
 - /CN=ubuntu
image signature certificates:
 - subject: /CN=ubuntu
   issuer:  /CN=ubuntu
```

Yeah, that boots too. And this is intentional, the system has Secure Boot enabled, is restricted to Microsoft’s Microsoft Windows UEFI Driver Publisher certificate because we previously installed it as the only certificate in the UEFI db variable, and will install/boot Ubuntu!

### Review bootloader code

If we want security from Secure Boot we need to minimize the trust chain to only allow bootloaders, kernels, and modules to execute if they are signed by certificates we explicitly trust. Another aside, the `grub-efi-amd64-signed` package is just a downloader/stager for pulling binaries from your apt mirror:

```python
#! /usr/bin/python3

import re
import shutil
from urllib.parse import urlparse, urlunparse
from urllib.request import urlopen

import apt

cache = apt.Cache()
grub_efi_amd64 = cache["grub-efi-amd64"].candidate
pool_parsed = urlparse(grub_efi_amd64.uri)
dists_dir = "/dists/%s/main/uefi/grub2-%s/current/" % (
    grub_efi_amd64.origins[0].archive, grub_efi_amd64.architecture)

for base in (
        "gcdx64.efi.signed",
        "grubx64.efi.signed",
        "grubnetx64.efi.signed",
        "version",
        ):
    dists_parsed = list(pool_parsed)
    dists_parsed[2] = re.sub(r"/pool/.*", dists_dir + base, dists_parsed[2])
    dists_uri = urlunparse(dists_parsed)
    print("Downloading %s ..." % dists_uri)
    with urlopen(dists_uri) as dists, open(base, "wb") as out:
        shutil.copyfileobj(dists, out)
```

Will pull down [http://us.archive.ubuntu.com/ubuntu/dists/trusty-updates/main/uefi/grub2-amd64/current/grubx64.efi.signed](http://us.archive.ubuntu.com/ubuntu/dists/trusty-updates/main/uefi/grub2-amd64/current/grubx64.efi.signed). This applies several patches against the grub2.02-beta2 sources to essentially include a linuxefi command. The `patches/linuxefi.patch` patch includes `grub_cmd_linux` with:

```c
[...]
  if (! grub_linuxefi_secure_validate (kernel, filelen))
    {
      grub_error (GRUB_ERR_INVALID_COMMAND, N_("%s has invalid signature"), argv[0]);
      grub_free (kernel);
      goto fail;
    }
[...]
```

And the following verify logic:

```c
grub_linuxefi_secure_validate (void *data, grub_uint32_t size)
{
  grub_efi_guid_t guid = SHIM_LOCK_GUID;
  grub_efi_shim_lock_t *shim_lock;

  shim_lock = grub_efi_locate_protocol(&guid, NULL);

  if (!shim_lock)
    return 1;

  if (shim_lock->verify(data, size) == GRUB_EFI_SUCCESS)
    return 1;

  return 0;
}
```

And finally the `patches/linuxefi_require_shim.patch` flips the missing shim check from a success to a failure. This is important because the grub bootloader is part of your trust chain. If you add an explicit trust for the signing certificate into your UEFI `db` variable you should NOT also enable verification bypasses.

So if your grub configuration is using the `linuxefi` command to load the kernel a signature will be required and verified! The problem is the `linux` command, patched with `patches/linuxefi_non_sb_fallback.patch` allows a fallback at all costs:

```diff
+#ifdef GRUB_MACHINE_EFI
+  using_linuxefi = 0;
+  if (grub_efi_secure_boot ())
+    {
+      /* Try linuxefi first, which will require a successful signature check
+	 and then hand over to the kernel without calling ExitBootServices.
+	 If that fails, however, fall back to calling ExitBootServices
+	 ourselves and then booting an unsigned kernel.  */
+      grub_dl_t mod;
+      grub_command_t linuxefi_cmd;
+
+      grub_dprintf ("linux", "Secure Boot enabled: trying linuxefi\n");
+
+      mod = grub_dl_load ("linuxefi");
+      if (mod)
+	{
+	  grub_dl_ref (mod);
+	  linuxefi_cmd = grub_command_find ("linuxefi");
+	  initrdefi_cmd = grub_command_find ("initrdefi");
+	  if (linuxefi_cmd && initrdefi_cmd)
+	    {
+	      (linuxefi_cmd->func) (linuxefi_cmd, argc, argv);
+	      if (grub_errno == GRUB_ERR_NONE)
+		{
+		  grub_dprintf ("linux", "Handing off to linuxefi\n");
+		  using_linuxefi = 1;
+		  return GRUB_ERR_NONE;
+		}
+	      grub_dprintf ("linux", "linuxefi failed (%d)\n", grub_errno);
+	      grub_errno = GRUB_ERR_NONE;
+	    }
+	}
+    }
+#endif
```

If Secure Boot, shim, and the Canonical-distributed and signed grub bootloader (`grub-efi-amd64-signed`) were providing security, this fallback would not be possible. The unsigned and signed-but-untrusted, kernel would not be bootable. The trust chain should start with the UEFI platform, assuming a protected way of writing firmware, the platform should use its `PK` and `KEK` to protect changes to `db` and `dbx`. The `db` and `dbx` should control what bootloader and or kernel can execute, and the kernel should control what modules can load. To inspect this chain:

```bash
$ sbverify --verbose /boot/efi/EFI/ubuntu/shimx64.efi 
 - /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/CN=Microsoft Corporation UEFI CA 2011
image signature certificates:
 - subject: /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/OU=MOPR/CN=Microsoft Windows UEFI Driver Publisher
   issuer:  /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/CN=Microsoft Corporation UEFI CA 2011
 - subject: /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/CN=Microsoft Corporation UEFI CA 2011
   issuer:  /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/CN=Microsoft Corporation Third Party Marketplace Root
$ sbverify --verbose /boot/efi/EFI/ubuntu/grubx64.efi.orig 
image signature issuers:
 - /C=GB/ST=Isle of Man/L=Douglas/O=Canonical Ltd./CN=Canonical Ltd. Master Certificate Authority
image signature certificates:
 - subject: /C=GB/ST=Isle of Man/O=Canonical Ltd./OU=Secure Boot/CN=Canonical Ltd. Secure Boot Signing
   issuer:  /C=GB/ST=Isle of Man/L=Douglas/O=Canonical Ltd./CN=Canonical Ltd. Master Certificate Authority
$ sbverify --verbose /boot/vmlinuz-3.19.0-26-generic.efi.signed 
image signature issuers:
 - /C=GB/ST=Isle of Man/L=Douglas/O=Canonical Ltd./CN=Canonical Ltd. Master Certificate Authority
image signature certificates:
 - subject: /C=GB/ST=Isle of Man/O=Canonical Ltd./OU=Secure Boot/CN=Canonical Ltd. Secure Boot Signing
   issuer:  /C=GB/ST=Isle of Man/L=Douglas/O=Canonical Ltd./CN=Canonical Ltd. Master Certificate Authority
# And of course our new kernel
$ sbverify --verbose /boot/vmlinuz-4.2.0ps1+
No signature table present
```

### Review shim implementation and trust

The trouble here is not the shim, but the Canonical-signed grubx64 binary that allows untrusted/unsigned kernel booting. Another aside, there may be arbitrary signed shims that extend trust beyond Microsoft. There is a walkthrough on the Archlinux wiki on how to apply for Microsoft shim signing: [https://en.altlinux.org/UEFI\_SecureBoot\_mini-HOWTO#Getting\_your\_shim\_signed](https://en.altlinux.org/UEFI_SecureBoot_mini-HOWTO#Getting_your_shim_signed).

The shim code that verifies grub is sound, and this is also the logic that backs the grub-called `SHIM_VERIFY` protocol:

```c
/*
 * Protocol entry point. If secure boot is enabled, verify that the provided
 * buffer is signed with a trusted key.
 */
EFI_STATUS shim_verify (void *buffer, UINT32 size)
{
	EFI_STATUS status = EFI_SUCCESS;
	PE_COFF_LOADER_IMAGE_CONTEXT context;

	loader_is_participating = 1;
	in_protocol = 1;

	if (!secure_mode())
		goto done;

	status = read_header(buffer, size, &context);
	if (status != EFI_SUCCESS)
		goto done;

	status = verify_buffer(buffer, size, &context);
done:
	in_protocol = 0;
	return status;
}
```

But both this Microsoft-signed shim and the Microsoft certificate cannot be used or trusted to enable security.

A shim is built using a vendor certificate: [https://github.com/rhinstaller/shim](https://github.com/rhinstaller/shim). In the case of canonical’s [shim-signed](https://launchpad.net/ubuntu/+source/shim-signed) package, the certificate is [Canonical Ltd. Secure Boot Signing](http://bazaar.launchpad.net/~ubuntu-bugcontrol/qa-regression-testing/master/download/head:/canonicalmasterpubli-20121127224415-zwfgigzh3kstgk0g-3/canonical-master-public.der):  
`SKID: AD:91:99:0B:C2:2A:B1:F5:17:04:8C:23:B6:65:5A:26:8E:34:5A:63 C=GB, ST=Isle of Man, L=Douglas, O=Canonical Ltd., CN=Canonical Ltd. Master Certificate Authority`

Since this certificate has signed a grub binary that allows fallback, we cannot trust it. Since the shim contains this vendor certificate and is signed by the Microsoft UEFI certificate we cannot trust that either. Also, we cannot audit what other shim-like bootloaders that certificate has signed, best to not include it in our trust chain.

## Control the bootloaders!

You can certainly build the shim from source and not include any `VENDOR_CERT_PUB`, to only include the `SHIM_VERIFY` protocol. Then build a grub with the `linuxefi` command and `SHIM_VERIFY` support:

```bash
$ git clone https://github.com/vathpela/grub2-fedora
$ cd grub2-fedora
$ git checkout sb
$ ./autogen.sh
$ ./configure --target=x86_64 --with-platform=efi
```

Then patch the `grub_linuxefi_secure_validate` method to require the shim protocol, and not return success if booting grub directly: [https://github.com/theopolis/grub2-fedora/commit/5fec86351d793cb2eef4dc5abddb22d193348be3](https://github.com/theopolis/grub2-fedora/commit/5fec86351d793cb2eef4dc5abddb22d193348be3)

```diff
- if (!shim_lock)
- return 1;
+  if (!shim_lock || shim_lock->verify(data, size) != GRUB_EFI_SUCCESS) {
+    /* The SHIM_LOCK protocol is missing or verification failed. */
+    return 0;
+  }
 
- if (shim_lock->verify(data, size) == GRUB_EFI_SUCCESS)
- return 1;
-
- return 0;
+  return 1;
 }
```

```bash
$ make -j 4
$ cd grub-core
$ ../grub-mkimage -d . -o bootx64-2.efi -O x86_64-efi -p /efi/ubuntu/ boot part_gpt part_msdos fat \
  ext2 normal configfile lspci ls reboot datetime loadenv search lvm help signature_test linuxefi
```

You will have a `shimx64.efi` and `bootx64.efi` in need of signing. In the previous article we create an authenticated UEFI variable with the Microsoft Certificate, we will now write the same variable with a custom certificate and **not include the Microsoft Certificate**.

```bash
$ openssl genrsa -des3 -out db-custom.key 2048
$ openssl req -new -utf8 -sha512 -days 3650 -key db-custom.key -out db-custom.csr
$ openssl ca -in db-custom.csr -config /etc/ssl/openssl.cnf 
$ cp newcerts/04.pem db-custom.crt
$ cert-to-efi-sig-list db-custom.crt db-custom.esl
$ sign-efi-sig-list -k kek.key -c kek.crt DB db-custom.esl db-custom.auth
```

You will need to update your UEFI variable store from within the UEFI shell. Refer to the previous article for help. Then use this new certificate to sign the shim, our patched grub2, and our custom 4.2.0 kernel. We will overwrite the `grubx64.efi` file, so please back it up-- the shim bootloader expects an explicit name for this EFI application.

```bash
$ sbsign --key db-custom.key --cert db-custom.crt \
  --output /boot/efi/EFI/ubuntu/shimx64-enforce.efi /path/to/compiled/shimx64.efi 
$ sbsign --key db-custom.key --cert db-custom.crt \
  --output /boot/efi/EFI/ubuntu/grubx64.efi /path/to/compiled/grub-core/bootx64.efi
$ sbsign --key db-custom.key --cert db-custom.crt  \
  --output /boot/vmlinuz-4.2.0ps1+.signed /boot/vmlinuz-4.2.0ps1+
```

Edit your `/boot/grub/grub.cfg` to use the “.signed” kernel file (or alternatively, overwrite the existing kernel):

```none
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-91e78c91-db22-429e-a481-3b8213577eb0' {
	linuxefi /boot/vmlinuz-4.2.0ps1+.signed  root=UUID=91e78c91-db22-429e-a481-3b8213577eb0 ro  quiet splash $vt_handoff
	initrdefi	/boot/initrd.img-4.2.0ps1+
}
```

And finally, if you’re a quick learner or have been paying a bit of attention, there are **STILL components left unsigned** or unverified:

- The `/boot/initrd.img-4.2.0ps1+` gzipped CPIO archive
- The set of `/boot/efi/EFI/ubuntu/grub.cfg` and `/boot/grub/grub.cfg` files
- The command line provided to the signed kernel specifying various options and most importantly the root partition
- Grub modules that could be loaded using the unsigned configurations

## Recommended approach to booting Linux securely

We could continue to patch features out of grub or [attempt](http://www.phoronix.com/scan.php?page=news_item&px=MTI5MzY) to add signing to the initrd and other configuration components. Instead let's optimize for simplicity and remove the shim and grub code from our trust chain. This reduces the flexibility of the platform, but the Minnow is a rather restrictive platform so we can cheat and optimize against flexibility.

We will follow Greg Kroah-Hardman’s [Booting a Self-signed Linux Kernel](http://www.kroah.com/log/blog/2013/09/02/booting-a-self-signed-linux-kernel/) guide but we will use the same `db-custom.crt` from before. This certificate is signed by our controlled CA, exists in our `db` variable and cannot be updated without control of our `KEK`/`PK` keying material.

This is possible because the Linux kernel can be configured to include an [EFI application stub](https://lwn.net/Articles/632528/). This can be helpful to reduce booting complexity regardless of UEFI, Secure Boot, or security and hardening concerns. Gentoo also has a nice guide for booting a Linux kernel directly: [https://wiki.gentoo.org/wiki/EFI\_stub\_kernel](https://wiki.gentoo.org/wiki/EFI_stub_kernel).

Essentially we will modify the kernel configuration ever so slightly and add the following:

```none
CONFIG_EFI=y
CONFIG_EFI_STUB=y
CONFIG_FB_EFI=y
CONFIG_CMDLINE_BOOL=y
CONFIG_CMDLINE="root=/dev/mmcblk0p2" #I’m replacing the nasty parition UUID
...
CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_SOURCE="initramfs.cpio"
```

Back to the ubuntu-trusted kernel source repo:

```bash
$ cp /boot/initrd.img-4.2.0ps2+ /path/to/ubuntu-trusty/initramfs.cpio.gz
$ gunzip initramfs.cpio.gz
$ make -j 5
$ ./arch/x86/boot/bzImage to /boot/vmlinuz-4.2.0ps1+
$ sbsign --key db-custom.key --cert db-custom.crt \
  --output /boot/efi/EFI/ubuntu/vmlinuz-4.2.0ps1+.signed /boot/vmlinuz-4.2.0ps1+
```

The last step is to edit your UEFI bootmanager to assign the new signed Linux binary as the default bootloader.

Congrats! You are now booting directly into your Linux kernel and no one boot into a replaced or altered kernel. If they rewrite your boot manager variables at run time the next boot will also fail! Every time you change or update your custom kernel you will need to resign with the `db-custom.crt` certificate.

The next article will start to focus on the TPM, dive into firmware modifications, and explore the great features [Vincent Zimmer](http://vzimmer.blogspot.com/) blogs about! Right now the kernel has TPM 2.0 support and will detect a firmware-based TPM from the Minnow and Intel's PTT implementation. However the 64-bit firmware that is defaultly installed (or avilable for download from Intel) does not enable the `fTPM`, even though you can toggle it in the UEFI Setup.

When this article was originally published, a commentor with the handle Hellishnoob posted:

> I share your idea about removing shim/grub from our trust chain completely. As for verifying initramfs and kernel command line arguments, I have found another approach that doesn't require rebuilding the kernel.
> I have stumbled upon this (https://harald.hoyer.xyz/2015/02/25/single-uefi-executable-for-kernelinitrdcmdline/) page and then found this script: https://github.com/systemd/systemd/blob/1f5dc27b662bdf098fa36a84d862295ed819c339/test/test-efi-create-disk.sh
> You can put your kernel, initramfs and cmdline arguments into a single UEFI-executable based on linuxx64.efi.stub from systemd package using objcopy from binutils and then sign it and boot from it. The advantage is that you can modify initramfs and cmdline without rebuilding the kernel. This script works for me:

```sh
echo -n "quiet splash" > cmdline
objcopy \
--add-section .osrel=/etc/os-release --change-section-vma .osrel=0x20000 \
--add-section .cmdline=cmdline --change-section-vma .cmdline=0x30000 \
--add-section .linux=/boot/vmlinuz-$(uname -r) --change-section-vma .linux=0x2000000 \
--add-section .initrd=/boot/initrd.img-$(uname -r) --change-section-vma .initrd=0x3000000 \
/usr/lib/systemd/boot/efi/linuxx64.efi.stub /boot/efi/EFI/BOOT/bootx64.efi
```

> You can also add signing to it and put it into `/etc/initramfs/post-update.d/` so it updates the UEFI-executable every time initramfs is regenerated.