## 第四章：記憶體相關觀念

<div style="background:#fff4e8; border-left: 4px solid #f0a35e; padding: 10px 14px; border-radius: 6px;">
這一章先處理一個很容易和 driver / DMA 混在一起的問題：kernel 到底怎麼知道系統有哪些 RAM 可以用，後面又是怎麼一步一步變成一般記憶體分配與 DMA 分配能力。
</div>

### 4.1 kernel 不是憑空知道有多少記憶體

當 kernel 開始執行時，它不是自己突然就知道：

- 系統總共有多少 RAM
- 哪些實體位址範圍是可用記憶體
- 哪些區域是保留區，不能碰

這些資訊通常要先由平台提供給 kernel。

可以先把整件事想成：

```text
平台 / 韌體 / bootloader
    -> 提供 memory map
kernel early boot
    -> 記住哪些 memory 可用、哪些 reserved
後面
    -> 才能做一般記憶體分配與 DMA 分配
```

所以重點不是「driver 在編譯時告訴 kernel 有多少 RAM」，而是：

**kernel 在 very early boot 就先拿到記憶體地圖，後面所有 allocator 都建立在這份地圖上。**

---

### 4.2 常見是誰把 memory map 告訴 kernel

不同平台來源不同，但概念都很像：  
先有人把「哪些實體位址範圍是 RAM」交給 kernel。

常見來源有幾種：

#### Device Tree

在很多 ARM、RISC-V、embedded 平台，Device Tree 會有 `/memory` node，用 `reg` 描述 RAM 的 base 和 size。

也就是說，kernel 早期解析 DT 時，就能知道：

- RAM 從哪個實體位址開始
- RAM 有多大

#### ACPI

在很多 PC-like、server 平台，kernel 會從 ACPI tables 拿到記憶體相關資訊。

#### EFI memory map

如果系統經過 UEFI 啟動，firmware 會交一份 EFI memory map 給 kernel，裡面會區分：

- 一般可用記憶體
- reserved memory
- firmware runtime 相關區域

#### x86 e820 map

傳統 x86 很常見的是 e820 memory map。  
kernel 會靠它知道哪些 range 是 RAM、哪些是 reserved、哪些不能使用。

#### bootloader 或平台固定描述

有些 SoC / embedded 平台，也可能由 bootloader 直接傳入 RAM base / size，或由平台碼用固定方式描述。

---

### 4.3 kernel 拿到 memory map 後，第一個記住它的是誰

在 Linux kernel very early boot 階段，最重要的早期記憶體管理者之一就是：

- `memblock`

它的角色可以先理解成：

**先把整張實體記憶體地圖記下來，區分哪些可用、哪些保留。**

也就是：

- usable memory
- reserved memory
- 之後可以再交給一般 allocator 的 memory

這個階段很重要，因為後面的 page allocator、zone、buddy allocator 都還沒完全建立起來；但 kernel 自己又已經要開始做一些初始化，所以需要先有一個 early memory manager。

可以粗略想成：

```text
memory map
   -> memblock
   -> page allocator / zone / buddy
   -> kmalloc / alloc_pages / dma alloc
```

---

### 4.4 reserved memory 是什麼意思

不是所有 RAM 都會被當成一般可自由分配的記憶體。

有些區域會先被標成 reserved，表示：

- 不是一般 page allocator 可以隨便拿來用的
- 可能保留給 firmware
- 可能保留給特定硬體需求
- 可能保留給 CMA 或其他特殊用途

所以「kernel 知道總共有多少 RAM」和「哪些 RAM 最後能交給一般 allocator」其實不是同一件事。

---

### 4.5 CMA 可以先怎麼理解

如果前面看到 `CMA`，可以先把它想成：

**kernel 先保留一塊比較適合做連續實體記憶體分配的區域，之後需要時再從那裡切。**

這種區域常常和 DMA 需求有關，因為某些硬體會需要：

- physically contiguous memory
- 比較容易 DMA 的配置方式

但這裡要注意：

- `CMA` 不等於 `coherent`
- `CMA` 比較像是在講「保留與分配策略」
- `coherent DMA` 比較是在講 DMA buffer 的一致性語意

這兩者很常在實務上一起出現，但不是同一個概念。

---

### 4.6 一般記憶體分配和 DMA 分配的關係

當 kernel 已經知道整張記憶體地圖，也建立好後續 allocator 之後，才會進入我們平常比較熟的世界：

- `alloc_pages()`
- `kmalloc()`
- `vmalloc()`
- `dma_alloc_coherent()`
- `dma_map_single()`

所以 `dma_alloc_coherent()` 不是自己憑空創造一塊記憶體。  
它背後仍然建立在 kernel 已知的實體記憶體與平台 DMA 規則之上。

可以先這樣看：

```text
平台先告訴 kernel 有哪些 RAM
kernel 再建立自己的 memory management
最後 driver 才透過一般 API 去分配
```

---

### 4.7 先把層次分開，很多問題就不會混掉

這一章最重要的不是一口氣背完所有 allocator，而是先分清楚不同層次：

1. 平台怎麼把 memory map 告訴 kernel
2. kernel early boot 怎麼先記住它
3. 哪些記憶體是 usable，哪些是 reserved
4. 後面 allocator 怎麼建立起來
5. driver 最後只是站在最上層使用分配 API

<div style="background:#e8f7e8; border-left: 4px solid #6bbf73; padding: 10px 14px; border-radius: 6px;">
先有 memory map，才有 memblock；先有 memblock，後面才有一般 allocator 和 DMA allocator。driver 平常呼叫的 API，都是建立在這條鏈上。
</div>
