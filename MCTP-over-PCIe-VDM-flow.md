# MCTP over PCIe VDM 流程整理

這份文件整理 Aspeed BMC 上 **MCTP over PCIe VDM** 的軟體路徑，並說明 Intel-BMC `libmctp` 與 Aspeed kernel driver 各自負責什麼。

重點先講：

- Intel-BMC `libmctp` 是舊的 userspace MCTP stack。
- `drivers/soc/aspeed/aspeed-mctp.c` 是真正控制 Aspeed MCTP PCIe 硬體的 kernel driver。
- `drivers/net/mctp/mctp-pcie-vdm.c` 是 Linux kernel socket-MCTP 的 PCIe VDM transport。
- 長期可以簡化成走 kernel AF_MCTP/socket-MCTP，不再擴充 Intel `libmctp`。

## 整體架構

目前可以分成兩條路。

### 舊路線：Intel libmctp + `/dev/aspeed-mctpX`

```text
Application / 測試工具 / 舊 SPDM 或 PLDM 程式
    |
    v
Intel-BMC libmctp
    core.c
    astpcie.c
    libmctp-astpcie.h
    |
    | open/read/write/ioctl
    v
/dev/aspeed-mctpX
    |
    v
drivers/soc/aspeed/aspeed-mctp.c
    |
    v
Aspeed MCTP PCIe hardware controller
    |
    v
PCIe fabric 上的 PCIe VDM packet
```

這條路是 Intel `libmctp` 自己在 userspace 做 MCTP core，然後透過 `astpcie.c` 把 packet 包成 PCIe VDM，再寫到 `/dev/aspeed-mctpX`。

### 新路線：kernel socket-MCTP + `mctppciX`

```text
Application / daemon
    |
    | AF_MCTP socket / mctp tools / mctpd
    v
Linux kernel MCTP core
    |
    v
mctppciX netdev
    |
    v
drivers/net/mctp/mctp-pcie-vdm.c
    |
    v
drivers/soc/aspeed/aspeed-mctp.c
    |
    v
Aspeed MCTP PCIe hardware controller
    |
    v
PCIe fabric 上的 PCIe VDM packet
```

這條路把 MCTP stack 放在 Linux kernel，userspace app 走 AF_MCTP socket 或 `mctpd`，不用直接碰 `/dev/aspeed-mctpX`。

## 角色分工

### Intel-BMC libmctp

位置：

```text
D:/Code/intel/libmctp
```

負責：

- userspace MCTP core
- MCTP header/tag/EID/message type
- 分段與重組
- callback dispatch
- PCIe VDM binding
- 透過 `/dev/aspeed-mctp` 跟 Aspeed kernel driver 溝通

它不是 PCIe driver，也不直接操作 Aspeed register。

### Aspeed kernel driver

主要檔案：

```text
D:/Code/aspeed/linux/drivers/soc/aspeed/aspeed-mctp.c
```

負責：

- 操作 Aspeed MCTP controller register
- 配置 TX/RX DMA buffer
- 維護 TX/RX command ring
- 處理 interrupt/tasklet
- 控制 PCIe VDM packet 實際收送
- 建立 `/dev/aspeed-mctpX`
- 提供 ioctl 給 userspace
- 可選擇註冊 `mctppciX` 給 kernel socket-MCTP 使用

這是硬體 driver，是 MCTP over PCIe VDM 能不能真的送出去的核心。

### kernel `mctp-pcie-vdm`

主要檔案：

```text
D:/Code/aspeed/linux/drivers/net/mctp/mctp-pcie-vdm.c
```

負責：

- Linux socket-MCTP 的 PCIe VDM transport
- 建立 `mctppciX` netdev
- 在 kernel 裡組/拆 PCIe VDM transport header
- 把 RX packet 送進 kernel MCTP networking stack
- TX 時呼叫下層硬體 driver callback

它本身不是 Aspeed 專屬硬體 driver，而是比較 generic 的 PCIe VDM transport layer。

## 主要 kernel 檔案

### `drivers/soc/aspeed/aspeed-mctp.c`

