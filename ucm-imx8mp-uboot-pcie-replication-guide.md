# UCM-iMX8M-Plus PCIe Root-Complex U-Boot Build Guide

This guide explains how to reproduce a U-Boot image for the CompuLab UCM-iMX8M-Plus / SBEV platform that initializes the i.MX8MP PCIe controller as a Root Complex. The intended use case is to let U-Boot perform the PCIe low-level bring-up, endpoint enumeration, and BAR assignment, and then jump to a bare-metal Cortex-A53 application that directly accesses an FPGA endpoint BAR.

The final validated flow is:

```text
Boot custom U-Boot from SD using ALT_BOOT
Run pci enum in U-Boot
U-Boot initializes PCIe Root Complex mode
U-Boot enumerates the PCIe endpoint and assigns BARs
Load a bare-metal application
Bare-metal application accesses the endpoint BAR directly
```

This was validated with an Intel AX210 PCIe endpoint before using the real FPGA endpoint.

---

## 1. Repository layout and branches

Use three directories:

```text
~/Documents/clue/firmware-pcie/bootloader/
├── u-boot-compulab-pcie-base/      # Working U-Boot tree
├── u-boot-compulab-2025.04/        # Donor U-Boot tree used for backports
└── build-ucm-imx8mp-pcie-base/     # Out-of-tree build directory
```

The working tree should be based on the CompuLab i.MX8MP U-Boot 2023.04 tree. The PCIe changes are kept in this branch:

```text
pcie-dw-imx8mp-backport
```

The final branch contains these commits on top of the CompuLab base:

```text
Backport i.MX8MP DesignWare PCIe support
Fix i.MX8MP PCIe runtime dependencies
Enable PCIe options in UCM-iMX8M-Plus defconfig
```

The third commit is important: without the defconfig update, the code may build only because an existing local `.config` already has PCIe enabled.

---

## 2. Clone the repositories

Clone the working U-Boot tree from the fork that contains the PCIe branch:

```bash
git clone https://github.com/juanea7/<your-u-boot-fork>.git u-boot-compulab-pcie-base
cd u-boot-compulab-pcie-base
git checkout pcie-dw-imx8mp-backport
```

Clone or keep a donor tree with newer i.MX8MP PCIe support. This is only needed if the PCIe branch has to be recreated manually from a clean vendor tree:

```bash
cd ~/Documents/clue/firmware-pcie/bootloader
git clone https://github.com/<vendor-or-donor>/u-boot-compulab-2025.04.git u-boot-compulab-2025.04
```

Set the common environment variables:

```bash
cd ~/Documents/clue/firmware-pcie/bootloader/u-boot-compulab-pcie-base

export BUILD=~/Documents/clue/firmware-pcie/bootloader/build-ucm-imx8mp-pcie-base
export DONOR=~/Documents/clue/firmware-pcie/bootloader/u-boot-compulab-2025.04
export CROSS_COMPILE=aarch64-linux-gnu-
```

Use only the intended build directory. Avoid reusing old build directories that may have accidentally selected `CONFIG_SANDBOX=y` or stale configuration values.

---

## 3. Summary of the required source changes

The final U-Boot tree needs the following functional changes:

```text
1. i.MX8MP DesignWare PCIe host driver
2. i.MX8M PCIe PHY driver
3. i.MX8MP PCIe clock definitions
4. i.MX8MP PCIe reset support
5. i.MX8MP HSIO block-control PCIe/PLL support
6. UCM-iMX8M-Plus device-tree PCIe Root Complex enablement
7. Non-fatal handling for missing board-level reset-gpio
8. PCIe-related defconfig options
```

If the branch already contains the commits, skip directly to section 8, **Build U-Boot**. The following sections explain how to recreate the patch set from a clean vendor tree.

---

## 4. Backport the PCIe and PHY drivers

Copy the i.MX DesignWare PCIe host driver from the donor tree:

```bash
cp ${DONOR}/drivers/pci/pcie_dw_imx.c drivers/pci/pcie_dw_imx.c
```

Ensure `drivers/pci/Makefile` contains:

```make
obj-$(CONFIG_PCIE_DW_IMX) += pcie_dw_imx.o
```

Ensure `drivers/pci/Kconfig` contains a `PCIE_DW_IMX` option. The exact donor Kconfig block can be copied from the donor tree.

Copy the i.MX8M PCIe PHY driver:

```bash
cp ${DONOR}/drivers/phy/phy-imx8m-pcie.c drivers/phy/phy-imx8m-pcie.c
```

