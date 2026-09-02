# FPGA 三平台开发调试方案(调研存档)

Zynq(MYIR Z-Turn 7020 / Digilent Zybo Z7-20)与国产紫光同创 Pango(Alinx PGL22G)三块开发板的双路线开发调试方案。含三平台硬件对比、逐级实操路线(L-1 → L7)、TCL/Verilog 示例、失败处理预案与官方资源汇总。

> 状态:方案已确认,待执行 · 2026-09-01
> 定位:硬件 / 嵌入式方向调研存档

## 路线划分

- **路线 A · Xilinx Zynq**(Z-Turn / Zybo Z7-20):芯片相同、工具链相同(Vivado + Vitis),可共享 90% 学习内容,能跑 Linux / 双核 AMP。
- **路线 B · 国产 Pango**(Alinx PGL22G):紫光同创 PGL22G 纯 FPGA,Pango Design Suite,独立学习路线。

## 文件

| 文件 | 说明 |
|---|---|
| [fpga-dev-platforms-plan.md](fpga-dev-platforms-plan.md) | 完整方案:三平台对比表 + 路线 A/B 逐级步骤(L-1→L7)+ 失败预案 + 工具链与资源汇总 |
