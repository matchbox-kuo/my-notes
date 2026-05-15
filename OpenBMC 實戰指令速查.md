# OpenBMC 實戰指令速查

> 這份文件只整理實際操作時會敲的命令，來源以 Mercury FPGA / QEMU / U-Boot / Linux 的現行流程為主。說明偏短，目標是拿來直接查指令。

---

## 目錄

- [1. 版本參考](#1-版本參考)
- [2. 環境初始化](#2-環境初始化)
- [3. Yocto / OpenBMC 建置](#3-yocto--openbmc-建置)
- [4. 使用本地 Linux / U-Boot 原始碼](#4-使用本地-linux--u-boot-原始碼)
- [5. QEMU 啟動與登入](#5-qemu-啟動與登入)
- [6. 更新 Linux defconfig](#6-更新-linux-defconfig)
- [7. PCIe EP 研究前確認](#7-pcie-ep-研究前確認)

---

## 1. 版本參考

### Git commit

- `openbmc-private`: `84885b18b0c5d73558b263c4957f8e03873b0107`
- `linux-private`: `20f99efa6d5d30cd85c41c507eea867aa9b192ad`
- `u-boot-private`: `f47891740787dccfdca693da6f3ec866705af72e`

### Repository / branch

- `openbmc-private`: `meta-jmicron` branch `jm-master`
- `u-boot-private`: branch `jmicron-v2025.01`
- `linux-private`: branch `jmicron-v6.18`

---

## 2. 環境初始化

### 啟用 SSH key

```bash
eval $(ssh-agent)
ssh-add /path/to/your-github-ssh.pem
```

### 建立 build 目錄

```bash
mkdir -p build
```

### 使用 Poky distro

```bash
TEMPLATECONF=meta-jmicron/meta-mercury-poky/conf/templates/default source oe-init-build-env build/mercury-fpga-poky
```

### 使用 OpenBMC phosphor distro on QEMU

```bash
. setup mercury-qemu-webui
```

### 使用 OpenBMC phosphor distro on FPGA

```bash
. setup mercury-fpga-webui
```

---

## 3. Yocto / OpenBMC 建置

### 建置 initramfs image

```bash
bitbake fpga-image-initramfs
```

### 建置含 webui 的 initramfs image

```bash
bitbake fpga-image-initramfs-webui
```

---

## 4. 使用本地 Linux / U-Boot 原始碼

以下內容是 `build/mercury-fpga/conf/local.conf` 常見設定。

### 設定 machine

```conf
MACHINE = "mercury-fpga-poky"
```

### 改用本地 U-Boot / Linux repo

```conf
INHERIT += "externalsrc"
EXTERNALSRC:pn-u-boot-jmicron-dev = "/path/to/u-boot-private"
EXTERNALSRC_BUILD:pn-u-boot-jmicron-dev = "/path/to/u-boot-private/build"
EXTERNALSRC:pn-linux-jmicron-dev = "/path/to/linux-private"
EXTERNALSRC_BUILD:pn-linux-jmicron-dev = "/path/to/linux-private/build"
```

---

## 5. QEMU 啟動與登入

### 指定映像路徑

```bash
U_BOOT_BIN="/home/matchbox/workspace/openbmc-private/build/mercury-qemu-webui/tmp/deploy/images/mercury-qemu-webui/u-boot.bin"
LINUX_FIT="/home/matchbox/workspace/openbmc-private/build/mercury-qemu-webui/tmp/deploy/images/mercury-qemu-webui/fitImage-qemu-image-initramfs-webui-mercury-qemu-webui-mercury-qemu-webui"
```

### 建立測試用 NVMe 映像檔

```bash
truncate -s 64M /tmp/mercury-nvme-test.img
```

### 啟動 QEMU

```bash
/home/matchbox/workspace/qemu-private/build/qemu-system-aarch64 -s -M jm-mercury-fpga -smp cpus=1 -nographic \
    -serial mon:stdio \
    -device loader,addr=0x80000000,cpu-num=0 \
    -device "loader,file=$U_BOOT_BIN,addr=0x80000000,force-raw=on" \
    -device "loader,file=$LINUX_FIT,addr=0x88000000,force-raw=on" \
    -drive file=/tmp/mercury-nvme-test.img,if=none,id=nvme0,format=raw \
    -device nvme,drive=nvme0,serial=mercury-nvme-0
```

### 在 U-Boot prompt 啟動 Linux

```bash
bootm
```

預設 `bootcmd`：

```text
bootm 0x88000000#conf-mercury_fpga_vu9p_mercury_ap.dtb
```

### Linux 登入

```text
user: root
password: 0penBmc
```

---

## 6. 更新 Linux defconfig

### 開啟 menuconfig

```bash
bitbake linux-jmicron-dev -c menuconfig
```

### 產生精簡 defconfig

```bash
bitbake linux-jmicron-dev -c savedefconfig
```

### 覆蓋 layer 內的 defconfig

```bash
cp /path/to/linux-private/build/defconfig meta-jmicron/recipes-kernel/linux/files/defconfig
```

> 手動直接改最終 `.config` 通常無法正確處理相依，建議走 `menuconfig` + `savedefconfig` 流程。

---

## 7. PCIe EP 研究前確認

### 目前 defconfig / DTS 狀態重點

- `mercury-qemu-webui` 使用的 kernel config 來源是 `meta-jmicron/recipes-kernel/linux/files/defconfig`
- 目前建置結果顯示 `CONFIG_PCI is not set`
- 目前產生的 `mercury-qemu.dts` 沒有 PCIe node
- 因此 Linux 開機後 `/sys/bus/pci` 不會存在

### 先打開這些 kernel option

```text
CONFIG_PCI=y
CONFIG_PCI_MSI=y
CONFIG_PCI_ENDPOINT=y
CONFIG_PCI_ENDPOINT_CONFIGFS=y
CONFIG_PCI_EPF_TEST=m
CONFIG_PCI_ENDPOINT_TEST=m
```
