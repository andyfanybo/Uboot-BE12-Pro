# Tenda BE12 Pro port for Yuzhii0718/bl-mt798x-dhcpd

Baseline: `Yuzhii0718/bl-mt798x-dhcpd` master commit `0f584098854f2956d5d36f3fbe6b786fafeb7e74` (2026-09-22 inspection).

## Hardware / bootloader evidence

The supplied 3 MiB `ImmortalWrt.mtd0.Bootloader.bin` was inspected directly:

- Size: `0x300000` (3 MiB)
- SHA256: `9b1aa7affa1f3704392cea4f92a185f11bb683d6f9815308edac7759b8fb762c`
- MediaTek SPI-NAND header `SPINAND!` starts at `0x000000`
- Main TF-A FIP ToC magic `0xAA640001` starts exactly at `0x100000`
- Therefore the OEM bootloader layout confirms:
  - BL2 area: `0x000000-0x0fffff` (1 MiB)
  - FIP area: `0x100000-0x2fffff` (2 MiB)
- OEM U-Boot string: `U-Boot 2025.04-rc3-svn3112 (Dec 02 2025 - 15:46:44 +0800)`
- OEM recovery UI identifies itself as `Tenda U-Boot System Recovery Mode`.

The OEM U-Boot embedded control DT confirms:

- GMAC0: 2500base-x -> AN8855, reset GPIO42, ACTIVE_HIGH
- GMAC1: XGMII -> MT7987 internal 2.5G PHY at MDIO address 15
- GMAC2: 2500base-x -> Airoha EN8811H at MDIO address 11, reset GPIO48, ACTIVE_HIGH
- SPI0: `#address-cells = <1>`, `#size-cells = <0>`, SPI-NAND at CS0, 52 MHz
- Reset button GPIO1 ACTIVE_LOW
- WPS button GPIO0 ACTIVE_LOW

The OEM control DT still declares a 256 MiB memory window, while the board Linux DTS reports 512 MiB. This port intentionally follows the upstream/OEM U-Boot control-DT convention and lets the MT7987 DRAM code perform runtime size detection instead of hard-coding a 512 MiB U-Boot control-DT node.

## ImmortalWrt/OpenWrt fixed MTD layout

The port uses the layout from the BE12 Pro ImmortalWrt/OpenWrt device tree, while splitting the 3 MiB Bootloader region into the confirmed BL2/FIP subregions:

```
0x0000000  0x0100000  bl2          1 MiB
0x0100000  0x0200000  fip          2 MiB
0x0300000  0x0080000  u-boot-env   512 KiB
0x0380000  0x0400000  factory      4 MiB
0x0780000  0x0600000  kernel       6 MiB
0x0d80000  0x5a00000  ubi          90 MiB
0x6780000  0x0400000  CFG          4 MiB
0x6b80000  0x0400000  MISC2        4 MiB
```

## Build

Copy the overlay files into a fork/clone of Yuzhii0718/bl-mt798x-dhcpd, then run:

```bash
BOARD=tenda_be12-pro VERSION=2025 VARIANT=default MULTI_LAYOUT=0 FIXED_MTDPARTS=1 ./build.sh
```

Or use this repository's `FIP Build` GitHub Action with:

- MODEL: `tenda_be12-pro`
- FIP_VERSION: `2025`
- VARIANT: `default`
- EXTRA_OPTION: `single_layout`
- COPY_BL2: `true`
- REPO_URL: `https://github.com/andyfanybo/Uboot-BE12-Pro`
- REPO_BRANCH: `master`

## Hardware validation note

The Linux/OpenWrt DTS describes the EN8811H reset GPIO48 with active-low polarity, while the inspected OEM U-Boot control DT and the MediaTek MT7987 RFB U-Boot description use active-high semantics. This port intentionally keeps the U-Boot/OEM active-high reset behavior; verify GMAC2/EN8811H link over UART during first hardware testing.

## Flashing safety

Do not flash BL2 before serial recovery has been validated. The uploaded bootloader confirms the FIP offset, but first hardware testing should still use UART and, where possible, test only FIP/recovery functions before replacing BL2.
