# Linux Kernel Study - Modern Debugging & Tracing

<div style="background:linear-gradient(135deg, #6e8efb, #a777e3); color: white; padding: 20px; border-radius: 12px; margin-bottom: 25px; box-shadow: 0 4px 15px rgba(0,0,0,0.1);">
現代 Linux 核心開發中，「追蹤 (Tracing)」與「分析 (Profiling)」已經不再只是找 Bug 的手段，而是理解系統運作行為、解決效能瓶頸的核心技術。本篇將介紹從傳統到當代最強大的幾種追蹤工具。
</div>

## 章節目錄

- [第一章：追蹤工具的地圖 (Tracing Landscape)](#chapter-1)
- [第二章：Ftrace — 核心瑞士刀](#chapter-2)
- [第三章：eBPF — 當代追蹤之王](#chapter-3)
- [第四章：Perf — 系統效能分析](#chapter-4)
- [第五章：動態與靜態探針 (Kprobes & Tracepoints)](#chapter-5)

---

<a id="chapter-1"></a>

## 第一章：追蹤工具的地圖 (Tracing Landscape)

在開始使用工具前，必須先分清楚三種不同的觀察方式：

| 分類 | 說明 | 代表工具 |
| :--- | :--- | :--- |
| **Tracing (追蹤)** | 記錄系統中發生的具體事件（例如：函數呼叫、中斷）。通常是事件觸發。 | Ftrace, eBPF |
| **Profiling (效能分析)** | 透過抽樣 (Sampling) 或統計，了解 CPU 時間都花在哪裡。 | Perf |
| **Logging (日誌)** | 程式碼主動輸出的文字訊息。 | printk, dmesg |

---

<a id="chapter-2"></a>

## 第二章：Ftrace — 核心瑞士刀

Ftrace (Function Tracer) 是 Linux 核心內建的官方追蹤器。它不需要安裝額外軟體，只要掛載 `tracefs` 就能使用。

### 2.1 核心概念：tracefs
Ftrace 的操作介面位於 `/sys/kernel/tracing` (舊版本在 `/sys/kernel/debug/tracing`)。

### 2.2 常用功能：Function Graph
這是 Ftrace 最迷人的地方，它可以畫出函數呼叫的層級圖：

```bash
# 進入追蹤目錄
cd /sys/kernel/tracing

# 設定追蹤器為 function_graph
echo function_graph > current_tracer

# 設定只追蹤特定函數 (例如 pci_proc_init)
echo pci_proc_init > set_graph_function

# 開啟追蹤
echo 1 > tracing_on

# (執行相關操作後) 關閉追蹤
echo 0 > tracing_on

# 查看結果
cat trace
```

**輸出結果範例：**
```text
 0)               |  pci_proc_init() {
 0)               |    proc_mkdir() {
 0)   0.123 us    |      __proc_mkdir();
 0)   0.556 us    |    }
 0)   2.134 us    |  }
```

---

<a id="chapter-3"></a>

## 第三章：eBPF — 當代追蹤之王

eBPF 是 Linux 核心近十年最重要的技術革新。它允許我們在核心內執行「沙盒化」的程式，而不需要重新編譯核心或載入 Kernel Module。

<div style="background:#f0f7ff; border-left: 5px solid #007bff; padding: 15px; border-radius: 8px;">
<strong>💡 為什麼 eBPF 這麼強？</strong><br/>
它能即時彙整資料。傳統追蹤工具會產生大量的資料並送到 User space（這很慢），eBPF 可以在核心內直接統計完（例如：計算平均延遲），最後只送出結果。
</div>

### 3.1 bpftrace (快速上手的 DTrace 替代品)
如果你只需要寫一行指令來解決問題，`bpftrace` 是最佳選擇。

**範例：統計系統中所有 `open()` 系統呼叫次數**
```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
```

### 3.2 BCC (BPF Compiler Collection)
如果你需要開發複雜的工具，BCC 提供 Python/C++ 的介面。
*   `execsnoop`: 追蹤所有新產生的行程。
*   `biolatency`: 統計磁碟 I/O 延遲分佈圖。

---

<a id="chapter-4"></a>

## 第四章：Perf — 系統效能分析

Perf 是與核心原始碼同步開發的工具，主要利用 CPU 內部的 **PMU (Performance Monitoring Unit)**。

### 4.1 常見指令
*   `perf top`: 像 `top` 指令一樣，即時顯示哪些核心函數最吃 CPU。
*   `perf record`: 紀錄一段時間的系統活動。
*   `perf report`: 分析 `perf record` 產生的資料。

### 4.2 火焰圖 (Flame Graph)
這是現代工程師最愛的視覺化工具，可以一眼看出效能瓶頸。
1.  使用 `perf record -g` 取樣。
2.  將輸出轉換為 SVG。
3.  **寬度** 代表該函數佔用的 CPU 時間比例。

---

<a id="chapter-5"></a>

## 第五章：動態與靜態探針 (Kprobes & Tracepoints)

這是追蹤技術的底層基礎。

### 5.1 Tracepoints (靜態探針)
*   **定義**：開發者預先寫在 Kernel 程式碼裡的追蹤點。
*   **優點**：穩定，核心升級後通常不會改變名稱。
*   **缺點**：數量有限，沒寫的地方就不能追蹤。

### 5.2 Kprobes (動態探針)
*   **定義**：可以在執行期，動態地插在 Kernel 的**任何一個指令**上。
*   **優點**：無孔不入，想看哪就看哪。
*   **缺點**：依賴核心符號，核心版本更動可能導致失效。

---

<div style="background:#fff4e5; border-left: 5px solid #ffa500; padding: 15px; border-radius: 8px; margin-top: 30px;">
<strong>🛠️ 實戰建議：</strong><br/>
1. 想要<strong>了解程式流程</strong>：先用 <code>Ftrace (function_graph)</code>。<br/>
2. 想要<strong>效能除錯</strong>：先用 <code>perf top</code> 或 <code>perf record</code>。<br/>
3. 想要<strong>自定義複雜邏輯</strong>：使用 <code>eBPF (bpftrace/BCC)</code>。
</div>
