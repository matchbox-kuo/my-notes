## 第三章：Driver 常用工具與 DMA 基礎

<div style="background:#fff4e8; border-left: 4px solid #f0a35e; padding: 10px 14px; border-radius: 6px;">
這一章整理 driver 實作時很常碰到的幾個主題：debug 訊息、register field 操作、DMA buffer 與 streaming DMA 的基本觀念。
</div>

### 3.1 Driver 怎麼印 debug 訊息

寫 driver 時，最常見的 debug 手段之一就是印 log。

初學先記這幾個最常用的就夠了：

- `printk()`
- `pr_info()`
- `pr_err()`
- `dev_info()`
- `dev_err()`
- `dev_dbg()`

#### `printk()`

這是最底層、最通用的 kernel log function，概念上很像 kernel world 的 `printf()`。

```c
printk(KERN_INFO "Mercury EP started\n");
```

不過在 driver 裡，現在通常更常用 `pr_*()` 或 `dev_*()` 這類包好的介面。

#### `pr_*()` 系列

這是一組比較方便的全域 log 巨集，例如：

- `pr_info()`
- `pr_warn()`
- `pr_err()`
- `pr_debug()`

```c
pr_err("failed to init Mercury EP\n");
```

這類寫法適合沒有特定 `struct device *dev` 可以掛訊息的情況。

#### `dev_*()` 系列

如果你手上有 `struct device *dev`，通常更推薦用：

- `dev_info(dev, ...)`
- `dev_warn(dev, ...)`
- `dev_err(dev, ...)`
- `dev_dbg(dev, ...)`

```c
dev_info(dev, "registered JMicron Mercury EP controller\n");
dev_err(dev, "failed to map registers\n");
dev_dbg(dev, "MSI phys addr = %pa\n", &ep->msi_phys_addr);
```

這類好處是 log 會自帶裝置脈絡，之後看訊息時比較容易知道是哪一個 device 印的。

#### 初學先怎麼選

可以先這樣記：

- 想快速印 kernel 訊息：`pr_info()` / `pr_err()`
- 在 driver 裡而且手上有 `dev`：優先用 `dev_info()` / `dev_err()`
- 想放比較多除錯訊息：用 `dev_dbg()`

<div style="background:#e8f7e8; border-left: 4px solid #6bbf73; padding: 10px 14px; border-radius: 6px;">
driver debug 最常用的是 `pr_*()` 和 `dev_*()`；如果手上有 `struct device *dev`，通常優先用 `dev_info()`、`dev_err()`、`dev_dbg()`。
</div>

---

### 3.2 register field 操作推薦用哪些

driver 很常需要讀寫 register，但實務上真正容易出錯的，常常不是整顆 register，而是其中某一個 bit 或某一段 field。

#### 單一 bit：`BIT(n)`

單一開關位元，通常用 `BIT(n)`：

```c
#define CTRL_ENABLE   BIT(0)
#define CTRL_INT_EN   BIT(3)
```

#### 連續 field：`GENMASK(h, l)`

一段連續 bit 代表模式、大小、狀態碼時，通常用 `GENMASK(high, low)`：

```c
#define CTRL_MODE_MASK   GENMASK(3, 1)
#define CTRL_SPEED_MASK  GENMASK(7, 4)
```

#### 寫入 field：`FIELD_PREP(mask, val)`

`FIELD_PREP()` 是把值準備成可以放進指定 field 的格式；它本身不會寫 register。

```c
val &= ~CTRL_MODE_MASK;
val |= FIELD_PREP(CTRL_MODE_MASK, mode);
```

#### 讀出 field：`FIELD_GET(mask, reg)`

```c
u32 mode = FIELD_GET(CTRL_MODE_MASK, val);
```

#### 如果你自己比較習慣 `SET_FIELD()`

如果你自己寫筆記或專案 helper，覺得每次手寫 `& ~mask` 和 `| FIELD_PREP(...)` 很煩，也可以自己包一層。

<div style="background:#ffe3ee; border-left: 4px solid #e26d93; padding: 10px 14px; border-radius: 6px;">
`SET_FIELD()` 不是 Linux kernel 內建標準巨集。它是你自己額外加的 helper；看別人的 kernel code 時，還是更常看到原生的 `FIELD_PREP()` 寫法。
</div>

```c
#define SET_FIELD(reg, mask, val) \
	(((reg) & ~(mask)) | FIELD_PREP((mask), (val)))
```

```c
reg = SET_FIELD(reg, CTRL_MODE_MASK, mode);
```

如果你希望讀取時也維持同樣語意，可以再自己包：

```c
#define GET_FIELD(reg, mask) \
	FIELD_GET((mask), (reg))
```

#### 一個常見、可讀性高的寫法

```c
#define REG_CTRL         0x00
#define CTRL_ENABLE      BIT(0)
#define CTRL_MODE_MASK   GENMASK(3, 1)

u32 val;

val = readl(base + REG_CTRL);
val |= CTRL_ENABLE;
val = SET_FIELD(val, CTRL_MODE_MASK, mode);
writel(val, base + REG_CTRL);
```

