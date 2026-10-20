# Radxa E24C: ports, MAC addresses, LEDs and buttons

Research for issue #7 (map #1). It answers: how the RJ45 jacks map to **switch** ports, where the MAC addresses come from, how each LED and button is wired, and which key codes OpenWrt should use.

Terms follow `GLOSSARY.md`. "Switch" always means the on-board RTL8367RB-VB.

## Sources

Square-bracket keys are used throughout the document.

| Key | Source |
| --- | --- |
| [SCH] | Radxa E24C schematic V1.20, dated 2025-05-19, 22 pages. https://dl.radxa.com/e/e24c/docs/radxa_e24c_v1200_schematic.pdf (sha256 `66ff50b5…09ac8d`). Cited as `SCH p<page>`, with net or reference designators. |
| [ML-BASE] | Jonas Karlman, *[PATCH 3/3] arm64: dts: rockchip: Add Radxa E24C*, 2025-07-27. Quoted in full in https://lists.infradead.org/pipermail/linux-rockchip/2025-August/061247.html. Cover letter: https://lkml.iu.edu/hypermail/linux/kernel/2507.3/03612.html |
| [ML-SW] | Jonas Karlman, *[PATCH 3/3] arm64: dts: rockchip: Add RTL8367RB-VB switch to Radxa E24C*, 2025-07-27. https://lists.infradead.org/pipermail/linux-rockchip/2025-July/060782.html. Review: https://lists.infradead.org/pipermail/linux-rockchip/2025-July/060821.html, https://lkml.iu.edu/hypermail/linux/kernel/2507.3/04653.html |
| [BSP] | Radxa vendor kernel, `radxa/kernel` branch `linux-6.1-stan-rkr5.1` @ `bd336601f157`, `arch/arm64/boot/dts/rockchip/rk3528-radxa-e24c.dts` and `rk3528-radxa-e24c-spi.dts` |
| [ISTORE] | iStoreOS (an OpenWrt derivative that Radxa promotes for the E-series) @ `45f8da64fc38`: `target/linux/rockchip/dts/rk3528/rk3528-e24c.dts` and `target/linux/rockchip/armv8/base-files/etc/board.d/02_network` |
| [ARMB] | Armbian `armbian/build` @ `c2d4d22b685c`, `patch/kernel/archive/rockchip64-7.3/dt/rk3528-radxa-e24c.dts` (mainline-based). Its EEPROM MAC change: commit `b34a1f27cd95`, PR https://github.com/armbian/build/pull/10817 |
| [ARMB-V] | Armbian vendor kernel `armbian/linux-rockchip`, commits `6b54ea8c59c9` ("fix Radxa E24C switch port labels"), `c5ab76fbdfb7` ("read Radxa E24C MAC addresses from EEPROM"), `7554973e9df0` ("fix … port LED triggers") and `428ab2713a61` ("point … ethernet0 at gmac1") |
| [DOCS] | Radxa docs, `radxa-docs/docs` @ `b63bfd07c1b4`: `i18n/en/.../e/e24c/hardware-usage/hardware-interface.md`, `getting-started/interface-usage.md` and the image `static/img/e/e24c/e24c-shell-interface.webp`. Rendered at https://docs.radxa.com/en/e/e24c |
| [LINUX] | torvalds/linux master @ `c03c1a14507d` |
| [UBOOT] | u-boot/u-boot master @ `ece349ade297` |

There is no `rk3528-radxa-e24c.dts` in torvalds/linux master. Neither the base series nor the switch series has been merged ([LINUX], directory listing of `arch/arm64/boot/dts/rockchip/`). Upstream OpenWrt has no E24C device either (`openwrt/openwrt` main, `target/linux/rockchip/image/armv8.mk`).

## 1. Ports

### Topology

