# UCM-iMX8M-Plus PCIe Root-Complex U-Boot Build Guide

This guide explains how to reproduce and validate a U-Boot image for the CompuLab UCM-iMX8M-Plus / SBEV platform with the i.MX8MP PCIe controller operating as a Root Complex. The target endpoint is a VCU128 FPGA using a Xilinx XDMA PCIe endpoint (`10ee:9031`), reached through the UCM M.2 A/E-key connector and a PCIe adapter.

The current validated FPGA bring-up flow is:

```text
Boot custom U-Boot from SD using ALT_BOOT
VCU128 connected but not yet programmed
Run pci enum once
  -> U-Boot initializes PCIe PHY/clocks
  -> link is down because the FPGA endpoint does not yet exist
  -> patched driver intentionally keeps PCIe PHY/clocks active
Program the VCU128 over JTAG
Run pci enum again
  -> driver reuses the already-active PCIe resources
  -> link reaches Gen1 x1 / L0
  -> Xilinx endpoint 10ee:9031 is enumerated
Assign/fix the FPGA BAR layout
Access the FPGA BARs directly from U-Boot or a bare-metal Cortex-A53 application
```

This flow has now been validated with the real VCU128 endpoint. An Intel AX210 native M.2 PCIe endpoint remains useful as a reference endpoint because it links normally on the first `pci enum`.

A second, independent issue remains in U-Boot's generic BAR autoconfiguration for the VCU128: the endpoint BARs fit in the available 127 MiB PCI memory window, but the default allocation order creates an alignment hole and causes `PCI: Failed autoconfig bar 18`. A manual BAR layout has been validated and MMIO reads work; a permanent allocator-side fix is still pending.

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

The branch contains the original PCIe backport work on top of the CompuLab base:

```text
Backport i.MX8MP DesignWare PCIe support
Fix i.MX8MP PCIe runtime dependencies
Enable PCIe options in UCM-iMX8M-Plus defconfig
```

The current tested working tree additionally contains the late-FPGA-endpoint bring-up fix described in Section 9. It should be checkpointed as a separate commit, for example:

```text
pci: imx8mp: support late FPGA endpoint bring-up
```

The defconfig commit is important: without it, the code may build only because an existing local `.config` already has PCIe enabled.

---

## 2. Clone the repositories

Clone the working U-Boot tree from the fork that contains the PCIe branch:

```bash
git clone https://github.com/juanea7/u-boot-compulab-pcie.git u-boot-compulab-pcie-base
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
9. Late FPGA-endpoint support:
   - keep PHY/clocks active after the first link-down
   - preserve that state across a failed Driver Model probe
   - reuse resources on the second probe instead of enabling them twice
10. Pending: permanent BAR allocation fix for the VCU128 multi-BAR layout
```

If the branch already contains the backport commits, skip the donor-copy steps and apply/verify the board-specific late-endpoint changes described below.

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

For the late-programmed FPGA endpoint flow, the effective U-Boot PCIe node must also contain:

```dts
u-boot,keep-pcie-resources-on-link-down;
```

The validated working tree initially placed this property in the board `&pcie` node. For repository hygiene it can instead be placed in `ucm-imx8m-plus-u-boot.dtsi`; if moved there, rebuild and revalidate the same sequence.

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

---

## 9. Support a late-programmed FPGA endpoint

### 9.1 Why the original U-Boot flow failed

The VCU128 endpoint is programmed after the UCM has already reached the U-Boot shell. A normal first `pci enum` therefore sees no active endpoint and the original i.MX PCIe driver follows its `err_link` cleanup path:

```c
err_link:
    generic_shutdown_phy(&priv->phy);
err_phy_power:
    generic_phy_exit(&priv->phy);
err_phy_init:
    clk_disable_bulk(&priv->clks);
```

That cleanup is correct for a conventional endpoint, but it is incompatible with this FPGA sequence. After the failed enumeration, the host-side PCIe resources disappear; programming the VCU128 afterwards leaves the XDMA core without the host clock environment it needs.

This was visible on the FPGA debug LEDs:

```text
Before fix, after programming VCU:
LED0  blinking  independent FPGA clock
LED1  ON        PERST# released
LED2  OFF       XDMA axi_aclk not running
LED3  OFF       XDMA AXI reset asserted
LED4  OFF       Detect.Active never reached
LED5  OFF       Polling never reached
LED6  OFF       L0 never reached
```

### 9.2 Board-specific property

The driver behavior is enabled only for the UCM/VCU flow with this DT property:

```dts
u-boot,keep-pcie-resources-on-link-down;
```

The driver reads it with:

```c
priv->keep_resources_on_link_down =
    dev_read_bool(dev, "u-boot,keep-pcie-resources-on-link-down");
```

Add the corresponding flag to the private structure:

```c
bool keep_resources_on_link_down;
```

### 9.3 Persistent resource state

A first implementation stored the "resources kept" state inside `struct pcie_dw_imx`. That was not sufficient because the Driver Model private instance does not preserve the state after a probe returns `-ENODEV`.

The validated implementation therefore keeps a small driver-level state:

```c
static bool pcie_resources_kept;
```

At probe entry:

```c
bool reuse_resources = priv->keep_resources_on_link_down &&
                       pcie_resources_kept;
```

The regulator, clocks, PHY initialization, and PHY power-on are only executed when the resources are not already active:

```c
if (!reuse_resources && priv->vpcie) {
    ret = regulator_set_enable(priv->vpcie, true);
    if (ret)
        return ret;
}

...

if (!reuse_resources) {
    ret = imx_pcie_clk_enable(priv);
    if (ret)
        goto err_clk;

    ret = generic_phy_init(&priv->phy);
    if (ret)
        goto err_phy_init;

    ret = generic_phy_power_on(&priv->phy);
    if (ret)
        goto err_phy_power;
} else {
    printf("PCIE-%d: Reusing active PCIe PHY/clocks\n",
           dev_seq(dev));
}
```

On a failed link attempt, the normal cleanup remains available for ordinary boards, but the UCM/VCU-specific property keeps the resources active:

```c
if (pcie_link_up(priv, LINK_SPEED_GEN_1)) {
    printf("PCIE-%d: Link down\n", dev_seq(dev));

    if (priv->keep_resources_on_link_down) {
        pcie_resources_kept = true;
        printf("PCIE-%d: Keeping PCIe PHY/clocks enabled for late endpoint\n",
               dev_seq(dev));
        return -ENODEV;
    }

    ret = -ENODEV;
    goto err_link;
}
```

On success, the state remains marked active for this board-specific flow:

```c
if (priv->keep_resources_on_link_down)
    pcie_resources_kept = true;
```

On real driver removal, clear the state before shutting down the PHY:

```c
pcie_resources_kept = false;
generic_shutdown_phy(&priv->phy);
```

This is intentionally board-specific. It should not silently replace the normal cleanup behavior for every i.MX8MP PCIe user.

### 9.4 Validated behavior

With the VCU connected but not programmed:

```text
ucm-imx8m-plus=> pci enum
PCIE-0: Link down
PCIE-0: Keeping PCIe PHY/clocks enabled for late endpoint
```

After programming the VCU128 over JTAG, the FPGA debug monitor shows the XDMA clock and LTSSM becoming active. A second enumeration gives:

```text
ucm-imx8m-plus=> pci enum
PCIE-0: Reusing active PCIe PHY/clocks
PCIE-0: Link up (Gen1-x1, Bus0)
PCI: Failed autoconfig bar 18
```

The BAR message is a separate allocation issue; the physical link and enumeration are successful.

The endpoint is visible:

```text
ucm-imx8m-plus=> pci
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x16c3     0xabcd     Bridge device           0x04
01.00.00   0x10ee     0x9031     Simple comm. controller 0x00
```

After link-up, the FPGA debug LEDs are all in the expected state:

```text
LED0  blinking
LED1  ON
LED2  blinking
LED3  ON
LED4  ON
LED5  ON
LED6  ON
```

This confirms REFCLK/XDMA clocking, PERST#, Detect, Polling, L0, and endpoint enumeration.

---

## 10. Enable PCIe in the board defconfig

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

## 11. Build U-Boot and flash.bin

Use the existing out-of-tree build directory:

```bash
cd ~/Documents/clue/firmware-pcie/bootloader/u-boot-compulab-pcie-base

export BUILD=~/Documents/clue/firmware-pcie/bootloader/build-ucm-imx8mp-pcie-base
export CROSS_COMPILE=aarch64-linux-gnu-

make O=${BUILD} ucm-imx8m-plus_defconfig
make O=${BUILD} olddefconfig
nice make -j"$(nproc)" O=${BUILD} flash.bin
```

The `source` entry inside the build directory resolves to the actual U-Boot source tree:

```bash
readlink -f ${BUILD}/source
```

Validated path:

```text
/home/juan/Documents/clue/firmware-pcie/bootloader/u-boot-compulab-pcie-base
```

The resulting boot image is:

```text
${BUILD}/flash.bin
```

For a clean reproducibility test, use a separate empty build directory rather than deleting the working build directory.

---

## 12. Flash to SD for ALT_BOOT

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