#### 不太推薦的寫法

- 直接寫 `0x40`、`0x8000` 這種 magic number
- 到處手寫 `(val >> 4) & 0x7` 卻沒有命名 field
- 寫 register 時整顆覆蓋，卻沒有先保留其他 bit

<div style="background:#e8f7e8; border-left: 4px solid #6bbf73; padding: 10px 14px; border-radius: 6px;">
如果是 Linux driver 裡的 register field 操作，最推薦先認得 `BIT()`、`GENMASK()`、`FIELD_PREP()`、`FIELD_GET()`；如果你想讓自己寫的 code 更順手，再額外包 `SET_FIELD()`。
</div>

---

### 3.3 Non-cache or HW-coherent Buffer DMA

這一節先只看一般使用情況下最常碰到的 coherent DMA buffer，也就是給 device 使用的一塊完整連續空間。

- non-coherent buffer 先不在這一節討論
- 平台到底是怎麼把 coherent 語意實作出來的，也先不在這一節展開
- 這裡先專心看 driver 端最常直接接觸到的 API
#### 3.3.1 設定 address bits

很多 driver 會先看到：

- `dma_set_mask_and_coherent()`

它是在告訴 kernel：這顆 device 最多能使用幾 bit 的 DMA address。

```c
ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
if (ret)
	return ret;
```

如果硬體只支援 32-bit DMA address，就要改成 `DMA_BIT_MASK(32)`。

這裡也很容易誤會：
`dma_set_mask_and_coherent()` 設的是這顆 device 能使用多寬的 DMA address，不是在設定它是不是 coherent。

#### 3.3.2 alloc buffer

`coherent` 講的是一致性，不是這塊 buffer 活多久。
可以先理解成：CPU 和 device 看同一塊 DMA buffer 時，彼此看到的內容會自動保持一致。

如果你需要這種 buffer，常見就是：

- `dma_alloc_coherent()`
- `dma_free_coherent()`

```c
void *cpu_addr;
dma_addr_t dma_addr;

cpu_addr = dma_alloc_coherent(dev, size, &dma_addr, GFP_KERNEL);
if (!cpu_addr)
	return -ENOMEM;

/* CPU 用 cpu_addr，device 用 dma_addr */

dma_free_coherent(dev, size, cpu_addr, dma_addr);
```

這類 buffer 常常剛好也會長期存在，所以常見於：

- descriptor ring
- command queue
- device 需要反覆讀寫的 shared buffer

#### 3.3.3 實做自己的 DMA trigger

前面兩步是在準備 DMA 能使用的 address 和 buffer；真正開始搬資料，通常還是要靠 driver 自己去接硬體的 trigger 流程。

常見做法是：

1. 把 `dma_addr` 寫進裝置的 DMA address register 或 descriptor
2. 把 length、control bits 一起設定好
3. 最後寫 start bit / kick bit / doorbell，讓硬體真的開始 DMA

也就是說，`dma_alloc_coherent()` 負責的是「準備好 coherent buffer」，不是「直接開始 DMA」。

可以先把分工想成：

```text
dma_set_mask_and_coherent()
    -> 這顆 device 可以用多寬的 DMA address
dma_alloc_coherent()
    -> 配出 coherent DMA buffer
driver 自己的 MMIO / descriptor setup
    -> 真的 trigger DMA
```

<div style="background:#ffe3ee; border-left: 4px solid #e26d93; padding: 10px 14px; border-radius: 6px;">
如果平台本身不是硬體自動 coherent，那麼 architecture / platform / DMA mapping backend 就要負責把 coherent 這個語意實作出來。也就是說，driver 使用 `dma_alloc_coherent()` 時，不應該再自己決定 flush 或同步方式；這個保證應該由平台層兌現。
</div>

---

### 3.4 streaming DMA sync API

這一節開始才看 streaming DMA 世界。
和前面的 coherent buffer 不一樣，streaming buffer 的資料同步不是自動保證的；CPU 和 device 之間什麼時候要同步、什麼時候要交還控制權，要靠 driver 依照 DMA API 規則去處理。

`streaming` 這個名字不是在講非同步，而是在講：

**這塊 buffer 是沿著一次次資料傳輸流程被暫時 map、使用、sync、再 unmap。**


#### 3.4.1 streaming DMA 是什麼

可以先把它想成：

```text
CPU 這邊本來有一塊 buffer
    -> kernel 幫你轉成 device 可用的 DMA address
    -> device 拿去做一次或一段時間的 DMA
    -> CPU / device 輪流接手時，要照規則 sync
```




這裡最容易誤會的一點是：
`streaming DMA` 通常不是先用專門 API 去配置一塊新 buffer；很多時候是你本來就有一塊一般記憶體 buffer，然後把它 map 成 DMA 可用。

常見會看到：

- `kmalloc()`
- driver 自己的 ring / queue buffer
- networking / block layer 上層交下來的 buffer

