---
title: "Harden your Linux UEFI Secure Boot using GRUB signature checking and a Yubikey"
created: 2020-05-24
categories: 
tags: 
  - uefi
---

The goal of this article is to walk through hardening your UEFI-supported Linux desktop’s boot. This is accomplished by replacing signature checking keys with your own, keeping a portion of that key chain in an HSM/PIV device, and enabling GRUB signature checking.

<!--more-->

**What problem are we solving?**

Most popular Linux distributions support UEFI Secure Boot to facilitate hardware enablement. This means they support Secure Boot for the to the extent needed to get you up and running without getting in your way, not to provide any in-depth security features. For example, distributions such as Ubuntu and Fedora intentionally do not verify signature checking of your initrd nor GRUB modules, fonts, themes, or graphics.

We want to harden our boot such that anything in the boot chain executed before Linux requires signature verification. Caveat, that we are going to implement verification to the extent possible, we are not going to guarantee everything executed is verified. For example, we are most likely not verifying any EC firmware, voltage regulator firmware, etc. This article will call out specifically what we are verifying.

****Important!**** If you want a turn-key, and more complete solution to implement now, please use [safeboot.dev](https://safeboot.dev/).

**What is the general threat model?**

The OS bring-up of our desktop should be deterministic and authenticated. Assume this desktop sits in a reasonably trusted location, for example an apartment. If someone enters our trusted location with the intent of compromising our desktop’s pre-OS boot they must disassemble the enclosure and tamper our R/W SPI flash contents or a similar component. We will protect against easier and quicker attacks such as modifying bootable disk content, attaching malicious devices, and interrupting and altering the boot logic.

The above scenario assumes more than this article covers, for practical purposes the desktop owner should lock the machine when not in use and implement full disk encryption. Additionally, if someone enters your apartment with malicious intent multiple times they may be able to install a simple keylogger followed by using the result to change UEFI Setup data.

In a nutshell what are we going to accomplish, we will:

- Generate private keys on a Yubikey 4 device (treat it as a personal HSM).
- Use the HSM keys to replace our desktop’s UEFI Secure boot platform key, key enrollment key, and allow-list db key.
- Create a standalone GRUB that enforces signature verification for any content used including a configuration that we will change often.
- Require signature verification for loading any initrd and kernel pairs.
- Password protect our UEFI Setup settings and grub boot-time configuration modification.

Heads up that this article is a recap of my experiences and not intended to be a "dummy’s guide". Hence I strongly recommend, if you are reading with the intent to DIY, that each step be accompanied with independent research and questioning.

The final boot flow will be: CPU bootstrap, UEFI platform code, Standalone Grub, Dynamic GRUB config, Linux initrd and kernel.

The source materials I used when researching and debugging are as follows:

- A simple end-to-end walkthrough with more discussion of the process, using a different HSM but without deep-diving into GRUB [https://diagprov.ch/posts/2017/01/own-and-lock-down-your-laptop-uefi-secure-boot.html](https://diagprov.ch/posts/2017/01/own-and-lock-down-your-laptop-uefi-secure-boot.html)
- GRUB's signature verification documentation: [https://www.gnu.org/software/grub/manual/grub/html\_node/Using-digital-signatures.html](https://www.gnu.org/software/grub/manual/grub/html_node/Using-digital-signatures.html)
- A walkthrough of setting up UEFI Security Boot variables and using a Standalone GRUB: [https://ruderich.org/simon/notes/secure-boot-with-grub-and-signed-linux-and-initrd](https://ruderich.org/simon/notes/secure-boot-with-grub-and-signed-linux-and-initrd)

## Make some backups and a restore USB

****Important!**** Let’s backup everything in `/boot` and everything in `/boot/EFI` as most likely your first GPT partitions is mounted to `/boot/EFI`.

Next follow [Ubuntu’s tutorial for creating](https://ubuntu.com/tutorials/tutorial-create-a-usb-stick-on-windows#1-overview) a Live USB, or have an existing Live USB ready to fix any errors. We may lock ourselves out of the OS. Though in most cases we can disable enforcement using a UEFI Setup password for Grub password.

## Remove existing packages

We are going to "take control" of the bootloading process and thus we need to prevent package manager updates from getting in the way. Otherwise a new version will overwrite our custom signed copies.

For Ubuntu this includes removing GRUB-EFI `grub-efi-*`, the signed versions `dpkg --list | grep grub | grep signed`, and any versions of `shim`. Any packages returned in these queries should be uninstalled. The signed version of `linux-image` can be removed as well `dpkg --list | grep linux-image | grep signed`.

## Generate UEFI keys and authenticated variables

We will generate three signing keys, a Platform Key, a Key Enrollment Key, and an Allow-List DB Key. Refer to [Antony Vennard’s descriptions](https://diagprov.ch/posts/2017/01/own-and-lock-down-your-laptop-uefi-secure-boot.html) for more information on these keys. We will keep the PK and KEK private keys in an HSM because they are rarely used. I choose to use a Yubikey 4 Nano and here is what was required to make this work.

Install the [Yubico PIV Tool](https://developers.yubico.com/yubico-piv-tool/) so we can generate keys on the Yubikey. I found that the usual pkcs11 tools and GPG are generally not great at interfacing with the Yubikey; specifically they cannot generate or use keys in the yubico-deprecated slots, go figure. ;)

```sh
$ yubico-piv-tool -s88 -agenerate -o yubikey-pk.pub
$ yubico-piv-tool -s88 \
  -S '/CN=My Platform Key/' \
  -averify -aselfsign \
  -i yubikey-pk.pub \
  -o yubikey-pk.pem
$ openssl x509 -outform DER -in yubikey-pk.pem -out yubikey-pk.der
$ yubico-piv-tool -a import-certificate -s88 -K DER -i yubikey-pk.der
```

Now the Yubikey has our PK private key and a self-signed certificate. Next we create an authenticated UEFI variable using this key.

```sh
$ uuidgen > uuid
$ cert-to-efi-sig-list -g `cat uuid` yubikey-pk.pem yubikey-pk.esl
```

****Important!**** We need a newer version of [efitools](https://git.kernel.org/pub/scm/linux/kernel/git/jejb/efitools.git), which has support for PKCS11 signing. I used version 1.9.2: [efitools-1.9.2.tar.gz](https://git.kernel.org/pub/scm/linux/kernel/git/jejb/efitools.git/snapshot/efitools-1.9.2.tar.gz). Past versions of efitools required you to generate a to-be-signed ESL that could be used with `openssl smime`. I found that UEFI code is picky about x509 options so for best results stay in efitools.

And I ran into an off-by-one reference counted object, which was fixed with a newer OpenSC installation. This is most likely referenced in the [GitHub #327 issue](https://github.com/OpenSC/libp11/issues/327) here. For posterity I used a commit hash: 1d93ed040930d60a8206bd839be0d61b269ac5d9. I built and installed this system-wide.

You may be affected by this bug if you see similar segfaults from efitools when trying to use the PKCS11 features.

```sh
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff6eeff2f in ?? () from /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so
(gdb) bt
#0  0x00007ffff6eeff2f in ?? () from /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so
#1  0x00007ffff6ef095e in ?? () from /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so
#2  0x00007ffff7ab9aa9 in RSA_sign () from /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
#3  0x00007ffff7ab8892 in ?? () from /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
#4  0x00007ffff7a7c144 in EVP_SignFinal () from /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
#5  0x00007ffff7aa1591 in PKCS7_dataFinal () from /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
#6  0x00007ffff7aa317c in PKCS7_final () from /usr/lib/x86_64-linux-gnu/libcrypto.so.1.1
#7  0x000055555555606c in sign_efi_var_ssl (payload=payload@entry=0x555555759670 "P", payload_size=payload_size@entry=84, pkey=pkey@entry=0x55555578a200, cert=cert@entry=0x555555776ea0, sig=sig@entry=0x7fffffffde10, sigsize=sigsize@entry=0x7fffffffde0c) at openssl_sign.c:24
#8  0x0000555555556346 in sign_efi_var (payload=0x555555759670 "P", payload_size=84, keyfile=0x7fffffffe600 "pkcs11:object=Private key for Retired Key 7;type=private", certfile=, sig=0x7fffffffde10, sigsize=0x7fffffffde0c, engine=0x7fffffffe5f6 "pkcs11") at openssl_sign.c:69
#9  0x0000555555555bc3 in main (argc=, argv=) at sign-efi-sig-list.c:251
```

Use the Yubikey’s key in slot 88 to self-sign an authenticated UEFI variable. The [PKCS11 key alias is documented](https://developers.yubico.com/yubico-piv-tool/YKCS11/Functions_and_values.html#_key_alias_per_slot_and_object_type) on Yubico’s website.

```sh
$ efitools-1.9.2/sign-efi-sig-list \
  -t "2020-01-01 00:00:00" \
  -e pkcs11 \
  -k "pkcs11:object=Private key for Retired Key 7;type=private" \
  -g `cat uuid` \
  -c yubikey-pk.pem \
  PK yubikey-pk.esl yubikey-pk.auth
```

Repeat the same process for the KEK, but use a different key slot and sign with the PK instead of self-signing.

```sh
$ yubico-piv-tool -s87 -agenerate -o yubikey-kek.pub
$ yubico-piv-tool -s87 \
  -S '/CN=My KEK/' \
  -averify -aselfsign \
  -i yubikey-kek.pub \
  -o yubikey-kek.pem
$ openssl x509 -outform DER -in yubikey-kek.pem -out yubikey-kek.der
$ yubico-piv-tool -a import-certificate -s 87 -K DER -i yubikey-kek.der
$ cert-to-efi-sig-list -g `cat uuid`yubikey-kek.pem yubikey-kek.esl
$ efitools-1.9.2/sign-efi-sig-list \
  -t "2020-01-01 00:00:00" \
  -e pkcs11 \
  -k "pkcs11:object=Private key for Retired Key 7;type=private" \
  -g `cat uuid` \
  -c yubikey-pk.pem \
  KEK yubikey-kek.esl yubikey-kek.auth
```

Finally, the DB key can be kept on-disk or in a third slot on the Yubikey. I choose to keep it online so I can keep the Nano/HSM offline and disconnected.

```sh
$ openssl req -new -x509 -newkey rsa:2048 -subj "/CN=My Signing Key/" -keyout signing.key -out signing.pem -days 9125 -nodes -sha256
$ cert-to-efi-sig-list -g `cat uuid` signing.pem signing.esl
```

Sign using the KEK on the Yubikey.

```sh
$ efitools-1.9.2/sign-efi-sig-list \
  -t "2020-01-01 00:00:00" \
  -e pkcs11 \
  -k "pkcs11:object=Private key for Retired Key 6;type=private" \
  -g `cat uuid` \
  -c yubikey-kek.pem \
  DB signing.esl signing.auth
```

In the next section we will create a GRUB binary and sign that with this DB key.

Create a standalone GRUB that enforces signature verification

I wanted to use the same DB key used to sign GRUB to sign any content that GRUB loaded. Since I wanted to change GRUB and my kernel/initrd often I would have to keep a secondary key "online" so there is no added security for multiple keys in my scenario.

GRUB uses GPG signatures to verify content so we need to import the DB private key into our root user's GPG keychain.

```sh
# apt install monkeysphere
# cat signing.key | pem2openpgp "My Signing Key " > signing.gpgkey
# gpg --import --allow-secret-key-import signing.gpgkey
# gpg --export > signing.pubgpg
# gpg --list-keys
[...]

pub   rsa2048 1970-01-01 [SCEA]
      2C5B596292929FB8DD9847F7C93B99D30E812A37
uid           [ unknown] My Signing Key 
```

The goal is to make a standalone GRUB that contains:

- All of the modules we want to use, ideally a limited set.
- A public GPG key used to verify signatures of any runtime content read.
- A basic configuration that enables signature verification and loads a larger config.

Create a `grub-initial.cfg` that will be "built in" to the standalone GRUB.

```sh
# Enforce that all loaded files must have a valid signature.
set check_signatures=enforce
export check_signatures

# Require a password to make boot-time changes
set superusers="root"
password_pbkdf2 root grub.pbkdf2.sha512.10000.HASH
export superusers

set root='hd3,gpt2' # Set this to the device/partition containing your root fs.
# See lsblk -o NAME,MOUNTPOINT,UUID for your UUID
search --no-floppy --fs-uuid --set=root $UUID # Replace '$UUID' with your root fs UUID.

configfile /boot/grub/grub.cfg

# If the config contains an error pause then reboot
sleep 10
reboot
```

You can read your current GRUB configuration, most likely at `/etc/grub/grub.cfg` and predict the minimum set of modules. On my host this was the following:

```sh
$ export MODULES="configfile echo normal ls \
  linux linuxefi \
  verify gcry_sha512 gcry_rsa \
  search search_fs_uuid part_gpt ext2 fat\
  all_video efi_gop efi_uga video_bochs video_cirrus gfxterm gettext \
  gzio xzio lzopio"
```

Then build grub using:

```sh
# gpg --default-key "2C5B596292929FB8DD9847F7C93B99D30E812A37" --detach-sign ./grub-initial.cfg
# grub-mkstandalone \
  --directory /usr/lib/grub/x86_64-efi \
  --modules "$MODULES" \
  --format x86_64-efi 
  --pubkey ./signing.pubgpg \
  -o /boot/efi/EFI/grubx64-standalone.efi \
 "boot/grub/grub.cfg=./grub-initial.cfg" \
 "boot/grub/grub.cfg.sig=./grub-initial.cfg.sig"
```

And sign using our DB signing key:

```sh
# sbsign \
  --key ./signing.key \
  --cert ./signing.pem \
  --output /boot/efi/EFI/grubx64-standalone.efi \
  /boot/efi/EFI/grubx64-standalone.efi
```

And now I can use my OS's GRUB configuration tooling and `update-grub` to build a more dynamic configuration saved to `/etc/grub/grub.cfg` and it will only boot if signed. That larger configuration has `insmod` calls that will fail if the module is not signed.

Finally, I used a small script to resign my GRUB config, my initrd, and kernel:

```sh
#!/bin/bash

set -e
set -x

KEY=2C5B596292929FB8DD9847F7C93B99D30E812A37

LINUX=/boot/vmlinuz-5.4.10
INITRD=/boot/initrd.img-5.4.10
GRUB=/boot/grub/grub.cfg

INITRD_HASH=$(shasum -a 256 $INITRD | awk '{print $1}')
INITRD_HASH_FROZEN=$(shasum -a 256 $INITRD.frozen | awk '{print $1}')

LINUX_HASH=$(shasum -a 256 $LINUX | awk '{print $1}')
LINUX_HASH_FROZEN=$(shasum -a 256 $LINUX.frozen | awk '{print $1}')

GRUB_HASH=$(shasum -a 256 $GRUB | awk '{print $1}')
GRUB_HASH_FROZEN=$(shasum -a 256 $GRUB.frozen | awk '{print $1}')

if [[ ! "$INITRD_HASH" = "$INITRD_HASH_FROZEN" ]]; then
  echo "Re-Signing $INITRD"
  gpg --default-key "$KEY" --detach-sign $INITRD
  cp $INITRD $INITRD.frozen
fi

if [[ ! "$LINUX_HASH" = "$LINUX_HASH_FROZEN" ]]; then
  echo "Re-Signing $LINUX"
  gpg --default-key "$KEY" --detach-sign $LINUX
  cp $LINUX $LINUX.frozen
fi

if [[ ! "$GRUB_HASH" = "$GRUB_HASH_FROZEN" ]]; then
  echo "Re-Signing $GRUB"
  gpg --default-key "$KEY" --detach-sign $GRUB
  cp $GRUB $GRUB.frozen
fi
```

This required me to add this “snip” to `/etc/grub.d/10_linux`, so that my `(.sig|.frozen)` files are not interpreted as optional kernel/initrd pairs.

```sh
grub_file_is_not_sig() {
  name="$1"
  case "$name" in
      *.sig) return 1 ;;
      *.frozen) return 1 ;;
  esac
}

machine=`uname -m`
case "x$machine" in
    xi?86 | xx86_64)
        list=
        for i in /boot/vmlinuz-* /vmlinuz-* /boot/kernel-* ; do
            if grub_file_is_not_garbage "$i" && grub_file_is_not_sig "$i" ; then list="$list $i" ; fi
        done ;;
```

Now boot and resolve any errors or typos. When you are finished and happy with the flow be sure to set up a UEFI Setup password.