位置：

```text
D:/Code/aspeed/linux/drivers/soc/aspeed/aspeed-mctp.c
```

這是最重要的檔案。

它負責：

- hardware register 操作
- PCIe link/reset 狀態處理
- TX/RX DMA ring
- TX/RX tasklet
- `/dev/aspeed-mctpX`
- ioctl
- kernel socket-MCTP callback

重要區塊：

- register 定義：檔案前段，`ASPEED_MCTP_CTRL`, `ASPEED_MCTP_TX_CMD`, `ASPEED_MCTP_RX_CMD`, `ASPEED_MCTP_INT_STS`
- channel 結構：`struct mctp_buffer`, `struct mctp_channel`, `struct aspeed_mctp`
- RX enable：`aspeed_mctp_rx_trigger()`
- TX trigger：`aspeed_mctp_tx_trigger()`
- TX command 建立：`aspeed_mctp_emit_tx_cmd()`
- RX tasklet：`aspeed_mctp_rx_tasklet()`
- TX tasklet：`aspeed_mctp_tx_tasklet()`
- userspace read/write：`aspeed_mctp_read()`, `aspeed_mctp_write()`
- ioctl：`aspeed_mctp_ioctl()`
- PCIe VDM callback：`aspeed_mctp_pcie_vdm_op_send_pkt()`, `aspeed_mctp_pcie_vdm_op_recv_pkt()`
- netdev 註冊：`aspeed_mctp_pcie_vdm_register()`
- misc device 建立：`misc_register(&priv->mctp_miscdev)`
- DTS match：`aspeed_mctp_match_table`

### `include/uapi/linux/aspeed-mctp.h`

位置：

```text
D:/Code/aspeed/linux/include/uapi/linux/aspeed-mctp.h
```

這是 userspace 看到的 ABI。

負責：

- 定義 `/dev/aspeed-mctpX` packet 格式
- 定義 ioctl number
- 定義 ioctl 用的 struct

重要項目：

```c
ASPEED_MCTP_PCIE_VDM_HDR_SIZE
ASPEED_MCTP_READY
ASPEED_MCTP_IOCTL_GET_BDF
ASPEED_MCTP_IOCTL_GET_MEDIUM_ID
ASPEED_MCTP_IOCTL_GET_MTU
ASPEED_MCTP_IOCTL_REGISTER_DEFAULT_HANDLER
ASPEED_MCTP_IOCTL_REGISTER_TYPE_HANDLER
ASPEED_MCTP_IOCTL_UNREGISTER_TYPE_HANDLER
ASPEED_MCTP_IOCTL_GET_EID_INFO
ASPEED_MCTP_IOCTL_SET_EID_INFO
ASPEED_MCTP_IOCTL_SET_OWN_EID
```

Intel `libmctp` 的 `astpcie.c` 就是用這些 ioctl 跟 driver 溝通。

### `include/linux/aspeed-mctp.h`

位置：

```text
D:/Code/aspeed/linux/include/linux/aspeed-mctp.h
```

這是 kernel 內部用的 header。

負責：

- 定義 PCIe transport header
- 定義 MCTP protocol header
- 定義 Aspeed driver 內部 packet buffer layout
- 定義 Aspeed MCTP MTU
- 宣告 Aspeed MCTP exported helper

重要定義：

```c
#define PCIE_VDM_HDR_SIZE 16
#define MCTP_BTU_SIZE 64
#define ASPEED_MCTP_MTU MCTP_BTU_SIZE
```

packet layout：

```c
struct mctp_pcie_packet_data {
    u32 hdr[PCIE_VDM_HDR_SIZE_DW];
    u32 payload[PCIE_VDM_DATA_SIZE_DW];
};

struct mctp_pcie_packet {
    struct mctp_pcie_packet_data data;
    u32 size;
};
```

目前這份 code 實際設定：

```text
PCIe VDM header = 16 bytes
MCTP payload area = 64 bytes
單包 buffer = 80 bytes
```

註解說 Aspeed MCTP MTU 可以是 64/128/256，但目前這份 tree 固定是 64。