接著再用這幾個 API：

| API | 用途 | 回傳值 / 判斷方式 |
| --- | --- | --- |
| `dma_map_single()` | 把既有 buffer 轉成 device 可用的 DMA address | 回傳 `dma_addr_t`，這就是要給硬體用的 DMA address |
| `dma_unmap_single()` | 告訴 kernel 這次 streaming DMA mapping 已經用完 | 沒有回傳值 |
| `dma_mapping_error()` | 檢查 `dma_map_single()` 有沒有失敗 | 回傳 true / false，用來判斷 mapping 是否成功 |

```c
dma_addr_t dma_addr;

dma_addr = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_addr))
	return -EIO;
```

這裡回傳的 `dma_addr`，就是要給硬體用的 DMA address。

#### 3.4.2 trigger write DMA 要注意什麼

如果是 CPU 準備好資料，要叫 device 去把資料讀走，常見方向是：

- `DMA_TO_DEVICE`

這種情況最重要的是：

1. CPU 先把 buffer 內容寫好
2. 如果這塊 buffer 之後還會重複拿來用，必要時先做 `dma_sync_single_for_device()`
3. 把 `dma_addr`、length、control bits 寫進硬體 register 或 descriptor
4. 最後 trigger DMA

sample code ：

```c
dma_addr_t dma_addr;

dma_addr = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_addr))
	return -EIO;

dma_sync_single_for_device(dev, dma_addr, len, DMA_TO_DEVICE);

writel(lower_32_bits(dma_addr), regs + DMA_ADDR_LO);
writel(upper_32_bits(dma_addr), regs + DMA_ADDR_HI);
writel(len, regs + DMA_LEN);
writel(DMA_START, regs + DMA_CTRL);

/* DMA 完成後 */
dma_unmap_single(dev, dma_addr, len, DMA_TO_DEVICE);
```

筆記：

- CPU 改過內容，要交回 device 前，呼叫 `dma_sync_single_for_device()`
- CPU 沒再動過內容，通常不需要多補一次 `dma_sync_single_for_device()`

#### 3.4.3 trigger read DMA 要注意什麼

如果是 device 要把資料寫回 buffer，常見方向是：

- `DMA_FROM_DEVICE`

這時候比較重要的是「DMA 完成後，CPU 接手前」那個同步點。

sample code ：

```c
dma_addr_t dma_addr;

dma_addr = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
if (dma_mapping_error(dev, dma_addr))
	return -EIO;

writel(lower_32_bits(dma_addr), regs + DMA_ADDR_LO);
writel(upper_32_bits(dma_addr), regs + DMA_ADDR_HI);
writel(len, regs + DMA_LEN);
writel(DMA_START, regs + DMA_CTRL);

/* 等 DMA 完成，例如 interrupt / polling */

dma_sync_single_for_cpu(dev, dma_addr, len, DMA_FROM_DEVICE);

/* CPU 現在開始讀 buf */

dma_unmap_single(dev, dma_addr, len, DMA_FROM_DEVICE);
```

如果這塊 buffer 之後還要再交回 device 重複使用，CPU 改完或看完之後，要再補：

```c
dma_sync_single_for_device(dev, dma_addr, len, DMA_FROM_DEVICE);
```

#### 3.4.4 `dma_sync_single_*()` 兩隻怎麼分

| API | 什麼時候用 | 最短記法 |
| --- | --- | --- |
| `dma_sync_single_for_cpu(dev, dma_addr, size, dir)` | device 剛剛動過資料，現在 CPU 準備讀或改這塊單一 buffer | `for_cpu` = 在 CPU 接手前先同步 |
| `dma_sync_single_for_device(dev, dma_addr, size, dir)` | CPU 剛剛動過資料，現在準備把這塊單一 buffer 再交回 device | `for_device` = 在 device 接手前先同步 |


<div style="background:#e8f7e8; border-left: 4px solid #6bbf73; padding: 10px 14px; border-radius: 6px;">
看 `dma_sync_*()` 時，先不要把它想成「開始 DMA」或「配置 DMA」。最穩的理解是：這是在 CPU 和 device 之間切換資料控制權時用的同步 API。
</div>

---

### 3.5 怎麼 alloc 一塊空間

一般記憶體：`kmalloc()` / `kzalloc()`

如果這塊空間只是給 driver 自己放資料結構、狀態、software queue，最常見就是：

- `kmalloc()`
- `kzalloc()`
- `devm_kzalloc()`

```c
struct my_priv *priv;

priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
if (!priv)
	return -ENOMEM;
```

這一類配置出來的是一般 kernel memory。
它不是 DMA API，也不保證你可以直接把這個 pointer 丟給硬體當 DMA address。

可以先這樣記：

- `kmalloc()`：配置一塊一般記憶體
- `kzalloc()`：配置後順便清成 0
- `devm_kzalloc()`：掛在 `dev` 生命周期上，之後由 managed resource 幫忙回收