Ensure `drivers/phy/Makefile` contains:

```make
obj-$(CONFIG_PHY_IMX8M_PCIE) += phy-imx8m-pcie.o
```

Ensure `drivers/phy/Kconfig` contains a `PHY_IMX8M_PCIE` option. The exact donor Kconfig block can be copied from the donor tree.

---

## 5. Backport reset and HSIO runtime dependencies

Copy the newer i.MX reset driver:

```bash
cp ${DONOR}/drivers/reset/reset-imx7.c drivers/reset/reset-imx7.c
```

The reset driver must support these i.MX8MP PCIe reset IDs:

```text
IMX8MP_RESET_PCIEPHY
IMX8MP_RESET_PCIEPHY_PERST
IMX8MP_RESET_PCIE_CTRL_APPS_EN
IMX8MP_RESET_PCIE_CTRL_APPS_TURNOFF
```

Copy the newer HSIO block-control driver:

```bash
cp ${DONOR}/drivers/power/domain/imx8mp-hsiomix.c \
  drivers/power/domain/imx8mp-hsiomix.c
```

This driver must provide PCIe-related HSIO support, including:

```text
clk_pcie
IMX8MP_HSIOBLK_PD_PCIE
IMX8MP_HSIOBLK_PD_PCIE_PHY
HSIO PLL clock provider
UCLASS_CLK support
```

This is required because the PCIe PHY obtains its reference clock from `hsio_blk_ctrl`.

---

## 6. Add PCIe clocks to the i.MX8MP clock driver

Edit:

```text
drivers/clk/imx/clk-imx8mp.c
```

Add the PCIe AUX parent list:

```c
static const char * const imx8mp_pcie_aux_sels[] = {
    "clock-osc-24m", "sys_pll2_200m", "sys_pll2_50m",
    "sys_pll3_out", "sys_pll2_100m", "sys_pll1_80m",
    "sys_pll1_160m", "sys_pll1_200m",
};
```

Register the PCIe AUX and root clocks:

```c
clk_dm(IMX8MP_CLK_PCIE_AUX,
       imx8m_clk_composite("pcie_aux", imx8mp_pcie_aux_sels, base + 0xa400));

clk_dm(IMX8MP_CLK_PCIE_ROOT,
       imx_clk_gate4("pcie_root_clk", "pcie_aux", base + 0x4250, 0));
```

After building, the image should contain the PCIe clock names:

```bash
strings ${BUILD}/u-boot | grep -E "pcie_aux|pcie_root_clk"
strings ${BUILD}/flash.bin | grep -E "pcie_aux|pcie_root_clk"
```

---

## 7. Update the board device tree

Edit:

```text
arch/arm/dts/ucm-imx8m-plus.dts
```

Add the PCIe PHY binding include near the top:

```dts
#include <dt-bindings/phy/phy-imx8-pcie.h>
```

Enable and describe the PCIe PHY:

```dts
&pcie_phy {
    fsl,refclk-pad-mode = <IMX8_PCIE_REFCLK_PAD_UNUSED>;
    clocks = <&hsio_blk_ctrl>;
    clock-names = "ref";
    status = "okay";
};
```

Enable the PCIe Root Complex node and provide its clocks:

```dts
&pcie {
    clocks = <&clk IMX8MP_CLK_HSIO_ROOT>,
             <&clk IMX8MP_CLK_HSIO_AXI>,
             <&clk IMX8MP_CLK_PCIE_ROOT>;
    clock-names = "pcie", "pcie_bus", "pcie_aux";
    assigned-clocks = <&clk IMX8MP_CLK_PCIE_AUX>;
    assigned-clock-rates = <10000000>;
    assigned-clock-parents = <&clk IMX8MP_SYS_PLL2_50M>;
    status = "okay";
};
```

Disable endpoint mode:

```dts
&pcie_ep {
    status = "disabled";
};
```

Make `hsio_blk_ctrl` usable as a clock provider:

```dts
&hsio_blk_ctrl {
    #clock-cells = <0>;
};
```

---

## 8. Make missing reset-gpio non-fatal

On this board, Linux PCIe works without a board-level PCIe `reset-gpio` property. U-Boot should therefore not fail if the property is absent.

Edit:

```text
drivers/pci/pcie_dw_imx.c
```

Change the `reset-gpio` request path from fatal to optional:

```diff
 ret = gpio_request_by_name(dev, "reset-gpio", 0, &priv->reset_gpio,
                            GPIOD_IS_OUT | GPIOD_IS_OUT_ACTIVE);
 if (ret) {
-        dev_err(dev, "unable to get reset-gpio\n");
-        goto err_gpio;
+        dev_dbg(dev, "no reset-gpio defined, continuing without PERST GPIO\n");
 }
```

Guard any `dm_gpio_free()` calls:

```diff
- dm_gpio_free(dev, &priv->reset_gpio);
+ if (dm_gpio_is_valid(&priv->reset_gpio))
+        dm_gpio_free(dev, &priv->reset_gpio);
```

The existing assert/deassert logic should also be guarded by `dm_gpio_is_valid()`.

---

## 9. Enable PCIe in the board defconfig

Edit:

```text
configs/ucm-imx8m-plus_defconfig
```

Add the PCIe options:

```text
# PCIe Root Complex support for i.MX8MP
CONFIG_CMD_PCI=y
CONFIG_PCI=y
CONFIG_DM_PCI=y
CONFIG_PCIE_DW_IMX=y
CONFIG_PHY=y
CONFIG_PHY_IMX8M_PCIE=y
CONFIG_DM_RESET=y
CONFIG_RESET_IMX7=y
```

Verify from a clean build directory, not from an old `.config`:

```bash
export BUILD=~/Documents/clue/firmware-pcie/bootloader/build-ucm-imx8mp-pcie-clean-test
rm -rf ${BUILD}

make O=${BUILD} ucm-imx8m-plus_defconfig
make O=${BUILD} olddefconfig

grep -E "CONFIG_CMD_PCI|CONFIG_PCI|CONFIG_DM_PCI|CONFIG_PCIE_DW_IMX|CONFIG_PHY_IMX8M_PCIE|CONFIG_DM_RESET|CONFIG_RESET_IMX7" \
  ${BUILD}/.config
```

Expected result:

```text
CONFIG_CMD_PCI=y
CONFIG_PCI=y
CONFIG_DM_PCI=y
CONFIG_PCIE_DW_IMX=y
CONFIG_PHY_IMX8M_PCIE=y
CONFIG_DM_RESET=y
CONFIG_RESET_IMX7=y
```

`CONFIG_PHY=y` may not appear explicitly depending on Kconfig selection, but `CONFIG_PHY_IMX8M_PCIE=y` must be present.

---

## 10. Build U-Boot and flash.bin

Use an out-of-tree build directory:

```bash
cd ~/Documents/clue/firmware-pcie/bootloader/u-boot-compulab-pcie-base

export BUILD=~/Documents/clue/firmware-pcie/bootloader/build-ucm-imx8mp-pcie-base
export CROSS_COMPILE=aarch64-linux-gnu-

rm -rf ${BUILD}
make O=${BUILD} ucm-imx8m-plus_defconfig
make O=${BUILD} olddefconfig
nice make -j"$(nproc)" O=${BUILD} flash.bin
```

The resulting image is:

```text
${BUILD}/flash.bin
```

This image contains the boot firmware. It is not a Linux kernel or root filesystem image.

---

## 11. Flash to SD for ALT_BOOT

Write `flash.bin` to the whole SD block device, not to a partition:

```bash
sudo dd if=${BUILD}/flash.bin of=/dev/sdX bs=1K seek=32 conv=fsync status=progress
sync
```

Replace `/dev/sdX` with the real SD card device.

The required offset is:

```text
seek=32
```

which corresponds to:

```text
0x8000
```

Boot the board with the SD card and ALT_BOOT selected.

The warning below is acceptable for a fresh SD environment:

```text
Loading Environment from MMC... *** Warning - bad CRC, using default environment
```

It only means U-Boot is using its default environment.

---

## 12. Verify the generated device tree

After building, dump the generated DTB:

```bash
DT=$(sed -n 's/^CONFIG_DEFAULT_DEVICE_TREE="\([^"]*\)"/\1/p' ${BUILD}/.config)

dtc -I dtb -O dts \
  -o ${BUILD}/${DT}.dump.dts \
  ${BUILD}/arch/arm/dts/${DT}.dtb
```

Check the PCIe PHY, HSIO block controller, and PCIe Root Complex nodes:

```bash
grep -n "pcie-phy@32f00000" -A35 ${BUILD}/${DT}.dump.dts
grep -n "blk-ctrl@32f10000" -A30 ${BUILD}/${DT}.dump.dts
grep -n "pcie@33800000" -A70 ${BUILD}/${DT}.dump.dts
```

