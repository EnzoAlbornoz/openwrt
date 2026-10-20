# U-Boot 2026.10 as the SPI bootloader on the Radxa E24C

Research for [#5](https://github.com/EnzoAlbornoz/openwrt/issues/5) (map: [#1](https://github.com/EnzoAlbornoz/openwrt/issues/1)).
Terms follow `GLOSSARY.md`: **SPI variant**, **SPI bootloader**, **boot medium**, **full-NOR image**.

Sources were checked on 2026-10-20 against:

- U-Boot `v2026.10` (commit `5508406582f6f7120dce0c4b3059819a41788db2`, "Prepare v2026.10"). This is the version in
  `package/boot/uboot-rockchip/Makefile:8`. In the links below, `UB` means
  `https://source.denx.de/u-boot/u-boot/-/blob/v2026.10` (the host now redirects to `git.u-boot-project.org`).
- This fork at `4e16c2ed8fc2be6971c2a2e358798eefca4f5d04`. In the links below, `OW` means
  `https://github.com/EnzoAlbornoz/openwrt/blob/4e16c2ed8fc2be6971c2a2e358798eefca4f5d04`.
- Jonas Karlman's (Kwiboo) WIP branch `rk3528` at
  `https://source.denx.de/u-boot/contributors/kwiboo/u-boot/-/commits/rk3528`. Its head is `997db7d7`, committed
  2026-05-01 on top of `v2026.07-rc1` (`4433253e`).
- U-Boot patchwork (`patchwork.ozlabs.org`, project `uboot`), linux-rockchip patchwork (`patchwork.kernel.org`),
  Linux `master` (7.3) and `linux-next` trees, TF-A on GitHub, and rkbin on GitHub.

## Answer

**Not with the stock tree. With three of Kwiboo's WIP commits carried as patches, it works, and that combination
has been tested on an E24C.**

Every driver the job needs is already in v2026.10: the SFC (SPI flash controller), the DesignWare PCIe controller,
the RK3528 PCIe mode of the Naneng combo PHY, NVMe, and the SFC clock. What v2026.10 lacks for RK3528 is the
glue:

- an SFC device-tree node
- a `BROM_BOOTSOURCE_SPINOR` entry that lets SPL keep loading from NOR
- an RK3528 layout for the SPI image
- an E24C device tree, board-ID entry and defconfig
- any RK3528 defconfig that enables SFC, PCI or NVMe

Kwiboo's WIP branch adds exactly these. His commit message says it was tested on an E24C v1.203 with "SD-card boot,
SPI flash boot, … PCIe/NVMe".

The boot order needs no change: U-Boot's default `boot_targets` for Rockchip is `mmc1 mmc0 nvme …`, and on RK3528
`mmc1` is the microSD slot, so U-Boot tries microSD before NVMe. Every RK3528 board, including the WIP E24C config,
keeps its environment nowhere (`ENV_IS_NOWHERE`, so nothing is saved to flash). U-Boot can keep its environment in
NOR, but if it does, the offset has to be chosen deliberately: the Rockchip default of `0x140000` can overlap the
U-Boot FIT.

## 1. Is there an E24C target in mainline U-Boot?

- **No E24C defconfig, DT or board entry in v2026.10.** `configs/` has only `generic-rk3528`, `nanopi-zero2-rk3528`,
  `radxa-e20c-rk3528`, `rock-2-rk3528` and `sige1-rk3528`. A case-insensitive grep for `e24c` across `arch/`,
  `board/`, `configs/`, `dts/` and `doc/` finds nothing RK3528-related, and `doc/board/rockchip/rockchip.rst:104-109`
  lists no E24C entry.
  Nothing changed after the release: `configs/radxa-e24c-rk3528_defconfig` returns 404 on both `master` and `next`
  (`https://source.denx.de/u-boot/u-boot/-/raw/{master,next}/configs/radxa-e24c-rk3528_defconfig`).
- **It was never posted to the U-Boot list.** A patchwork search for `E24C` in project `uboot` returns nothing
  (`https://patchwork.ozlabs.org/api/patches/?project=uboot&q=E24C`). The RK3528 series on that list cover other
  boards: NanoPi R28S, USB, OTP and clock fixes, with Kwiboo's most recent ones dated 2026-10-15
  (`https://patchwork.ozlabs.org/api/patches/?project=uboot&q=rk3528`).
- **No E24C DT in Linux either.** U-Boot uses `OF_UPSTREAM`, which takes board DTs from the Linux tree, and
  `rk3528-radxa-e24c.dts` is missing from Linux `v6.18`, `master` (7.3) and `linux-next`. Patchwork still marks all
  three patches of the Jul 2025 kernel series as `new` (not applied):
  - [1/3 "Add SFC node for RK3528"](https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-2-jonas@kwiboo.se/)
  - [2/3 bindings](https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-3-jonas@kwiboo.se/)
  - [3/3 board DTS](https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-4-jonas@kwiboo.se/)

  The [cover letter](https://lkml.iu.edu/hypermail/linux/kernel/2507.3/03612.html) says the board "can be booted from
  SD-card using the U-Boot 2025.07 generic-rk3528 target or preferably using the work-in-progress patches" at
  `source.denx.de/u-boot/contributors/kwiboo/u-boot/-/commits/rk3528`.
- **Kwiboo's WIP, the three commits that matter** (each linked as `…/-/commit/<sha>`):
  - `821e379f` *rockchip: rk3528: Add support for booting from SPI flash*
    ([patch](https://source.denx.de/u-boot/contributors/kwiboo/u-boot/-/commit/821e379f02808d157542f9feeb30f2ce2e6056d5)).
    It adds an `spi2 = &sfc` alias, `bootph` tags on `sfc`, `fspi_pins` and `fspi_csn0`, and
    `u-boot,spl-sfc-no-dma`. It also adds a `[BROM_BOOTSOURCE_SPINOR] = "/soc/spi@ffc00000/flash@0"` entry in
    `rk3528.c`, and a binman override that writes the SPI idbloader in **`rksd`** format at offset **`0x8000`**.
  - `9c77b725` *WIP: board: rockchip: Add Radxa E24C*
    ([patch](https://source.denx.de/u-boot/contributors/kwiboo/u-boot/-/commit/9c77b72594954bcb122de2795e29dbdd9cc2233d)).
    It adds `configs/radxa-e24c-rk3528_defconfig` and `arch/arm/dts/rk3528-radxa-e24c-u-boot.dtsi`, and adds the
    E24C hardware-ID range `{ 745, 846, "rockchip/rk3528-radxa-e24c.dtb" }` to the ADC table in
    `board/radxa/rock-2-rk3528/rock-2-rk3528.c`. It also adds the E24C DT to `OF_LIST` in the E20C and ROCK 2
    defconfigs, and enables SFC, PCI and NVMe in all three. Tested on "E24C v1.203 (D4E0S16)".
  - `dc903763` *HACK: rockchip: dts/upstream: use work-in-progress RK3528 DTs*
    ([patch](https://source.denx.de/u-boot/contributors/kwiboo/u-boot/-/commit/dc9037637306eaca353f65ae9d5f5f59df51da39)).
    It adds the `sfc: spi@ffc00000` node (`compatible = "rockchip,sfc"`) to `dts/upstream/.../rk3528.dtsi` and adds
    the whole `rk3528-radxa-e24c.dts`. That DTS describes the NOR as `jedec,spi-nor` at 50 MHz with RX bus width 4,
    turns PCIe on with `vpcie3v3-supply = <&vcc3v3_mkey>` (GPIO4_C6) and `reset-gpios = <&gpio1 RK_PA2 …>`, and sets
    `sdhci` to `status = "fail"`: eMMC and FSPI share pins.
- **Rebasing onto v2026.10 looks small.** The branch's `rk3528-u-boot.dtsi` differs from v2026.10 only by the SPI
  additions above and by the `otp` node (`UB/arch/arm/dts/rk3528-u-boot.dtsi:26-29`), which the branch removes
  because its HACK DTs supply it. A board patch that places the DTS under `dts/upstream/src/arm64/rockchip/` already
  has a precedent in this tree: `OW/package/boot/uboot-rockchip/patches/107-board-rockchip-add-HINLINK-H28K.patch`
  (it creates `dts/upstream/src/arm64/rockchip/rk3528-hinlink-h28k.dts`).

## 2. Can RK3528 SPL load from SPI NOR, and which TPL and BL31 does it use?

**What v2026.10 already has:**

- The SFC driver, `ROCKCHIP_SFC` (`UB/drivers/spi/Kconfig:475`; compatible `rockchip,sfc`,
  `UB/drivers/spi/rockchip_sfc.c:645`), including the SFC v8 handling (`rockchip_sfc.c:596`). The RK3528 clock
  driver can read and set `SCLK_SFC` (`UB/drivers/clk/rockchip/clk_rk3528.c:926-969,1373,1490`).
- Opened by SPL, `arch_cpu_init()` already lets the FSPI master access DDR (`FW_DDR_MST7_REG`,
  `UB/arch/arm/mach-rockchip/rk3528/rk3528.c:63-65`).
- The generic SPI image recipe: `ROCKCHIP_SPI_IMAGE` (`UB/arch/arm/mach-rockchip/Kconfig:746`) makes binman write
  `u-boot-rockchip-spi.bin` with an idbloader (TPL + SPL) and a FIT at `CONFIG_SYS_SPI_U_BOOT_OFFS`
  (`UB/arch/arm/dts/rockchip-u-boot.dtsi:197-228`).
  The docs say to "Write u-boot-rockchip-spi.bin to offset 0 of SPI flash", and give a `sf probe` /
  `sf update $fileaddr 0 $filesize` recipe for doing it from U-Boot (`UB/doc/board/rockchip/rockchip.rst:388-399`).

**What v2026.10 lacks for RK3528:**

- **No SFC node.** The RK3528 SoC DT (`UB/dts/upstream/src/arm64/rockchip/rk3528.dtsi`) has a `qos_fspi` node but
  no `sfc` controller node. `spi0` and `spi1` at `0xff9c0000` and `0xff9d0000` are ordinary SPI controllers, not the
  FSPI. Linux `master` and `linux-next` have no RK3528 SFC node either.
- **No `BROM_BOOTSOURCE_SPINOR` mapping.** `boot_devices[]` lists only eMMC and SD
  (`UB/arch/arm/mach-rockchip/rk3528/rk3528.c:21-24`; unchanged on `next`). RK3568 and RK3588 do have it
  (`rk3568.c:95`, `rk3588.c:57-59`).
  In SPL, `"same-as-spl"` resolves through `board_spl_was_booted_from()`. With no mapping it returns NULL, and the
  `same-as-spl` entry is skipped (`UB/arch/arm/mach-rockchip/spl.c:51-75`,
  `UB/arch/arm/mach-rockchip/spl-boot-order.c:140-144`).
  The RK3528 order is `"same-as-spl", &sdmmc, &sdhci` (`UB/arch/arm/dts/rk3528-u-boot.dtsi:12`). So an SPL built
  from stock sources and booted from NOR would move on and look for U-Boot proper on microSD rather than in NOR.
- **No RK3528 binman override.** RK3528 needs the idbloader in `rksd` format at `0x8000`, as Kwiboo's `821e379f`
  shows. RK3328 already has the identical override, `rksd` @ `0x8000` (`UB/arch/arm/dts/rk3328-u-boot.dtsi:157-166`), and
  `rk3326` has `rksd` @ `0x10000` (`UB/arch/arm/dts/rk3326-u-boot.dtsi:29-38`). RK3528 has none in v2026.10.
- **No RK3528 defconfig enables SFC, SPI flash, PCI or NVMe.** See `UB/configs/*rk3528_defconfig`.

**Layout that Kwiboo's WIP produces:** `u-boot-rockchip-spi.bin` written at NOR offset 0. It holds:

| NOR offset | Contents | Set by |
|---|---|---|
| `0x8000` | idbloader: rkbin TPL (DDR init) + U-Boot SPL, `rksd` format | `821e379f`, binman override |
| `0x60000` | FIT: U-Boot proper + BL31 + DTBs | `CONFIG_SYS_SPI_U_BOOT_OFFS=0x60000` in `9c77b725` |

The WIP defconfig also sets `SF_DEFAULT_BUS=2`, `SF_DEFAULT_SPEED=50000000`, `SF_DEFAULT_MODE=0x2000`,
`SPL_SPI_LOAD`, `SPI_FLASH_SFDP_SUPPORT` and **only** `SPI_FLASH_MACRONIX` (`9c77b725`).

**TPL and BL31 come from rkbin blobs, not TF-A 2.15:**

- **TPL (DDR init):** `ROCKCHIP_EXTERNAL_TPL` is implied for RK3528 (`UB/arch/arm/mach-rockchip/Kconfig:394`), so
  TPL is the rkbin DDR blob passed as `ROCKCHIP_TPL`.
- **BL31:** upstream TF-A has no RK3528 platform. `plat/rockchip/` at `v2.15.0` has only `common px30 rk3288 rk3328
  rk3368 rk3399 rk3568 rk3576 rk3588` (`https://github.com/ARM-software/arm-trusted-firmware/tree/v2.15.0/plat/rockchip`),
  and `master` adds only `rv1126b`. This tree's TF-A package builds only `rk3328 rk3399 rk3576 rk3588`
  (`OW/package/boot/arm-trusted-firmware-rockchip/Makefile:46-50`).
- **What this tree uses:** `U-Boot/rk3528/Default` depends on `trusted-firmware-a-rk3528` from the **rkbin**
  package and uses `ATF:=rk3528_bl31_v1.20.elf` and `TPL:=rk3528_ddr_1056MHz_v1.11.bin`
  (`OW/package/boot/uboot-rockchip/Makefile:177-182`; `OW/package/boot/rkbin/Makefile:40-44`). rkbin is pinned to
  `74213af1e952c4683d2e35952507133b61394862` (2025-06-13, `OW/package/boot/rkbin/Makefile:12-13`).
- **Newer rkbin:** `master` (`3e288fe8`, 2026-06-26) has `rk3528_bl31_v1.21.elf` and `rk3528_ddr_1056MHz_v1.14.bin`
  (`https://github.com/rockchip-linux/rkbin/tree/master/bin/rk35`). The blobs are passed through as
  `BL31=…`/`ROCKCHIP_TPL=…` (`OW/package/boot/uboot-rockchip/Makefile:495-497`).

## 3. Can U-Boot boot from NVMe, and in what order?

- **PCIe works in principle on v2026.10 with config only.**
  - The RK3528 `pcie@fe000000` node is in `dts/upstream`
    (`UB/dts/upstream/src/arm64/rockchip/rk3528.dtsi:286-337`, `compatible = "rockchip,rk3528-pcie",
    "rockchip,rk3568-pcie"`, `status = "disabled"`).
  - `PCIE_DW_ROCKCHIP` (`UB/drivers/pci/Kconfig:414`) matches the `rockchip,rk3568-pcie` fallback
    (`UB/drivers/pci/pcie_dw_rockchip.c:492`) and turns on the `vpcie3v3` supply before PHY init (`:304-316`).
  - The combo PHY has an RK3528 PCIe mode (`UB/drivers/phy/rockchip/phy-rockchip-naneng-combphy.c:249-345,748`).
  - `NVME_PCI` exists (`UB/drivers/nvme/Kconfig:24`).
  - U-Boot has no Rockchip power-domain driver (nothing under `UB/drivers/power/domain/`), so the node's
    `power-domains = <&power RK3528_PD_VPU>` is not acted on.

  What is missing is the E24C board DT that enables `&pcie` and `&combphy` with the M.2 power and reset GPIOs, and a
  defconfig with `PCI`, `PCIE_DW_ROCKCHIP` and `NVME_PCI`. Both are in `9c77b725` and `dc903763`, which were tested
  with PCIe/NVMe.
- **How U-Boot boots from NVMe (bootstd).** RK3528 implies `BOOTSTD_FULL`
  (`UB/arch/arm/mach-rockchip/Kconfig:380`). When the bootflow scan reaches `nvme`, the NVMe hunter runs
  `pci_init()` and `nvme_scan_namespace()` (`UB/drivers/nvme/nvme-uclass.c:24-40`). The script boot method then sets
  `devtype`/`devnum` (`UB/boot/bootmeth_script.c:200-227`), which is what OpenWrt's `boot.scr` uses: `part uuid
  ${devtype} ${devnum}:2 uuid` and `load ${devtype} ${devnum}:1 … kernel.img`
  (`OW/target/linux/rockchip/image/default.bootscript:1,21`; RK3528 console at `:9-10`).
- **Order (microSD before NVMe):** `boot_targets` defaults to `BOOT_TARGETS` = `"mmc1 mmc0 nvme scsi usb pxe dhcp
  spi"` (`UB/include/configs/rockchip-common.h:16-17`, used by `UB/include/configs/rk3528_common.h:36`). The RK3528
  aliases make `mmc0 = &sdhci` (eMMC) and `mmc1 = &sdmmc` (microSD) (`UB/arch/arm/dts/rk3528-u-boot.dtsi:6-9`). So
  U-Boot proper tries microSD first, then eMMC (marked `fail` on the SPI variant, so it is skipped), then NVMe. That
  is the order the map requires, with no override. It can be pinned explicitly later through a
  `CONFIG_BOOTCOMMAND` or env `boot_targets`, but nothing requires that.
- **SPL order is a separate setting.** It only decides where SPL loads the U-Boot FIT from
  (`"same-as-spl", &sdmmc, &sdhci`), not where OpenWrt is loaded from.

## 4. Can U-Boot keep its environment in NOR?

- **Today the environment is nowhere.** No RK3528 defconfig in v2026.10, and not the WIP E24C defconfig, selects an
  `ENV_IS_IN_*` option, so `ENV_IS_NOWHERE` applies (`UB/env/Kconfig:100-110`) and U-Boot always starts with its
  built-in environment. OpenWrt's rockchip `uboot-envtools` config only sets up an environment for the OrangePi R1
  Plus, and that one is on MMC (`OW/package/boot/uboot-tools/uboot-envtools/files/rockchip_armv8:13-17`). OpenWrt's
  rockchip boot flow uses `boot.scr` and needs no saved environment.
- **NOR environment is possible.** `ENV_IS_IN_SPI_FLASH` uses Rockchip defaults of `ENV_OFFSET=0x140000`,
  `ENV_SIZE=0x2000` and `ENV_SECT_SIZE=0x2000` (`UB/env/Kconfig:649,706,716`).
  **Caution:** with the FIT at `0x60000`, the default offset leaves only `0xE0000` (896 KiB) for U-Boot proper +
  BL31 + DTBs. binman does not check that space against the environment.
  An RK3399 precedent sets an explicit `ENV_OFFSET=0x3F8000` and `ENV_SIZE=0x8000` with `SYS_SPI_U_BOOT_OFFS=0xE0000`
  (`UB/configs/rockpro64-rk3399_defconfig:8-9,28,42`).
- **Choosing size and offset is a layout decision for #10.** If U-Boot keeps an environment in NOR, it belongs at a
  fixed, erase-aligned offset inside the bootloader region and above the FIT's maximum size, so that it doesn't eat
  into space a future **full-NOR image** would need. Leaving it nowhere keeps NOR read-only at runtime, which fits
  "sysupgrade never writes NOR".

## 5. The `U-Boot/radxa-e20c-rk3528` pattern in this tree

- **Build:**
  - `U-Boot/rk3528/Default` sets `BUILD_SUBTARGET:=armv8`, the rkbin dependency, `ATF` and `TPL`
    (`OW/package/boot/uboot-rockchip/Makefile:177-182`).
  - `U-Boot/radxa-e20c-rk3528` adds only `NAME:=E20C` and `BUILD_DEVICES:=radxa_e20c` (`:191-196`).
  - The variant name is the upstream defconfig name, used as `UBOOT_CONFIG` (`OW/include/u-boot.mk:73,121-126`). It
    is listed in `UBOOT_TARGETS` (`:460`).
- **Image:**
  - `Device/radxa_e20c` sets `UBOOT_DEVICE_NAME := radxa-e20c-rk3528` (`OW/target/linux/rockchip/image/armv8.mk:274-282`).
  - `pine64-img` writes `$(UBOOT_DEVICE_NAME)-u-boot-rockchip.bin` at sector 64 of the SD/NVMe image
    (`OW/target/linux/rockchip/image/Makefile:27-44`).
- **Only one artifact is staged:** `Build/InstallDev` copies just `u-boot-rockchip.bin`
  (`OW/package/boot/uboot-rockchip/Makefile:509-512`). Nothing in `uboot-rockchip` or the rockchip image code
  mentions SPI. Shipping `u-boot-rockchip-spi.bin` needs a small `InstallDev` change, or a separate recipe.
- **What a `U-Boot/radxa-e24c-rk3528` entry would look like:**
  `$(U-Boot/rk3528/Default)`, `NAME:=E24C`, `BUILD_DEVICES:=radxa_e24c`, plus carried patches that add
  `configs/radxa-e24c-rk3528_defconfig`, the E24C DT, the RK3528 SFC node and the SPI-boot glue. That is the pattern
  `107-board-rockchip-add-HINLINK-H28K.patch` already uses.
- **The E20C or ROCK 2 binaries can't be reused as they are.** They pick their DTB through an ADC hardware-ID table
  that has no E24C range (`UB/board/radxa/rock-2-rk3528/rock-2-rk3528.c:16-20`). With no match,
  `board_fit_config_name_match()` matches nothing (`:52-59`), so the first `OF_LIST` entry is used: the E20C DT.

## Decision-relevant surprises

1. **"TF-A 2.15" plays no part on RK3528.** BL31 is the rkbin blob `rk3528_bl31_v1.20.elf`. Upstream TF-A has no
   RK3528 platform, even on `master`. Newer blobs exist (BL31 v1.21, DDR v1.14).
2. **No RK3528 SFC node exists anywhere upstream.** It is missing from U-Boot v2026.10 and `next`, and from Linux
   v6.18, `master` and `linux-next`. Kernel patch 1/3 has been `new` since Jul 2025. U-Boot needs it as a carried
   DT change. A 6.18 OpenWrt kernel also cannot see the NOR without a pending patch. That matters for any on-device
   install or update path for the SPI bootloader (#10), and for a future **full-NOR image**.
3. **E24C U-Boot support exists only as WIP.** It was never posted to the U-Boot list. Kwiboo's branch was last
   rebased on 2026-05-01 (`v2026.07-rc1`). Carrying it means 7xx-style pending patches in `uboot-rockchip`.
4. **The RK3528 SPI idbloader sits at `0x8000` in `rksd` format,** not `rkspi` at 0 as in the generic recipe. Any NOR
   layout decision must reserve `0x0–0x8000` (left as `0xff` padding) plus the idbloader and FIT regions.
5. **The WIP defconfig enables only the Macronix SPI-NOR driver.** If the factory NOR is from another vendor, the
   matching `SPI_FLASH_<vendor>` option has to be added. The JEDEC ID will come from #2 (the factory NOR backup).
6. **`uboot-rockchip` stages only `u-boot-rockchip.bin`.** No rockchip board in this tree ships an SPI image yet.

## Still to verify on hardware

- **Which medium the RK3528 BootROM tries first, NOR or microSD.** I found no primary source for it. Radxa's E24C
  docs cover flashing NOR only through Maskrom with `rkdeveloptool wl 0 spi.img`
  ([docs](https://docs.radxa.com/en/e/e24c/getting-started/install-os/nvme-system/spi-flash)), and say nothing about
  boot priority. This decides two things: whether the U-Boot embedded in the microSD image ever runs on the SPI
  variant, and whether a microSD card can recover a board whose NOR holds a bad loader. Either way, U-Boot proper
  still tries microSD before NVMe for OpenWrt itself.
- **How large the FIT built for the E24C actually is,** which fixes how much room the NOR bootloader region needs.
