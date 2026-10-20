# How the rockchip image finds its root filesystem, and whether it works from NVMe

Research for [#6](https://github.com/EnzoAlbornoz/openwrt/issues/6) (map: [#1](https://github.com/EnzoAlbornoz/openwrt/issues/1)).
Terms follow `GLOSSARY.md` (SPI bootloader, boot medium).

All in-repo citations are at commit `4e16c2ed8f` (the base of this branch).
External sources are pinned to the versions this tree builds: U-Boot `v2026.10`
(`package/boot/uboot-rockchip/Makefile:8`), firmware-utils `0179ab1e`
(`tools/firmware-utils/Makefile:14-15`), fstools `2aa16407`
(`package/system/fstools/Makefile:16-17`) and Linux `v6.18`.

## Answer in brief

- The image is an **MBR disk with two partitions**: p1 is an ext4 "kernel" partition
  holding `boot.scr` and a FIT `kernel.img` (kernel + DTB), p2 is the root filesystem.
  U-Boot sits raw at sector 64 inside the first 32 MiB.
- The kernel gets **`root=PARTUUID=<disk-signature>-02`**. The boot script computes it at
  boot time with `part uuid ${devtype} ${devnum}:2`, i.e. from whatever medium U-Boot
  loaded the script from.
- The boot script has **no mmc-only commands**. It uses `${devtype}`/`${devnum}` that
  U-Boot's script bootmeth sets to `nvme`/`0` for NVMe. Upstream fixed this in 2024 for
  the NanoPi R5S booting from NVMe.
- sysupgrade (`legacy-sdcard`) and preinit find the boot disk by matching the MBR disk
  signature from `root=` against every block device, then picking partitions by `DISKSEQ`.
  **Nothing depends on the device name**, so `nvme0n1` works.
- **So the same image works unchanged from NVMe**, provided U-Boot can read NVMe (ticket #5).
  The kernel side is in place too, since NVMe, PCIe and the RK3528 PCIe DTS are built in.
- **The catch is that every rockchip image has the same disk signature, `0x5452574F` ("OWRT").**
  With OpenWrt on *both* microSD and NVMe, `root=PARTUUID=5452574f-02` matches both disks.
  The kernel and sysupgrade then each pick "the first one they find", and that need not be
  the boot medium U-Boot loaded the kernel from.

## 1. Image layout

| Step | What it does | Source |
|---|---|---|
| `IMAGE/sysupgrade.img.gz` | `boot-common \| boot-script $(BOOT_SCRIPT) \| pine64-img \| gzip \| append-metadata` | `target/linux/rockchip/image/Makefile:53` |
| `boot-common` | copies the kernel artifact to `kernel.img` in a staging dir | `Makefile:13-20` |
| `KERNEL` | `kernel-bin \| lzma \| fit lzma <dtb>`, so `kernel.img` is a FIT with kernel+DTB, booted with `bootm` | `Makefile:49`, `default.bootscript:23` |
| `boot-script` | `mkimage -T script` of `$(BOOT_SCRIPT).bootscript` (default `default`) to `boot.scr` | `Makefile:22-25` |
| `pine64-img` | `gen_image_generic.sh` with `PADDING=1`, kernel part size, rootfs part size, alignment `32768` KiB, so partitions start on 32 MiB boundaries (`-l` is in KiB, [ptgen.c:635,851](https://github.com/openwrt/firmware-utils/blob/0179ab1e367f756ac108a485f26585661d4c528c/src/ptgen.c#L851)) | `Makefile:36-40` |
| U-Boot | `dd` `$(UBOOT_DEVICE_NAME)-u-boot-rockchip.bin` at sector 64 | `Makefile:43` |
| Partition table | `ptgen` builds an MBR. No `GUID`, so no GPT, and **no `SIGNATURE`** is passed | `scripts/gen_image_generic.sh:24` |
| p1 "kernel" | ext4 (`make_ext4fs -L kernel`), because `GUID` is unset | `gen_image_generic.sh:52-58` |
| p2 rootfs | raw `IMAGE_ROOTFS` (squashfs or ext4; target `FEATURES` include both) | `gen_image_generic.sh:49-50`, `target/linux/rockchip/Makefile:7` |
| Sizes | `TARGET_KERNEL_PARTSIZE` default 16 MiB, `TARGET_ROOTFS_PARTSIZE` default 104 MiB | `config/Config-images.in:343-356` |

**Disk signature.** `ptgen` defaults the MBR signature to `0x5452574F` ('OWRT') unless
`-S` is given ([ptgen.c:694](https://github.com/openwrt/firmware-utils/blob/0179ab1e367f756ac108a485f26585661d4c528c/src/ptgen.c#L694),
help text [L641](https://github.com/openwrt/firmware-utils/blob/0179ab1e367f756ac108a485f26585661d4c528c/src/ptgen.c#L641)).
`gen_image_generic.sh` only passes `-S` if `SIGNATURE` is set (`gen_image_generic.sh:24`), and
rockchip's `pine64-img` does not set it (`Makefile:36`). x86, armsr, bcm27xx, mvebu, tegra and
loongarch64 pass `SIGNATURE="$(IMG_PART_SIGNATURE)"` (e.g. `target/linux/x86/image/Makefile:47`).
That value is a per-build hash of `SOURCE_DATE_EPOCH`+vermagic (`include/image.mk:55`).
**So every rockchip image, of any build or version, has PARTUUIDs `5452574f-01`/`-02`.**

**The image carries U-Boot.** `pine64-img` needs `$(UBOOT_DEVICE_NAME)-u-boot-rockchip.bin`
(`Makefile:43`), and `Device/Default` derives `UBOOT_DEVICE_NAME` (`Makefile:56`). The E24C image
therefore needs a `uboot-rockchip` build (or a recipe override) even though it boots from the
SPI bootloader. That copy is written to sector 64 of whatever medium the image lands on.

## 2. How `root=` is formed: the boot script

`target/linux/rockchip/image/default.bootscript`:

- `:1` `part uuid ${devtype} ${devnum}:2 uuid` reads p2's PARTUUID from the medium U-Boot is
  booting from.
- `:9-10` for `soc = rk3528`: `serial_addr=',0xff9f0000'; serial_port='ttyS0'`.
- `:19` `bootargs = "console=${serial_port},1500000 earlycon=uart8250,mmio32${serial_addr} root=PARTUUID=${uuid} rw rootwait"`.
- `:21` `load ${devtype} ${devnum}:1 ${kernel_addr_r} kernel.img`, then `:23` `bootm`.

The device-specific scripts `radxa-e25.bootscript` and `rock-pi-s.bootscript` follow the same
pattern (`:1`, `:3`, `:5`) with fixed consoles. Per-device scripts are selected with `BOOT_SCRIPT :=`
(`target/linux/rockchip/image/armv8.mk:289`, `:456`).

**Assumptions about the loading device.** Partition numbers `:1`/`:2` are hard-coded, and the
script ignores `distro_bootpart`. It assumes `devtype`/`devnum` are set, and uses only generic
commands (`part`, `load`, `bootm`). It needs U-Boot built with `CMD_PART`, ext4 read support and
NVMe/PCIe drivers, which ticket #5 covers.

**Who sets `devtype`/`devnum`.** U-Boot's standard-boot script bootmeth, in `script_boot()`, sets
`devtype` to `blk_get_devtype()` of the block device, or `usb`/`sata` in special cases. It also
sets `devnum` (hex) and `distro_bootpart`, and only then runs the script
([boot/bootmeth_script.c:197-217 @ v2026.10](https://github.com/u-boot/u-boot/blob/v2026.10/boot/bootmeth_script.c#L197-L217)).
For an NVMe namespace this yields `devtype=nvme devnum=0`. U-Boot looks for `boot.scr` on any
partition it scans, which here is p1 at its root
([bootmeth_script.c:65-115](https://github.com/u-boot/u-boot/blob/v2026.10/boot/bootmeth_script.c#L65-L115)).

**History.** The script used to hard-code `part uuid mmc ${devnum}:2`. That failed when booting
`nvme#0.blk#1.bootdev.part_1` on a NanoPi R5S. Commit `4d49ce1bde` ("rockchip: fix boot from
non-MMC devices", Jan 2024, fixes [openwrt#14420](https://github.com/openwrt/openwrt/issues/14420))
renamed `mmc.bootscript` to `default.bootscript` and switched to `${devtype}`. Its message notes
that boards with only a BootROM may need U-Boot on eMMC/SD to reach NVMe. The E24C sidesteps this
because U-Boot lives in SPI NOR.

## 3. How the kernel resolves `root=PARTUUID=`

`devt_from_partuuid()` calls `class_find_device(&block_class, NULL, &cmp, &match_dev_by_uuid)` and
takes the **first** block device whose partition UUID matches
([block/early-lookup.c:46-71 @ v6.18](https://github.com/torvalds/linux/blob/v6.18/block/early-lookup.c#L46-L71)).
`rootwait` makes the kernel wait for a match, which is what lets the PCIe/NVMe probe finish.

Kernel support for an NVMe root is built in, not modular, in `target/linux/rockchip/armv8/config-6.18`:
- `CONFIG_BLK_DEV_NVME=y` (`:117`) and `CONFIG_NVME_CORE=y` (`:458`).
- `CONFIG_PCI=y` (`:483`), `CONFIG_PCIE_DW*=y` and `CONFIG_PCIE_ROCKCHIP_DW_HOST=y` (`:491-498`).
- `CONFIG_PHY_ROCKCHIP_NANENG_COMBO_PHY=y` (`:523`).
- `CONFIG_EXT4_FS=y` (`:249`).

The RK3528 PCIe node is backported in
`target/linux/rockchip/patches-6.18/070-26-v6.19-arm64-dts-rockchip-Add-PCIe-Gen2x1-controller-for-RK3528.patch`
(E20C enable: `071-14-v6.19-...`). Whether this covers the E24C's M.2 slot is ticket #4.
`kmod-nvme` in `DEVICE_PACKAGES` is redundant here, because its `KCONFIG` symbols are already `=y`
(`package/kernel/linux/modules/block.mk:493-506`).

## 4. Overlay (squashfs images)

fstools finds `rootfs_data` from the device actually mounted on `/` (or `/rom`). It `stat()`s the
mountpoint and scans `/dev` for the matching `dev_t`, then puts the overlay after the squashfs/erofs
end on that same partition
([libfstools/rootdisk.c:45-85, 160-197 @ 2aa16407](https://github.com/openwrt/fstools/blob/2aa16407fdedb3f1e1b915879f4d6772c5fe896c/libfstools/rootdisk.c#L45-L197)).
This is name-agnostic and follows whatever the kernel picked.

## 5. sysupgrade and config restore: finding the boot disk

`target/linux/rockchip/armv8/base-files/lib/upgrade/platform.sh` is generic:
- `platform_check_image` calls `legacy_sdcard_check_image` (`:3-5`).
- `platform_do_upgrade` calls `legacy_sdcard_do_upgrade` (`:17-19`).
- `platform_copy_config` writes the backup to partition 1 via `export_partdevice partdev 1` (`:7-15`).

Preinit `lib/preinit/79_move_config:6` moves it back with `export_bootdevice && export_partdevice partdev 1`.

`export_bootdevice` (`package/base-files/files/lib/upgrade/common.sh:168-224`):
- Reads `root=` from the cmdline (`:171`).
- For an MBR `PARTUUID=xxxxxxxx-NN`, it loops over `$(find /dev -type b)`, reads the 4 bytes at
  offset 440 (MBR disk signature) and **breaks on the first match** (`:174-183`).
- It also handles GPT PARTUUIDs (`:185-196`), `/dev/*` paths (`:198-199`) and major:minor (`:201-212`).
- It exports the disk's `DISKSEQ` as `BOOTDEV_DISKSEQ` (`:219`).

`export_partdevice` then picks the disk (`partn=0`) or partition N with the same `DISKSEQ` from
`/sys/class/block/*/uevent` (`:226-243`). Commit `63d0b5c243` switched from MAJOR/MINOR to
`DISKSEQ` specifically because NVMe minors interleave (its example is `nvme0n1`/`nvme1n1`).

`legacy_sdcard_do_upgrade` (`package/base-files/files/lib/upgrade/legacy-sdcard.sh:108-154`) runs in
one of two modes:
- **full**: `dd`s the whole image to `/dev/$diskdev`, including the image's U-Boot at sector 64,
  then `partx -d/-a` (`:121-128`).
- **partitions**: writes each partition, then copies the image's 4 signature bytes at 440 onto the
  disk (`:129-141`). **Either way the disk ends up with signature `OWRT` again.**

**Verdict: nothing in this path is mmc-specific, and `nvme0n1` works.** No rockchip-specific
change is needed for an NVMe boot medium.

**Existing NVMe users in tree.** Several armv8 devices ship `kmod-nvme`:
- `friendlyarm_nanopi-m5` (`armv8.mk:95`) and `linkease_easepi-r1` (`:238`).
- Radxa ROCK 5 ITX/5A/5B/5B+/5C/5T (`:375-421`).

None documents booting OpenWrt *from* NVMe; the NanoPi M5 commit `f9dd769052` boots from microSD.
The only in-tree evidence of NVMe boot is the NanoPi R5S log in `4d49ce1bde`. That device has no
definition in this tree.

## 6. What breaks, or could break, from NVMe

1. **U-Boot side (not OpenWrt's image).** The SPI bootloader must have NVMe, PCIe and the RK3528
   combo PHY, and must scan the `nvme` bootdev. The image needs nothing beyond that. This is ticket #5.
2. **Duplicate PARTUUID when both media hold OpenWrt.** This is the real hazard. Every rockchip image
   has signature `0x5452574F` (§1). Take the case where microSD and NVMe both contain an OpenWrt
   rockchip image, of any version. This is the natural state while installing to NVMe from a
   microSD boot, or when a rescue card is left in.
   - U-Boot loads `kernel.img` from the medium it tries first (microSD, per the map).
   - The kernel mounts the **first** block device in class order with PARTUUID `5452574f-02`
     (`early-lookup.c:71`). That depends on whether the SD controller or the NVMe/PCIe probes first,
     not on where the kernel came from. The result can be a kernel from one disk with the rootfs
     from the other.
   - `export_bootdevice` independently takes the **first** `/dev` block device with that signature
     (`common.sh:177-182`). That is `find` order, which may differ from the kernel's choice. So
     sysupgrade can write, and config backup or restore can touch, a disk that is not the running root.
   - Writing the same image to both media makes the two disks identical by design, so this cannot
     be avoided with a per-build `IMG_PART_SIGNATURE` alone. Re-randomising the signature after
     install would not survive sysupgrade either (§5, `legacy-sdcard.sh:122` and `:141`).
3. **The image carries its own U-Boot** (§1). Every full sysupgrade re-writes it to the boot medium.
   On NVMe it is inert. On microSD, whether the BootROM prefers it over the SPI bootloader depends on
   RK3528's BootROM order (tickets #5/#11). It also means the E24C device definition needs a
   `uboot-rockchip` entry or a recipe override (ticket #10/#11).
4. **Console on rk3528 is hard-coded** to `ttyS0`@`0xff9f0000`, 1500000 baud
   (`default.bootscript:9-10,19`). This is fine if the E24C debug UART is UART0 (ticket #7).

## 7. Options for the duplicate-PARTUUID problem (input to #11, not decided here)

- **Per-device boot script that passes a device path.** `BOOT_SCRIPT := radxa-e24c` would emit
  `root=/dev/nvme0n1p2` when `devtype = nvme` and `root=/dev/mmcblkXp2` otherwise. Precedents:
  `armv8.mk:289`, `:456`. `export_bootdevice` already supports `/dev/*` (`common.sh:198-199`).
  This needs a stable `mmcblk` index (DT `mmc` aliases) and diverges from the shared `default.bootscript`.
- **Accept the restriction.** Document that only one boot medium may hold OpenWrt at a time, and
  wipe or repartition microSD after installing to NVMe.
- **Unique signature per medium.** Set it at install time and teach `legacy-sdcard` to preserve the
  disk's existing signature instead of copying the image's (`legacy-sdcard.sh:141`). This is a
  generic base-files change that affects every legacy-sdcard target.
