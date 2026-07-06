# Kria K26 Custom BSP Build Instruction (AMD EDF)

This document describes the detailed steps to build a custom BSP for a Kria K26 carrier board using a `.xsa` hardware description file and the AMD Embedded Development Framework (EDF).

## 1. Prerequisites

- Linux host with required Yocto/EDF dependencies installed.
- Vivado/Vitis installed and available for `stdgen` / `gen-machine-conf`.
- XSA hardware file for the Kria K26 custom board.
- Yocto `repo` tool installed.
- EDF repository checked out or in the process of being initialized.

## 2. Prepare the EDF Build Workspace

```bash
mkdir -p ~/xilinx/edf-build
cd ~/xilinx/edf-build

repo init -u https://github.com/Xilinx/yocto-manifests.git \
          -b refs/tags/amd-edf-rel-v25.11 \
          -m default-edf.xml
repo sync
```

## 3. Initialize the EDF Build Environment

```bash
source ./edf-init-build-env
```

This command changes your working directory into the build workspace and prepares the `conf/` files.

## 4. Generate System Device Tree (SDT) from the Kria K26 XSA

Source the Xilinx toolchain, then generate the SDT from the `.xsa` file.

```bash
source /home/<user>/tools/Xilinx/2025.2/Vitis/settings64.sh
mkdir -p ~/xilinx/kria-k26-sdt
cd ~/xilinx/kria-k26-sdt
stdgen
```

Inside the `stdgen` prompt:

```txt
stdgen% set_dt_param -dir ./sdt
stdgen% set-dt-param -xsa /path/to/your/kria-k26-design.xsa
stdgen% generate_sdt
stdgen% exit
```

Confirm output:

```bash
ls -lh ~/xilinx/kria-k26-sdt/sdt
```

## 5. Create and Register a Custom BSP Layer

```bash
cd ~/xilinx/edf-build
bitbake-layers create-layer ../sources/meta-custom-kria-k26-bsp
bitbake-layers add-layer ../sources/meta-custom-kria-k26-bsp
rm -rf ../sources/meta-custom-kria-k26-bsp/recipes-example/
```

## 6. Generate a Custom Machine Configuration

Use `gen-machine-conf` to create a machine config from the generated SDT.

```bash
cd ~/xilinx/edf-build
gen-machine-conf parse-sdt \
    --hw-description ~/xilinx/kria-k26-sdt/sdt \
    -c ../sources/meta-custom-kria-k26-bsp/conf \
    --machine-name k26-custom
```

If your version of EDF supports direct XSA input, you may use:

```bash
gen-machine-conf --hw-description /path/to/your/kria-k26-design.xsa \
    -c ../sources/meta-custom-kria-k26-bsp/conf \
    --machine-name k26-custom
```

## 7. Create a Custom Device Tree Layer

```bash
cd ~/xilinx/edf-build
bitbake-layers create-layer ../sources/meta-kria-k26-device-tree
bitbake-layers add-layer ../sources/meta-kria-k26-device-tree
mkdir -p ../sources/meta-kria-k26-device-tree/recipes-bsp/device-tree/files
```

Create `../sources/meta-kria-k26-device-tree/recipes-bsp/device-tree/device-tree.bbappend` with:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
EXTRA_DT_INCLUDE_FILES:append:linux = " system-user.dtsi"
EXTRA_DT_INCLUDE_FILES:append:linux-gnueabi = " system-user.dtsi"
```

Create `../sources/meta-kria-k26-device-tree/recipes-bsp/device-tree/files/system-user.dtsi` with the board-specific overlay changes, for example:

```dts
/ {
    chosen {
        bootargs = "clk_ignore_unused";
    };
};
```

## 8. Configure `conf/local.conf`

Open `conf/local.conf` and add or update these entries:

```bitbake
DL_DIR = "${HOME}/xilinx/edf-pkg/downloads"
SSTATE_DIR = "${HOME}/xilinx/edf-pkg/sstate-cache"
SANITY_TEST_SUPERCEDED = "1"
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
MACHINE = "k26-custom"
BBMULTICONFIG += "fsbl-fw"
```

If your board requires a prebuilt FSBL instead of building one in Yocto, add:

```bitbake
FSBL_FILE = "/path/to/prebuilt/fsbl-k26-custom.elf"
```

If BitBake reports that `virtual/dtb` is missing, define the device tree variable:

```bitbake
CONFIG_DTFILE = "k26-custom"
```

or, for an explicit system device tree:

```bitbake
SYSTEM_DTFILE = "/path/to/your/system-top.dts"
```

## 9. Optional: Kernel and U-Boot Configuration

To configure the Linux kernel:

```bash
MACHINE=k26-custom bitbake -c menuconfig linux-xlnx
```

To configure U-Boot:

```bash
MACHINE=k26-custom bitbake -c menuconfig u-boot-xlnx
```

## 10. Build the BSP Images

Build the boot binaries and full Linux image.

```bash
MACHINE=k26-custom bitbake xilinx-bootbin
MACHINE=k26-custom bitbake kria-image-full-cmdline
```

If your custom board is best served by the generic EDF Linux image instead, use:

```bash
MACHINE=k26-custom bitbake edf-linux-disk-image
```

## 11. Verify Output Artifacts

Inspect the deploy directory:

```bash
ls -lh tmp/deploy/images/k26-custom/
```

Common output files:

- `boot.bin`
- `Image`
- `devicetree.dtb`
- `kria-image-full-cmdline-k26-custom.wic`
- `edf-linux-disk-image-k26-custom.rootfs.wic`

## 12. Flash the WIC Image to SD Card

```bash
sudo dd if=tmp/deploy/images/k26-custom/kria-image-full-cmdline-k26-custom.wic of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

Replace `/dev/sdX` with your actual SD card device.

## 13. Notes and Troubleshooting

- If `bitbake` reports `Nothing PROVIDES 'virtual/fsbl'`, ensure `BBMULTICONFIG += "fsbl-fw"` or `FSBL_FILE` is set.
- If `virtual/dtb` is missing, confirm `CONFIG_DTFILE` or `SYSTEM_DTFILE` is defined.
- Use the generated machine name from `gen-machine-conf` consistently in `MACHINE=` and `conf/local.conf`.
- Keep `stdgen` and `gen-machine-conf` output directories clean and point to the correct `.xsa` / `sdt` paths.
- If you choose `kria-image-full-cmdline`, the deploy directory and recipe dependencies are specific to Kria boards.

---

This instruction set is based on the AMD EDF custom BSP flow shown in `zynq_memo.txt`, adapted for a Kria K26 carrier board with a `.xsa` hardware description.