### `drivers/net/mctp/mctp-pcie-vdm.c`

位置：

```text
D:/Code/aspeed/linux/drivers/net/mctp/mctp-pcie-vdm.c
```

這是 kernel socket-MCTP 的 PCIe VDM transport。

負責：

- 建立 `mctppciX`
- TX 時建立 PCIe VDM header
- RX 時把 packet 轉成 skb
- 把 skb 餵進 Linux MCTP stack

重要項目：

```c
MCTP_PCIE_VDM_HDR_SIZE
MCTP_PCIE_VDM_MSG_CODE
MCTP_PCIE_VDM_VENDOR_ID
struct mctp_pcie_vdm_hdr
mctp_pcie_vdm_xmit()
mctp_pcie_vdm_hdr_create()
mctp_pcie_vdm_receive_packet()
mctp_pcie_vdm_add_dev()
mctp_pcie_vdm_remove_dev()
```

它定義的 netdev MTU：

```c
#define MCTP_PCIE_VDM_MIN_MTU (64 + 4)
#define MCTP_PCIE_VDM_MAX_MTU 512
```

注意：netdev 宣告最大 512，不代表 Aspeed driver 目前就能用 512。實際限制要看 Aspeed driver 的 `ASPEED_MCTP_MTU`，目前是 64。

### `include/linux/mctp-pcie-vdm.h`

位置：

```text
D:/Code/aspeed/linux/include/linux/mctp-pcie-vdm.h
```

這是 hardware driver 與 generic PCIe VDM transport 之間的 callback interface。

核心結構：

```c
struct mctp_pcie_vdm_ops {
    int (*send_packet)(struct device *dev, u8 *data, size_t len);
    u8 *(*recv_packet)(struct device *dev);
    void (*free_packet)(void *packet);
    void (*uninit)(struct device *dev);
};
```

`aspeed-mctp.c` 實作這組 callback，然後傳給 `mctp_pcie_vdm_add_dev()`。

## kernel build 相關檔案

### `drivers/soc/aspeed/Kconfig`

位置：

```text
D:/Code/aspeed/linux/drivers/soc/aspeed/Kconfig
```

重要 config：

```text
CONFIG_ASPEED_MCTP
```

用途：

```text
啟用 Aspeed MCTP Controller driver
```

### `drivers/soc/aspeed/Makefile`

位置：

```text
D:/Code/aspeed/linux/drivers/soc/aspeed/Makefile
```

重要規則：

```make
obj-$(CONFIG_ASPEED_MCTP) += aspeed-mctp.o
```

### `drivers/net/mctp/Kconfig`

位置：

```text
D:/Code/aspeed/linux/drivers/net/mctp/Kconfig
```

重要 config：

```text
CONFIG_MCTP_TRANSPORT_PCIE_VDM
```

用途：

```text
啟用 kernel socket-MCTP 的 PCIe VDM transport
```

### `drivers/net/mctp/Makefile`

位置：

```text
D:/Code/aspeed/linux/drivers/net/mctp/Makefile
```

重要規則：

```make
obj-$(CONFIG_MCTP_TRANSPORT_PCIE_VDM) += mctp-pcie-vdm.o
```

### defconfig

AST2600：

```text
D:/Code/aspeed/linux/arch/arm/configs/aspeed_g6_defconfig
```

重要設定：

```text
CONFIG_ASPEED_MCTP=y
```

AST2700：

```text
D:/Code/aspeed/linux/arch/arm64/configs/aspeed_g7_defconfig
```

重要設定：

```text
CONFIG_MCTP_TRANSPORT_PCIE_VDM=y
CONFIG_ASPEED_MCTP=y
```

## Device Tree 相關檔案

### AST2600

檔案：

```text
D:/Code/aspeed/linux/arch/arm/boot/dts/aspeed/aspeed-g6.dtsi
```

MCTP controller node：

