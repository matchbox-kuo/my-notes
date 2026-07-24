## 第四章：記憶體相關觀念

<div style="background:#fff4e8; border-left: 4px solid #f0a35e; padding: 10px 14px; border-radius: 6px;">
這一章處理一個很容易和 driver / DMA 混在一起的問題：kernel 到底怎麼知道系統上有哪些 RAM，之後又是怎麼變成我們熟悉的記憶體分配與 DMA 分配能力。
</div>

### 4.1 kernel 不是憑空知道有多少記憶體

kernel 剛開始執行時，並不知道：

- 系統總共有多少 RAM
- 哪些實體位址範圍是可用記憶體
- 哪些區域是保留區，不能碰

這些資訊要由平台提供。整條路徑大致是：

| 階段 | 做的事 |
| --- | --- |
| 平台 / 韌體 / bootloader | 提供 memory map |
| kernel early boot | 記住哪些 memory 可用、哪些 reserved |
| 之後 | 才能做一般記憶體分配與 DMA 分配 |

所以不是「driver 在編譯時告訴 kernel 有多少 RAM」，而是 kernel 在 very early boot 就拿到記憶體地圖，後面所有 allocator 都建立在這份地圖上。

---

### 4.2 誰把 memory map 告訴 kernel

來源依平台而異，但概念一致：有人先把「哪些實體位址範圍是 RAM」交給 kernel。

#### Device Tree

ARM、RISC-V、embedded 平台常見。DT 裡的 `/memory` node 用 `reg` 描述 RAM 的 base 和 size，kernel 早期解析 DT 時就能知道 RAM 從哪個實體位址開始、有多大。

#### ACPI

PC-like 與 server 平台上，kernel 從 ACPI tables 取得記憶體資訊。

#### EFI memory map

經 UEFI 啟動的系統，firmware 會交一份 EFI memory map 給 kernel，裡面區分一般可用記憶體、reserved memory，以及 firmware runtime 相關區域。

#### x86 e820 map

傳統 x86 的做法。kernel 靠它判斷哪些 range 是 RAM、哪些是 reserved、哪些不能使用。

#### bootloader 或平台固定描述

部分 SoC / embedded 平台由 bootloader 直接傳入 RAM base / size，或由平台碼寫死。

---

### 4.3 kernel 拿到 memory map 之後

very early boot 階段負責記住這張地圖的是 `memblock`。它把整張實體記憶體地圖記下來，並區分：

- usable memory
- reserved memory
- 之後可以交給一般 allocator 的 memory

這個階段之所以需要獨立的 early memory manager，是因為 page allocator、zone、buddy allocator 都還沒建立起來，但 kernel 已經要開始做初始化、需要配記憶體。

由下而上的層次：

| 層 | 內容 |
| --- | --- |
| 1 | memory map |
| 2 | memblock |
| 3 | page allocator / zone / buddy |
| 4 | kmalloc / alloc_pages / dma alloc |

---

### 4.4 reserved memory 是什麼意思

不是所有 RAM 都會被當成一般可自由分配的記憶體。有些區域會先標成 reserved，表示不歸一般 page allocator 管，可能是留給 firmware、留給特定硬體需求，或留給 CMA 之類的特殊用途。

因此「kernel 知道總共有多少 RAM」和「哪些 RAM 最後能交給一般 allocator」是兩件事。

---

### 4.5 CMA

CMA 是 kernel 先保留一塊適合做連續實體記憶體分配的區域，需要時再從那裡切。這種區域常和 DMA 需求有關，因為某些硬體要求 physically contiguous memory。

要注意 `CMA` 不等於 `coherent`：

- `CMA` 講的是保留與分配策略
- `coherent DMA` 講的是 DMA buffer 的一致性語意

實務上兩者常一起出現，但不是同一個概念。

---

### 4.6 一般記憶體分配和 DMA 分配的關係

等 kernel 知道整張記憶體地圖、也建好後續 allocator，才進入平常比較熟的那些 API：

- `alloc_pages()`
- `kmalloc()`
- `vmalloc()`
- `dma_alloc_coherent()`
- `dma_map_single()`

`dma_alloc_coherent()` 不會憑空生出一塊記憶體，它仍然建立在 kernel 已知的實體記憶體與平台 DMA 規則之上。順序是：平台先告訴 kernel 有哪些 RAM，kernel 再建立自己的 memory management，最後 driver 才透過 API 去分配。

---

### 4.7 這幾個 API 分別怎麼用

先看差異：

