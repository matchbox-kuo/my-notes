# BMC / PCIe Study Notes

這個 repository 收集 AST2700、OpenBMC、PCIe Endpoint、MCTP over PCIe VDM 等主題的學習與實務筆記。

GitHub Markdown 不支援像 Excel sheet 那樣的原生分頁，因此這裡保留多個 `.md` 檔，並用 README 作為入口頁。

## 快速入口

| 筆記 | 內容 |
| --- | --- |
| [AST2700 PCIe 韌體開發與 OpenBMC 實務手冊](./AST2700%20PCIe%20韌體開發與%20OpenBMC實務手冊.md) | AST2700、OpenBMC、PCIe、DC-SCM、LTPI、MCTP、PLDM 與 GPIO 控制流程的整體整理。 |
| [PCIe 協定與韌體開發核心](./PCIe%20協定與韌體開發核心.md) | PCIe TLP、Configuration Space、Link Training、Enumeration、Doorbell、Interrupt、SQ/CQ、Mailbox、MMBI、MCTP over PCIe VDM 與 MFD。 |
| [OpenBMC 開發環境與工具速查](./OpenBMC%20開發環境與工具速查.md) | OpenBMC 編譯、QEMU、常用指令、ConfigFS、Kconfig 與 Makefile 速查。 |
| [MCTP over PCIe VDM 流程整理](./MCTP-over-PCIe-VDM-flow.md) | Intel libmctp、Aspeed kernel driver、kernel socket-MCTP、TX/RX flow、Device Tree、Yocto recipe 與 VDM packet 限制。 |

## 建議閱讀順序

1. [AST2700 PCIe 韌體開發與 OpenBMC 實務手冊](./AST2700%20PCIe%20韌體開發與%20OpenBMC實務手冊.md)
2. [PCIe 協定與韌體開發核心](./PCIe%20協定與韌體開發核心.md)
3. [MCTP over PCIe VDM 流程整理](./MCTP-over-PCIe-VDM-flow.md)
4. [OpenBMC 開發環境與工具速查](./OpenBMC%20開發環境與工具速查.md)

## 使用方式

在 GitHub 上可以直接從上面的連結切換不同筆記。若要找特定主題，建議使用瀏覽器搜尋或 GitHub repository 搜尋。