```dts
mctp0: mctp@1e6e8000 {
    compatible = "aspeed,ast2600-mctp";
    reg = <0x1e6e8000 0x30>;
    interrupt-names = "mctp", "pcie";
    resets = <&syscon ASPEED_RESET_DEV_MCTP>;
    aspeed,scu = <&syscon>;
    aspeed,pcieh = <&pcie_phy0>;
    status = "disabled";
};

mctp1: mctp@1e6f9000 {
    compatible = "aspeed,ast2600-mctp";
    reg = <0x1e6f9000 0x30>;
    interrupt-names = "pcie";
    pcie_rc;
    resets = <&syscon ASPEED_RESET_RC_MCTP>, <&syscon ASPEED_RESET_DEV_MCTP>;
    aspeed,scu = <&syscon>;
    aspeed,pcieh = <&pcie_phy1>;
    status = "disabled";
};
```

board-level DTS 需要把對應 node 改成：

```dts
status = "okay";
```

有些平台也會配置 reserved memory 給 MCTP DMA buffer。

### AST2700

檔案：

```text
D:/Code/aspeed/linux/arch/arm64/boot/dts/aspeed/aspeed-g7.dtsi
```

MCTP controller node：

```dts
mctp0: mctp0@12c06000 {
    compatible = "aspeed,ast2700-mctp0";
};

mctp1: mctp1@12c07000 {
    compatible = "aspeed,ast2700-mctp1";
};
```

常見 board DTS：

```text
D:/Code/aspeed/linux/arch/arm64/boot/dts/aspeed/ast2700-dcscm.dts
D:/Code/aspeed/linux/arch/arm64/boot/dts/aspeed/ast2700a1-dcscm.dts
D:/Code/aspeed/linux/arch/arm64/boot/dts/aspeed/ast2700-evb.dts
D:/Code/aspeed/linux/arch/arm64/boot/dts/aspeed/ast2700a1-evb.dts
```

## Intel libmctp 相關檔案

位置：

```text
D:/Code/intel/libmctp
```

### `core.c`

負責：

- MCTP core
- packet buffer
- message TX/RX
- EID bus registration
- tag handling
- fragment/reassembly
- callback dispatch

### `libmctp.h`

負責：

- 對 application 暴露的 public API
- core structure 定義
- binding interface 定義

### `astpcie.c`

這是 Intel `libmctp` 的 Aspeed PCIe VDM binding。

負責：

- open `/dev/aspeed-mctp`
- poll/read/write `/dev/aspeed-mctp`
- TX 時組 PCIe VDM header
- RX 時拆 PCIe VDM header
- ioctl get BDF
- ioctl get medium id
- ioctl register default handler
- ioctl register type handler
- ioctl get/set EID mapping

重要函式：

```c
mctp_astpcie_init()
mctp_astpcie_start()
mctp_astpcie_tx()
mctp_astpcie_rx()
mctp_astpcie_poll()
mctp_astpcie_get_bdf()
mctp_astpcie_register_default_handler()
mctp_astpcie_register_type_handler()
```

### `libmctp-astpcie.h`

負責：

- 對 app 暴露 Aspeed PCIe binding API
- 定義 PCIe routing type
- 定義 binding private data

重要 routing：

```c
PCIE_ROUTE_TO_RC = 0
PCIE_ROUTE_BY_ID = 2
PCIE_BROADCAST_FROM_RC = 3
```

binding private data：

```c
struct mctp_astpcie_pkt_private {
    enum mctp_astpcie_msg_routing routing;
    uint16_t remote_id;
};
```

`remote_id` 是 PCIe BDF。TX 時表示 target BDF，RX 時表示 source/requester BDF。

### `astpcie.h`

負責：

- 定義 userspace 看的 PCIe VDM header layout
- 定義 DMTF VDM constant
- 定義 routing、payload length、requester BDF、target BDF、padding 相關 macro
- 定義預設 device node

```c
#define AST_DRV_FILE "/dev/aspeed-mctp"
```

### `linux/aspeed-mctp.h`

這是 Intel repo 內附的 Aspeed kernel UAPI copy。

負責：

- 讓 Intel `libmctp` build 時有 Aspeed ioctl 定義
- 定義 `/dev/aspeed-mctp` packet format
- 定義 `ASPEED_MCTP_IOCTL_*`