## 13. Verify the generated device tree

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
grep -n "pcie@33800000" -A80 ${BUILD}/${DT}.dump.dts
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
u-boot,keep-pcie-resources-on-link-down;
```

---

## 14. PCIe validation with the Intel reference endpoint

The native Intel AX210 remains a useful sanity check because it is present and ready when U-Boot probes PCIe.

Validated output:

```text
ucm-imx8m-plus=> pci enum
PCIE-0: Link up (Gen1-x1, Bus0)

ucm-imx8m-plus=> pci
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x16c3     0xabcd     Bridge device           0x04
01.00.00   0x8086     0x2725     Network controller      0x80
```

A validated Intel BAR assignment was:

```text
ucm-imx8m-plus=> pci bar 01.00.00
ID   Base                Size                Width  Type
----------------------------------------------------------
 0   0x0000000018200000  0x0000000000004000  64     MEM
```

The PCIe memory window is:

```text
0x18000000 - 0x1fefffff
```

---

## 15. PCIe validation with the VCU128 FPGA endpoint

### 15.1 Bring-up sequence

Use this exact sequence:

```text
1. Connect the VCU128 but leave it unprogrammed.
2. Boot the UCM into the custom U-Boot.
3. Run `pci enum`.
4. Verify that U-Boot reports Link down but keeps PCIe resources active.
5. Program the VCU128 bitstream over JTAG.
6. Verify the FPGA LEDs show XDMA clocking / LTSSM activity.
7. Run `pci enum` again.
8. Verify Gen1 x1 link-up and endpoint enumeration.
```

Validated console output:

```text
ucm-imx8m-plus=> pci enum
PCIE-0: Link down
PCIE-0: Keeping PCIe PHY/clocks enabled for late endpoint

ucm-imx8m-plus=> pci enum
PCIE-0: Reusing active PCIe PHY/clocks
PCIE-0: Link up (Gen1-x1, Bus0)
PCI: Failed autoconfig bar 18
```

The Xilinx endpoint is then visible:

```text
ucm-imx8m-plus=> pci
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x16c3     0xabcd     Bridge device           0x04
01.00.00   0x10ee     0x9031     Simple comm. controller 0x00
```

### 15.2 VCU128 BAR requirements

The endpoint exposes three 64-bit memory BARs:

```text
BAR0/1: 32 MiB
BAR2/3: 64 MiB
BAR4/5: 64 KiB
```

U-Boot initially reports:

```text
ID   Base                Size                Width  Type
----------------------------------------------------------
 0   0x000000001a000000  0x0000000002000000  64     MEM
 1   0xfffffffffc000000  0x0000000004000000  64     MEM
 2   0x000000001c000000  0x0000000000010000  64     MEM
```

The second BAR is unassigned because of U-Boot's allocation order/alignment, not because the total memory requirement is too large.

---

## 16. Manual VCU128 BAR layout and MMIO validation

### 16.1 Why the automatic allocation fails

The available PCI memory window is:

```text
0x18000000 - 0x1fefffff
size = 0x07f00000 = 127 MiB
```

The BARs need only about 96 MiB in total, but the default order places the 32 MiB BAR first:

```text
BAR0 32 MiB -> 0x1a000000
```

The following 64 MiB BAR must be 64 MiB aligned, so its next valid start is `0x1c000000`. That would extend to `0x1fffffff`, just beyond the PCI memory window. U-Boot therefore prints:

```text
PCI: Failed autoconfig bar 18
```

Here `18` is configuration-space offset `0x18`, the low dword of the BAR2/3 64-bit BAR pair.

### 16.2 Validated manual layout

A working packing is:

```text
BAR2/3  64 MiB  -> 0x18000000
BAR0/1  32 MiB  -> 0x1c000000
BAR4/5  64 KiB  -> 0x1e000000
```

Program it in U-Boot:

```bash
# Temporarily disable Memory Space Enable, keep Bus Master Enable
pci write.w 1.0.0 04 0004

# BAR0/1: 32 MiB @ 0x1c000000
pci write.l 1.0.0 10 1c000004
pci write.l 1.0.0 14 00000000

# BAR2/3: 64 MiB @ 0x18000000
pci write.l 1.0.0 18 18000004
pci write.l 1.0.0 1c 00000000

# BAR4/5: 64 KiB @ 0x1e000000
pci write.l 1.0.0 20 1e000004
pci write.l 1.0.0 24 00000000