| API | 拿到什麼 | 實體連續 | 給誰用 | 釋放 |
| --- | --- | --- | --- | --- |
| `alloc_pages()` | `struct page *` | 是 | 需要整頁、自己管的場合 | `__free_pages()` |
| `kmalloc()` | 虛擬位址 | 是 | 小塊、要做 DMA 的 buffer | `kfree()` |
| `vmalloc()` | 虛擬位址 | 否 | 大塊、不做 DMA | `vfree()` |
| `dma_alloc_coherent()` | 虛擬位址 + `dma_addr_t` | 是 | 長期存在的 DMA buffer | `dma_free_coherent()` |
| `dma_map_single()` | `dma_addr_t` | — | 把既有 buffer 暫時交給 device | `dma_unmap_single()` |

#### alloc_pages()

以 page 為單位分配，`order` 是 2 的次方，`order = 2` 就是 4 個連續 page。回傳的是 `struct page *`，要用 `page_address()` 換成可以直接存取的虛擬位址。

```c
struct page *p = alloc_pages(GFP_KERNEL, 2);   /* 4 pages */
void *va = page_address(p);
__free_pages(p, 2);
```

比較底層，driver 裡不常直接用，通常是需要自己控制 page 或要拿整頁對齊的記憶體時才會碰。

#### kmalloc()

最常用的一般分配。實體連續、虛擬也連續，背後是 slab allocator，適合幾個 byte 到數個 page 的小配置。因為實體連續，kmalloc 出來的 buffer 可以直接拿去做 DMA。

```c
void *buf = kmalloc(512, GFP_KERNEL);
kfree(buf);
```

gfp flag 決定能不能睡：`GFP_KERNEL` 允許 sleep，只能在 process context 用；在 interrupt handler、spinlock 內要用 `GFP_ATOMIC`。要配大塊連續記憶體時 kmalloc 容易失敗，這是正常的，實體連續本來就難湊。

#### vmalloc()

在虛擬位址空間切出一段連續區域，但底下的實體 page 是散的。好處是大塊配置容易成功，代價是要額外建 page table、存取略慢，而且**不能拿去做 DMA**（device 看到的是實體位址，那邊根本不連續）。

```c
void *big = vmalloc(4 * 1024 * 1024);
vfree(big);
```

用在單純給 CPU 用的大 buffer，例如載入韌體、暫存大量資料。

#### dma_alloc_coherent()

一次拿到兩個位址：CPU 用的虛擬位址，以及 device 用的 `dma_addr_t`。這塊記憶體是 coherent 的，CPU 寫完 device 就看得到，不需要手動做 cache 同步。

```c
dma_addr_t dma;
void *va = dma_alloc_coherent(dev, size, &dma, GFP_KERNEL);
/* va 給 CPU 用，dma 寫進硬體 register */
dma_free_coherent(dev, size, va, dma);
```

適合生命週期長、CPU 和 device 會反覆存取的東西，典型是 descriptor ring、command queue。缺點是平台上可能被配成 uncached，CPU 大量存取會比較慢。

#### dma_map_single()

不是分配，是**把已經存在的 buffer 暫時映射給 device**（streaming DMA）。傳一塊 kmalloc 來的記憶體進去，拿到 device 用的 `dma_addr_t`，傳輸結束再 unmap。

```c
dma_addr_t dma = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma))
        return -ENOMEM;
/* 啟動硬體傳輸、等完成 */
dma_unmap_single(dev, dma, len, DMA_TO_DEVICE);
```

幾個重點：

- direction 要給對（`DMA_TO_DEVICE` / `DMA_FROM_DEVICE` / `DMA_BIDIRECTIONAL`），cache 操作靠它決定
- map 到 unmap 之間，buffer 的所有權在 device 手上，CPU 不該去碰
- 回傳值一定要用 `dma_mapping_error()` 檢查
- 來源不能是 vmalloc 的記憶體，也不能是 stack 上的變數

適合每次傳輸內容都不同的一次性 buffer，例如網路封包、SPI/I2C 的 transfer buffer。

---

### 4.8 把層次分開

這一章的重點不是背完所有 allocator，而是分清楚層次：

1. 平台怎麼把 memory map 告訴 kernel
2. kernel early boot 怎麼先記住它
3. 哪些記憶體是 usable，哪些是 reserved
4. 後面 allocator 怎麼建立起來
5. driver 最後只是站在最上層使用分配 API

<div style="background:#e8f7e8; border-left: 4px solid #6bbf73; padding: 10px 14px; border-radius: 6px;">
先有 memory map，才有 memblock；先有 memblock，後面才有一般 allocator 和 DMA allocator。driver 平常呼叫的 API，都是建立在這條鏈上。
</div>