## OpenBMC / Yocto 相關檔案

位置：

```text
D:/Code/aspeed/openbmc
```

### Intel libmctp recipe

檔案：

```text
meta-aspeed-sdk/recipes-phosphor/pmci/libmctp-intel_git.bb
```

負責：

- fetch Intel-BMC/libmctp
- 用 CMake build
- 產生 `libmctp-intel`

### Intel libmctp bbappend

檔案：

```text
meta-aspeed-sdk/recipes-phosphor/pmci/libmctp-intel_git.bbappend
```

負責：

- 對 Intel `libmctp` 套 Aspeed patch

重要 patch：

```text
meta-aspeed-sdk/recipes-phosphor/pmci/libmctp-intel/0003-Add-function-to-change-the-file-name-of-mctp-device.patch
```

這個 patch 讓 `astpcie` binding 可以指定 device node，不必固定只用 `/dev/aspeed-mctp`。

### libmctp-intel test recipe

檔案：

```text
meta-aspeed-sdk/recipes-phosphor/pmci/libmctp-intel-test_git.bb
```

負責：

- build Intel libmctp 測試程式
- depends on `libmctp-intel`

### PCIe 測試工具

檔案：

```text
meta-aspeed-sdk/recipes-phosphor/pmci/libmctp-intel-test/mctp-astpcie-test.c
```

負責：

- 用 Intel `libmctp` 測 PCIe VDM
- get BDF
- requester mode
- fake responder mode
- 指定 routing type
- 指定 destination bus/dev/function
- 指定 destination EID/source EID
- 指定 MCTP message type 與 command payload

## MCTP 上層協定與傳輸內容

MCTP 本身只負責端點之間的訊息封包化、路由、分片與重組；真正的管理語意通常由上層協定承載。換句話說，PCIe VDM 是 transport，MCTP 是 message transport layer，而 SPDM、PLDM、NVMe-MI 這類協定才是實際要傳遞的管理內容。

| 類別 | Protocol over MCTP | 實際傳輸內容 | 常見用途 |
| --- | --- | --- | --- |
| 安全 (Security) | SPDM | 憑證、金鑰交換、硬體指紋驗證 (Attestation) | 裝置身分驗證、量測啟動狀態、建立安全通道 |
| 監控 (Monitoring) | PLDM for Monitoring | 感測器數值、報警訊息、門檻值設定 | 溫度、電壓、電流、風扇、錯誤事件監控 |
| 韌體 (Firmware) | PLDM for FW Update | 韌體 binary、更新進度、版本回傳 | 裝置韌體更新、更新狀態查詢、版本管理 |
| 配置 (Control) | PLDM / NVMe-MI | 功耗限制 (Power Cap)、風扇轉速需求、硬碟格式化指令 | 平台控制、儲存裝置管理、操作參數設定 |
| 拓樸 (Topology) | PLDM PDR | 設備內部硬體結構圖、slot 位置資訊 | 建立 inventory、描述 FRU/slot/sensor 關係 |

可以把資料流想成下面這種分層：

```text
SPDM / PLDM / NVMe-MI payload
    |
    v
MCTP message: EID, message type, tag, fragmentation
    |
    v
PCIe VDM transport packet
    |
    v
Aspeed MCTP hardware / PCIe fabric
```

在 BMC 實作上，application 或 daemon 通常只關心 SPDM、PLDM、NVMe-MI 的 request/response；MCTP stack 負責把這些 payload 封裝成 MCTP message，再交給 PCIe VDM transport 送到對應 endpoint。

## `/dev/aspeed-mctpX` TX flow

這是 legacy Intel `libmctp` 路線。

```text
Application
    |
    v
Intel libmctp core.c
    |
    v
Intel astpcie.c
    |
    v
write(/dev/aspeed-mctpX)
    |
    v
aspeed_mctp_write()
    |
    v
mctp_client.tx_queue
    |
    v
aspeed_mctp_tx_tasklet()
    |
    v
aspeed_mctp_emit_tx_cmd()
    |
    v
aspeed_mctp_tx_trigger()
    |
    v
Aspeed MCTP hardware DMA
    |
    v
PCIe VDM packet
```

