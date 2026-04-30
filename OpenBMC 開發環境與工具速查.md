# 🛠️ OpenBMC 開發環境與工具速查

> 本文件摘錄自《AST2700 PCIe 韌體開發與 OpenBMC 實務手冊》第四、五、六章，涵蓋 Yocto/WSL 編譯配置、QEMU 模擬流程與常用指令速查。

---

## 目錄

- [一、 開發環境建置與優化 (WSL + Yocto)](#一-開發環境建置與優化-wsl--yocto)
  - [1.1 編譯配置建議](#11-編譯配置建議)
  - [1.2 繞過網路限制策略](#12-繞過網路限制策略)
- [二、 QEMU 模擬與除錯實務](#二-qemu-模擬與除錯實務)
- [三、 常用指令速查表](#三-常用指令速查表)
- [四、 ConfigFS 虛擬檔案系統：PCIe EP 設定的控制面板](#四-configfs-虛擬檔案系統pcie-ep-設定的控制面板)
  - [4.1 存放位置與生命週期](#41-存放位置與生命週期)
  - [4.2 存在的目的](#42-存在的目的)
  - [4.3 指令背後的真實意義](#43-指令背後的真實意義pcie-ep-驗證關鍵)

---


### 1.1 編譯配置建議

編譯 AST2700 映像檔 (obmc-phosphor-image) 非常消耗資源：

* **16GB RAM 環境**：務必在 `local.conf` 限制執行緒，如 `BB_NUMBER_THREADS = "4"` 以避免 OOM 錯誤。
* **32GB RAM 環境**：可調升至 12 執行緒以上大幅縮短編譯時間。

### 1.2 繞過網路限制策略

1. 在有網路的環境執行 `bitbake obmc-phosphor-image --runall=fetch`。
2. 將 `build/ast2700-default/downloads` 目錄完整搬移至受限環境。
3. 在受限環境執行編譯，BitBake 會優先使用本地快取的原始碼。


## 🐛 二、 QEMU 模擬與除錯實務

QEMU 模擬 AST2700 虛擬硬體，流程如下：

1. 執行啟動指令後，觀察 `char device redirected to /dev/pts/X` 訊息。
2. 使用 `tio /dev/pts/X` 連接序列埠 Console。
3. 在 QEMU 視窗輸入 `c` 啟動系統執行。
4. 登入憑據：帳號 `root` / 密碼 `0penBmc`。


## 📝 三、 常用指令速查表

* \# 初始化編譯環境
  `. setup ast2700-default`
* \# 執行編譯
  `bitbake obmc-phosphor-image`
* \# 修改原始碼 (以 obmc-ikvm 為例)
  `devtool modify obmc-ikvm`
* \# 連接虛擬串口
  `tio /dev/pts/2`

---

## 🔧 四、 ConfigFS 虛擬檔案系統：PCIe EP 設定的控制面板

FAT、NTFS、ext4 是我們熟知的**「實體檔案系統（Disk-based File System）」**；而 ConfigFS（以及 Linux 常見的 `sysfs`、`procfs`）則屬於**「虛擬檔案系統（Virtual / Pseudo File System）」**。

它們雖然都叫「檔案系統」，也都可以用 `ls`、`cd`、`mkdir` 來操作，但本質與用途有天壤之別。

### 4.1 存放位置與生命週期

| | 實體檔案系統（FAT / NTFS / ext4）| 虛擬檔案系統（ConfigFS）|
|:--|:--|:--|
| **儲存位置** | 實體硬碟 / SSD / 隨身碟 | 完全存在於 **RAM（記憶體）** 中 |
| **生命週期** | 斷電後資料仍保留 | **重開機或斷電即消失** |
| **由誰建立** | 格式化時寫入磁碟結構 | Linux 核心**開機後動態生成** |

> [!IMPORTANT]
> 這也是為什麼凡是透過 ConfigFS 設定的 PCIe EP 功能，**每次開機都必須重跑設定腳本**——所有設定都存在記憶體中，重開機後一片空白。

### 4.2 存在的目的

- **實體檔案系統**：目的是「**儲存資料**」。就像一個檔案櫃——把文件放進去，下次需要時再拿出來。

- **ConfigFS**：目的是提供一個**「讓 User-space（使用者空間）控制 Kernel-space（核心空間）的操作面板」**。

  Linux 核心為了讓工程師不需要撰寫 C 語言程式就能動態設定底層硬體，便把「設定開關」**偽裝成「資料夾與檔案」的外形**，讓你用熟悉的 Shell 指令去操作。

### 4.3 指令背後的真實意義（PCIe EP 驗證關鍵）

這是理解 ConfigFS 最關鍵的一點——**相同的指令，在不同的檔案系統下，觸發的是截然不同的動作：**

#### mkdir：建資料夾 vs. 實體化核心物件

```bash
# 在 NTFS / ext4 下
mkdir new_folder
# → 在硬碟上劃分一小塊空間，記錄「這裡有一個空目錄」。僅此而已。

# 在 ConfigFS 下
mkdir /sys/kernel/config/pci_ep/functions/pci_epf_test/func1
# → 這不是在建空資料夾！
#   Linux 核心攔截到這個動作後，會在底層呼叫 pci_epf_test 模組的回呼函數，
#   在系統記憶體中真正分配並實體化出一個「虛擬 PCIe Endpoint Function 物件」。
```

#### echo：寫入字元 vs. 設定核心資料結構

```bash
# 在 NTFS / ext4 下
echo "1987" > file.txt
# → 把四個字元寫進硬碟磁區保存。

# 在 ConfigFS 下
echo 0x1987 > vendorid
# → 觸發核心執行一段程式碼，
#   把剛剛生出來的那個虛擬 PCIe 裝置結構體裡的 vendor_id 變數賦值為 0x1987。

echo 1 > start
# → 觸發核心呼叫 Skeleton EPC driver 裡的 start() callback 函數，
#   讓 PCIe EP 硬體開始實際運作。
```

#### 整體對照

| Shell 指令 | 在實體 FS 的效果 | 在 ConfigFS 的效果 |
|:--|:--|:--|
| `mkdir func1` | 建立空目錄 | **實體化一個 PCIe EPF 核心物件** |
| `echo 0x1987 > vendorid` | 寫入字串到檔案 | **設定 Vendor ID 到裝置結構體** |
| `echo 1 > start` | 寫入字元「1」 | **呼叫 EPC driver 的 start() 啟動硬體** |
| `rmdir func1` | 刪除空目錄 | **釋放 PCIe EPF 物件、停止 EP 功能** |

> [!NOTE]
> 你可以把 ConfigFS 想像成一台機器（Linux 核心）的「**控制面板**」：上面的每一個「資料夾」都是一個「可擴充的模組插槽」，每一個「檔案」都是一個「旋鈕或開關」。我們只是借用了「檔案系統」這層外皮，達到設定底層系統與硬體的目的——這正是 AST2700 能夠在不燒錄任何韌體的情況下，透過幾行 Shell 指令就讓 Host 看見一張全新虛擬 PCIe 裝置的根本原因。

---

## 五、 Kconfig 與 Makefile 的協作關係

Linux 核心原始碼目錄下的各個子資料夾中都散佈著 `Kconfig` 檔案。當開發者或系統管理員準備編譯核心（例如執行 `make menuconfig`）時，建構系統就會收集這些分散的 `Kconfig`，組合出龐大且階層分明的選單介面，讓使用者可以輕鬆勾選想要納入的硬體驅動或系統功能。

在 Linux Kernel (包含 OpenBMC 底層) 的編譯系統中，`Kconfig` 與 `Makefile` 扮演著不同的角色，我們可以將其比喻為「點餐系統」與「廚房」：

1. **`Kconfig` (菜單)**：定義了有哪些功能（菜色）可以開啟、模組化或關閉，以及它們之間的相依性。
2. **`make menuconfig` (點餐過程)**：讀取 `Kconfig` 產生互動式圖形/文字介面，讓開發者勾選需要的功能。
3. **`.config` (訂單)**：點餐完成後，系統會生成這個隱藏檔案，記錄所有的設定值（例如 `CONFIG_MCTP=y`）。
4. **`Makefile` (廚房)**：`Makefile` 程式碼本身通常不需要修改，它會去讀取 `.config` (訂單) 的內容，並根據裡面的變數 (如 `obj-$(CONFIG_MCTP)`) 來決定要編譯哪些 `.c` 原始碼檔案成 `.o` 或是 `.ko` 核心模組。

**總結**：`Kconfig` 負責提供設定介面並產出 `.config` 訂單，而 `Makefile` 負責根據訂單精準執行編譯，這種設計讓超級龐大的專案也能完美解耦。



