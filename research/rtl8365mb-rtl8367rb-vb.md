# Can mainline `rtl8365mb` drive the E24C's RTL8367RB-VB switch on 6.18?

Research for [issue #3](https://github.com/EnzoAlbornoz/openwrt/issues/3), part of the map in [issue #1](https://github.com/EnzoAlbornoz/openwrt/issues/1). Terms follow `GLOSSARY.md` (**switch** = the on-board RTL8367RB-VB).
Sources were read on 2026-10-20. Mainline HEAD was v7.3, and net-next is heading for v7.4.

## Answer

**Yes. A stock 6.18 `rtl8365mb` can run the switch, and it needs no driver patch.**

- The chip has been recognised since **v5.18**.
- The board DTS has to describe the switch **with an `mdio` child node**. The driver needs that node. The upstream dt-schema forbids it, but OpenWrt never runs dt-schema.

Upstream, two things are missing, and neither blocks OpenWrt:

1. **No E24C DTS in mainline.** It isn't in v7.3, linux-next or net-next.
2. **The driver and the binding disagree about the `mdio` node.** Jonas Karlman's follow-up series was reviewed and marked changes-requested in July 2025, and **no v2 was ever posted**.

Three runtime issues need workarounds in the board DTS or config:

- **stmmac TSO and RX checksum offload break traffic** through the switch's DSA conduit (GMAC1). Every working downstream DTS deletes `snps,tso`.
- **`reset-gpios` on the switch node can make probe fail with `-EBUSY`, but only with `CONFIG_RESET_GPIO=y`.** OpenWrt leaves that option unset.
- **The WAN port number is disputed downstream.** Some trees say port 0, others say port 3.

## 1. Chip ID support

- **The v6.18 driver lists the chip.** The `rtl8365mb_chip_infos[]` table has an `RTL8367RB-VB` entry: `chip_id = 0x6367`, `chip_ver = 0x0020`, with external interfaces on port 6 (EXT1) and port 7 (EXT2), each MII/TMII/RMII/RGMII. See [rtl8365mb.c#n547 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/dsa/realtek/rtl8365mb.c?h=v6.18#n547).
- **Added in v5.18.** The commit is [`84a10aecdcc0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=84a10aecdcc013b390547d0b84c75eefb84bf828) "net: dsa: realtek: rtl8365mb: add RTL8367RB-VB support" (Luiz Angelo Daros de Luca, Jan 2022). It is in v5.18 and not in v5.17: the GitHub compare API returns `behind` for v5.18 and `diverged` for v5.17. It came in the same series that added the MDIO management interface ([patchwork series cover](https://patchwork.kernel.org/project/netdevbpf/patch/20220128060509.13800-1-luizluca@gmail.com/)).
  - A later commit, [`5eb1a2384066`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5eb1a2384066ea11b49065ababd2277f9de7adde) in v6.0, only renamed the macro.
- **Real hardware detects it.** Jonas Karlman's boot log on an E24C shows `rtl8365mb-mdio stmmac-0:1d: found an RTL8367RB-VB switch` ([series cover letter](https://patchwork.kernel.org/project/netdevbpf/cover/20250727180305.381483-1-jonas@kwiboo.se/)).
- **The internal PHYs bind to the Realtek PHY driver.** The same log shows the user ports' PHYs bound to `RTL8365MB-VC Gigabit Ethernet`, from `drivers/net/phy/realtek`. In this tree, `kmod-dsa-realtek` depends on `kmod-phy-realtek` ([netdevices.mk:915](../package/kernel/linux/modules/netdevices.mk)), and rockchip/armv8 builds `CONFIG_REALTEK_PHY=y` (`target/linux/rockchip/armv8/config-6.18:568`).
- **The Radxa BSP uses the same driver.** Its 6.1 kernel has the same chip table ([radxa/kernel `rtl8365mb.c` @ linux-6.1-stan-rkr5.1](https://github.com/radxa/kernel/blob/bd336601f1572b7bbd7c2fa5a9e07b684a4dc084/drivers/net/dsa/realtek/rtl8365mb.c#L543-L545)).

## 2. How the switch is managed and connected

**Management is MDIO on GMAC1's `mdio1` bus, not GPIO-bitbanged SMI.**

- **Schematic (V1.2, page 19 "RTL8367RB_VB_1000M").** The switch's management pins carry nets `GMAC1_MDC` and `GMAC1_MDIO`. Its RGMII EXT1 group (`RG1_*`) carries `GMAC1_TXD0-3/TXEN/TXCLK` and `GMAC1_RXD0-3/RXDV/RXCLK`. Other nets: reset `GMAC1_RSTn_L`, interrupt `RTL8367RB_EINT` on chip pin `GPIO57/INTERRUPT`, and supply enable `GPIO_8367_EN`. Strap pins `SMI_SEL/P0LED0` and `MID29/P1LED0` select MDIO mode and the PHY address. ([schematic PDF](https://dl.radxa.com/e/e24c/docs/radxa_e24c_v1200_schematic.pdf); text taken with pypdf. Net-to-pin connectivity was not traced visually.)
- **MDIO address is 29 (0x1d).** The Radxa BSP (`switch0@29`, `reg = <29 0>`) and Jonas's DTS (`ethernet-switch@1d`) agree, and Jonas's log shows the device at `stmmac-0:1d`.
- **CPU port is switch port 6 (EXT1) to GMAC1, fixed-link 1000/full.** Every source agrees: Radxa BSP, Jonas, Armbian and Lede.
- **RGMII delays: the switch adds 2 ns in both directions, and the MAC adds none.**
  - Jonas: `&gmac1 { phy-mode = "rgmii-id"; }` with no MAC delays, and switch port 6 `phy-mode = "rgmii-id"` with `rx/tx-internal-delay-ps = <2000>` ([patch 3/3](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-4-jonas@kwiboo.se/)).
  - Radxa BSP: same result, written as `&gmac1 { phy-mode = "rgmii"; tx_delay = <0>; rx_delay = <0>; }` with switch port 6 at `tx/rx-internal-delay-ps = <2000>` ([BSP DTS L366-L474](https://github.com/radxa/kernel/blob/bd336601f1572b7bbd7c2fa5a9e07b684a4dc084/arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts#L366-L474)).
  - In the v6.18 driver, TX delay is 0 or 2 ns, and RX delay is 0–7 steps of about 0.3 ns. Both 2000 ps and Armbian's 2100 ps round to step 7 ([rtl8365mb.c#n868 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/dsa/realtek/rtl8365mb.c?h=v6.18#n868)).
  - Russell King's review notes that writing `rgmii-id` at both ends of a MAC-to-MAC link is semantically questionable ([comment](https://patchwork.kernel.org/comment/26489655/)). Andrew Lunn, on where the 2 ns goes: "It should not matter. 2ns is 2ns" ([comment](https://patchwork.kernel.org/comment/26489643/)).
- **Supply.** GPIO1_C3 (`gpio_8367_en`) switches one regulator that feeds both AVDDL 1.1 V and AVDDH 3.3 V. Jonas attaches it as `&gmac1 { phy-supply }`, and the BSP marks it `regulator-always-on` ([Jonas's reply](https://patchwork.kernel.org/comment/26490794/)).
- **Reset.** GPIO4_C2, active low. Jonas puts it on the `&mdio1` bus node (`reset-gpios`, 25 ms / 100 ms) to avoid the `-EBUSY` problem in §4.2. The BSP uses the stmmac `snps,reset-gpio`.
- **Interrupt.** GPIO1_C2 (`rtl8367rb_eint`), level-low.
- **Mainline SoC side in v6.18.** `rk3528.dtsi` has `gmac1: ethernet@ffbe0000` with `snps,tso` and an `mdio1: mdio` child ([rk3528.dtsi#n1004 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/boot/dts/rockchip/rk3528.dtsi?h=v6.18#n1004)), and `dwmac-rk.c` has RK3528 support ([dwmac-rk.c#n865 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/ethernet/stmicro/stmmac/dwmac-rk.c?h=v6.18#n865)).

## 3. The "dt-binding and/or driver changes", and their status

The base series [*arm64: dts: rockchip: Add Radxa E24C*](https://patchwork.kernel.org/series/986207/) (Jonas Karlman, 2025-07-27, v1) left the switch out because "support for the switch requires dt-binding and/or driver changes". In review he described the conflict as "the dt-bindings prevents describing a mdio child node, yet the driver requires it" ([comment](https://patchwork.kernel.org/comment/26489575/)).

- **Binding (v6.18, unchanged in v7.3 and net-next).** If `reg` is present (an MDIO-connected switch), then `mdio: false` ([realtek.yaml#n111 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/net/dsa/realtek.yaml?h=v6.18#n111)).
- **Driver (v6.18, unchanged in v7.3 and net-next).** `rtl83xx_setup_user_mdio()` returns `-ENODEV` with "no MDIO bus node" when the `mdio` child is missing ([rtl83xx.c#n79 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/dsa/realtek/rtl83xx.c?h=v6.18#n79)).
  - The requirement came in with [`bba140a566ed`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bba140a566ed075304c49c52ab32c0016cab624a) "net: dsa: realtek: use the same mii bus driver for both interfaces" (v6.9). Before that, only the SMI interface needed the node ([Chukun Pan](https://patchwork.kernel.org/comment/26491759/)).

Follow-up series [*net: dsa: realtek: Add support for use of an optional mdio node*](https://patchwork.kernel.org/series/986229/) (net-next, 2025-07-27, v1):

| Patch | Patchwork state |
|---|---|
| [1/3 net: dsa: realtek: remove unused user_mii_bus from realtek_priv](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-2-jonas@kwiboo.se/) | changes-requested (series) |
| [2/3 net: dsa: realtek: Add support for use of an optional mdio node](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-3-jonas@kwiboo.se/) | changes-requested |
| [3/3 arm64: dts: rockchip: Add RTL8367RB-VB switch to Radxa E24C](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-4-jonas@kwiboo.se/) | changes-requested |

How the review went:

- **Andrew Lunn asked for a binding change, not a driver change.** "I think you should allow an MDIO node in DT. This is the only DSA driver that i know of which does not allow it" ([comment](https://patchwork.kernel.org/comment/26489691/)).
- **Jonas planned a v2:** a new dt-bindings patch to allow `mdio` when `reg` is present, a `mdio` node in the E24C switch node, PHY interrupts described through the switch's interrupt controller, and a Fixes tag ([comment](https://patchwork.kernel.org/comment/26492434/)).
- **No v2 exists.** Patchwork searches for "Radxa E24C", "RTL8367RB", "optional mdio node" and "dt-bindings: net: dsa: realtek" return only the v1 patches and unrelated work. The E24C DTS is in neither mainline v7.3 nor linux-next: `rk3528-radxa-e24c.dts` returns 404, and the v7.3 rockchip `Makefile` has no e24c entry.
- **Other netdev work on the same driver:**
  - Oleksij Rempel's supplies series (`avddh-supply`, `dvddio-supply`, …) was accepted into net-next on 2026-08-31 and targets v7.4, so it is not in v6.18:
    - [`f4d025418721`](https://patchwork.kernel.org/project/netdevbpf/patch/20260831123637.666457-2-o.rempel@pengutronix.de/) dt-bindings
    - [`f82364499682`](https://patchwork.kernel.org/project/netdevbpf/patch/20260831123637.666457-3-o.rempel@pengutronix.de/) rtl83xx
  - Mieczyslaw Nalewaj's RTL8367S-VB "family D" series ([v6, 2026-10-16](https://patchwork.kernel.org/project/netdevbpf/patch/3cb754f2-dc21-4de3-b245-b970719925fd@yahoo.com/)) is for a different chip and does not touch RB-VB.

**What this means for OpenWrt:** OpenWrt compiles DTS with dtc only and never runs dt-schema. A DTS with an `mdio` child under the switch node therefore probes on an unpatched 6.18. Jonas confirmed this: "With a mdio node the driver is happy and dtschema is sad" ([comment](https://patchwork.kernel.org/comment/26490815/)). The Armbian 6.18/7.3, Lede and Radxa BSP DTS all use an `mdio` child.

## 4. Runtime issues found in review and downstream

### 4.1 stmmac TSO and RX checksum offload on the DSA conduit

- **The symptom.** With `snps,tso`, iperf3 ran at about 930–940 Mbit/s in one direction and **about 1–2 Mbit/s in the other** ([patch 3/3 notes](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-4-jonas@kwiboo.se/)).
- **Disabling both offloads fixed it.** Jonas: "disable of TSO and RX checksum offload on the conduit interface help fix any TCP speed issue and reduced UDP packet loss to a minimum" ([comment](https://patchwork.kernel.org/comment/26490518/)).
- **Andrew Lunn's explanation** is that the stmmac TSO path likely doesn't replicate the DSA header on every segment ([comment](https://patchwork.kernel.org/comment/26489643/)).
- **Chukun Pan bisected it** to [`041cc86b3653`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=041cc86b3653cbcdf6ab96c2f2ae34f3d0a99b0a) "net: stmmac: Enable TSO on VLANs" (v6.11). Kernels 6.1 and 6.6 were fine ([comment](https://patchwork.kernel.org/comment/26504957/)). Andrew and Vladimir Oltean questioned that reading, and the thread ends there unresolved ([comment](https://patchwork.kernel.org/comment/26505017/)).
- **There is no upstream fix.** Patchwork shows stmmac TSO fixes through Sep 2026, but none for this DSA case.
- **What downstreams do.** Andrew asked that the workaround stay in the DTS ([comment](https://patchwork.kernel.org/comment/26492468/)). The Armbian 6.18/7.3 and Lede DTS both carry `/delete-property/ snps,tso;` in `&gmac1`: [Armbian L204](https://github.com/armbian/build/blob/c2d4d22b685c6f340c074ae8c0f189804d4a4820/patch/kernel/archive/rockchip64-6.18/dt/rk3528-radxa-e24c.dts#L204) and [Lede L178](https://github.com/coolsnowwolf/lede/blob/07cebbcf28f61637047645c3fedc829ee9cc3688/target/linux/rockchip/files/arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts#L178). No DT property disables RX checksum offload, so that half would need `ethtool -K eth0 rx off` or a driver patch.
- **Possibly related, pending.** On some stmmac conduits, a hardware-accelerated VLAN tag gets inserted after the DSA CPU tag.
  - Amitesh Singh's [tag_rtl8_4 fix](https://patchwork.kernel.org/project/netdevbpf/patch/20260908040041.124640-1-singh.amitesh@gmail.com/) got changes-requested.
  - It was reworked as a DSA-core fix, [*net: dsa: push hwaccel VLAN tag into payload in dsa_user_xmit()*](https://patchwork.kernel.org/project/netdevbpf/patch/20260930033239.57093-1-singh.amitesh@gmail.com/) (v2, changes-requested).
  - It matters only for VLAN subinterfaces on user ports. Not verified on RK3528.

### 4.2 `reset-gpios` on the switch node can fail with `-EBUSY`

- **The failure.** `rtl83xx_probe()` calls `devm_reset_control_get_optional()` and then `devm_gpiod_get_optional(dev, "reset")` ([rtl83xx.c#n187 @ v6.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/dsa/realtek/rtl83xx.c?h=v6.18#n187)). With `CONFIG_RESET_GPIO`, the reset framework claims `reset-gpios` itself, so the second probe after a deferral returns `-EBUSY`.
  - The cause is the pairing of [`c721f189e89c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c721f189e89c0d4db119d7bb2b46768d0fb5f6b1) (v6.9) and [`56998aa6b7f0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=56998aa6b7f0f31ce8df23c00701af2d8e8a1f1a) (v6.9).
  - Sources: Jonas's [reply](https://patchwork.kernel.org/comment/26490794/), Chukun's [analysis](https://patchwork.kernel.org/comment/26491886/), and the Armbian patch commit message.
- **Unchanged upstream.** The code is the same in v7.3 and net-next.
- **A fix exists only in Armbian.** Jonas wrote "net: dsa: realtek: fix reset-gpios handling" and Armbian carries it, but it was never posted (patchwork has no match). It sits in [Armbian `rk3528-net-dsa-realtek-fixes-for-radxa-e24c-switch-chip.patch`](https://github.com/armbian/build/blob/c2d4d22b685c6f340c074ae8c0f189804d4a4820/patch/kernel/archive/rockchip64-6.18/rk3528-net-dsa-realtek-fixes-for-radxa-e24c-switch-chip.patch) alongside Jonas's two unmerged rtl83xx patches and two **WIP** dwmac-rk patches. In the 7.3 copy, the dwmac-rk WIP patches are dropped.
- **Not a problem in OpenWrt.** `CONFIG_RESET_GPIO` is unset in `target/linux/generic/config-6.18:5560` and not set in `target/linux/rockchip/armv8/config-6.18`. Nothing under `drivers/` in 6.18.55 `select`s it (`sound/` was not checked). So `reset-gpios` on the switch node works without the fix. Jonas's alternative, putting the reset on the `&mdio1` bus, also avoids it.

## 5. What Radxa and other downstreams use

| Source | Kernel | Switch driver | WAN port | Notes |
|---|---|---|---|---|
| Radxa's official OpenWrt image `openwrt_rk3528_e24c_R25.05.07_k6.1.115-rk35xx-flippy-2505a` ([Radxa docs download.md L17-L19](https://github.com/radxa-docs/docs/blob/b63bfd07c1b41f5a57d900f3235e699eae0397d8/i18n/en/docusaurus-plugin-content-docs/current/e/e24c/download.md?plain=1#L17-L19)) | 6.1.115-rk35xx (unifreq/flippy) | **swconfig `realtek,rtl8367b`**, `mii-bus = <&mdio1>`, `phy-id = <29>`, extif 6 ([unifreq DTS L246-L252](https://github.com/unifreq/linux-6.1.y-rockchip/blob/4d8ee3d3745fd381d79294d35b670817e123afdb/arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts#L246-L252)) | port 3 (`"0:lan" "1:lan" "2:lan" "3:wan" "6t@eth0"`, [packit 99-default_network L65](https://github.com/unifreq/openwrt_packit/blob/a46b4f5d201ab99eba0ddc7c2e7cd67412fc3d5c/files/rk3528/e24c/board.d/99-default_network#L65)) | Same case as E54C. The unifreq DTS comment calls port 0 "wan" (L699). |
| iStoreOS (also listed by Radxa) | 6.1 BSP | **swconfig `realtek,rtl8367b`**. DSA `switch0@29` present but `status = "disabled"` ([istoreos DTS L109-L120, L234-L236](https://github.com/istoreos/istoreos/blob/45f8da64fc38cf31096659f5e19f3e43a92cc9bd/target/linux/rockchip/dts/rk3528/rk3528-e24c.dts#L109-L120)) | port 0 (`"0:wan" "1:lan" "2:lan" "3:lan"`, [02_network L75-L77](https://github.com/istoreos/istoreos/blob/45f8da64fc38cf31096659f5e19f3e43a92cc9bd/target/linux/rockchip/armv8/base-files/etc/board.d/02_network#L75-L77)) | |
| Radxa BSP kernel | 6.1 (`linux-6.1-stan-rkr5.1`) | DSA `rtl8365mb` with `mdio` child, no IRQ, `pause` on fixed-links ([BSP DTS L366-L446](https://github.com/radxa/kernel/blob/bd336601f1572b7bbd7c2fa5a9e07b684a4dc084/arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts#L366-L446)) | port 3 (labels port0=lan3 … port3=wan) | The Radxa 6.12 and 6.18 branches have no E24C DTS. |
| Jonas Karlman (lore v1) | next-20250722 | DSA `rtl8365mb`, IRQ, no `mdio` child (needs his patch) | port 0 | Deletes `snps,tso`. |
| Armbian `rockchip64-6.18` / `-7.3` | 6.18 / 7.3 | DSA `rtl8365mb`, `mdio` child + IRQ, `reset-gpios` on the switch ([DTS L374-L458](https://github.com/armbian/build/blob/c2d4d22b685c6f340c074ae8c0f189804d4a4820/patch/kernel/archive/rockchip64-6.18/dt/rk3528-radxa-e24c.dts#L374-L458)) | port 0 | Deletes `snps,tso`. Carries Jonas's unposted reset fix and optional-mdio patch. |
| coolsnowwolf/lede | Lede kernel | DSA `rtl8365mb`, `mdio` child + IRQ, `reset-gpios` on the switch ([DTS L374](https://github.com/coolsnowwolf/lede/blob/07cebbcf28f61637047645c3fedc829ee9cc3688/target/linux/rockchip/files/arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts#L374)) | port 0 | Deletes `snps,tso`. |

**Summary of the table.** Radxa ships swconfig `rtl8367b` in its OpenWrt image. Its own BSP kernel already describes the switch for the mainline DSA driver. Every recent community port uses DSA `rtl8365mb`. The map's standing decision (DSA only, never swconfig) is consistent with this.

## 6. Commits a 6.18 kernel needs for the switch

Minimum for a working DSA switch on 6.18: **none.** The driver, tagger (`tag_rtl8_4`), MDIO interface and RTL8367RB-VB ID are all mainline before 6.18:

| Commit | Subject | Status |
|---|---|---|
| [`84a10aecdcc0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=84a10aecdcc013b390547d0b84c75eefb84bf828) | net: dsa: realtek: rtl8365mb: add RTL8367RB-VB support | mainline **v5.18** |
| [`5eb1a2384066`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5eb1a2384066ea11b49065ababd2277f9de7adde) | net: dsa: realtek: rtl8365mb: rename macro RTL8367RB -> RTL8367RB_VB | mainline v6.0 |
| [`bba140a566ed`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bba140a566ed075304c49c52ab32c0016cab624a) | net: dsa: realtek: use the same mii bus driver for both interfaces (makes `mdio` child mandatory) | mainline v6.9 |

**Already carried in this tree** (`target/linux/generic/backport-6.18/`). These add VLAN, FDB, bridge offload and related cleanups. They are not needed to probe, but they give real bridge offload instead of software bridging. Mainline release checked with the GitHub compare API against torvalds/linux tags:

| Patch file | Upstream commit | Status |
|---|---|---|
| `940-01-v7.1-net-dsa-tag_rtl8_4-update-format-description.patch` | `297e1f411ed4` | mainline v7.1 |
| `940-02-v7.1-net-dsa-tag_rtl8_4-set-KEEP-flag.patch` | `82f37bd9a4d7` | mainline v7.1 |
| `942-01`…`942-09-v7.2-…` (ERR_PTR, reject unsupported topologies, DSA port helpers, multi-file split, table lookup, **VLAN**, **FDB**, **bridge join/leave**, **bridge port flags**) | `a543687227d8`, `36c572fd60d6`, `51a4a22301b0`, `0e692c27fe84`, `fbafdd3b224a`, `9da2c8672f77`, `336e3e4a1ab3`, `183bd68b1fe1`, `660a9e399ab0` | mainline v7.2 |
| `943-01/02-v7.3-…` SGMII/HSGMII for RTL8367S | `0b577e2fe06c`, `987137345f33` | mainline v7.3 (irrelevant to RB-VB, which has no SerDes) |
| `944-01…04-v7.2-…` devm_mutex_init cleanups | `70fd0cf29bc4`, `050e07f8765d`, `a95f3e9b8985`, `442ecdc83d00` | mainline v7.2 |
| `946-v7.4-…` define masks using BIT()/GENMASK() | `9b84b36a527c` | **not in v7.3** (net-next; the file name says v7.4) |

None of these change the `mdio`-node requirement or the reset handling. `942-02` ("reject unsupported topologies") requires a CPU port and rejects DSA links. The E24C, with one CPU port (6) and no cascade, is fine.

**Optional or pending, only if wanted:**

| Item | Status | Needed for OpenWrt 6.18? |
|---|---|---|
| Jonas: [remove unused user_mii_bus](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-2-jonas@kwiboo.se/) | pending (changes-requested, abandoned) | no |
| Jonas: [optional mdio node](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-3-jonas@kwiboo.se/) | pending (changes-requested; maintainers prefer a binding change) | no, if the DTS has an `mdio` child |
| dt-binding: allow `mdio` when `reg` is present | **never posted** (Jonas's plan, Andrew's ask) | no (OpenWrt doesn't run dt-schema); yes for an upstream DTS |
| Jonas: "net: dsa: realtek: fix reset-gpios handling" | **never posted**; in Armbian only | no, while `CONFIG_RESET_GPIO` is unset or the reset sits on `&mdio1` |
| Oleksij Rempel: realtek supplies binding + rtl83xx regulator support | accepted to net-next 2026-08-31 (`f4d025418721`, `f82364499682`), targets v7.4 | no (regulator can stay as `&gmac1` `phy-supply`) |
| stmmac TSO and RX-csum fix for DSA conduits | **no fix exists upstream** | worked around with `/delete-property/ snps,tso;` plus RX-csum off |
| DSA core: push hwaccel VLAN tag before CPU tag ([v2](https://patchwork.kernel.org/project/netdevbpf/patch/20260930033239.57093-1-singh.amitesh@gmail.com/)) | pending (changes-requested) | only if VLANs on user ports show leaked 0x8899 tags |
| E24C board DTS ([base series](https://patchwork.kernel.org/series/986207/), [switch patch](https://patchwork.kernel.org/project/netdevbpf/patch/20250727180305.381483-4-jonas@kwiboo.se/)) | pending v1 (2025-07); no v2; not in v7.3 or linux-next | yes, carried as an OpenWrt DTS |

## 7. Surprises and open points

1. **Downstreams disagree on which port is WAN.**
   - Port 0 = WAN: Jonas, Armbian, Lede, iStoreOS swconfig, and the unifreq DTS comment.
   - Port 3 = WAN: Radxa BSP DSA labels and flippy's `99-default_network`, which is shared with the E54C.
   - Schematic page 20 text lists `RJ45-0..3` with `WAN LAN1 LAN2 LAN3` and the `P0..P3` MDI nets in the same order. That suggests port 0 = WAN, but it was not traced visually. This belongs to issue #7 and needs checking on the board.
2. **The mainline E24C DTS stalled at v1 in July 2025, and so did the switch series.** Neither has a v2. So the OpenWrt port carries its own DTS, with no `0xx-vX.Y` backport available for the board.
3. **There is no upstream fix for the TSO/RX-csum problem.** Expect to carry `/delete-property/ snps,tso;` plus an RX-checksum-off step (for example `ethtool -K eth0 rx off` in board init), or a stmmac patch. The RX-csum part is decision-relevant for issue #8.
4. **Radxa's official OpenWrt image uses swconfig `rtl8367b` on a 6.1 BSP kernel.** That is the path the map rules out, so it is not a usable reference for the DSA port beyond wiring facts.
5. **There is a newer upstream supplies binding** (`avddh-supply` etc., v7.4). An upstream-quality DTS might later move the GPIO1_C3 regulator from `&gmac1 phy-supply` to the switch node.