詳細流程：

1. Application 呼叫 Intel `libmctp` API，例如 `mctp_message_tx()`。

2. `core.c` 建立 MCTP message，處理 EID/tag/header。

3. `astpcie.c` 根據 `mctp_astpcie_pkt_private` 取得：

```text
routing type
remote BDF
```

4. `astpcie.c` 組 PCIe VDM header。

會填：

- route type
- data length
- requester BDF
- target BDF
- padding length
- DMTF VDM vendor ID
- PCIe VDM message code

5. `astpcie.c` 對 `/dev/aspeed-mctpX` 做 `write()`。

寫入內容是：

```text
PCIe VDM header + MCTP packet
```

6. kernel 進入 `aspeed_mctp_write()`。

7. driver 把 userspace buffer copy 成 `struct mctp_pcie_packet`。

8. packet 放進該 client 的 `tx_queue`。

9. driver schedule TX tasklet。

10. `aspeed_mctp_tx_tasklet()` 從 client TX queue 取 packet。

11. `aspeed_mctp_emit_tx_cmd()` 做兩件事：

- 把 packet bytes copy 到 `tx->data` DMA buffer
- 在 `tx->cmd` command ring 建立一筆 TX command

12. TX command 告訴硬體：

- packet data DMA address
- packet size
- target BDF
- routing type
- padding length
- 是否送完 interrupt

13. `aspeed_mctp_emit_tx_cmd()` 更新：

```c
tx->wr_ptr = (tx->wr_ptr + 1) % TX_PACKET_COUNT;
```

14. `aspeed_mctp_tx_trigger()` 通知硬體。

它會：

- 必要時在最後一筆 command 設 `TX_INTERRUPT_AFTER_CMD`
- 寫 `ASPEED_MCTP_TX_BUF_WR_PTR`
- 設 `ASPEED_MCTP_CTRL.TX_CMD_TRIGGER`
- polling 等硬體清掉 `TX_CMD_TRIGGER`

15. Aspeed MCTP hardware 透過 DMA 讀取 command 和 packet data。

16. hardware 把 packet 作為 PCIe VDM 送到 PCIe fabric。

17. TX 完成後，硬體產生 interrupt 或更新狀態。

18. driver 更新 TX ring 狀態並釋放 packet。

## TX channel 裡面有什麼

定義：

```c
struct mctp_channel {
    struct mctp_buffer data;
    struct mctp_buffer cmd;
    struct tasklet_struct tasklet;
    u32 buffer_count;
    u32 rd_ptr;
    u32 wr_ptr;
    bool stopped;
};
```

buffer 定義：

```c
struct mctp_buffer {
    void *vaddr;
    dma_addr_t dma_handle;
};
```

TX channel 裡的東西：

### `tx.data`

真正要給硬體送出去的 packet DMA buffer。

內容通常是：

```text
PCIe VDM header + MCTP payload
```

對 AST2600/AST2700 新格式，`vdm_hdr_direct_xfer = true`，driver 會把整個 `struct mctp_pcie_packet_data` copy 到 `tx.data`。

### `tx.cmd`

硬體 TX command ring。

每一筆 command 描述：

- packet data 在 DMA memory 的位置
- packet size
- target ID/BDF
- routing type
- tag owner
- padding length
- 是否最後一筆
- 是否送完 interrupt

AST2700/G7 需要 64-bit DMA address，所以 command 格式是：

```c
struct aspeed_g7_mctp_tx_cmd {
    u32 tx_lo;
    u32 tx_mid;
    u32 tx_hi;
    u32 reserved;
};
```

AST2600 等舊格式：

```c
struct aspeed_mctp_tx_cmd {
    u32 tx_lo;
    u32 tx_hi;
};
```

### `tx.tasklet`

TX bottom-half。

用途：

- 從 `mctp_client.tx_queue` 取 packet
- 填入 `tx.data`
- 建立 `tx.cmd`
- 呼叫 `aspeed_mctp_tx_trigger()`

