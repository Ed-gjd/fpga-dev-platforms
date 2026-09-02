# FPGA 三平台开发调试方案汇总

  本文档覆盖三块开发板的完整开发调试路线，按芯片厂分两条独立路线。

  编写日期：2026-09-01
  状态：方案已确认，待执行

---

## 1. 三平台硬件概览

### 1.1 对比表

| 项 | MYIR Z-Turn 7020 | Digilent Zybo Z7-20 | Alinx PGL22 |
|---|---|---|---|
| 芯片 | Xilinx Zynq-7020 | Xilinx Zynq-7020 | 紫光同创 PGL22G |
| 架构 | ARM + FPGA (SoC) | ARM + FPGA (SoC) | 纯 FPGA |
| Part 参数 | xc7z020clg400-1 | xc7z020clg400-1 | PGL22G-6MBG324 |
| DDR3 | 1GB | 1GB | 256MB (16bit) |
| Flash | 16MB QSPI | 16MB QSPI | 16MB QSPI |
| USB 芯片 | FT2232HL 双通道 | FT2232HQ 双通道 | FT2232HL 双通道 |
| JTAG 通道 | 通道 A | 通道 A | 通道 A |
| UART 通道 | 通道 B → COM 口 | 通道 B → COM6 | 通道 B → COM 口 |
| PS UART | UART1 (MIO 48..49) | UART1 (MIO 48..49) | N/A（纯 FPGA） |
| 波特率 | 115200 | 115200 | 115200 |
| 能跑 Linux | ✅ 双核 Cortex-A9 | ✅ 双核 Cortex-A9 | ❌ 无硬 ARM 核 |
| 开发工具 | Vivado + Vitis | Vivado + Vitis | Pango Design Suite |
| 工具大小 | ~40-100 GB | ~40-100 GB | ~6 GB |
| 板级预设 | MYIR 官方提供 | Digilent GitHub 维护 | Alinx 配套资料 |
| 生态成熟度 | 中 | 高 | 低（国产小众） |
| 价格参考 | ~¥600 | ~¥300 | ~¥300 |

### 1.2 路线划分

  路线 A — Xilinx Zynq 系列（Z-Turn / Zybo Z7-20）
    两块板子芯片相同，工具链相同，仅板级预设和外设引脚分配不同。
    可共享 90% 的学习内容。

  路线 B — 国产 Pango 系列（Alinx PGL22）
    完全不同的芯片架构和工具链，独立学习路线。
    纯 FPGA，无 ARM 核，不能直接跑 Linux。

---

## 2. 路线 A · Xilinx Zynq（Z-Turn / Zybo Z7-20）