- The SoC's **GMAC1** connects over RGMII to the switch's **RG1 / extension interface 1**. The `RG1_*` pins pass through 22 Ω series resistors to the `GMAC1_*` nets (SCH p19, R241–R259). The switch's RG2 pins are unconnected (SCH p19, pins 41–52 marked NC). The block diagram labels this link "RGMII1" (SCH p3).
- In mainline `rtl8365mb`, extension interface 1 on the RTL8367RB-VB (chip id `0x6367`, version `0x0020`) is **switch port 6** ([LINUX] `drivers/net/dsa/realtek/rtl8365mb_main.c:753-761`, `extints = { { 6, 1, … RGMII }, { 7, 2, … } }`). All vendor and mainline device trees put the CPU port at `reg = <6>` ([ML-SW], [BSP] l.408, [ARMB]).
- The switch is managed over **MDIO on GMAC1's MDIO bus** (`GMAC1_MDC`/`GMAC1_MDIO` = GPIO3_B6/B7, SCH p13). The lines reach the switch's SCK/SDA pins (93/94) through 0 Ω resistors R245 and R251. The alternative path from the switch to `I2C_SCL`/`I2C_SDA` goes through R247 and R8, which are not populated (NC) (SCH p19). Every device tree puts the switch at MDIO address **29 (0x1d)** ([ML-SW] `ethernet-switch@1d`, [BSP] `switch0@29`, [ISTORE] `phy-id = <29>`).
- The SoC's embedded 10/100 PHY (GMAC0) is not used for any port (SCH p3 shows all four RJ45 ports behind the switch; [ML-BASE] enables only `gmac1`).

### Port map

| Switch port | MDI nets (SCH p19/p20) | Magnetics / jack (SCH p20) | Schematic label (SCH p20) | Board silkscreen | Case label (rear, left→right after 12V, HDMI) | In-jack LEDs |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | P0MDI[A-D][PN] | J6702 / J6703 "RJ45-0" | **WAN** | RJ45_0 | **WAN** (1st) | LED0_G/LED0_Y |
| 1 | P1MDI… | J6704 / J6705 "RJ45-1" | **LAN1** | RJ45_1 | **LAN1** (2nd) | LED1_G/LED1_Y |
| 2 | P2MDI… | J6706 / J6707 "RJ45-2" | **LAN2** | RJ45_2 | **LAN2** (3rd) | LED2_G/LED2_Y |
| 3 | P3MDI… | J6708 / J6709 "RJ45-3" | **LAN3** | RJ45_3 | **LAN3** (4th) | LED3_G/LED3_Y |
| 4 | P4MDI… | not connected | — | — | — | — |
| 6 | RG1 | to GMAC1 | CPU | — | — | — |

Evidence for each column:

- **Schematic.** SCH p20 draws the four jacks with large WAN / LAN1 / LAN2 / LAN3 labels next to RJ45-0…RJ45-3. Each jack's magnetics input is the matching `P<n>MDI*` net.
- **Silkscreen and case.** The bare-board photo in [DOCS] (`e24c-interface.webp`) shows silkscreen RJ45_3, RJ45_2, RJ45_1, RJ45_0 from left to right when viewed from the component side, with RJ45_0 next to the HDMI and DC jacks. The enclosed-version photo in [DOCS] (`e24c-shell-interface.webp`) shows the rear panel as 12V, HDMI, **WAN, LAN1, LAN2, LAN3**. Seen from outside, RJ45_0 sits next to HDMI, so the two photos agree.
- **Prior art that agrees.** [ML-SW] (Jonas Karlman, tested on hardware: "iperf3 … ~930-940 Mbits/sec") labels `port@0 = wan`, `1..3 = lan1..lan3`. [ISTORE] `02_network:75-78` configures swconfig with `"0:wan" "1:lan" "2:lan" "3:lan" "6t@eth0"`. [ARMB-V] commit `6b54ea8c59c9` corrected the vendor labels to the same mapping: "Port 0 is the WAN port and ports 1 to 3 are LAN1 to LAN3".
- **Prior art that disagrees: the Radxa BSP is wrong.** [BSP] l.381-403 labels port 0 `lan3`, port 1 `lan2`, port 2 `lan1` and port 3 `wan`. That is exactly the E54C mapping (`radxa,e54c` in [ISTORE] `02_network:79-81`: `"3:wan:0" "2:lan:1" "1:lan:2" "0:lan:3"`), which suggests the E24C DTS was copied from the E54C. Do not follow the BSP on this point.