### `tx.wr_ptr`

下一筆 TX command 要寫到哪個 ring slot。

`aspeed_mctp_emit_tx_cmd()` 會用它來決定 offset。

### `tx.rd_ptr`

TX ring 已完成/已讀取的位置。

interrupt handler 或 tasklet 會用它追蹤硬體送到哪裡。

### `tx.stopped`

channel 停止狀態。TX 主要 flow 裡不是最核心，但用來避免硬體/driver 狀態不一致時繼續送。

## `/dev/aspeed-mctpX` RX flow

這也是 legacy Intel `libmctp` 路線。

```text
PCIe VDM packet
    |
    v
Aspeed MCTP hardware
    |
    v
RX DMA buffer
    |
    v
interrupt
    |
    v
aspeed_mctp_rx_tasklet()
    |
    v
aspeed_mctp_dispatch_packet()
    |
    v
mctp_client.rx_queue
    |
    v
read(/dev/aspeed-mctpX)
    |
    v
Intel astpcie.c
    |
    v
Intel libmctp core.c
    |
    v
Application callback
```

詳細流程：

1. 遠端 endpoint 送出 PCIe VDM MCTP packet。

2. Aspeed MCTP hardware 收到 packet。

3. hardware 透過 DMA 寫入 RX buffer。

4. hardware 更新 RX ring/command 狀態。

5. hardware 產生 RX interrupt。

6. interrupt handler schedule RX tasklet。

7. `aspeed_mctp_rx_tasklet()` 掃 RX buffer。

8. driver 檢查 packet length 是否超過 `ASPEED_MCTP_MTU`。

9. driver 配置 `struct mctp_pcie_packet`。

10. driver 複製 PCIe VDM header 與 payload。

11. driver 依需要做 VDM header endian swap。

12. driver 呼叫 dispatch。

dispatch 可能送到：

- default client
- 註冊特定 MCTP message type 的 client
- kernel socket-MCTP PCIe VDM path

13. userspace client `poll()` 看到可讀。

14. userspace 對 `/dev/aspeed-mctpX` 做 `read()`。

15. Intel `astpcie.c` 收到 packet。

16. Intel `astpcie.c` 解析 PCIe VDM header，取出 routing/source BDF。

17. Intel `libmctp` core 處理 MCTP packet 並 dispatch 給 app callback。

## kernel socket-MCTP TX flow

這是建議長期走的方向。

```text
Application
    |
    v
AF_MCTP socket
    |
    v
Linux kernel MCTP core
    |
    v
mctppciX
    |
    v
mctp-pcie-vdm.c
    |
    v
aspeed_mctp_pcie_vdm_op_send_pkt()
    |
    v
aspeed_mctp_send_packet()
    |
    v
Aspeed TX tasklet/ring
    |
    v
Aspeed MCTP hardware
```

詳細流程：

1. Application 用 AF_MCTP socket send。

2. kernel MCTP core 根據 route/neigh 選到 `mctppciX`。

3. `mctp-pcie-vdm.c` 建立 PCIe VDM header。

4. `mctp-pcie-vdm.c` 呼叫硬體 callback：

```c
vdm_dev->callback_ops->send_packet()
```

5. 在 Aspeed 平台，callback 是：

```c
aspeed_mctp_pcie_vdm_op_send_pkt()
```

6. `aspeed_mctp_pcie_vdm_op_send_pkt()` 配置 `struct mctp_pcie_packet`。

7. 呼叫：

```c
aspeed_mctp_send_packet()
```

8. 後面走同一套 Aspeed TX tasklet、TX ring、DMA、hardware flow。

## kernel socket-MCTP RX flow

```text
PCIe VDM packet
    |
    v
Aspeed MCTP hardware
    |
    v
aspeed-mctp.c RX tasklet
    |
    v
mctp_pcie_vdm_receive_packet()
    |
    v
mctp-pcie-vdm.c
    |
    v
Linux MCTP networking stack
    |
    v
AF_MCTP socket user
```

詳細流程：