The generated PHY node should contain:

```dts
status = "okay";
fsl,refclk-pad-mode = <0x00>;
clocks = <...>;
clock-names = "ref";
```

The generated HSIO block-control node should contain:

```dts
#clock-cells = <0x00>;
```

The generated PCIe node should contain:

```dts
status = "okay";
phy-names = "pcie-phy";
phys = <...>;
clock-names = "pcie", "pcie_bus", "pcie_aux";
```

---

## 13. U-Boot PCIe validation

Stop autoboot and run:

```bash
pci enum
pci
pci regions
pci header 00.00.00
pci header 01.00.00
pci bar 01.00.00
```

A validated result with an Intel AX210 endpoint was:

```text
ucm-imx8m-plus=> pci enum
PCIE-0: Link up (Gen1-x1, Bus0)

ucm-imx8m-plus=> pci
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x16c3     0xabcd     Bridge device           0x04
01.00.00   0x8086     0x2725     Network controller      0x80
```

The validated BAR assignment was:

```text
ucm-imx8m-plus=> pci bar 01.00.00
ID   Base                Size                Width  Type
----------------------------------------------------------
 0   0x0000000018200000  0x0000000000004000  64     MEM
```

For that endpoint, BAR0 was mapped at:

```text
0x18200000 - 0x18203fff
```

The PCIe memory window was:

```text
0x18000000 - 0x1fefffff
```

---

## 14. Bare-metal handoff validation

After `pci enum`, load and run the bare-metal test:

```bash
pci enum
tftpboot ${loadaddr} pcie-test.bin
go ${loadaddr}
```

The validated bare-metal test read the PCIe DBI registers:

```text
DBI vendor/device @ 0x33800000 = 0xabcd16c3
DBI command/status @ 0x33800004 = 0x00100007
DBI class/revision @ 0x33800008 = 0x06040001
DBI BAR0 @ 0x33800010 = 0x18000004
DBI bus numbers @ 0x33800018 = 0x00010100
```

It also read the endpoint BAR directly:

```text
BAR0 + 0x000 @ 0x18200000 = 0x00880000
BAR0 + 0x004 @ 0x18200004 = 0x00000000
BAR0 + 0x008 @ 0x18200008 = 0x00000000
BAR0 + 0x00c @ 0x1820000c = 0x00000000
```

This validates that U-Boot can initialize PCIe and then hand off to bare metal while leaving DBI and endpoint BAR MMIO reachable.

---

## 15. Notes about config-space access from bare metal

Do not use a naive ECAM formula such as:

```c
PCIE_CFG_BASE + (bus << 20) + (dev << 15) + (func << 12) + offset
```

The i.MX8MP DesignWare config aperture used here is not a full simple ECAM window. In the validated setup:

```text
PCIE_CFG_BASE = 0x1ff00000
```

The naive bus 1 address becomes:

```text
0x20000000
```

which is outside the configured 512 KiB config aperture and causes a synchronous abort.

For the register-only FPGA endpoint use case, this is not required. U-Boot performs enumeration and BAR assignment; the bare-metal application only needs the assigned endpoint BAR base.

---

## 16. Final FPGA endpoint flow

With the FPGA endpoint connected, use U-Boot to enumerate and discover the BAR assignment:

```bash
pci enum
pci
pci bar 01.00.00
```

Then use the printed BAR base in the bare-metal application:

```c
#define FPGA_BAR0_BASE 0x18xxxxxxUL

#define REG_ID        0x00
#define REG_VERSION   0x04
#define REG_CONTROL   0x08
#define REG_STATUS    0x0c

uint32_t id      = mmio_read32(FPGA_BAR0_BASE + REG_ID);
uint32_t version = mmio_read32(FPGA_BAR0_BASE + REG_VERSION);
uint32_t status  = mmio_read32(FPGA_BAR0_BASE + REG_STATUS);

mmio_write32(FPGA_BAR0_BASE + REG_CONTROL, 0x1);
```

For a register-only FPGA endpoint, no DMA support is required.

---

## 17. Final validated result

The final result is a reproducible U-Boot build that:

```text
boots from SD using ALT_BOOT
initializes PCIe Root Complex mode
trains a PCIe link
enumerates a PCIe endpoint
assigns endpoint BARs
hands off to a bare-metal Cortex-A53 application
leaves DBI and endpoint BAR MMIO accessible from bare metal
```

This provides the required foundation for a bare-metal application that controls a register-only FPGA endpoint over PCIe.