**Conclusion:** the port labelled WAN on the case is **switch port 0**, and LAN1–LAN3 are **switch ports 1–3**. The standing decision "the port labelled WAN is `wan`" therefore means DSA `port@0 label = "wan"`, with `lan1`/`lan2`/`lan3` on ports 1/2/3.

### Switch control lines (for #3 and #8)

| Function | SoC pin | Net (SCH p13) | Switch side (SCH p19) | Notes |
| --- | --- | --- | --- | --- |
| Switch power enable | GPIO1_C3 | `GPIO_8367_EN` | EN of both SY8088 DC-DCs U29 (AVDDL_1V1) and U30 (AVDDH_3V3), via R96008/R96009 10K | The switch has **no power until this pin is driven high**. Modelled as a fixed regulator in [ML-BASE] (`avdd_rtl8367rb`, `startup-delay-us = <10000>`), [BSP] and [ARMB]. |
| Switch reset | GPIO4_C2 | `GMAC1_RSTn_L` | `nRESET` (pin 91) via R239 0R, 100K pull-up R360 to DVDDIO | Active low. [ML-BASE] puts it on `&mdio1 reset-gpios` (25 ms assert, 100 ms post-delay). The reviewer in [ML-SW] asked to move it to the switch node. |
| Switch interrupt | GPIO1_C2 | `RTL8367RB_EINT` | `GPIO57/INTERRUPT` via R7 0R | [ML-SW] and [ARMB] use `IRQ_TYPE_LEVEL_LOW` with an internal pull-up. |
| Switch EEPROM | — | — | U90008 "NC/BL24C16A-RRRC" | Not fitted. The `DISAUTOLOAD` strap is pulled high (see LEDs), which matches. |

## 2. MAC addresses

### Sources on the board

1. **EEPROM (primary source).** U2702 is a **BL24C16F** (2 KiB, 24c16-compatible) on **I2C1 (M0 pins: GPIO4_A2 SDA, GPIO4_A3 SCL)** at address **0x50** (SCH p22 "EEPROM", p13). Write protect is active by default: WP is pulled up to 3V3 by R2716 10K and is only pulled low (writable) when **GPIO4_C0** (`GPIO4_C0_EEP_WP`) switches on Q1 (SCH p22). The SoC pin defaults to pull-down (`GPIO4_C0_d`, SCH p13), so the EEPROM is read-only unless software drives the pin high. Every DTS describes it ([ML-BASE] `eeprom@50`, `belling,bl24c16a`, `read-only`; [BSP] l.659).
   - **Layout.** The EEPROM holds a Radxa "RADX" record container. Record type 5 carries the serial number followed by **four sequential unicast MACs at offsets `0x9e`, `0xa4`, `0xaa` and `0xb0`**, assigned in switch-port order: wan, lan1, lan2, lan3 ([ARMB-V] commit `c5ab76fbdfb7` message; [ARMB] l.347-380 and 413-443 wire them as `nvmem-cells` on the DSA user ports). This is consistent with the E20C, whose EEPROM has its two MACs at the same first two offsets, `0x9e` and `0xa4` (this repo, `target/linux/rockchip/patches-6.18/165-arm64-dts-rockchip-Use-MAC-address-from-EEPROM-for-.patch`).
   - Radxa's docs say the E24C "has a unique and fixed MAC address that remains consistent across power cycles, reboots, and even after reflashing the firmware" ([DOCS] `interface-usage.md:233-235`). The EEPROM is the only on-board store that fits that description.
   - **Caveat:** no Radxa-authored description of the RADX format was found. The layout claim rests on Armbian's reading of real boards. See [On-board checks](#5-on-board-checks).
