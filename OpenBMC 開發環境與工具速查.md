# OpenBMC 開發環境與工具速查

> 這份筆記整理 OpenBMC 開發時最常碰到的幾個主題：`Yocto`、`BitBake`、交叉編譯、QEMU、`ConfigFS`、`Kconfig` 與 `Makefile`。重點放在「它們是什麼、彼此怎麼接起來、實際開發時要先做什麼」。

---

## 目錄

- [1. OpenBMC 建置全貌](#1-openbmc-建置全貌)
- [2. Yocto 與 BitBake 是什麼](#2-yocto-與-bitbake-是什麼)
- [3. 編譯前要先準備什麼](#3-編譯前要先準備什麼)
- [4. AST2700 的異質交叉編譯觀念](#4-ast2700-的異質交叉編譯觀念)
- [5. QEMU 模擬與除錯實務](#5-qemu-模擬與除錯實務)
- [6. ConfigFS：用檔案系統外觀操作核心](#6-configfs用檔案系統外觀操作核心)
- [7. Kconfig 與 Makefile 的協作關係](#7-kconfig-與-makefile-的協作關係)

---

## 1. OpenBMC 建置全貌

OpenBMC 不是單純把一個應用程式編成執行檔，而是要一路產出：

- Bootloader，例如 `U-Boot`
- 安全韌體，例如 `TF-A`
- Linux Kernel
- Root File System
- 各種 BMC user-space 服務，例如 `bmcweb`、`phosphor-*`
- 最後可燒錄到 SPI Flash 的整包 image

可以把它想成一座自動化工廠：

```mermaid
flowchart LR
    A[原始碼與配方<br/>recipes layers patches] --> B[Yocto / BitBake]
    B --> C[下載原始碼<br/>do_fetch]
    B --> D[套用修補<br/>do_patch]
    B --> E[設定與編譯<br/>do_configure do_compile]
    B --> F[安裝與打包<br/>do_install do_rootfs do_image]
    F --> G[OpenBMC 映像檔]
```

---

## 2. Yocto 與 BitBake 是什麼

### Yocto 是什麼

`Yocto Project` 是 Linux Foundation 旗下的開源專案，用來建立嵌入式 Linux 發行版。它提供：

- 建置框架
- metadata 格式
- layer 機制
- cross toolchain 產生流程
- 可重複建置的工作模式

它不是 `.bat`，也不是單一 shell script。它比較像「一套用 metadata 驅動的發行版建置平台」。

### BitBake 是什麼

`BitBake` 是 Yocto 的任務引擎。你敲的通常是：

```bash
bitbake obmc-phosphor-image
```

它做的不是照順序把幾個命令跑完，而是：

1. 讀 recipe 與設定
2. 算出依賴圖
3. 決定哪些 task 要跑
4. 平行執行可並行的工作
5. 使用快取避免重編

### 一句話記法

- `Yocto`：整個建置生態與方法論
- `BitBake`：實際執行配方與任務的引擎

### Layer 是什麼

Yocto 最重要的概念之一就是 layer。每一層負責不同範圍：

| Layer 類型 | 典型內容 |
|:--|:--|
| `meta-phosphor` | OpenBMC 共通框架與服務 |
| `meta-aspeed` | ASPEED SoC/BSP 相關內容 |
| `meta-<vendor>` | 系統廠共用客製化 |
| `meta-<board>` | 特定機種、特定板子的設定 |

這種分層設計的好處是：你可以在自己的 layer 覆寫或追加設定，而不是直接魔改上游原始碼。

### `.bb` 與 `.bbappend` 是什麼

在 Yocto 裡，最常看到的兩種 metadata 檔案就是 `.bb` 與 `.bbappend`。

| 檔案類型 | 用途 |
|:--|:--|
| `.bb` | 定義一個 recipe，也就是「這個套件要怎麼抓、怎麼補 patch、怎麼編、怎麼安裝」 |
| `.bbappend` | 對既有 recipe 做追加或覆寫，不直接改原本那份 `.bb` |

可以把它們想成：

- `.bb`：原始食譜
- `.bbappend`：你自己加註的修改單

### `.bb` 會寫什麼

一個 `.bb` 通常會描述：

- 套件名稱與版本
- 原始碼從哪裡抓
- 要套哪些 patch
- 相依哪些套件
- 怎麼 configure / compile / install
- 最後安裝哪些檔案到 image

例如常會看到這類內容：

```conf
DESCRIPTION = "Example package"
LICENSE = "MIT"
SRC_URI = "git://example.com/project.git;branch=main"
S = "${WORKDIR}/git"

DEPENDS += "openssl"
```

BitBake 會依據這些內容去產生對應 task，例如 `do_fetch`、`do_patch`、`do_compile`、`do_install`。

### `.bbappend` 會寫什麼

當上游已經有某個 recipe，但你想針對自己平台加東西，通常就不要直接改原本 `.bb`，而是在自己的 layer 寫 `.bbappend`。

常見用途有：

- 追加 patch
- 補 device tree 檔案
- 改安裝內容
- 加入額外相依
- 覆寫某些變數

例如：

```conf
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://0001-fix-board-init.patch"
```

這表示：保留原 recipe，本 layer 額外再補一個 patch 進去。

### 為什麼 `.bbappend` 很重要

`.bbappend` 是 OpenBMC / Yocto 客製化最常用的做法，因為它有幾個好處：

- 不必直接改上游 layer
- 上游更新時比較容易跟進
- 客製內容集中在自己的 layer，比較好維護
- 同一份上游 recipe 可以被不同 board layer 各自追加設定

### 一句話記法

- `.bb`：定義這個套件本來怎麼 build
- `.bbappend`：在不改上游 recipe 的前提下，替它加料或微調

---

## 3. 編譯前要先準備什麼

很多新手以為 `bitbake` 是起點，其實它比較像「最後按下開始」的那一步。前面的環境準備才是建置能不能順利的關鍵。

### 3.1 載入環境

常見做法是先執行專案提供的初始化腳本，例如：

```bash
. setup ast2700-default
```

或是較標準的 Yocto 形式：

```bash
export TEMPLATECONF=meta-xxx/conf/templates/default
source oe-init-build-env build
```

這一步通常會完成：

- 建立 `build/` 工作目錄
- 匯入 BitBake/Yocto 需要的環境變數
- 準備 `conf/` 下的設定檔

### 3.2 `bblayers.conf`

`build/conf/bblayers.conf` 決定這次建置要載入哪些 layer。

BitBake 在啟動並解析環境時，就會先讀這個檔案。換句話說，當你執行：

```bash
bitbake obmc-phosphor-image
```

它很早就會用到 `bblayers.conf`，因為 BitBake 得先知道「要去哪些 layer 找 recipes、append、classes 與 machine 設定」。

最重要的內容通常是 `BBLAYERS`，例如：

```conf
BBLAYERS ?= " \
  /path/to/poky/meta \
  /path/to/poky/meta-poky \
  /path/to/openbmc/meta-phosphor \
  /path/to/openbmc/meta-aspeed \
  /path/to/meta-myboard \
"
```

你可以把它理解成這次建置的「資料來源清單」。

`bblayers.conf` 主要提供的資訊包括：

- 要載入哪些 layer
- 每個 layer 的實體路徑
- BitBake 可以去哪裡找 `.bb` recipes
- BitBake 可以去哪裡找 `.bbappend`
- 哪些 machine、distro、class、patch 與設定片段有機會被看到

它最實際的作用是：

- 決定哪些 recipe 看得見
- 決定哪些 `.bbappend` 會生效
- 決定你的自家 board layer 是否真的有被納入
- 決定 `MACHINE` 對應內容能不能被找到

如果缺 layer，常見症狀是：

- 找不到 recipe
- 找不到 machine 設定
- append 沒有生效

可以用一句話記：

- `bblayers.conf`：告訴 BitBake「去哪裡找東西」
- `local.conf`：告訴 BitBake「這次怎麼 build」

### 3.3 `local.conf`

`build/conf/local.conf` 比較像這台 build machine 的在地設定。最常看的幾項：

| 參數 | 用途 |
|:--|:--|
| `MACHINE` | 指定這次要替哪一塊板子建置，例如選哪份 machine 設定、device tree、kernel 設定與 image 組合 |
| `DL_DIR` | 指定下載快取目錄，集中存放 `do_fetch` 抓回來的原始碼、壓縮檔與 git mirror，避免每次換 build 目錄都重新下載 |
| `SSTATE_DIR` | 指定 shared state cache 目錄，保存可重用的中間建置成果，讓沒變動的 recipe 不必每次從頭重編 |
| `BB_NUMBER_THREADS` | 控制 BitBake 可同時執行多少個 task，影響整體平行建置效率 |
| `PARALLEL_MAKE` | 傳給底層編譯器的平行編譯參數，例如 `make -j` 的行為，影響單一 recipe 的編譯速度 |

其中 `DL_DIR` 與 `SSTATE_DIR` 特別重要，它們不是編譯指令，而是 Yocto 建置時最關鍵的兩個快取目錄：

- `DL_DIR` 比較像原料倉庫，負責保存下載回來的 source
- `SSTATE_DIR` 比較像半成品倉庫，負責保存可重用的建置成果

如果這兩個目錄沒有妥善規劃，常見結果就是：

- 每次重建 build 目錄都重新下載大量原始碼
- 改一小部分內容卻要重等很久
- 不同 build 目錄或不同使用者之間無法共享快取成果

### 3.4 常見修改入口

OpenBMC 專案裡，常見的客製方式不是直接改上游，而是：

- 在自己的 layer 新增 recipe
- 寫 `.bbappend`
- 加 patch
- 覆寫 device tree
- 調整 kernel config

例如：

```text
meta-myboard/
  recipes-kernel/
    linux/
      linux-aspeed_%.bbappend
      files/
        0001-my-fix.patch
        ast2700-myboard.dts
```

---

## 4. AST2700 的異質交叉編譯觀念

AST2700 不是「三顆 CPU 就要手動編三次」的意思，而是「建置系統要替不同 CPU 類型安排不同 toolchain 與產物」。

### 4.1 核心觀念

- `Cortex-A35`：64-bit，通常跑 Linux
- `Cortex-A32`：32-bit，常見用於 RTOS 或 bare-metal 任務

所以會牽涉不同交叉編譯器，例如：

- `aarch64-linux-gnu-gcc`
- `arm-none-eabi-gcc`

### 4.2 不是你手動編三次

通常情況下，你仍然只下類似一條主命令：

```bash
bitbake obmc-phosphor-image
```

接著由 Yocto/BitBake 自動安排：

```mermaid
flowchart TD
    A[bitbake obmc-phosphor-image] --> B[A32 韌體建置]
    A --> C[A35 Linux 與 OpenBMC 建置]
    B --> D[A32 產物<br/>elf bin]
    C --> E[A35 產物<br/>kernel rootfs services]
    D --> F[整合打包]
    E --> F
    F --> G[最終燒錄 image]
```

### 4.3 開機後怎麼接上

典型流程是：

1. A35 端 Linux 先開機
2. Linux 從檔案系統找 A32 韌體
3. 透過 `remoteproc` 載入到指定記憶體
4. 啟動 A32 核心執行對應工作

所以「多核心、多架構」在建置時是多套產物，在使用者操作上通常還是一套整合流程。

---

## 5. QEMU 模擬與除錯實務

QEMU 常拿來先驗證 boot flow、console、服務啟動與基本行為。

### 基本流程

1. 啟動 QEMU 後，觀察 `char device redirected to /dev/pts/X`
2. 用 `tio /dev/pts/X` 連上虛擬序列埠
3. 若 QEMU 停在等待狀態，在 QEMU 視窗輸入 `c`
4. 使用 `root / 0penBmc` 登入

### 適合用 QEMU 先驗證的事

- 開機流程是否正常
- systemd 服務是否起得來
- 基本 shell / log 行為
- 某些 user-space 程式是否能運作

### 不要對 QEMU 期待過高的事

- 真實 SoC 周邊時序
- 實體 PCIe/I2C/GPIO 細節
- 板級硬體互動問題

---

## 6. ConfigFS：用檔案系統外觀操作核心

`ConfigFS` 看起來像檔案系統，但它的重點不是存資料，而是把核心物件與控制介面「包裝成目錄和檔案」給使用者操作。

### 6.1 它和 ext4 / NTFS 的差別

| 項目 | ext4 / NTFS | ConfigFS |
|:--|:--|:--|
| 主要目的 | 儲存資料 | 操作核心物件 |
| 資料位置 | 磁碟 | RAM |
| 重開機後 | 保留 | 消失 |
| 建立 `mkdir` 的效果 | 建空目錄 | 建立核心中的一個物件 |

### 6.2 用在 PCIe Endpoint 時的理解方式

把 ConfigFS 想成「控制面板」最容易理解：

- `mkdir`：新增一個功能物件
- `echo ... > vendorid`：改這個物件的屬性
- `echo 1 > start`：通知核心啟動對應硬體流程

### 6.3 指令背後真正發生的事

```bash
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func1
```

這不是單純建資料夾，而是要求 kernel 建立一個 PCIe EP function 物件。

```bash
echo 0x1987 > vendorid
```

這不是寫文字到普通檔案，而是在改核心裡裝置結構的欄位值。

```bash
echo 1 > start
```

這通常會觸發核心 callback，讓 EP 控制器真的開始工作。

### 6.4 開發上要記住的一句話

ConfigFS 的操作表面上像檔案 IO，本質上其實是在「呼叫核心邏輯」。

---

## 7. Kconfig 與 Makefile 的協作關係

Linux 核心與很多 OpenBMC 底層元件的建置，都依賴 `Kconfig` 與 `Makefile` 的組合。

### 最簡單的理解

- `Kconfig`：定義有哪些功能可選、彼此相依怎麼算
- `make menuconfig`：提供互動式設定介面
- `.config`：最後的設定結果
- `Makefile`：依據 `.config` 決定哪些檔案要編譯

### 可以把它想成

| 元件 | 比喻 |
|:--|:--|
| `Kconfig` | 菜單 |
| `make menuconfig` | 點餐過程 |
| `.config` | 訂單 |
| `Makefile` | 廚房照單出菜 |

### 實際關係

如果某功能在 `.config` 裡是：

```text
CONFIG_MCTP=y
```

那對應 `Makefile` 可能會有：

```make
obj-$(CONFIG_MCTP) += mctp.o
```

意思是：只有在這個功能被打開時，`mctp.o` 才會被納入編譯。

---

## 速記總結

- `Yocto` 是建置平台，`BitBake` 是任務引擎。
- OpenBMC 的建置不是單編程式，而是整個韌體映像的組裝流程。
- AST2700 屬於異質多核心平台，但通常仍由同一套建置流程統一產出。
- `DL_DIR` 和 `SSTATE_DIR` 決定你是在高效率開發，還是在反覆重煉地獄。
- `ConfigFS` 看似檔案系統，實際上是核心控制介面。
- `Kconfig` 決定能選什麼，`Makefile` 決定真的編什麼。