### 2.1 工具链安装

  Vivado + Vitis 一体化安装，WebPACK 免费版够用。

  安装步骤：
    1. 下载 Vivado 2022.2 或 2023.2（约 50-100GB）
    2. 安装时勾选：Vivado + Vitis + SDK
    3. 器件选择：只勾 Zynq-7000 系列（可瘦身到 40GB）
    4. 安装 License（WebPACK 免费自动激活）

  板级预设安装：

    Zybo Z7-20：
      git clone https://github.com/digilent/vivado-boards.git
      拷贝 vivado-boards/new/board_files/* 到 Vivado 安装目录的 board_files/

    Z-Turn 7020：
      从 MYIR 官网下载 Z-Turn BSP 包
      或手动创建工程时选 xc7z020clg400-1 + 手动配 PS 参数

  验证安装：
    vivado -version
    xsct -version
    xsdb -version

### 2.2 学习级别

#### L-1 · 串口连通验证

  目标：确认 Win 能看到板子串口输出

  步骤：
    1. 设备管理器找 COM 口（FT2232 通道 B）
    2. PuTTY 打开：putty -serial COMx -sercfg 115200,8,n,1
    3. 按板子复位键

  预期结果：
    情况 A — 有输出：板子有 Demo 固件，能看到 UART 菜单或启动日志
    情况 B — 无输出：空片或固件损坏，需烧录

  验证标准：PuTTY 窗口出现字符输出

#### L0 · 板级预设 & 工程模板

  目标：Vivado 新建工程能看到板子型号

  验证：新建工程 → Board 选择页 → 搜 "Zybo" 或手动选 xc7z020clg400-1

#### L1 · Hello World（CLI 全流程）

  目标：Vivado 生成硬件 → Vitis 编译 C → xsdb 烧录 → 串口看到 Hello

  TCL 脚本（硬件设计）：
    create_project zynq_l1 ./proj_l1 -part xc7z020clg400-1
    set_property board_part <board_preset> [current_project]
    create_bd_design "system"
    create_bd_cell -type ip -vlnv xilinx.com:ip:processing_system7:5.5 ps_0
    apply_bd_automation -rule xilinx.com:bd_rule:processing_system7 \
        -config {make_external "FIXED_IO, DDR" apply_board_preset "1"} \
        [get_bd_cells ps_0]
    # 关键：确认 UART1 启用（Z-Turn/Zybo 都是 UART1）
    generate_target all [get_files system.bd]
    write_hw_platform -fixed -force -include_bit -file ./zturn_l1.xsa

  执行：vivado -mode batch -source l1_hello.tcl

  C 代码（hello.c）：
    #include <stdio.h>
    #include "platform.h"
    #include "xil_printf.h"
    int main() {
        init_platform();
        print("Hello from Zynq-7020!\r\n");
        cleanup_platform();
        return 0;
    }

  xsct 脚本（编译）：
    hsi::open_hw_design ./zturn_l1.xsa
    hsi::set_property CONFIG.stdout "ps7_uart_1" [hsi::current_sw_processor]
    hsi::generate_app -hw ./zturn_l1.xsa -os standalone \
        -proc ps7_cortexa9_0 -app hello_world -compile

  xsdb 脚本（烧录）：
    connect
    target 4
    FPGA -f ./system.bit
    target 2
    dow ./hello_world.elf
    con
    disconnect

  验证：PuTTY 看到 "Hello from Zynq-7020!"

#### L2 · PS 外设控制（GPIO/Timer/中断）

  目标：按键控制 LED，定时器中断，串口回显

  关键命令：
    xsdb> rwr 0xe000a000 0xFF    # 写 GPIO 寄存器
    xsdb> rrd 0xe000a000         # 读 GPIO

  验证：按按键 LED 亮灭 + 串口打印 "Button pressed"

#### L3 · PL 自定义 AXI IP

  目标：FPGA 逻辑区写自定义 IP，PS 通过 AXI 总线访问

  步骤：
    1. Vivado 创建自定义 IP（AXI4-Lite 从接口）
    2. 连到 AXI Interconnect
    3. 生成地址映射
    4. PS 端 C 代码读写 PL 寄存器

  验证：xsdb 读写 PL 寄存器，读回值与写入一致

#### L4 · JTAG 调试 + ILA

  目标：断点调试 ARM 程序 + 抓 FPGA 内部信号波形

  xsdb 调试命令：
    xsdb> b main                  # 设断点
    xsdb> con                     # 运行到断点
    xsdb> step                    # 单步
    xsdb> print var               # 查看变量
    xsdb> mrd 0x10000 16          # 读内存

  ILA 抓波形：
    1. Vivado 中添加 ILA IP，连到要观测的信号
    2. 综合生成比特流
    3. xsdb> ila::read_data ila_0
    4. xsdb> ila::write_vcd ila_0 waveform.vcd

  验证：断点命中 + VCD 波形用 GTKWave 打开看时序

#### L5 · Linux on Zynq

  目标：板子跑 Linux，串口当 console

  步骤：
    1. 编译 U-Boot（Xilinx 分支）
    2. 编译 Linux Kernel（Xilinx 分支）
    3. Buildroot 构建根文件系统
    4. bootgen 打包 BOOT.BIN（FSBL + bitstream + u-boot）
    5. SD 卡启动 或 JTAG 加载

  验证：串口出现 Linux login 提示，能登录 shell

#### L6 · 高级调试

  内容：
    GDB 远程调试（gdbserver over Ethernet）
    perf 性能分析
    ftrace 内核追踪
    双核 AMP（核0 Linux + 核1 裸机实时）

  验证：远程 GDB 命中断点

#### L7 · 实战项目

  建议项目（从简到难）：
    1. 串口回显机器人（UART 中断 + 状态机）
    2. AXI DMA 高速数据搬运
    3. HDMI 视频输出（Z-Turn 有 HDMI 接口）
    4. Linux 自写内核驱动
    5. 双核 AMP 架构

### 2.3 Z-Turn vs Zybo 差异点

  虽然芯片相同，以下参数需分别配置：

| 参数 | Z-Turn 7020 | Zybo Z7-20 |
|---|---|---|
| board_part | MYIR 预设（手动配） | digilentinc.com:zybo_z7_20:part0:1.2 |
| PS UART | UART1 (MIO 48..49) | UART1 (MIO 48..49) |
| LED 引脚 | 查 MYIR 原理图 | MIO LED 或 PL LED |
| 按键引脚 | 查 MYIR 原理图 | 查 Digilent 参考手册 |
| 参考文档 | MYIR 官方手册 | Zybo Z7 Reference Manual |

### 2.4 失败处理预案

#### 串口无输出

  排查：
    1. 试不同波特率（9600 / 57600 / 115200）
    2. 检查 COM 口是否正确（设备管理器）
    3. 按复位键看有无反应
    4. 若完全无反应 → 空片 → 需 JTAG 烧录

#### JTAG 连不上

  排查：
    1. 确认 USB 是 FT2232（不是单通道 CH340）
    2. hw_server -d 启动调试服务器
    3. xsdb> connect 重试
    4. 若芯片锁死 → 需 JTAG 强制擦除

#### BSP STDIO 配错

  症状：程序跑但串口没输出
  处理：xsct 里强制设 hsi::set_property CONFIG.stdout "ps7_uart_1"

#### Vivado 综合报错

  常见原因：
    Part 选错 → 改 -part 参数
    板级预设没装 → 手动 import
    License 问题 → WebPACK 免费版够用

---

## 3. 路线 B · Pango PGL22G（Alinx PGL22）

### 3.1 工具链安装

  Pango Design Suite (PDS)，约 6GB，比 Vivado 小 16 倍。

  下载：
    官网：https://www.pangomicro.com/resources/software/pds/
    或从 Alinx 配套资料获取

  安装步骤：
    1. 下载 PDS 安装包（约 1.4-2.4 GB）
    2. 解压到无中文无空格的路径（如 C:\pango\）
    3. 运行安装程序 → 默认装到 C:\pango\PDS_2022.1
    4. 导入 License 文件（联系 Alinx 客服或紫光同创申请，学习免费）

  系统要求：
    内存：8GB 最低，推荐 16GB
    硬盘：安装约 6GB，推荐预留 10GB+
    系统：Win 7/10/11 64 位

  验证安装：
    打开 PDS → 新建工程 → Family: Logos → Device: PGL22G → Package: BG324 → Speed: -6

### 3.2 学习级别

#### L-1 · 串口连通验证

  目标：确认 Win 能看到板子串口输出

  步骤：
    1. 设备管理器找 COM 口（FT2232 通道 B，之前确认是 COM6）
    2. PuTTY 打开：putty -serial COM6 -sercfg 115200,8,n,1
    3. 按板子复位键

  预期结果：
    情况 A — 有输出：板子有 Demo → 先玩 Demo
    情况 B — 无输出：空片 → 装 PDS 烧程序

  验证标准：PuTTY 窗口出现字符

#### L0 · PDS 安装 & 新建工程

  目标：PDS 能正常打开，能新建 PGL22G 工程

  验证：新建工程向导能选到 Logos / PGL22G / BG324 / -6

#### L1 · LED 流水灯

  目标：第一个 Verilog 程序，控制板载 LED

  步骤：
    1. PDS 新建工程（PGL22G-6MBG324, -6）
    2. 写 Verilog 流水灯模块
    3. 写引脚约束文件（.pcf）—— 查 Alinx P22 用户手册的 LED 引脚表
    4. 综合 → 布局布线 → 生成比特流
    5. JTAG 下载到板子

  Verilog 示例（led_blink.v）：
    module led_blink(
        input clk,           // 50MHz 板载晶振
        output reg [3:0] led // 4 个 LED
    );
        reg [25:0] counter;
        always @(posedge clk) begin
            counter <= counter + 1;
            if (counter == 0) led <= ~led;
        end
    endmodule

  引脚约束示例（led.pcf）：
    # 查 Alinx P22 用户手册确认实际引脚
    IO_LOC "clk" P1;
    IO_LOC "led[0]" R1;
    IO_LOC "led[1]" T1;
    IO_LOC "led[2]" U1;
    IO_LOC "led[3]" V1;
    IO_TYPE "clk" LVCMOS33;
    IO_TYPE "led" LVCMOS33;

  验证：板上 LED 按约 0.5 秒节奏闪烁

#### L2 · UART 串口收发

  目标：Verilog 实现 UART TX/RX，通过 COM6 与 PC 通信

  步骤：
    1. 写 UART TX 模块（并转串，115200 波特率）
    2. 写 UART RX 模块（串转并）
    3. 顶层模块例化 TX + RX，做回环（收到什么发回什么）
    4. 约束 UART TX/RX 引脚（查原理图 FT2232 通道 B 连的 FPGA 引脚）

  验证：PuTTY 发字符，板子回显相同字符

#### L3 · 按键消抖 + 状态机

  目标：按键控制 LED 模式切换

  内容：
    按键消抖模块（20ms 定时器采样）
    状态机（模式0:全灭 → 模式1:流水 → 模式2:呼吸 → 循环）

  验证：按一次按键，LED 切换到下一模式

#### L4 · DDR3 控制器调用

  目标：调用 PGL22G 内置 DDR3 硬核控制器，读写测试

  步骤：
    1. PDS IP Catalog 例化 DDR3 控制器 IP
    2. 配置参数：256MB, 16bit, 800Mbps
    3. 写测试模块：伪随机数写入 → 读回 → 比对
    4. 约束 DDR3 引脚（严格参照 Alinx 原理图）

  验证：读写一致性 100%，错误计数为 0

  注意：DDR3 引脚约束极其严格，必须完全照搬参考设计，差一个引脚都不行。

#### L5 · HDMI 视频输出

  目标：生成测试图案，通过 HDMI 输出

  步骤：
    1. 写 VGA/HDMI timing 生成模块
    2. 生成彩条或渐变测试图案
    3. 约束 HDMI TMDS 引脚
    4. 下载运行

  验证：HDMI 显示器出现测试图案

#### L6 · JTAG 逻辑分析仪（ILA）

  目标：抓 FPGA 内部信号波形，调试时序问题

  步骤：
    1. PDS 中设置要观测的信号为 debug 标记
    2. 综合时自动插入 ILA 逻辑
    3. JTAG 下载后，用 PDS 调试面板读取波形

  验证：能看到信号时序波形，与预期一致

#### L7 · RISC-V 软核（可选进阶）

  目标：在 PGL22G FPGA 里跑 RISC-V 软核 CPU

  参考项目：
    https://github.com/JerryYin777/FPGA_Competition-RISC-V_Processor_in_PGL22G

  步骤：
    1. 下载参考工程
    2. PDS 打开综合
    3. JTAG 下载
    4. 通过 UART 串口与 RISC-V 交互

  验证：串口出现 RISC-V shell 提示符

### 3.3 失败处理预案

#### PDS 安装失败

  常见原因：
    路径有中文/空格 → 改纯英文路径
    杀毒软件拦截 → 安装前关闭杀软
    License 不对 → 联系 Alinx 客服重新申请

#### JTAG 下载失败

  排查：
    1. 确认 FT2232 驱动正常（设备管理器无感叹号）
    2. PDS 里选对 JTAG 下载器型号
    3. 检查 USB 线是否支持数据传输（不是纯充电线）

#### DDR3 读写失败

  原因：
    引脚约束错 → 严格对照 Alinx 原理图
    时序参数不对 → 用 PDS 提供的 DDR3 IP 默认参数
    信号完整性问题 → 检查电源纹波

  降级：先跳过 DDR3，做 L1-L3 纯逻辑实验

#### 串口无输出

  排查：
    1. 确认波特率 115200
    2. 确认 COM 口（设备管理器 → COM6）
    3. 确认 UART TX/RX 引脚约束正确（TX/RX 不能接反）
    4. 用示波器或逻辑分析仪看 TX 引脚有没有波形

---

## 4. 通用知识（两条路线都要学）

### 4.1 Verilog HDL 基础

  模块结构（module / endmodule）
  组合逻辑（assign / always @*）
  时序逻辑（always @(posedge clk)）
  状态机（三段式写法）
  参数化设计（parameter）

### 4.2 FPGA 设计流程

  代码编写 → 综合（Synthesis）→ 布局布线（Place & Route）→ 比特流生成 → JTAG 下载 → 调试

### 4.3 UART 串口原理

  波特率、起始位、数据位、停止位、校验位
  115200 8N1 = 115200 bps, 8 数据位, 无校验, 1 停止位
  TX = 发送, RX = 接收（交叉连接）

### 4.4 JTAG 调试原理

  JTAG 四线：TCK, TMS, TDI, TDO
  功能：边界扫描、在系统编程（ISP）、逻辑分析
  FT2232 双通道：通道 A = JTAG, 通道 B = UART

---

## 5. 工具链汇总

| 工具 | 适用路线 | 大小 | CLI 支持 | 用途 |
|---|---|---|---|---|
| Pango Design Suite | B (PGL22) | ~6 GB | pango_run 批处理 | PGL22G 全流程 |
| Vivado WebPACK | A (Zynq) | ~40-100 GB | vivado -mode batch | Zynq FPGA 设计 |
| Vitis | A (Zynq) | 随 Vivado | xsct script.tcl | Zynq C/C++ 开发 |
| xsdb | A (Zynq) | 随 Vivado | xsdb script.tcl | JTAG 调试/烧录 |
| PuTTY | 通用 | ~1 MB | putty -serial ... | 串口终端 |
| GTKWave | 通用 | ~50 MB | CLI 可驱动 | 看 VCD 波形 |

---

## 6. 资源链接汇总

### 6.1 Alinx PGL22（路线 B）

  PGL22G 芯片官方页：
    https://www.pangomicro.com/product/logos_family/1797.html
  PDS 工具下载：
    https://www.pangomicro.com/resources/software/pds/
  Alinx P22 用户手册：
    https://www.alinx.com/public/upload/file/P22_UG.pdf
  Alinx PGL22G 产品页：
    https://www.alinx.com/detail/325
  PDS 开发流程教程（CSDN）：
    https://blog.csdn.net/hejinjing_tom_com/article/details/142875663
  PDS 安装指南（知乎）：
    https://zhuanlan.zhihu.com/p/601403824
  FPGA 程序固化教程（CNBlogs）：
    https://www.cnblogs.com/Tronlong818/p/17167091.html
  RISC-V 软核参考项目：
    https://github.com/JerryYin777/FPGA_Competition-RISC-V_Processor_in_PGL22G

### 6.2 Zybo Z7-20（路线 A）

  Zybo Z7 Reference Manual：
    https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual
  Digilent vivado-boards（板级预设）：
    https://github.com/digilent/vivado-boards
  Zybo Z7 入门页：
    https://digilent.com/reference/programmable-logic/zybo-z7/start

### 6.3 MYIR Z-Turn 7020（路线 A）

  MYIR Z-Turn Hello World 教程：
    https://www.linkedin.com/pulse/hello-world-tutorial-myir-z-turn-board-zynq-7020-soc-juan-abelaira
  FPGA Developer 入门指南：
    https://www.fpgadeveloper.com/2017/10/getting-started-with-the-myir-z-turn.html
  MYIR JTAG FAQ：
    https://www.myirtech.com/faq_list.asp?id=545
  MYIR Z-Turn 调试视频 Part 1：
    https://www.youtube.com/watch?v=-YTHzU0stfw
  MYIR Z-Turn 调试视频 Part 2：
    https://www.youtube.com/watch?v=TsxdXC5rLmY

### 6.4 Xilinx 官方文档

  Vitis 调试器教程：
    https://xilinx.github.io/Embedded-Design-Tutorials/docs/2020.2/build/html/docs/Introduction/ZynqMPSoC-EDT/5-debugging-with-vitis-debugger.html
  JTAG UART (UG1400)：
    https://docs.amd.com/r/2020.2-English/ug1400-vitis-embedded/Using-JTAG-UART
  串口控制台设置：
    https://xilinx-wiki.atlassian.net/wiki/display/A/Setup+a+Serial+Console

---

## 7. 执行顺序建议

  第一步（10 分钟，不装任何工具）：
    三块板子分别上电，PuTTY 连串口，按复位，记录每块板的串口输出情况。

  第二步（根据结果决定）：
    有 Demo 输出的板子 → 先玩 Demo，熟悉串口交互
    空片板子 → 装对应工具链，烧录 Hello World

  第三步（按级别推进）：
    PGL22 工具小（6GB），建议先从路线 B 的 L1 开始
    Zynq 工具大（100GB），安装时间长，可以后台装着，先做路线 B

  长期路线：
    路线 B（PGL22）走 L1-L6 建立 FPGA 基础
    路线 A（Zynq）走 L1-L7 发挥 SoC 优势（Linux/双核/AMP）
