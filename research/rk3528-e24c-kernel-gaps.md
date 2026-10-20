# What E24C and RK3528 kernel support is missing from 6.18?

Research for [EnzoAlbornoz/openwrt#4](https://github.com/EnzoAlbornoz/openwrt/issues/4) (map: #1).
This note leaves the **switch** driver to #3 and its DT integration to #8.

Researched 2026-10-20 against:

- OpenWrt fork `main` at `4e16c2ed8f`. Rockchip uses `KERNEL_PATCHVER:=6.18` (`target/linux/rockchip/Makefile:10`), pinned to **6.18.55** (`target/linux/generic/kernel-6.18:1-2`).
- The pristine `linux-6.18.55.tar.xz` from cdn.kernel.org, checked against the sha256 in `kernel-6.18`. "Patched tree" below means 6.18.55 plus the RK3528-relevant patches from `target/linux/rockchip/patches-6.18` (038-*, 039-*, 070-26, 071-14, 072-*, 073-*, 104, 138, 161-*). All of them apply cleanly.
- Mainline `torvalds/linux` master (Makefile says **7.3**; tags go up to `v7.3`). Release versions were worked out with `GET /repos/torvalds/linux/compare/<tag>...<sha>` ("behind" means the tag contains the commit).
- Heiko Stübner's Rockchip tree `mmind/linux-rockchip.git` `for-next` (https://git.kernel.org/pub/scm/linux/kernel/git/mmind/linux-rockchip.git/plain/arch/arm64/boot/dts/rockchip/rk3528.dtsi?h=for-next).
- patchwork.kernel.org (the linux-rockchip project) for postings and review threads. lore.kernel.org answered 403 to scripted access, so threads are cited through patchwork.
- Jonas Karlman's public work branch `Kwiboo/linux-rockchip` `next-20260430-rk3528` (https://github.com/Kwiboo/linux-rockchip/tree/next-20260430-rk3528). This is the author's own tree and has not been posted. It is cited only as evidence of where his E24C series is heading.

## Answer in one paragraph

Almost all the **SoC** support the E24C needs is already in OpenWrt's 6.18. Some of it is in 6.18.55 itself: GMAC1 RGMII, combphy, SARADC, pinctrl, CPU OPPs up to 2016 MHz, RK805 and the generic SFC driver. The rest is in existing `patches-6.18` backports: PCIe node, USB nodes and inno-usb2 RK3528, OTP, and TSADC. Three things are **missing**, and none of them is in mainline (v7.3) or in Rockchip `for-next`. The first is the RK3528 **SFC (FSPI) DT node**, without which Linux cannot see the SPI NOR. The second is the **E24C board DTS**, together with its dt-binding compatible. The third is (optionally) the **watchdog node**, which is in mainline v7.2. The only posting of the E24C DTS is v1 from July 2025, and it no longer matches today's `rk3528.dtsi`. It uses the old USB phy labels and has no PCIe/NVMe node. The author's unposted branch has a fixed version that is the better base. No driver backport is needed for anything except the switch, which belongs to #3.

## 1. Is `rk3528-radxa-e24c.dts` in mainline?

**No.** Not in v6.18.55, not in mainline master (v7.3), and not in Rockchip `for-next`.

- Mainline master `arch/arm64/boot/dts/rockchip/` holds RK3528 DTS files for armsom-sige1, hinlink-h28k, nanopi-zero2, radxa-e20c and rock-2a/2f. There is no E24C file. Source: `gh api repos/torvalds/linux/contents/arch/arm64/boot/dts/rockchip`, and `commits?path=…/rk3528-radxa-e24c.dts` returns nothing.
- The Rockchip `for-next` `Makefile` lists rk3528 armsom-sige1, hinlink-h28k, mangopi-m28k, nanopi-r28s, nanopi-zero2, radxa-e20c and rock-2a/2f. E24C is not there. Source: https://git.kernel.org/pub/scm/linux/kernel/git/mmind/linux-rockchip.git/plain/arch/arm64/boot/dts/rockchip/Makefile?h=for-next, lines 96-103.
- The only posting is **v1**, *[PATCH 0/3] arm64: dts: rockchip: Add Radxa E24C*, from Jonas Karlman on 2025-07-27. Patchwork series 986207 (https://patchwork.kernel.org/project/linux-rockchip/cover/20250727144409.327740-1-jonas@kwiboo.se/) contains:
  1. `[1/3] arm64: dts: rockchip: Add SFC node for RK3528`. Patchwork state is "new". https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-2-jonas@kwiboo.se/
  2. `[2/3] dt-bindings: arm: rockchip: Add Radxa E24C`. Acked by Krzysztof Kozlowski, state "new". https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-3-jonas@kwiboo.se/
  3. `[3/3] arm64: dts: rockchip: Add Radxa E24C`. Andrew Lunn gave Reviewed-by for the gmac1 node only. State "new". https://patchwork.kernel.org/project/linux-rockchip/patch/20250727144409.327740-4-jonas@kwiboo.se/
- No v2 was ever posted. A patchwork search for `E24C` returns only this series and the separate switch series 986228, which is #3's (`https://patchwork.kernel.org/api/patches/?q=E24C`). Review on the v1 thread asked for these changes:
  - SFC: put `clocks` before `interrupts`, and decide where the pinctrl goes. Chukun Pan and Jonas, 2025-07-28 (patchwork comments on 14169993).
  - DTS: rename the switch regulator to `vdd_switch` with `avddl_1v1`/`avddh_3v3` labels, use `#clock-cells = <0>` on RK805, and move the switch reset from `&mdio1` onto the switch node. Chukun Pan and Jonas, 2025-07-28/29.
  - DTS: drop the deprecated LED `label`s in favour of `function-enumerator`, and keep `BTN_1` for USER as on the E20C. FUKAUMI Naoki of Radxa and Heiko Stübner, 2025-08-15 to 2025-09-10 (patchwork comments on 14170006).

**Dependencies of v1 against the patched 6.18 tree.** I extracted the v1 DTS and resolved every `&label` it references against the patched `rk3528.dtsi`, `rk3528-pinctrl.dtsi` and the DTS itself. Only two kinds of label are unresolved:

| Unresolved label | Why | Fix |
|---|---|---|
| `&sfc` | No SFC node anywhere (see §3) | Pending patch [1/3] |
| `&u2phy`, `&u2phy_host`, `&u2phy_otg` | v1 was written against an earlier USB series. Mainline and OpenWrt `073-1` name them `usb2phy`, `usb2phy_host` and `usb2phy_otg` (patched `rk3528.dtsi:1406-1432`, upstream `5f3ae9b12a6c`) | Use the new labels |

v1 also **leaves out PCIe**. It defines `vcc3v3_mkey` and the `mkey_pwr_en` pinctrl but has no `&combphy` or `&pcie` node, so NVMe would not work. At the time it was posted, `rk3528.dtsi` had no PCIe node yet; that node arrived in v6.19.

**The author's current version.** It is unposted and lives in `Kwiboo/linux-rockchip` `next-20260430-rk3528`. It is the v1 series rebased and fixed:

- `132d82695f31` *arm64: dts: rockchip: Add Radxa E24C* adds `&combphy { status = "okay"; }` and `&pcie { pinctrl-0 = <&pciem1_pins>; reset-gpios = <&gpio1 RK_PA2 GPIO_ACTIVE_HIGH>; vpcie3v3-supply = <&vcc3v3_mkey>; }`. These are the same PCIe pins and PERST# as the E20C. It also takes up the review changes: `vdd_switch` regulator, RK805 `#clock-cells = <0>`, and the switch reset removed from `&mdio1`.
- `1ea810e218c5` *Enable USB ports on Radxa E24C* uses the `usb2phy*` labels.
- `6bf983983dfb` *Add RTL8367RB-VB switch to Radxa E24C* is #3/#8's territory. It is noted here because it also does `/delete-property/ snps,tso;` on `&gmac1`, and its message says *"TSO and/or RX checksum offload may need to be disabled on the conduit interface (gmac1)"*.
- `008276e5e201` *Add SFC node for RK3528* is [1/3] with the `clocks` and `interrupts` order fixed.
- The LED `label`s are still present in this version, so the review comment about them has not been addressed.

## 2. RK3528 GMAC (dwmac-rk) and GMAC1/RGMII in 6.18

**Present in 6.18.55, and already exercised in this tree.** The ticket assumed GMAC1/RGMII had never been used here because the E20C uses r8169. That assumption is wrong: the E20C's `ethernet0` *is* `gmac1` in RGMII mode, with an external PHY. r8169 on PCIe is its second port.

- Driver: `rockchip,rk3528-gmac` → `rk3528_ops`, including `rk3528_set_to_rgmii` for GMAC1 through `VPU_GRF_GMAC_CON5/6` (`drivers/net/ethernet/stmicro/stmmac/dwmac-rk.c:865-954,1815` in 6.18.55). It arrived with mainline `1725f0eb37d6` *net: stmmac: dwmac-rk: Add GMAC support for RK3528*, which is in **v6.15**.
- DT: the `gmac1: ethernet@ffbe0000` node with `mdio1` is at `rk3528.dtsi:1004-1050` in 6.18.55. It came from mainline `5eb28f461a1b` *Add GMAC nodes for RK3528*, in **v6.16**.
- In-tree users of `&gmac1` with `phy-mode = "rgmii-id"` and `clock_in_out = "output"`: `rk3528-radxa-e20c.dts:208-217`, `rk3528-rock-2a.dts:28`, `rk3528-armsom-sige1.dts:251` and `rk3528-nanopi-zero2.dts:211`. The E20C one came from mainline `10b9ef4a514b`, in **v6.16**. OpenWrt's `radxa_e20c` uses it (`target/linux/rockchip/image/armv8.mk:274-282`), and `CONFIG_DWMAC_ROCKCHIP=y` is set (`target/linux/rockchip/armv8/config-6.18:241`).
- The stable fix `7f864458e9a6` *dwmac-rk: Fix disabling set_clock_selection* (Cc: stable) is in 6.18.55 (`dwmac-rk.c:1451-1455`).
- The large `net: stmmac: rk:` refactor (Nov 2025 to Feb 2026, for example `d7d92037cbd8` *rk3528: gmac0 only supports RMII*) is cleanup and GMAC0/RMII work. None of it is needed for GMAC1 RGMII.

What *is* new on the E24C is a **fixed-link** to a switch, where the E20C has a PHY on `mdio1`. That raises questions about RGMII delays and offloads (the TSO note above), and those belong to #3 and #8.

## 3. SFC (SPI NOR controller) on RK3528

- **Driver: present and sufficient.** `drivers/spi/spi-rockchip-sfc.c` matches the generic `"rockchip,sfc"` (`:823`) and detects the IP version at runtime (`:211-213,691`), so it needs no RK3528-specific code. Both mainline fixes since 6.17 are in 6.18.55: `ee795e82e101` *Fix DMA-API usage* (`dma_map_single` at `:707`) and `111e2863372c` *Fix double-free in remove()* (`spi_register_controller` at `:715`). `CONFIG_SPI_ROCKCHIP_SFC=y` and `CONFIG_MTD_SPI_NOR=y` are already set (`config-6.18:665,425`).
- **Clocks, power domain and pins: present.** `SCLK_SFC`/`HCLK_SFC` are in `include/dt-bindings/clock/rockchip,rk3528-cru.h:150-151` and `clk-rk3528.c:957,989`. `RK3528_PD_VPU` is in `rockchip,rk3528-power.h:17`. `fspi_pins` and `fspi_csn0` are in `rk3528-pinctrl.dtsi:143-170`.
- **DT node: missing.** It is not in 6.18.55 `rk3528.dtsi`, not in any OpenWrt patch, not in mainline master (the full `rk3528.dtsi` history has no SFC commit), and not in Rockchip `for-next` (only `qos_fspi` appears). The only sources are pending patch [1/3] and its reordered form `008276e5e201` on Kwiboo's branch: `sfc: spi@ffc00000`, `compatible = "rockchip,sfc"`, IRQ `GIC_SPI 135`, `PD_VPU`.
- **Pins are shared with eMMC.** FSPI uses GPIO1_PC4-PC7, PD0 and PD5 with mux 2 (`rk3528-pinctrl.dtsi:145-164`). eMMC uses the same pins with mux 1 (`emmc_bus8`/`emmc_clk`, `:48-73`). That is why the E24C DTS sets `&sdhci { status = "fail"; }` and `&sfc { status = "okay"; }`, so the DTS describes the **SPI variant** only. Radxa confirmed that eMMC variants of the E24C exist (FUKAUMI Naoki, 2025-08-18, patchwork comments on 14170006).

## 4. Coverage of the other E24C peripherals in `patches-6.18`

| Block | 6.18.55 vanilla | OpenWrt `patches-6.18` | Mainline origin | Gap for E24C? |
|---|---|---|---|---|
| **PCIe** Gen2x1 DT node | missing | `070-26-v6.19-…Add-PCIe-Gen2x1-controller-for-RK3528` (`263fac6b09b4`) | v6.19 | none. Board needs `&combphy`/`&pcie` in its DTS (§1) |
| PCIe driver (`pcie-dw-rockchip`, `rk3568` fallback) | present, incl. L1SS config (`pcie-dw-rockchip.c:239`) | – | – | none |
| **combphy** RK3528 | present (`phy-rockchip-naneng-combphy.c:1427`, `bbcca4fac873`, v6.18); L1ss fix `a2a18e5da64f` (v6.19) already backported to 6.18.y (`:515`) | `104-phy-rockchip-naneng-combphy-Fix-TX-detect-RX` | – | none |
| **USB** DT nodes (dwc3, EHCI/OHCI, usb2phy) | missing | `073-1-v7.2-…Add-USB-nodes-for-RK3528` (`5f3ae9b12a6c`) | v7.2 | none (labels: `usb2phy*`) |
| USB2 PHY driver RK3528 | missing | `039-1/2/3-v7.2-phy-rockchip-inno-usb2-…` (039-3 = `864b3617df82`) | v7.2 | none |
| **SARADC** (MASKROM adc-key) | present: node `rk3528.dtsi:932` (`6e58302c84ce`, v6.15), driver `rockchip_saradc.c:362` | – | – | none. Package `kmod-input-adc-keys` as on E20C (`armv8.mk:280`) |
| **TSADC** | missing (no `rk3528` in `rockchip_thermal.c`) | `161-01/02/03` (Jonas Karlman, Mar 2025, out-of-tree) | **never posted upstream**; not in mainline or `for-next` | none for the build. It is carried, and its naming is not `7xx-` |
| **OTP** (TSADC trim, CPU leakage) | missing | `038-1..4-v7.1-nvmem-rockchip-otp-…` (038-4 = `a255f352b0e0`), `072-3-v7.1-…Enable-OTP-controller-for-RK3528` (`8c2c73084bd1`) | v7.1 | none |
| **GPIO LEDs** | `gpio-leds`, `LEDS_GPIO=y` (`config-6.18:371`), heartbeat and netdev triggers built in (`generic/config-6.18:3504,3508`) | `138-…Update-LED-properties-for-Radxa-E20C` is the E20C pattern for `led-boot`/`led-running` aliases | – | none for the kernel. The OpenWrt alias patch is #9/#12 |
| CPU OPP / DVFS | OPP table with 1200-2016 MHz, max 1.1 V (`rk3528.dtsi:103-136`; `42bbc32c7e9e`, v6.18) | – | – | none for the kernel. The E24C is the **first RK3528 board here whose `cpu-supply` is an RK805** (DCDC1 on i2c0). Every other in-tree RK3528 DTS uses `pwm-regulator`. RK805 support is mature (`MFD_RK8XX_I2C`, `REGULATOR_RK808` at `config-6.18:395-396,581`) |
| RTC hym8563, EEPROM at24 | present; `RTC_DRV_HYM8563=y`, `EEPROM_AT24=y` (`config-6.18:600,245`) | – | – | none |
| **Watchdog** | missing (no `wdt` node) | – | `29d71b732bdc` *dt-bindings: watchdog: Add watchdog compatible for RK3528* (v7.2), `eb24b60553e0` *arm64: dts: rockchip: Add watchdog node for RK3528* (v7.2) | optional. `DW_WATCHDOG=y` is already set (`config-6.18:242`), so with the node, procd would get `/dev/watchdog` |

All the Kconfig symbols the E24C DTS needs are already set in `target/linux/rockchip/armv8/config-6.18`: `PHY_ROCKCHIP_NANENG_COMBO_PHY` (:523), `PCIE_ROCKCHIP_DW_HOST` (:497), `PHY_ROCKCHIP_INNO_USB2` (:522), `USB_DWC3` (:734), `ROCKCHIP_SARADC` (:594), `ROCKCHIP_THERMAL` (:595), `NVMEM_ROCKCHIP_OTP` (:456), `BLK_DEV_NVME` (:117) and `NET_DSA` (:435). The DSA Realtek symbol belongs to #3.

USB note: the single combphy goes to PCIe, so the dwc3 runs USB 2.0 only (`maximum-speed = "high-speed"`, `phys = <&usb2phy_otg>`), the same as the E20C (`rk3528-radxa-e20c.dts:364-371`). The OTG/peripheral role is the map's open "USB-C OTG port role" item.

## 5. Backport list for a 6.18 E24C (excluding the switch)

| # | Commit | Status | Suggested OpenWrt name | Needed? |
|---|---|---|---|---|
| 1 | *arm64: dts: rockchip: Add SFC node for RK3528*, Jonas Karlman (v1 `20250727144409.327740-2-jonas@kwiboo.se`; reordered `Kwiboo/linux-rockchip@008276e5e201`) | **pending**: posted v1 2025-07-27, no v2, not in v7.3 or `for-next` | `7xx-` pending | **yes**: Linux access to the SPI NOR |
| 2 | *dt-bindings: arm: rockchip: Add Radxa E24C* (`…-3-jonas@kwiboo.se`, Acked-by Krzysztof Kozlowski) | **pending** | `7xx-` pending (or skipped; it is binding only) | only for upstream shape |
| 3 | *arm64: dts: rockchip: Add Radxa E24C* (v1 `…-4-jonas@kwiboo.se`; current `Kwiboo/linux-rockchip@132d82695f31`) | **pending**: v1 is stale (u2phy labels, no PCIe); the branch version is not posted | `7xx-` pending, based on `132d82695f31` | **yes** |
| 4 | *arm64: dts: rockchip: Enable USB ports on Radxa E24C* (`Kwiboo/linux-rockchip@1ea810e218c5`) | **pending** (unposted; folded into v1's DTS) | merge into #3 or a separate `7xx-` | yes |
| 5 | `29d71b732bdc` dt-bindings: watchdog: Add watchdog compatible for RK3528 | **mainline v7.2** | `0xx-v7.2-` | optional |
| 6 | `eb24b60553e0` arm64: dts: rockchip: Add watchdog node for RK3528 | **mainline v7.2** | `0xx-v7.2-` | optional |
| – | Switch DTS `Kwiboo/linux-rockchip@6bf983983dfb` and the realtek DSA driver changes | – | – | for #3/#8 |

Already carried, no action: `070-26` (v6.19), `071-14` (v6.19, E20C PCIe, the pattern to copy), `072-3` + `038-1..4` (v7.1), `073-1` + `039-1..3` (v7.2), `161-01..03` (out-of-tree TSADC), and `104`.
Already in 6.18.55, no action: dwmac-rk RK3528 (v6.15), GMAC nodes (v6.16), SARADC (v6.15), combphy (v6.18), CPU OPPs (v6.18), SFC driver fixes, and the combphy L1ss fix (v6.19 → 6.18.y).

## 6. Decision-relevant surprises

1. **There is no "mainline E24C DTS" to backport.** The map's prior art is a July 2025 v1 that never went past v1. The board DTS will be a `7xx-` pending patch, and its best base is the author's unposted `next-20260430-rk3528` branch, not the patchwork v1.
2. **Linux cannot see the SPI NOR without a pending patch.** The SFC DT node is in neither mainline nor `for-next`. This matters for any OpenWrt-side NOR access, such as reading the U-Boot env with `uboot-envtools`, updating the SPI bootloader from Linux (#10), or the future full-NOR image. It does not matter for booting.
3. **v1 DTS has no NVMe.** It is missing `&combphy`/`&pcie`. The branch version adds them with the E20C's `pciem1_pins` and `gpio1 RK_PA2` PERST#. #7 should confirm this against the schematic.
4. **GMAC1/RGMII is not untested.** The E20C already runs `gmac1` in RGMII in this tree. The risk on the E24C is the fixed-link to the switch, and the author's DTS **disables TSO** on gmac1 (`/delete-property/ snps,tso`) and warns that RX checksum offload may also need to go. Hand this to #8.
5. **One DTB cannot serve both board variants.** FSPI and eMMC share pins, so the E24C DTS marks `sdhci` `"fail"`. A microSD image built from it will boot on an eMMC-variant board, but it will not see the eMMC. Radxa says eMMC variants exist.
6. **TSADC support is out-of-tree and has never been posted.** It is carried as `161-0x` rather than under the `7xx-` convention. The map's "CPU OPPs and thermal" item depends on it staying in the tree.
7. **RK805 as CPU regulator is new for RK3528 in this tree.** Every other RK3528 board uses a PWM regulator. RK805 itself is mature (RK3328).

## 7. Open question surfaced

- **SPI vs eMMC variant in one image.** With `sdhci` set to `"fail"` and `sfc` set to `"okay"`, the E24C DTB describes only the SPI variant. Should the OpenWrt device stay SPI-only, as the map already scopes it? Or should something make eMMC usable on eMMC-variant boards, such as a U-Boot DT fixup, a second DTB/device, or an overlay? No current ticket (#2-#12) owns this.