# Re-enable Memory Space Enable + Bus Master Enable
pci write.w 1.0.0 04 0006
```

Validated result:

```text
ucm-imx8m-plus=> pci bar 1.0.0
ID   Base                Size                Width  Type
----------------------------------------------------------
 0   0x000000001c000000  0x0000000002000000  64     MEM
 1   0x0000000018000000  0x0000000004000000  64     MEM
 2   0x000000001e000000  0x0000000000010000  64     MEM
```

The configuration header confirms:

```text
base address 0 = 0x1c000004
base address 1 = 0x00000000
base address 2 = 0x18000004
base address 3 = 0x00000000
base address 4 = 0x1e000004
base address 5 = 0x00000000
command register ID = 0x0006
```

### 16.3 MMIO reads from the UCM

Direct U-Boot reads prove that the Root Complex can issue MMIO transactions into the FPGA:

```text
ucm-imx8m-plus=> md.l 0x1c000000 10
1c000000: 00000000 ffffffff 00000000 ffffffff
1c000010: 00000000 ffffffff 00000000 ffffffff
1c000020: 00000000 ffffffff 00000000 ffffffff
1c000030: 00000000 ffffffff 00000000 ffffffff

ucm-imx8m-plus=> md.l 0x18000000 10
18000000: 00000010 454e4554 00000000 ffffffff
18000010: 00000002 00000000 00000004 0b0b0700
18000020: 40010000 00000002 ffffffff ffffffff
18000030: 00000000 00000000 ffffffff ffffffff

ucm-imx8m-plus=> md.l 0x1e000000 10
1e000000: 00000000 00000000 00000000 00000000
1e000010: 00000000 00000000 00000000 00000000
1e000020: 00000000 00000000 00000000 00000000
1e000030: 00000000 00000000 00000000 00000000
```

The structured values at `0x18000000` provide a particularly strong confirmation that the FPGA address space is reachable over PCIe.

Do not perform arbitrary writes until each BAR has been mapped back to the XDMA/Vivado AXI address map, because some registers may have side effects.

---

## 17. Bare-metal handoff notes

The earlier Intel-endpoint bare-metal test validated that U-Boot can initialize PCIe and hand off while leaving DBI and endpoint BAR MMIO reachable.

Example validated DBI reads:

```text
DBI vendor/device @ 0x33800000 = 0xabcd16c3
DBI command/status @ 0x33800004 = 0x00100007
DBI class/revision @ 0x33800008 = 0x06040001
DBI BAR0 @ 0x33800010 = 0x18000004
DBI bus numbers @ 0x33800018 = 0x00010100
```

Do not use a naive ECAM formula such as:

```c
PCIE_CFG_BASE + (bus << 20) + (dev << 15) + (func << 12) + offset
```

The i.MX8MP DesignWare config aperture used here is not a full simple ECAM window. With:

```text
PCIE_CFG_BASE = 0x1ff00000
```

a naive bus-1 calculation becomes `0x20000000`, outside the configured 512 KiB config aperture and can trigger a synchronous abort.

For the FPGA migration, U-Boot can perform enumeration and BAR setup; the bare-metal application can then access the assigned BAR base directly.

---

## 18. Current validated result and remaining work

The current UCM/VCU128 setup has validated all of the following:

```text
Custom U-Boot boots from SD using ALT_BOOT
i.MX8MP PCIe Root Complex initializes
PCIe PHY/REFCLK resources can be intentionally kept alive after an initial Link down
VCU128 can be programmed after U-Boot has already booted
Second PCI probe reuses the active resources instead of re-enabling them
XDMA reaches L0
Link comes up as Gen1 x1
Xilinx 10ee:9031 endpoint enumerates
All three VCU128 BAR sizes are detected correctly
A valid manual BAR packing fits inside the UCM PCI memory window
Direct MMIO reads from the UCM into the FPGA BARs succeed
```

The remaining PCIe firmware task is:

```text
Make the VCU128 BAR placement permanent so that no manual pci write commands are required.
```

The known-good target layout is:

```text
BAR2/3  64 MiB  @ 0x18000000
BAR0/1  32 MiB  @ 0x1c000000
BAR4/5  64 KiB  @ 0x1e000000
```

After that is integrated, the intended runtime sequence becomes:

```text
Boot U-Boot
pci enum                          # keeps host PCIe resources alive if FPGA is absent
Program VCU128
pci enum                          # reuses resources, reaches L0, enumerates FPGA
Automatic valid BAR assignment
Load/run bare-metal Cortex-A53 software
Access the FPGA/Ethernet IP through the assigned BARs
```

This is the foundation required to migrate the existing MicroBlaze-side Ethernet control software to the UCM Cortex-A53, with the vendor Ethernet IP registers reached through PCIe MMIO rather than local MicroBlaze AXI accesses.
