---
title: "Exploring Universal Flash Storage (UFS) Write Protection on the HiKey960"
created: 2018-10-06
tags: 
  - hardware
  - hikey
  - secure-boot
  - firmware
---

In my [previous post](https://casualhacking.io/blog/2018/7/8/diy-root-of-trust-using-arm-trusted-firmware-on-the-96boards-hikey) I gave an overview of basic "do it yourself" root-of-trust creation through MMC boot region write-protection. I used this on sample HiKey (original) devices to authenticate ARM-Trusted-Firmware code beyond BL2, authenticating the OPTEE OS and U-Boot as BL33.

This post explores the same concept on a HiKey960. The 960 version has an onboard [Universal Flash Storage](https://en.wikipedia.org/wiki/Universal_Flash_Storage) (UFS) devices, the future of MMC, and much faster! UFS as a standard has existed since 2011, but seems only since 2017 they were generally obtainable. Support in Linux has experienced very-recent updates (4.15+).

The unfortunate news is I have not found a way to implement an open-source/do-it-yourself Root of Trust on the HiKey960. I will explain why during this post, but this post will focus more on UFS write-protection.

<!--more-->

## Background information on the HiKey960

As usual, the focus is on booting.

The original HiKey's CPU booted directly to open source `l-loader` code, placing the CPU into AARCH64 mode and then executing open source ARM-Trusted-Firmware BL1 or BL2. The HiKey960 uses a closed-source `xloader` binary stored in the UFS's Logical Unit Number (LUN) 0 disk. This loader searches for ARM-Trusted-Firmware's BL2 code in a GPT partition called `fastboot` on the UFS's LUN3 disk (\*). _This statement is an assumption based on extremely naive inferences about the `xloader` behavior. Instead of a hardcoded partition it could be referencing an offset into LUN3._

The boot flows is as follows:

- LUN0:0x0 `xloader`, closed source
- LUN3:0x200000 (`fastboot` partition) ARM-Trusted-Firmware's `BL2`, open source
- LUN3:0x1400000 (`fip` partition) ARM-Trusted-Firmware's FIP container, various
- `mcuimage` or `lpm3` included as `SCP_BL2` in the FIP container, closed source
- Optional OPTEE included as `BL32` in the FIP container, open source
- UEFI included as `BL33` in the FIP container, open source
- Your OS, open source

If you read the HiKey960 [platform defines](https://github.com/ARM-software/arm-trusted-firmware/blob/master/plat/hisilicon/hikey960/hikey960_def.h) in ARM-Trusted-Firmware you can find the offsets to `fip`, and the references to LUN3. This means we can place the FIP in another LUN or at another location in LUN3.

## UFS Write-Protection

All of my (limited) knowledge on UFS write-protection comes from [`JESD220B`](assets/JESD220B.pdf). This specification is freely available on JEDEC's website with a free account. Newer versions of the specification are not as free.

Write-protection can be applied at the Logical Unit (LU) granularity. At manufacture time the LU's are configured. The UFS device maintains an internal structure for each Unit called the Device Descriptor. Within the descriptor includes a 8-bit flag called `bLUWriteProtect`. Section **14.1.6.5** defines this as offset `0x5` within the descriptor. A value of `0x0` means no protection, `0x1` means write-protection until the next power reset, and `0x2` means permanent write-protection. By default you can configure this flag at runtime, it is not write-once.

The writability of `bLUWriteProtect` on all Logical Units is determined by flags on the UFS device. Section **14.2** defines the device flags. Specifically flag `fPermanentWPEn` at offset `x02` and flag `fPowerOnWPEn` at offset `0x3`. These flags can be on or off, true or false, 0 or 1.

- `fPowerOnWPEn` can be enabled and will remain enabled until device reset
- `fPermanentWPEn` is write-once forever, so be very careful

If a Logical Unit's `bLUWriteProtect` is written the device flags are checked. If the device flags are enabled then the potential configuration values are accepted or rejected. For example if you set `bLUWriteProtect` to `0x1` and `fPowerOnWPEn` is false, the value will be rejected. To say this a different way, if you want LUN0 to be write-protected until the next power-on you must first set the UFS's device flag `fPowerOnWPEn==1` then write `bLUWriteProtect` on LUN0's device descriptor.

## Inspecting UFS Write-Protection statuses

Thanks to recent commits to Linux from Stanislav Nijnikov, this is rather straight forward.

```bash

commit d10b2a8ea8fd0d6c8a667dc1950c8c061bfbbcdd
Author: Stanislav Nijnikov 
Date:   Thu Feb 15 14:14:10 2018 +0200

    scsi: ufs: sysfs: flags
    
    This patch introduces a sysfs group entry for the UFS flags. The group adds
    "flags" folder under the UFS driver sysfs entry
    (/sys/bus/platform/drivers/ufshcd/*). The flags are shown as boolean value
    ("true" or "false"). The full information about the UFS flags could be
    found at UFS specifications 2.1.
    
    Signed-off-by: Stanislav Nijnikov <stanislav.nijnikov@wdc.com>
    Reviewed-by: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
    Signed-off-by: Martin K. Petersen <martin.petersen@oracle.com>

commit d829fc8a1058851f1058b4a29ea02da125c1684a
Author: Stanislav Nijnikov 
Date:   Thu Feb 15 14:14:09 2018 +0200

    scsi: ufs: sysfs: unit descriptor
    
    This patch introduces a sysfs group entry for the UFS unit descriptor
    parameters. The group adds "unit_descriptor" folder under the corresponding
    SCSI device sysfs entry (/sys/class/scsi_device/*/device/). The parameters
    are shown as hexadecimal numbers. The full information about the parameters
    could be found at UFS specifications 2.1.
    
    Signed-off-by: Stanislav Nijnikov <stanislav.nijnikov@wdc.com>
    Reviewed-by: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
    Signed-off-by: Martin K. Petersen <martin.petersen@oracle.com>
```

There are not available in the Hikey960's 4.15 kernel. Pull Linux master and apply the 4.15's defconfig and everything should work great. The only change I made was disabling loadable modules.

On my sample HiKey960:

```bash
$ uname -a
Linux linaro-developer 4.19.0-rc2-00285-gf8f6538 #20 SMP PREEMPT Sat Sep 8 19:37:44 EDT 2018 aarch64 GNU/Linux
```

To inspect the UFS flags:

```bash
$ cat /sys/devices/platform/soc/ff3b0000.ufs/flags/permanent_wpe 
false
linaro@linaro-developer:/sys/devices/platform/soc/ff3b0000.ufs$ cat /sys/devices/platform/soc/ff3b0000.ufs/flags/power_on_wpe  
false
$ ls -la /sys/devices/platform/soc/ff3b0000.ufs/flags/permanent_wpe 
-r--r--r-- 1 root root 4096 Sep  8 23:40 /sys/devices/platform/soc/ff3b0000.ufs/flags/permanent_wpe
```

Good thing this is `0x444`, but there's no `.store` on these device attributes so you are super safe against bricking your HiKey960.

To inspect the unit descriptors for each LUN:

```bash
$ cat /sys/devices/platform/soc/ff3b0000.ufs/host0/target0\:0\:0/0\:0\:0\:0/unit_descriptor/lun_write_protect 
0x00
linaro@linaro-developer:/sys/devices/platform/soc/ff3b0000.ufs$ cat /sys/devices/platform/soc/ff3b0000.ufs/host0/target0\:0\:0/0\:0\:0\:*/unit_descriptor/lun_write_protect 
0x00
0x00
0x00
0x00
cat: '/sys/devices/platform/soc/ff3b0000.ufs/host0/target0:0:0/0:0:0:49456/unit_descriptor/lun_write_protect': Invalid argument
0x00
cat: '/sys/devices/platform/soc/ff3b0000.ufs/host0/target0:0:0/0:0:0:49488/unit_descriptor/lun_write_protect': Invalid argument
```

The `Invalid arguments` come from special Units called Well-Known Units that do not support write-protection, we can ignore these right now.

So this says my HiKey960 is implementing no write-protection at all. Let's change that!

## Enable `fPermanentWPEn`

This is a hacky example of enabling this flag through a patch to the `ufshcd.c` code. It is not recommended to apply this as having this functionality means permanent configuration is possible at run-time.

```diff
diff --git a/drivers/scsi/ufs/ufs.h b/drivers/scsi/ufs/ufs.h
index 54deeb7..717d03e 100644
--- a/drivers/scsi/ufs/ufs.h
+++ b/drivers/scsi/ufs/ufs.h
@@ -513,6 +514,8 @@ struct ufs_vreg_info {
 
 struct ufs_dev_info {
    bool f_power_on_wp_en;
+   bool f_perm_wp_en;
    /* Keeps information if any of the LU is power on write protected */
    bool is_lu_power_on_wp;
 };
diff --git a/drivers/scsi/ufs/ufshcd.c b/drivers/scsi/ufs/ufshcd.c
index 794a460..c86229a 100644
--- a/drivers/scsi/ufs/ufshcd.c
+++ b/drivers/scsi/ufs/ufshcd.c
@@ -3944,6 +3944,38 @@ static int ufshcd_complete_dev_init(struct ufs_hba *hba)
    return err;
 }
 
+static int ufshcd_write_wp(struct ufs_hba *hba, enum flag_idn flag) {
+   int i;
+   int err;
+   bool flag_res = 0;
+
+   err = ufshcd_query_flag_retry(hba, UPIU_QUERY_OPCODE_SET_FLAG, flag,
+       NULL);
+   if (err) {
+       dev_err(hba->dev,
+           "%s setting WP flag failed with error %d\n",
+           __func__, err);
+       goto out;
+   }
+
+   /* poll for max. 1000 iterations for fDeviceInit flag to clear */
+   for (i = 0; i < 1000 && !err && flag_res; i++)
+       err = ufshcd_query_flag_retry(hba, UPIU_QUERY_OPCODE_READ_FLAG,
+           flag, &flag_res);
+
+   if (err)
+       dev_err(hba->dev,
+           "%s reading WP flag failed with error %d\n",
+           __func__, err);
+   else if (flag == QUERY_FLAG_IDN_PWR_ON_WPE)
+       hba->dev_info.f_power_on_wp_en = flag_res;
+   else if (flag == QUERY_FLAG_IDN_PERMANENT_WPE)
+       hba->dev_info.f_perm_wp_en = flag_res;
+
+out:
+   return err;
+}
+
 /**
  * ufshcd_make_hba_operational - Make UFS controller operational
  * @hba: per adapter instance
@@ -4313,14 +4345,13 @@ static int ufshcd_get_lu_wp(struct ufs_hba *hba,
  *
  */
 static inline void ufshcd_get_lu_power_on_wp_status(struct ufs_hba *hba,
- struct scsi_device *sdev)
+                           int lun)
 {
    if (hba->dev_info.f_power_on_wp_en &&
        !hba->dev_info.is_lu_power_on_wp) {
        u8 b_lu_write_protect;
 
- if (!ufshcd_get_lu_wp(hba, ufshcd_scsi_to_upiu_lun(sdev->lun),
- &b_lu_write_protect) &&
+       if (!ufshcd_get_lu_wp(hba, lun, &b_lu_write_protect) &&
            (b_lu_write_protect == UFS_LU_POWER_ON_WP))
            hba->dev_info.is_lu_power_on_wp = true;
    }
@@ -4350,7 +4381,8 @@ static int ufshcd_slave_alloc(struct scsi_device *sdev)
 
    ufshcd_set_queue_depth(sdev);
 
- ufshcd_get_lu_power_on_wp_status(hba, sdev);
+   ufshcd_get_lu_power_on_wp_status(hba,
+       ufshcd_scsi_to_upiu_lun(sdev->lun));
 
    return 0;
 }
@@ -6391,6 +6423,9 @@ static int ufshcd_probe_hba(struct ufs_hba *hba)
        if (!ufshcd_query_flag_retry(hba, UPIU_QUERY_OPCODE_READ_FLAG,
                QUERY_FLAG_IDN_PWR_ON_WPE, &flag))
            hba->dev_info.f_power_on_wp_en = flag;
+       if (!ufshcd_query_flag_retry(hba, UPIU_QUERY_OPCODE_READ_FLAG,
+               QUERY_FLAG_IDN_PERMANENT_WPE, &flag))
+           hba->dev_info.f_perm_wp_en = flag;
 
        if (!hba->is_init_prefetch)
            ufshcd_init_icc_levels(hba);
@@ -7693,16 +7728,101 @@ static void ufshcd_add_spm_lvl_sysfs_nodes(struct ufs_hba *hba)
        dev_err(hba->dev, "Failed to create sysfs for spm_lvl\n");
 }
 
+static ssize_t ufshcd_wp_show(struct device *dev,
+       struct device_attribute *attr, char *buf)
+{
+   struct ufs_hba *hba = dev_get_drvdata(dev);
+   int curr_len;
+   int state = 0;
+
+   if (hba->dev_info.f_perm_wp_en)
+       state = 2;
+   else if (hba->dev_info.f_power_on_wp_en)
+       state = 1;
+
+   curr_len = snprintf(buf, PAGE_SIZE, "%d\n", state);
+   return curr_len;
+}
+
+static ssize_t ufshcd_wp_store(struct device *dev,
+       struct device_attribute *attr, const char *buf, size_t count)
+{
+   struct ufs_hba *hba = dev_get_drvdata(dev);
+   unsigned long flags;
+   int ret = 0;
+
+   if (count != 1)
+       return -EINVAL;
+
+   spin_lock_irqsave(hba->host->host_lock, flags);
+   if (strncmp(buf, "1", 1))
+       ret = ufshcd_write_wp(hba, QUERY_FLAG_IDN_PWR_ON_WPE);
+   else if (strncmp(buf, "2", 1))
+       ret = ufshcd_write_wp(hba, QUERY_FLAG_IDN_PERMANENT_WPE);
+   else
+       ret = -EINVAL;
+   spin_unlock_irqrestore(hba->host->host_lock, flags);
+
+   return ret;
+}
+
+static void ufshcd_add_wp_sysfs_nodes(struct ufs_hba *hba)
+{
+   hba->wp_attr.show = ufshcd_wp_show;
+   hba->wp_attr.store = ufshcd_wp_store;
+   sysfs_attr_init(&hba->wp_attr.attr);
+   hba->wp_attr.attr.name = "wp";
+   hba->wp_attr.attr.mode = 0644;
+   if (device_create_file(hba->dev, &hba->wp_attr))
+       dev_err(hba->dev, "Failed to create sysfs for wp\n");
+}
+
 static inline void ufshcd_add_sysfs_nodes(struct ufs_hba *hba)
 {
    ufshcd_add_rpm_lvl_sysfs_nodes(hba);
    ufshcd_add_spm_lvl_sysfs_nodes(hba);
+   ufshcd_add_wp_sysfs_nodes(hba);
 }
 
 static inline void ufshcd_remove_sysfs_nodes(struct ufs_hba *hba)
 {
    device_remove_file(hba->dev, &hba->rpm_lvl_attr);
    device_remove_file(hba->dev, &hba->spm_lvl_attr);
+   device_remove_file(hba->dev, &hba->wp_attr);
 }
 
 /**
diff --git a/drivers/scsi/ufs/ufshcd.h b/drivers/scsi/ufs/ufshcd.h
index cdc8bd0..727a874 100644
--- a/drivers/scsi/ufs/ufshcd.h
+++ b/drivers/scsi/ufs/ufshcd.h
@@ -526,6 +526,9 @@ struct ufs_hba {
    enum ufs_pm_level spm_lvl;
    struct device_attribute rpm_lvl_attr;
    struct device_attribute spm_lvl_attr;
+   struct device_attribute wp_attr;
+
    int pm_op_in_progress;
 
    struct ufshcd_lrb *lrb;
```

Now you can enable the write protection:

```bash
echo -n '2' | sudo tee /sys/devices/platform/soc/ff3b0000.ufs/wp
```

Meaning `fPermanentWPEn` will be enabled. No write-protection will occur, but now values of `0x2` written to Unit Descriptor's `bLUWriteProtect` offset will make entire Logic Unit's data write-protected permanently.

## How to create a Root of Trust on HiKey960

I am referring to a do-it-yourself Root of Trust where no manufacture NDA or collaboration is required.

To allow this capability a change to the HiSilicon-provided, closed source, `xloader.bin` is needed. It will need the capability to boot `BL2` from a LUN that is not LUN3. If that capability exists then a Root of Trust can be created by permanent write-protecting LUN0 (where the `xloader` lives), and either LUN1 or LUN2, whichever contains `BL2`. This is because ARM-Trusted-Firmware's BL2 for the HiKey960 contains a root public key used to verify FIP, which can live on the RW LUN3.

All of this assumes there is no software programmable way to cause `xloader` to boot from a different location.