1. Aspeed hardware 收到 PCIe VDM packet。

2. `aspeed-mctp.c` RX tasklet 收 packet。

3. driver 通知 generic PCIe VDM transport：

```c
mctp_pcie_vdm_receive_packet(priv->ndev)
```

4. `mctp-pcie-vdm.c` 呼叫 callback：

```c
vdm_dev->callback_ops->recv_packet()
```

5. 在 Aspeed 平台，callback 是：

```c
aspeed_mctp_pcie_vdm_op_recv_pkt()
```

6. generic PCIe VDM transport 把 packet 轉成 skb。

7. skb 進入 Linux MCTP networking stack。

8. AF_MCTP socket user 收到 message。

## VDM packet 大小上限

這裡要分層看。

### Aspeed internal driver 目前設定

檔案：

```text
D:/Code/aspeed/linux/include/linux/aspeed-mctp.h
```

目前定義：

```c
#define PCIE_VDM_HDR_SIZE 16
#define MCTP_BTU_SIZE 64
#define ASPEED_MCTP_MTU MCTP_BTU_SIZE
```

所以目前實際 buffer 是：

```text
PCIe VDM header: 16 bytes
MCTP payload area: 64 bytes
total: 80 bytes
```

### Aspeed 硬體/driver 註解

同一個 header 註解寫：

```text
The MTU of the ASPEED MCTP can be 64/128/256
```

也就是設計上可支援 64/128/256，但這份 tree 目前選 64。

### kernel `mctp-pcie-vdm.c` netdev 設定

檔案：

```text
D:/Code/aspeed/linux/drivers/net/mctp/mctp-pcie-vdm.c
```

定義：

```c
#define MCTP_PCIE_VDM_MIN_MTU (64 + 4)
#define MCTP_PCIE_VDM_MAX_MTU 512
```

這是 socket-MCTP netdev 層的設定。實際能不能送到 512，還要看下層 hardware driver 的 packet buffer 與 `ASPEED_MCTP_MTU`。

### 實際結論

目前不改 code/config 時，保守看：

```text
單包 MCTP payload area <= 64 bytes
單包 PCIe VDM header + payload <= 80 bytes
```

如果要改成 128 或 256，需要同步確認：

- `ASPEED_MCTP_MTU`
- TX/RX packet buffer layout
- hardware maximum payload size 設定
- `ASPEED_MCTP_ENGINE_CTRL` 裡 TX/RX max payload 設定
- socket-MCTP netdev MTU
- 對端 endpoint 支援的 MCTP BTU/MTU

## 建議簡化方向

因為 Intel-BMC `libmctp` 已停止維護，長期建議不要再擴這條路：

```text
Application
    |
    v
Intel libmctp
    |
    v
/dev/aspeed-mctpX
```

建議改成：

```text
Application
    |
    v
AF_MCTP socket / mctpd / kernel MCTP
    |
    v
mctppciX
    |
    v
mctp-pcie-vdm.c
    |
    v
aspeed-mctp.c
    |
    v
Aspeed PCIe MCTP hardware
```

建議保留：

- `drivers/soc/aspeed/aspeed-mctp.c`
- `drivers/net/mctp/mctp-pcie-vdm.c`
- `include/linux/mctp-pcie-vdm.h`
- `include/linux/aspeed-mctp.h`
- `include/uapi/linux/aspeed-mctp.h`
- MCTP controller DTS node
- `CONFIG_ASPEED_MCTP`
- `CONFIG_MCTP`
- `CONFIG_MCTP_TRANSPORT_PCIE_VDM`

建議逐步避免擴充：

- Intel `libmctp` `astpcie.c`
- 直接操作 `/dev/aspeed-mctpX` 的新 userspace 程式
- `mctp-astpcie-test` 作為主要驗證工具

短期可以雙軌共存：

```text
舊 app/test tool -> /dev/aspeed-mctpX
新 app/daemon   -> AF_MCTP socket / mctppciX
```

長期收斂到 kernel socket-MCTP，維護成本會低很多，也比較貼近 OpenBMC upstream 的方向。
