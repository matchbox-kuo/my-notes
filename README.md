# BMC / PCIe Study Notes

這個 repository 收集 AST2700、OpenBMC、PCIe Endpoint、MCTP over PCIe VDM 等主題的學習與實務筆記。

GitHub Markdown 不支援像 Excel sheet 那樣的原生分頁，因此這裡保留多個 `.md` 檔，並用 README 作為入口頁。

## 快速入口

| 筆記 | 內容 |
| --- | --- |
| [AST2700 DC-SCM 平台管理架構與 OpenBMC 實務](./AST2700%20DC-SCM%20平台管理架構與%20OpenBMC%20實務.md) | 以 AST2700 與 DC-SCM 為主軸，整理 OpenBMC、PCIe RC/EP、Caliptra、LTPI、MCTP/PLDM 與 GPIO 控制。 |
| [PCIe 協定與韌體開發核心](./PCIe%20協定與韌體開發核心.md) | PCIe TLP、PCIe Configuration Space、Link Training、Enumeration、Doorbell、Interrupt、SQ/CQ、Mailbox、MMBI、MCTP over PCIe VDM 與 MFD。 |
| [OpenBMC 開發環境與工具速查](./OpenBMC%20開發環境與工具速查.md) | OpenBMC 編譯、QEMU、常用指令、ConfigFS、Kconfig 與 Makefile 速查。 |
| [AST2700 MCTP TX/RX Code Flow](./MCTP_AST2700_flow_notes.md) | AST2700 kernel socket-MCTP、PCIe VDM netdev 與 TX/RX code flow。 |

## 建議閱讀順序

1. [AST2700 DC-SCM 平台管理架構與 OpenBMC 實務](./AST2700%20DC-SCM%20平台管理架構與%20OpenBMC%20實務.md)
2. [PCIe 協定與韌體開發核心](./PCIe%20協定與韌體開發核心.md)
3. [AST2700 MCTP TX/RX Code Flow](./MCTP_AST2700_flow_notes.md)
4. [OpenBMC 開發環境與工具速查](./OpenBMC%20開發環境與工具速查.md)

## 使用方式

在 GitHub 上可以直接從上面的連結切換不同筆記。若要找特定主題，建議使用瀏覽器搜尋或 GitHub repository 搜尋。