2. **SoC OTP (fallback).** Mainline U-Boot's `rockchip_setup_macaddr()` hashes the SoC `cpuid#` (read from OTP/efuse) with SHA-256 and sets `ethaddr` and `eth1addr` ([UBOOT] `arch/arm/mach-rockchip/board.c:325-359`). U-Boot's `fdt_fixup_ethernet()` then writes `ethaddr` to the node behind the `ethernet0` alias. That is why [ML-BASE] sets `ethernet0 = &gmac1`, and why [ARMB-V] `428ab2713a61` had to fix a vendor DTS whose alias pointed at the unused gmac0. RK3528 OTP support is already carried in-tree (`target/linux/rockchip/patches-6.18/038-4-v7.1-nvmem-rockchip-otp-Add-support-for-RK3528.patch`, `072-3-v7.1-…Enable-OTP-controller-for-RK3528.patch`). This only gives a stable address for the **conduit (gmac1)**, and only when the SPI bootloader is mainline U-Boot (#5, #11).
3. **Not used.** The Radxa BSP hard-codes placeholder addresses `00:48:54:20:00:0x` on every port ([BSP] l.382-411; copied unchanged into [ISTORE] l.250-279). iStoreOS ignores them and derives WAN/LAN MACs from the boot MMC's CID ([ISTORE] `02_network:89-101` default branch). On the SPI variant without eMMC, that ties the MAC to whichever microSD card is inserted, which is unsuitable for NVMe boot.

### Count

**Four MAC addresses** are available, one per front-panel port. The conduit and CPU port have no EEPROM entry of their own.

### Implications for the port (input to #8 / #12, not decided here)

- In DSA, put `nvmem-cells = <&mac_wan>` and so on on each user port, as [ARMB] does. This needs `CONFIG_NVMEM_LAYOUTS` (already `=y`, `target/linux/rockchip/armv8/config-6.18:454`) and `CONFIG_EEPROM_AT24` (`=y`, same file l.245).
- The conduit `eth0` needs a MAC too. Options: reuse `mac_wan` or `mac_lan1` on `gmac1`, or let U-Boot fill it in from the cpuid hash. OpenWrt's DSA bridge `br-lan` takes its MAC from its first member port unless one is set in board.d.

## 3. LEDs

### Front-panel LEDs: SoC GPIOs

There are five front-panel LEDs, all green. They are labelled SYS, WAN, LAN1, LAN2, LAN3 on the case ([DOCS] `e24c-shell-interface.webp`, item 5). The block diagram has "LAN LED ×3, WAN LED, SYS LED" (SCH p3). These are **SoC GPIOs, not switch outputs**.

| LED | Part (SCH p22) | GPIO (SCH p13) | Net | Drive circuit (SCH p22) | DT polarity | Function |
| --- | --- | --- | --- | --- | --- | --- |
| SYS | LED2701 green | **GPIO4_A1** | `SYS_LED_G` | VCC_3V3 → 4.7K R2720 → LED → GND. NMOS Q2701 (WNM2021) shorts the anode to GND when the gate is high. Gate has a 100K pull-down (R2721). | **`GPIO_ACTIVE_LOW`** | Lit by hardware from power-on until software drives the pin high. Radxa docs call it the power LED ([DOCS] `interface-usage.md:212`). Use it for OpenWrt `led-boot`/`-failsafe`/`-running`/`-upgrade`, as `138-…Update-LED-properties-for-Radxa-E20C.patch` does for the E20C. |
| WAN | LED2703 green | **GPIO4_A0** | `WAN_LED_G` | GPIO → 4.7K R2706 → LED → GND | `GPIO_ACTIVE_HIGH` | Link/activity of switch port 0 |
| LAN1 | LED2705 green | **GPIO1_B0** | `LAN1_LED_G` | GPIO → 4.7K R96077 → LED → GND | `GPIO_ACTIVE_HIGH` | Port 1 |
| LAN2 | LED2706 green | **GPIO1_B1** | `LAN2_LED_G` | GPIO → 4.7K R96078 → LED → GND | `GPIO_ACTIVE_HIGH` | Port 2 |
| LAN3 | LED2704 green | **GPIO1_B4** | `LAN3_LED_G` | GPIO → 4.7K R96076 → LED → GND | `GPIO_ACTIVE_HIGH` | Port 3 |

The GPIO numbers and polarities match all of [ML-BASE], [BSP] and [ISTORE]. Because these are plain GPIO LEDs, link and activity indication is a software job. The current options are:

- OpenWrt `netdev` triggers on the DSA user netdevs, set from `01_leds`. This is the E20C's approach (`target/linux/rockchip/armv8/base-files/etc/board.d/01_leds:54-59`). `CONFIG_LEDS_TRIGGER_NETDEV=y` is already in `target/linux/generic/config-6.18:3508`.
- Per-PHY `…:link` triggers ([ARMB], [ARMB-V] `7554973e9df0`). These need `CONFIG_LED_TRIGGER_PHY`, which is **not set** in OpenWrt (`target/linux/generic/config-6.18:3518`).

Which to use is a #12 decision.

### In-jack LEDs: switch-driven

Each HR5622S jack has a green and a yellow LED. Both are wired to the **switch's LED pins**, not to the SoC (SCH p19 "Port n LED" blocks, SCH p20 `LEDn_G±`/`LEDn_Y±`):

| Jack | Green LED | Yellow LED |
| --- | --- | --- |
| RJ45-0 (WAN) | P0LED0, **active high** (G+ via R12 0R, G− via R74 470R to GND) | P0LED2, active low (Y− via R10 0R, Y+ via R75 470R to DVDDIO) |
| RJ45-1 (LAN1) | P1LED0, **active low** (G− via R96016 0R, G+ via R96015 470R to DVDDIO) | P1LED2, active low |
| RJ45-2 (LAN2) | P2LED0, **active low** | P2LED2, active low |
| RJ45-3 (LAN3) | P3LED0, **active high** | P3LED2, active low |

The polarity mix follows the switch's **strapping pins**, because several LED pins double as straps (pin names from SCH p19):

| Pin | Strap function | Pulled to | LED wiring |
| --- | --- | --- | --- |
| P0LED0 | `SMI_SEL` | low | active-high LED |
| P0LED2 | `EN_PHY` | high | active-low LED |
| P1LED0 | `MID29` | high | active-low LED |
| P2LED0 | `DISAUTOLOAD` | high | active-low LED |
| P3LED0 | `RESERVED` | low | active-high LED |
| P3LED2 | `EN_PWRLIGHT` | high | active-low LED |

`SMI_SEL` low together with `MID29` high is consistent with MDIO management at address 29. This is an inference: no RTL8367RB datasheet was consulted. P4LED0 (`EEPROM_MOD`) has no fitted resistors. The serial LED pins (`LED_CK`/`LED_DA`) are unused.

Mainline `rtl8365mb` has **no LED support** (unlike `rtl8366rb-leds.c`; [LINUX] `drivers/net/dsa/realtek/`). The in-jack LEDs therefore show whatever the switch's power-on LED configuration does. Whether that is useful link/activity behaviour, and which colour means what, can only be settled on the board.

## 4. Buttons

| Button | Ref (SCH p22) | Wiring | How Linux sees it | Prior-art key codes | Location on case ([DOCS]) |
| --- | --- | --- | --- | --- | --- |
| USER | SW2702 | `GPIO0_A0_USER`: GPIO0_A0 → 100R R2713 → switch to GND, ESD diode ED2702 (SCH p22, p5). Needs the SoC pull-up (no external pull-up drawn). | `gpio-keys`, `GPIO_ACTIVE_LOW`, pinctrl `pcfg_pull_up` ([ML-BASE]) | [ML-BASE]/[ARMB]: `BTN_1`. [BSP]: `KEY_F15` (l.118-119). [ISTORE]: **`KEY_RESTART`** (l.126-127) | Side panel, between USB-A and DEBUG Type-C |
| MASKROM | SW2701 | `SARADC_IN0_BOOT`: SARADC channel 0, 100K pull-up R1503 to VCC_1V8. The button shorts it to GND through 100R R2705 (SCH p9, p22). Released ≈ 1.8 V, pressed ≈ 0 V. The boot-mode table on SCH p9 makes "LEVEL1 / ADC 0" mean USB Maskrom. | `adc-keys` on `saradc 0`, `keyup-threshold-microvolt = <1800000>`, `press-threshold-microvolt = <0>` ([ML-BASE]) | [ML-BASE]/[ARMB]: `KEY_SETUP` (same as in-tree H28K, `101-arm64-dts-rockchip-Add-HINLINK-H28K.patch:69`) | Side panel, between the two USB-A ports |
| POWER | SW2703 | `PWRON` → 100R R96084 → RK805 PMIC `PWRON` pin (SCH p22, p14). Not a SoC GPIO. | `rk805-pwrkey` input device, created automatically by the RK805 MFD ([LINUX] `drivers/mfd/rk8xx-core.c:73-82`). Always reports **`KEY_POWER`** (`drivers/input/misc/rk805-pwrkey.c:22,53`). `CONFIG_INPUT_RK805_PWRKEY=y` is already set (`target/linux/rockchip/armv8/config-6.18:339`). | fixed | Front panel, left |

Other key-related nets: `SARADC_IN1_RECOVER` has a 10K pull-up but goes only to test point TP2704, so there is no recovery button. `SARADC_IN2_HW_ID` is a 100K/330K divider (≈1.38 V), probably a board ID (SCH p9, p22).

### What OpenWrt does with these codes

`kmod-button-hotplug` maps `KEY_RESTART`→`reset`, `KEY_POWER`→`power`, `KEY_SETUP`→`setup`, and `BTN_0`/`BTN_1` to themselves (`package/kernel/button-hotplug/src/button-hotplug.c:76-90`). Only `reset`, `power`, `reboot`, `failsafe` and `rfkill` have handlers in `package/base-files/files/etc/rc.button/`:

- **USER → `KEY_RESTART`.** This gives the standard OpenWrt reset/failsafe behaviour. It is what iStoreOS ships, and what in-tree patches do for other boards (`140-arm64-dts-rockchip-add-and-set-buttons-for-NanoPi-M5.patch`, `401-2-nanopi-r6-reset-button.patch`). It needs a DTS patch on top of the mainline `BTN_1`. `gpio-keys` is handled by `kmod-gpio-button-hotplug`, a default package (`target/linux/rockchip/Makefile:18`).
- **POWER → `KEY_POWER`.** This is fixed by the driver. `/etc/rc.button/power` will power the board off, which only works once `kmod-button-hotplug` is installed, as for the E20C (`target/linux/rockchip/image/armv8.mk:280`).
- **MASKROM → `KEY_SETUP`.** This keeps the mainline code. It has no default handler, so it is harmless, and the button is mainly a boot-ROM strap. It needs `kmod-input-adc-keys` and `kmod-button-hotplug`, the same `DEVICE_PACKAGES` as `radxa_e20c` and `radxa_rock-2a/2f`.

Choosing the final codes belongs to #12. The facts above are its input.

## 5. On-board checks

These cannot be settled from documents. They belong in the hardware acceptance plan:

1. **EEPROM MAC layout.** Run `hexdump -C /sys/bus/nvmem/devices/1-00500/nvmem` (or `i2cdump -y 1 0x50`) and confirm the four valid, sequential, globally administered MACs at `0x9e`/`0xa4`/`0xaa`/`0xb0`. Check whether they match the sticker or label on the unit, if there is one.
2. **Port order.** With DSA up, plug one cable into each jack in turn. Confirm WAN = `port@0` and LAN1–3 = ports 1–3. This independently confirms [SCH] p20, [ML-SW] and [ISTORE] against the reversed [BSP].
3. **SYS LED behaviour.** Confirm it lights at power-on before Linux runs, and that `GPIO_ACTIVE_LOW` turns it on.
4. **In-jack LEDs.** Check whether the green and yellow jack LEDs show link/activity without any switch LED configuration under mainline `rtl8365mb`, and what each colour means. If they stay dark or behave oddly, decide whether that matters (#8).
5. **Power button.** Check short-press `KEY_POWER` delivery through `rk805-pwrkey`, and whether a long press forces a hardware power-off in the PMIC.
6. **MASKROM as an ADC key.** Confirm the SARADC reads about 0 when pressed and about 1023 when released.

## 6. Contradictions and loose ends

- **Port labels.** The Radxa BSP ([BSP] l.381-403) labels the ports in reverse and conflicts with the schematic, the case, mainline and iStoreOS. It is treated as a copy-paste error from the E54C.
- **Radxa docs.** The "Gigabit Network Controller" section of [DOCS] `hardware-interface.md` says the ports are "driven by the network controller integrated in the Rockchip RK3528A SoC", which contradicts the schematic. The ports are behind the switch. That section is generic marketing text and should not be used as a source.
- **The conduit's MAC.** Nothing in the hardware assigns the gmac1 conduit a MAC. This needs a decision (see §2).
