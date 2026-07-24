# Linux Kernel Study - Debugging & Tracing

追蹤 (Tracing) 與效能分析 (Profiling) 除了用來找 Bug，也常用於確認核心實際執行路徑、量測延遲來源。本篇整理常用工具的定位、使用方式與各自的限制。

## 章節目錄

- [第一章：追蹤工具的分類](#chapter-1)
- [第二章：Ftrace](#chapter-2)
- [第三章：eBPF](#chapter-3)
- [第四章：Perf](#chapter-4)
- [第五章：Kprobes 與 Tracepoints](#chapter-5)
- [第六章：工具選擇與注意事項](#chapter-6)

---

<a id="chapter-1"></a>

## 第一章：追蹤工具的分類

| 分類 | 說明 | 資料取得方式 | 代表工具 |
| :--- | :--- | :--- | :--- |
| **Tracing (追蹤)** | 記錄具體事件（函數呼叫、中斷、排程切換） | 事件觸發 | Ftrace, eBPF, LTTng |
| **Profiling (效能分析)** | 以抽樣統計 CPU 時間分佈 | 週期性取樣 | Perf |
| **Logging (日誌)** | 程式碼主動輸出的訊息 | 手動埋點 | printk, dev_dbg, dmesg |
| **Debugging (除錯)** | 停下來檢查狀態 | 中斷執行 | KGDB, KDB, crash |

Tracing 與 Profiling 的差別在於前者記錄「發生了什麼」，後者回答「時間花在哪」。兩者資料量與 overhead 的量級也不同：Profiling 的取樣頻率固定（例如 99 Hz），Tracing 的資料量則取決於事件發生頻率，高頻事件容易塞爆 ring buffer。

---

<a id="chapter-2"></a>

## 第二章：Ftrace

Ftrace 是核心內建的追蹤框架，不需額外安裝套件，掛載 `tracefs` 即可使用。需要核心編譯時開啟 `CONFIG_FUNCTION_TRACER`、`CONFIG_FUNCTION_GRAPH_TRACER`、`CONFIG_DYNAMIC_FTRACE` 等選項。

### 2.1 操作介面：tracefs

介面位於 `/sys/kernel/tracing`（舊版本在 `/sys/kernel/debug/tracing`）。若未自動掛載：

```bash
mount -t tracefs nodev /sys/kernel/tracing
```

主要檔案：

| 檔案 | 用途 |
| :--- | :--- |
| `current_tracer` | 選擇追蹤器（`nop` / `function` / `function_graph` / `irqsoff` 等） |
| `available_tracers` | 列出此核心支援的追蹤器 |
| `set_ftrace_filter` | 限制只追蹤指定函數（支援 `*` 萬用字元） |
| `set_ftrace_notrace` | 排除指定函數 |
| `set_ftrace_pid` | 只追蹤指定 PID |
| `trace` | 讀取 ring buffer 內容（不清空） |
| `trace_pipe` | 串流讀取，讀過即消耗 |
| `buffer_size_kb` | 每 CPU 的 ring buffer 大小 |
| `tracing_on` | 追蹤開關 |
| `events/` | 靜態 tracepoint 事件目錄 |

### 2.2 Function Graph

`function_graph` 會輸出函數呼叫的層級與各層耗時：

```bash
cd /sys/kernel/tracing

# 先清空舊資料並停用追蹤
echo 0 > tracing_on
echo > trace

echo function_graph > current_tracer

# 只追蹤特定進入點及其子呼叫
echo pci_proc_init > set_graph_function

# 限制深度，避免資料量爆炸
echo 5 > max_graph_depth

echo 1 > tracing_on
# (執行相關操作)
echo 0 > tracing_on

cat trace
```

輸出範例：

```text
 0)               |  pci_proc_init() {
 0)               |    proc_mkdir() {
 0)   0.123 us    |      __proc_mkdir();
 0)   0.556 us    |    }
 0)   2.134 us    |  }
```

行首數字是 CPU 編號，`us` 欄位是該函數的執行時間。時間超過門檻的行會被標上 `+`（>10us）或 `!`（>100us），可用來快速定位慢的路徑。

### 2.3 事件追蹤 (Event Tracing)

不需要 `function_graph`，也能單獨開啟特定 tracepoint：

```bash
cd /sys/kernel/tracing

# 開啟單一事件
echo 1 > events/sched/sched_switch/enable

# 開啟整個子系統
echo 1 > events/irq/enable

# 加上過濾條件
echo 'prev_comm == "ipmid"' > events/sched/sched_switch/filter

cat trace_pipe
```

### 2.4 trace-cmd

直接操作 tracefs 步驟繁瑣，`trace-cmd` 是官方的命令列前端：

```bash
# 追蹤某個指令執行期間的 I2C 事件
trace-cmd record -e i2c -F ipmitool sensor list

# 追蹤全系統 10 秒
trace-cmd record -p function_graph -g i2c_transfer sleep 10

# 檢視結果
trace-cmd report
```

`trace-cmd` 會把資料寫成 `trace.dat`，可帶回主機用 `kernelshark` 做圖形化檢視。在 BMC 這種資源受限的環境，通常在 target 上 `record`、在 host 上 `report`。

### 2.5 其他 tracer

| Tracer | 用途 |
| :--- | :--- |
| `irqsoff` | 記錄中斷被關閉最久的區段 |
| `preemptoff` | 記錄搶佔被關閉最久的區段 |
| `wakeup_rt` | 量測 RT 任務的喚醒延遲 |
| `hwlat` | 偵測硬體/韌體造成的延遲（SMI 等） |

這幾個在 real-time 或即時性問題排查時特別有用。

---

<a id="chapter-3"></a>

## 第三章：eBPF

eBPF 允許把受驗證器 (verifier) 檢查過的程式載入核心執行，不需重新編譯核心或載入模組。相較於傳統做法，主要優勢在於可以**在核心內先做彙整**（例如直接累加成 histogram），只把結果送到 user space，減少資料搬運成本。

限制也需要一併理解：

- 需要較新的核心（實務上 tracing 相關功能建議 5.x 以上）與 `CONFIG_BPF_SYSCALL`、`CONFIG_DEBUG_INFO_BTF`
- 程式受 verifier 限制：不能有無界迴圈、stack 大小有限、指標存取需通過檢查
- 載入需要 `CAP_BPF` / `CAP_PERFMON`（舊核心是 root）
- 在 BMC 這類嵌入式環境，核心可能未開啟 BTF，或工具鏈不完整，未必能直接使用

### 3.1 bpftrace

適合寫一次性的檢查腳本：

```bash
# 統計各行程呼叫 openat 的次數
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'

# 量測 i2c_transfer 的延遲分佈
bpftrace -e '
kprobe:i2c_transfer { @start[tid] = nsecs; }
kretprobe:i2c_transfer /@start[tid]/ {
    @us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

# 列出可用的探針
bpftrace -l 'tracepoint:i2c:*'
```

### 3.2 BCC

需要開發較複雜的工具時，BCC 提供 Python / C++ 介面，並附帶一批現成工具：

| 工具 | 用途 |
| :--- | :--- |
| `execsnoop` | 追蹤新產生的行程 |
| `opensnoop` | 追蹤檔案開啟 |
| `biolatency` | 磁碟 I/O 延遲分佈 |
| `funclatency` | 指定核心函數的延遲分佈 |
| `stackcount` | 統計到達某函數的呼叫堆疊 |
| `offcputime` | 分析行程沒在 CPU 上的時間花在哪 |

BCC 在執行時才編譯 BPF 程式，需要核心標頭檔；`libbpf` + CO-RE (Compile Once, Run Everywhere) 是較新的做法，編譯出的執行檔可跨核心版本執行，比較適合部署到嵌入式目標。

---

<a id="chapter-4"></a>

## 第四章：Perf

Perf 與核心原始碼同步開發，底層利用 `perf_event` 子系統，可存取 CPU 的 PMU (Performance Monitoring Unit)、軟體事件與 tracepoint。

### 4.1 常見指令

```bash
# 即時顯示佔用 CPU 最多的函數
perf top

# 統計整體事件計數（cache miss、branch miss 等）
perf stat -e cycles,instructions,cache-misses ./workload

# 取樣並記錄呼叫堆疊
perf record -F 99 -a -g -- sleep 30

# 分析結果
perf report --stdio

# 直接列出每筆取樣
perf script
```

`-F` 是取樣頻率，`-a` 為全系統，`-g` 收集 call graph。call graph 的展開方式（`--call-graph fp` / `dwarf` / `lbr`）會影響準確度與 overhead：`fp` 需要編譯時保留 frame pointer，`dwarf` 較準但資料量大。

### 4.2 火焰圖 (Flame Graph)

用來把取樣資料的呼叫堆疊聚合成圖：

```bash
perf record -F 99 -a -g -- sleep 30
perf script > out.perf
./stackcollapse-perf.pl out.perf > out.folded
./flamegraph.pl out.folded > out.svg
```

讀圖方式：

- **寬度**代表該函數出現在取樣中的比例，越寬表示佔用 CPU 越多
- **高度**是呼叫堆疊深度，沒有時間先後的意義
- **x 軸順序**是字母排序，不是時間軸

需要注意的是：火焰圖顯示的是 on-CPU 時間。若問題是等待 I/O 或鎖，on-CPU 火焰圖看不出來，要改用 off-CPU 分析（`offcputime`）。

### 4.3 在嵌入式平台的限制

- 部分 SoC 的 PMU 中斷未接好，`perf record` 只能退回軟體事件 (`cpu-clock`)
- 需要 `CONFIG_PERF_EVENTS`，以及 `kernel.perf_event_paranoid` 設定允許
- 符號解析需要對應的 `vmlinux` 或未 strip 的 binary，通常在 host 端做

---

<a id="chapter-5"></a>

## 第五章：Kprobes 與 Tracepoints

這兩者是上述工具共用的底層機制。

### 5.1 Tracepoints (靜態探針)

- **定義**：由開發者預先寫在核心程式碼中的追蹤點，未啟用時的成本接近零（透過 static key / jump label 實作）
- **優點**：介面相對穩定，欄位有明確定義，核心升級後名稱通常不變
- **缺點**：只能追蹤已經被埋點的位置

可用的事件列在 `/sys/kernel/tracing/available_events`。

### 5.2 Kprobes (動態探針)

- **定義**：執行期動態插入，理論上可掛在核心大部分指令位址上；`kretprobe` 則掛在函數返回點
- **優點**：不需修改核心程式碼，涵蓋範圍廣
- **缺點**：
  - 依賴核心符號與函數簽章，核心版本更動可能失效
  - 部分函數被標記為 `NOKPROBE_SYMBOL` 或位於 `.noinstr` 區段，無法插入
  - 被 inline 或最佳化掉的函數不會有對應符號
  - 每次觸發都有 trap 成本，高頻函數上 overhead 明顯

### 5.3 其他探針類型

| 類型 | 位置 | 說明 |
| :--- | :--- | :--- |
| `uprobe` / `uretprobe` | User space | 追蹤使用者程式的函數 |
| `fentry` / `fexit` | Kernel | 較新的機制，overhead 低於 kprobe，需 BTF 支援 |
| USDT | User space | 應用程式內建的靜態探針 |

原則上：**有 tracepoint 就用 tracepoint，沒有才退而使用 kprobe**。

---

<a id="chapter-6"></a>

## 第六章：工具選擇與注意事項

### 6.1 依目的選擇

| 目的 | 建議工具 |
| :--- | :--- |
| 了解程式執行流程 | Ftrace `function_graph` |
| 找出 CPU 熱點 | `perf top` / `perf record` + 火焰圖 |
| 量測特定函數延遲分佈 | bpftrace / `funclatency` |
| 分析等待與阻塞 | `offcputime`、`wakeup` tracer |
| 排查中斷延遲 | Ftrace `irqsoff` / `hwlat` |
| 觀察系統呼叫行為 | `strace`（單行程）、`perf trace`（全系統） |
| 硬體事件計數 | `perf stat` |

### 6.2 共通注意事項

- **Overhead**：追蹤本身會影響被觀察的系統，高頻事件尤其明顯。先用過濾條件縮小範圍，再逐步放寬。
- **資料遺失**：`trace` 輸出若出現 `LOST EVENTS`，表示 ring buffer 不足，需調大 `buffer_size_kb` 或降低事件量。
- **符號問題**：找不到符號通常是核心 strip 過、或未開啟 `CONFIG_KALLSYMS_ALL`。
- **記得關閉**：測完務必 `echo 0 > tracing_on` 並把 `current_tracer` 設回 `nop`，否則會持續消耗效能。
