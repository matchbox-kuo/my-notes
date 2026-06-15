# Linux Kernel Study - Device Tree System (DTS)

<div style="background:linear-gradient(135deg, #1e3c72, #2a5298); color: white; padding: 20px; border-radius: 12px; margin-bottom: 25px; box-shadow: 0 4px 15px rgba(0,0,0,0.1);">
Device Tree 是嵌入式系統中描述硬體配置的數據結構。它將硬體資源的「描述」與核心的「邏輯實作」分離，使同一份作業系統核心能透過加載不同的 Device Tree Binary (DTB) 來支援多樣化的硬體平台。
</div>

## 章節目錄

1. [Device Tree 定義與必要性](#chapter-1)
2. [檔案格式：dts, dtsi, dtb](#chapter-2)
3. [核心匹配機制 (Matching Mechanism)](#chapter-3)
4. [DTS 語法結構](#chapter-4)
5. [關鍵屬性與資源描述](#chapter-5)
6. [驅動程式端的資源獲取 API](#chapter-6)
7. [除錯診斷與驗證](#chapter-7)

---

<a id="chapter-1"></a>

## 第一章：Device Tree 定義與必要性

### 1.1 背景
在嵌入式 Linux 環境中，許多非即插即用 (Non-discoverable) 的硬體元件（如 SoC 內部的 UART、I2C 控制器、GPIO 等）無法像 PCI 或 USB 裝置般進行自動枚舉 (Enumeration)。過去這類資訊硬編碼 (Hard-coded) 於核心原始碼中，導致代碼冗餘。

### 1.2 核心職能
Device Tree 提供了一種標準化的硬體拓撲描述機制，告知核心：
- **硬體實例**：系統中存在哪些控制器與外設。
- **資源分配**：包括暫存器位址 (Base Address)、中斷號 (IRQ)、時鐘 (Clock) 及復位訊號 (Reset)。
- **配置參數**：操作頻率、DMA 通道分配及引腳多路複用 (Pinmux) 設定。

**原則：DTS 負責硬體描述，Driver 負責邏輯實作。**

---

<a id="chapter-2"></a>

## 第二章：檔案格式：dts, dtsi, dtb

### 2.1 檔案類型定義
- **`*.dts` (Device Tree Source)**：Board-level描述文件。定義特定硬體載板的最終配置，通常包含 SoC 級別的 `dtsi` 並針對具體硬體進行客製化。
- **`*.dtsi` (Device Tree Source Include)**：共用片段文件。類似於 C 語言的標頭檔，用於定義 SoC 內部的通用 IP 控制器或多個平台共用的硬體描述。
- **`*.dtb` (Device Tree Blob)**：二進位目標文件。由編譯器產生，供 Bootloader (如 U-Boot) 加載並傳遞給核心解析。

### 2.2 編譯流程
使用 **DTC (Device Tree Compiler)** 進行格式轉換：
- **編譯 (Source to Binary)**：`dtc -I dts -O dtb -o board.dtb board.dts`
- **反編譯 (Binary to Source)**：`dtc -I dtb -O dts -o source.dts board.dtb`

---

<a id="chapter-3"></a>

## 第三章：核心匹配機制 (Matching Mechanism)

### 3.1 `compatible` 屬性
這是 Device Tree 最核心的屬性。核心透過匹配 `compatible` 字串來決定由哪個驅動程式接管該硬體Node。

**DTS Node示例：**
```dts
uart0: serial@1e784000 {
	compatible = "aspeed,ast2600-uart", "ns16550a";
};
```

**Driver 匹配表示例：**
```c
static const struct of_device_id my_uart_match[] = {
	{ .compatible = "aspeed,ast2600-uart" },
	{ }
};
MODULE_DEVICE_TABLE(of, my_uart_match);
```

### 3.2 流程概述
1. 核心解析 DTB 並實例化為 `device_node`。
2. 針對具備 `compatible` 屬性的Node創建 `platform_device`。
3. 核心總線 (Bus) 匹配機制觸發，將驅動與設備綁定 (Binding)。
4. 執行驅動程式的 `probe()` 函式。

---

<a id="chapter-4"></a>

## 第四章：DTS 語法結構

### 4.1 Node 定義與層級結構

Node 是 Device Tree 描述硬體物件的最小單位。整份 DTS 以樹狀結構組織硬體拓樸。

#### 4.1.1 Node 語法結構
```dts
label: node-name@unit-address {
    property-name = "value";
    child-node { ... };
};
```
- **Label**：節點標籤，用於在其他位置透過 `&label` 語法引用此節點（建立 phandle）。
- **Node-name**：節點名稱，描述硬體類別。
- **Unit-address**：裝置單元位址，通常與 `reg` 屬性的起始位址一致。

#### 4.1.2 樹狀結構示例
```dts
/ { // Root Node: 代表整個系統
    model = "My SoC Board";

    soc { // Bus/Container Node: 代表片上系統匯流排
        uart0: serial@1e784000 { // Device Node: 實際的硬體裝置
            compatible = "ns16550a";
            reg = <0x1e784000 0x1000>;
        };
    };
};
```

#### 4.1.3 Node 與 Device 的對照關係
需注意，**並非所有 Node 都會被核心實例化為驅動程式可匹配的 Device**：
- **可實例化裝置 (Probe-able Device)**：通常具備 `compatible` 屬性，且位於根目錄或 `simple-bus` 底下的 Node，核心會為其創建 `platform_device`。
- **配置/容器 Node**：
    - **匯流排/容器**：如 `soc` 或 `i2c@...`，主要作為管理與定址空間的容器。
    - **系統配置**：如 `chosen` (傳遞 Bootargs)、`memory` (描述 RAM 佈局) 或 `aliases`，僅供核心讀取參數，不會與驅動匹配。
    - **子裝置**：掛載於 I2C 或 SPI 匯流排下的 Node，會由所屬的總線驅動程式負責解析與實例化。

因此，精確的理解是：**Node 是硬體資源的描述單位，而 Device 是核心根據這些描述所建立的運行實體。**

### 4.2 屬性類別 (Property Types)
- **Strings**：雙引號字串，如 `status = "okay";`。
- **Cells**：角括號內的 32 位元整數，如 `reg = <0x1000 0x100>;`。
- **Phandles**：Node引用，如 `clocks = <&sysclk 5>;`。
- **Boolean**：空屬性代表 True，如 `dma-coherent;`。

### 4.3 Node 引用與屬性覆寫 (Overriding)

在 Board-level 開發中，最常見的模式是透過引用既有節點來修改其屬性，而不需重新定義整個硬體描述。

#### 4.3.1 引用語法
使用 `&label` 語法可直接定位到 `dtsi` 檔案中已定義的節點。

#### 4.3.2 實務示例
假設 `soc.dtsi` 已定義了 UART 控制器，Board-level `dts` 可進行如下覆寫：
```dts
#include "soc.dtsi"

&uart0 {
	status = "okay"; // 啟用原先在 dtsi 中為 disabled 的裝置
	current-speed = <115200>; // 額外增加或修改參數
};
```
這表示核心會先加載 `soc.dtsi` 的通用描述，再根據 `dts` 中的定義進行覆蓋或增補，最後合併生成完整的硬體描述樹。

---

<a id="chapter-5"></a>

## 第五章：關鍵屬性與資源描述

### 5.1 `reg` (Registers)

`reg` 屬性定義了裝置在系統中的位址資源（通常為 MMIO 空間）。

#### 5.1.1 基本格式與多組資源
裝置可擁有單組或多組暫存器區域：
```dts
mydev@10000000 {
	compatible = "vendor,mydev";
	reg = <0x10000000 0x1000>, // 第一組資源：控制暫存器
	      <0x10010000 0x1000>; // 第二組資源：數據緩衝區
	reg-names = "ctrl", "data";
};
```

#### 5.1.2 驅動端獲取方式
驅動程式可透過索引或名稱獲取對應的映射資源：
- **依索引獲取**：
  ```c
  ctrl = devm_platform_ioremap_resource(pdev, 0);
  data = devm_platform_ioremap_resource(pdev, 1);
  ```
- **依名稱獲取 (具備較佳可讀性)**：
  ```c
  ctrl = devm_platform_ioremap_resource_byname(pdev, "ctrl");
  data = devm_platform_ioremap_resource_byname(pdev, "data");
  ```

#### 5.1.3 定址長度規範 (Cells)
`reg` 屬性中每個數值（Cell）的數量並非固定，而是由其 **父 Node (Parent Node)** 的屬性決定：
- **`#address-cells`**：定義起始位址所需的 32-bit Cell 數量。
- **`#size-cells`**：定義長度範圍所需的 32-bit Cell 數量。

**範例：64-bit 定址系統**
若父 Node 宣告 `#address-cells = <2>; #size-cells = <2>;`，則 `reg` 會呈現如下格式：
```dts
reg = <0x0 0x10000000 0x0 0x1000>; // 每一組資源佔用 4 個 Cells
```
這確保了 Device Tree 能夠彈性支援不同位元寬度的處理器架構。

### 5.2 `interrupts` (Interrupts)
定義中斷源。具體格式取決於所屬的中斷控制器 (Interrupt Controller)。
```dts
interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
```

### 5.3 時鐘與復位 (Clocks & Resets)

在 SoC 開發中，時鐘與復位訊號通常是硬體 IP 正常運作的前提。

#### 5.3.1 `clocks` 與 `clock-names`
定義裝置所需的時鐘源。驅動端通常使用 `devm_clk_get()` 並搭配時鐘名稱來獲取：
```dts
clocks = <&sysclk 5>, <&apbclk 2>;
clock-names = "uart", "apb";
```

#### 5.3.2 `resets` (Reset Control)
描述裝置的復位線連接情況。許多 SoC IP 在開起時鐘後，仍需解除復位 (Deassert Reset) 才能正常存取暫存器。
```dts
resets = <&rst 10>;
reset-names = "uart";
```

#### 5.3.3 實務初始化順序
驅動程式在 `probe()` 階段的標準初始化流程建議如下：
1. **獲取時鐘** 並啟用 (`clk_prepare_enable`)。
2. **獲取復位控制** 並執行 **Deassert** (`reset_control_deassert`)。
3. 開始存取裝置暫存器。
4. (移除驅動時) 反向操作：**Assert Reset** 並 **Disable Clock**。

### 5.4 裝置綁定規範 (Binding)

`Binding` 是 Device Tree 的格式合約（規格書）。它規定了特定 `compatible` 裝置必須提供的屬性及其格式。目前核心開發趨勢是使用 **YAML** 格式進行定義，存放於 `Documentation/devicetree/bindings/`。

### 5.5 `status`

控制 Node 是否參與核心的實例化過程：
- `"okay"`：核心將為此 Node 創建設備並進行驅動匹配。
- `"disabled"`：核心將忽略此 Node，常用於在 `dtsi` 中宣告硬體但在 `dts` 中根據實際電路決定是否啟用。

---

<a id="chapter-6"></a>

## 第六章：驅動程式端的資源獲取 API

核心提供一系列 `of_*` 或 `devm_*` API 用於從 `device_node` 中提取資源：

| 資源 | 常用 API |
| :--- | :--- |
| **MMIO** | `devm_platform_ioremap_resource(pdev, index)` |
| **IRQ** | `platform_get_irq(pdev, index)` |
| **Clock** | `devm_clk_get(dev, "name")` |
| **Reset** | `devm_reset_control_get_exclusive(dev, "name")` |
| **Custom** | `of_property_read_u32(np, "prop-name", &out_val)` |

---

<a id="chapter-7"></a>

## 第七章：除錯診斷與驗證

### 7.1 運行時檢查
系統啟動後，可透過虛擬檔案系統檢查當前生效的 Device Tree：
- `/proc/device-tree/`：以目錄結構呈現的原始Node資訊。
- `/sys/firmware/devicetree/base/`：與核心內部數據結構對應的視圖。

### 7.2 常見失敗原因
1. **Compatible 字串不匹配**：導致驅動無法進入 `probe()`。
2. **Status 未設置為 "okay"**：Node被核心忽略。
3. **資源定義錯誤**：如 `reg` 範圍重疊或中斷號配置錯誤，導致資源獲取失敗。
4. **依賴關係缺失**：必要的時鐘或電源管理Node未正常初始化。

### 7.3 實用除錯技巧
- **DTB 反編譯**：若無法確定最終燒錄至系統的配置，可反編譯 `dtb` 檔案以確認 `include` 或 `override` 後的最終結果：
  ```bash
  dtc -I dtb -O dts -o debug.dts board.dtb
  ```
- **確認資源獲取**：若 `probe()` 失敗，應檢查核心日誌 (dmesg)，常見錯誤如 `failed to get clock` 或 `invalid IRQ` 通常指向 DTS 定義錯誤。

### 7.4 更新 DTB 的邏輯
修改 DTS 後，必須確保新的 `dtb` 被加載流程使用。根據平台架構的不同，可能需要：
- **獨立更新**：僅替換 Boot Partition 內的 `.dtb` 檔案。
- **重新打包**：若 DTB 被包裝在 FIT Image 或核心 Image 中，則需重新執行打包流程（如 Yocto 的 `bitbake`）。
