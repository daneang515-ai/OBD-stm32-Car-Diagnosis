# 🚗  OBD-stm32-Car-Diagnosis

设计与启动使用方案可直接落地的软件架构 + 启动流程，分 STM32 固件、鸿蒙 App、联调启动三部分。


[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Platform](https://img.shields.io/badge/platform-STM32F103C8T6-orange.svg)](https://www.st.com/en/microcontrollers-microprocessors/stm32f103c8.html)
[![App](https://img.shields.io/badge/app-HarmonyOS%20ArkTS-green.svg)](https://developer.harmonyos.com/)
[![Protocol](https://img.shields.io/badge/protocol-ISO15765--4%20CAN--OBD-red.svg)]()

---

## 📑 目录

- [🎯 项目简介](#-项目简介)
- [📦 硬件清单](#-硬件清单)
- [🔌 硬件接线](#-硬件接线)
- [💻 STM32 固件](#-stm32-固件)
- [📱 鸿蒙 App](#-鸿蒙-app)
- [⚡ 快速开始](#-快速开始)
- [📋 OBD-II PID 命令表](#-obd-ii-pid-命令表)
- [📂 目录结构](#-目录结构)
- [⚠️ 重要避坑](#-重要避坑)
- [❓ 常见问题 FAQ](#-常见问题-faq)
- [📄 License](#-license)

---

## 🎯 项目简介

一、整体软件架构

```
┌─────────────────────────────────────────────────────┐
│  鸿蒙 App (ArkTS)                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ 蓝牙管理  │→│ 命令调度  │→│ 报文解析  │          │
│  │ BtManager│  │ ObdCmd   │  │ ObdParser│          │
│  └──────────┘  └──────────┘  └──────────┘          │
│        ↓                            ↓               │
│  ┌──────────┐              ┌──────────┐            │
│  │ 原始报文  │              │ 数据模型  │            │
│  │ 显示     │              │ + 曲线    │            │
│  └──────────┘              └──────────┘            │
└─────────────────────┬───────────────────────────────┘
                      │ 蓝牙 SPP (9600, 8N1)
                      │ 文本协议: "010C\r"
┌─────────────────────┴───────────────────────────────┐
│  STM32 固件 (HAL)                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ UART2 RX │→│ 命令解析  │→│ CAN1 TX  │          │
│  │ 中断      │  │ 状态机    │  │ 0x7DF    │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ UART2 TX │←│ 帧封装    │←│ CAN1 RX  │          │
│  │          │  │ 转 hex    │  │ 0x7E8    │          │
│  └──────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────┘
                      │ CAN 500kbps
┌─────────────────────┴───────────────────────────────┐
│  汽车 ECU (OBD-II)                                   │
└─────────────────────────────────────────────────────┘
```

设计原则：STM32 只做「透传 + 帧格式转换」，所有 PID 解析在 App。

---

二、通信协议设计（App ↔ STM32）

这是整个项目的核心契约，先定好协议再写代码。

2.1 下行命令（App → STM32，ASCII 文本，\r 结尾）

命令 功能 示例
AT 心跳/测试 AT\r
010C 透传 OBD 请求（十六进制） 010C\r
03 读故障码 03\r
BAUD:500 切换 CAN 波特率 BAUD:500\r
BAUD:250 切换 CAN 波特率 BAUD:250\r
RESET 软复位 RESET\r

2.2 上行响应（STM32 → App）

统一格式： +CAN:ID:DLC:D0D1D2...D7\r\n

响应 含义
+CAN:7E8:8:04410C1A3C000000\r\n ECU 应答，ID=0x7E8
+OK:BAUD=500\r\n 命令成功
+ERR:UNKNOWN_CMD\r\n 未知命令
+ERR:CAN_TIMEOUT\r\n CAN 无应答

💡 用 + 前缀便于 App 过滤；\r\n 结尾便于解析。

---

三、STM32 固件设计

3.1 模块划分

```
src/
├── main.c              # 初始化 + 主循环
├── uart_handler.c/h    # UART2 中断收发 + 环形缓冲
├── can_handler.c/h     # CAN1 收发 + 过滤器
├── cmd_parser.c/h      # 下行命令解析状态机
├── frame_codec.c/h     # CAN 帧 ↔ ASCII hex 转换
└── baud_switch.c/h     # CAN 波特率切换
```

**核心架构：**

```
汽车 ECU ⇄ OBD-II ⇄ TJA1050 ⇄ STM32 ⇄ HC-05 ⇄ 华为鸿蒙平板
            (CAN总线)        (CAN)      (UART串口)   (蓝牙SPP)
```

STM32 作为 **CAN-蓝牙透传网桥**，只负责转发报文；**报文解析全部放在鸿蒙 App**，架构清晰、易扩展。

---

## 📦 硬件清单

| 器件 | 用途 | 参考价 |
|------|------|--------|
| STM32F103C8T6 最小系统板 | 主控，CAN-蓝牙报文网桥 | ¥15 |
| ST-Link V2 | 程序烧录调试（**车上使用必须拔掉**） | ¥20 |
| TJA1050 CAN 收发器模块 | 汽车 CAN 电平转换（带 120Ω 终端电阻） | ¥10 |
| HC-05 蓝牙模块 | STM32 ↔ 鸿蒙平板 SPP 串口蓝牙 | ¥15 |
| OBD2 16Pin 公头线束 | 车辆 OBD 诊断插座 | ¥12 |
| 12V→5V 电源模块 | 汽车 12V 取电给整套系统供电（MP1584/LM2596） | ¥8 |
| 杜邦线、面包板 | 硬件调试接线 | ¥10 |

> 💰 **总计约 ¥90**，比市面 OBD 工具便宜太多 😎

---

## 🔌 硬件接线

### STM32 ↔ HC-05（蓝牙串口）

| STM32F103 | HC-05 |
|-----------|-------|
| PA2 (USART2_TX) | RX |
| PA3 (USART2_RX) | TX |
| 5V | VCC |
| GND | GND |

> ⚠️ HC-05 参数：**波特率 9600，8N1（数据位 8，停止位 1，无校验）**，STM32 串口需保持一致。

### STM32 ↔ TJA1050（CAN 总线）

| STM32F103 | TJA1050 |
|-----------|---------|
| PA11 (CAN1_RX) | RX |
| PA12 (CAN1_TX) | TX |
| 5V | VCC |
| GND | GND |

### TJA1050 ↔ OBD2 16Pin

| TJA1050 | OBD 引脚 |
|---------|----------|
| CAN_H | Pin 6 |
| CAN_L | Pin 14 |
| GND | Pin 4 / 5（车身地） |

- **OBD Pin 16**：汽车 12V 输入 → 12V→5V 电源模块 → 给整套系统供电
- ⚠️ **务必共地**：所有 GND 必须连在一起，否则 CAN 乱码无通信

### ST-Link 烧录接线（仅开发阶段）

| ST-Link | STM32F103 |
|---------|-----------|
| SWDIO | PA13 |
| SWCLK | PA14 |
| GND | GND |

> ❗ **禁止接 3.3V**；下载完成上车使用**务必拔掉 ST-Link**，否则干扰 CAN 总线。

### 接线总览图

```
汽车 OBD-II 接口
┌──────────────────────────────────────┐
│ Pin 4/5 (GND) ───────────────┐      │
│ Pin 6  (CAN_H) ────┐         │      │
│ Pin 14 (CAN_L) ──┐ │         │      │
│ Pin 16 (12V) ──┐  │ │         │      │
└────────────────│──│─│─────────│──────┘
             12V→5V │ │         │
             模块   │ │         │
        ┌───────────┘ │         │
        │             │         │
   TJA1050       TJA1050     STM32F103C8T6
   ┌────────┐        │         ┌──────────────┐
   │ VCC 5V │◄───────┘         │ PA11 CAN_RX  │◄── CAN_H
   │ GND    │◄─────────────────│ PA12 CAN_TX  │──► CAN_L
   │ RX     │─────────────────►│ PA2  USART2_TX
   │ TX     │◄─────────────────│ PA3  USART2_RX
   └────────┘                  │ PA13 SWDIO   │◄── ST-Link
                               │ PA14 SWCLK   │◄── ST-Link
                               │ 5V  ─────────┼──► HC-05 VCC
                               │ GND ─────────┼──► HC-05 GND
                               │ PA2(TX) ─────┼──► HC-05 RX
                               │ PA3(RX) ◄────┼──── HC-05 TX
                               └──────────────┘
                                      │
                                 HC-05 蓝牙 (SPP)
                                      │
                                华为鸿蒙平板
```


0. 加一个「最小验证」清单

在快速开始里加一个 30 秒自检：

```
□ 万用表测 5V 母线 = 4.9~5.1V
□ 串口助手连 HC-05，发 AT 返回 OK
□ CAN 回环（PA11/PA12 短接）自发自收
□ 接车后 0100 返回非空
```

---

📝 小笔误

· 「计算负载值」应为「计算负载值」→ 实际是「发动机负荷」，建议改 01 04 为「发动机负荷」
· 01 0A 燃油压力公式 A×3 单位 kPa 正确，但部分车是 A×3 还是 A×0.079 取决于 PID，建议标注「依车型」
· 故障码示例 0x01 0x33 → P0133：首字节 0x01 高 2 位 = 00 → P，bit5-4 = 00 → 0，低 4 位 = 1，第二字节 0x33 → 33，合起来 P0133 ✅ 正确

---

## 💻 STM32 固件

### 工程配置（STM32CubeIDE）

1. 新建工程，选择芯片 **STM32F103C8T6**
2. **SYS → Debug**：Serial Wire（防止芯片锁死）
3. **RCC → HSE**：外部 8M 晶振
4. **USART2**：异步串口，波特率 **9600**，开启 NVIC 中断
5. **CAN1**：Normal 模式，波特率 **500kbit/s**
6. 时钟树配置：**HSE 8M → 系统时钟 72MHz**
7. 生成 HAL 工程

> ⚠️ STM32CubeIDE **不要安装到 C 盘**，会出现图形卡死蓝屏；建议安装路径 `D:\ST\STM32CubeIDE`。

### 固件逻辑

STM32 作为 **CAN-蓝牙透传网桥**：

1. **蓝牙串口**接收鸿蒙平板下发的 OBD 指令
2. 转为 **CAN 报文**发送给 ECU（**请求 ID 0x7DF**）
3. 接收 ECU 返回的 CAN 报文（**应答 ID 0x7E8**）
4. 通过蓝牙**原样转发**给鸿蒙平板
5. **报文解析全部放在鸿蒙 App**，STM32 只负责转发

### CAN 波特率

| 车辆类型 | 波特率 | 切换命令 |
|----------|--------|----------|
| 主流车型（2008+） | 500Kbps（默认） | `BAUD:500` |
| 部分老车 | 250Kbps | `BAUD:250` |

### 构建

```bash
cd src/
# 方式1：用 STM32CubeIDE 打开工程，Build Project
# 方式2：用 CMake
mkdir build && cd build
cmake ..
make
```

烧录：

```bash
# 用 tools/stlink_flash.bat 一键烧录（Windows）
# 或修改路径后运行
```

---

## 📱 鸿蒙 App

- **开发设备**：华为平板 11.5，鸿蒙系统（HarmonyOS 4.0+）
- **通信方式**：蓝牙 SPP 串口
- **开发工具**：DevEco Studio（≥4.0）

### 功能

- 🔍 扫描、连接 HC-05 蓝牙模块
- 📤 下发 OBD PID 命令
- 📥 显示原始 CAN 十六进制报文
- 📊 解析**故障码、转速、水温、车速**等数据
- 📈 实时曲线（转速趋势、水温趋势）
- ⚠️ 故障记录查看
- 🔧 波特率切换（250K / 500K）

### 运行

```bash
cd harmony_app/
# 用 DevEco Studio 打开，选择华为平板真机 → Run
```

> ⚠️ **鸿蒙模拟器无蓝牙，必须使用真机平板调试！**

### 蓝牙配对

1. 平板进入 **系统设置 → 蓝牙**
2. 搜索设备，找到 `HC-05`
3. 点击配对，密码 `1234`
4. 配对成功后打开 OBD App

---

## ⚡ 快速开始

详细步骤见 [docs/QUICKSTART.md](docs/QUICKSTART.md)，快速流程：

```bash
# 1. 克隆仓库
git clone https://github.com/daneang515-ai/OBD-ESP32-Car-Diagnosis.git
cd OBD-ESP32-Car-Diagnosis

# 2. 烧录 STM32 固件
#    用 STM32CubeIDE 打开 src/，Build → Debug 烧录

# 3. 安装鸿蒙 App
#    用 DevEco Studio 打开 harmony_app/，真机运行

# 4. 接线（参考上方硬件接线）

# 5. 蓝牙配对 HC-05（密码 1234）

# 6. 接车测试（车辆点火 ON，不启动发动机）
```

### 验证步骤

| 步骤 | 命令 | 预期结果 |
|------|------|----------|
| 1 | `0100` | 返回支持 PID 列表 |
| 2 | `010C` | 返回发动机转速 |
| 3 | `010D` | 返回车速 |
| 4 | `0105` | 返回水温 |
| 5 | `03` | 返回故障码 |

---

## 📋 OBD-II PID 命令表（ISO15765-4）

**CAN 请求 ID：0x7DF ； ECU 应答 ID：0x7E8**

### Mode 01 - 显示当前数据

| 命令 | 功能 | 计算公式 | 单位 |
|------|------|----------|------|
| `01 00` | 查询支持 PID 列表 | - | - |
| `01 04` | 计算负载值 | A×100/255 | % |
| `01 05` | 冷却液温度 | A−40 | ℃ |
| `01 0A` | 燃油压力 | A×3 | kPa |
| `01 0B` | 进气歧管压力 | A | kPa |
| `01 0C` | 发动机转速 | (A×256+B)/4 | rpm |
| `01 0D` | 车辆速度 | A | km/h |
| `01 0F` | 进气温度 | A−40 | ℃ |
| `01 10` | 空气流量 MAF | (A×256+B)/100 | g/s |
| `01 11` | 节气门位置 | A×100/255 | % |
| `01 42` | 系统电压 | (A×256+B)/100 | V |

### Mode 03 / 04 - 故障码

| 命令 | 功能 |
|------|------|
| `03` | 读取故障码 DTC |
| `04` | 清除故障码 & 冻结帧 |

> 📖 完整 PID 速查表见 [docs/OBD_PID_REFERENCE.md](docs/OBD_PID_REFERENCE.md)

### CAN 帧结构

```
标准 OBD 请求帧 (ID 0x7DF):
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ DLC  │ Mode │ PID  │  -   │  -   │  -   │  -   │  -   │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  1字节   1字节  1字节   填充0

ECU 响应帧 (ID 0x7E8):
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ DLC  │ 4X   │ PID  │ DataA│ DataB│  -   │  -   │  -   │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  1字节  响应模式  PID   数据字节
```

### 故障码 DTC 编码规则

```
首字节高2位 → 系统类型: 00=P, 01=C, 10=B, 11=U
首字节bit5-4 → 第一位数 (0-3)
首字节低4位 + 第二字节 → 后4位十六进制

示例: 0x01 0x33 → P0133 (氧传感器响应慢)
```

---

## 📂 目录结构

```
OBD-ESP32-Car-Diagnosis/
├── README.md                    # 项目说明（本文件）
├── .gitignore                   # 过滤编译产物
├── LICENSE                      # MIT License
├── docs/                        # 文档
│   ├── QUICKSTART.md            # 快速上手指南
│   ├── OBD_PID_REFERENCE.md     # PID 速查表
│   ├── PROTOCOL_DETAIL.md       # 协议详解
│   └── TROUBLESHOOTING.md       # 故障排查
├── schematics/                  # 硬件接线图
│   └── breadboard_wiring.png
├── src/                         # STM32 HAL 固件
│   ├── CMakeLists.txt
│   ├── main.c
│   ├── can_handler.c/h
│   ├── uart_handler.c/h
│   ├── obd_parser.c/h
│   ├── baud_switch.c/h
│   └── ...
├── harmony_app/                 # 鸿蒙 ArkTS App
│   ├── build.gradle
│   └── entry/src/main/ets/
│       ├── pages/Index.ets
│       ├── model/ObdManager.ets
│       └── components/LineChart.ets
└── tools/                       # 辅助工具
    ├── stlink_flash.bat         # 一键烧录脚本
    └── can_sniffer.py           # PC 端 CAN 调试工具
```

---

## ⚠️ 重要避坑

- 🔴 **ST-Link 仅烧录使用，上车必须拔掉**，否则干扰 CAN 总线
- 🔴 **TJA1050 必须共地**，OBD GND（Pin4/5）一定要接好，地不共会 CAN 乱码无通信
- 🔴 **HC-05 波特率严格 9600**，STM32 串口波特率保持一致
- 🟡 **部分老车 CAN 速率 250Kbps**，App 支持波特率切换（`BAUD:250`）
- 🟡 **先面包板回环测试 CAN 收发**，硬件调试完成后再接真实汽车 OBD 口
- 🟡 **鸿蒙模拟器无蓝牙**，必须用真机平板调试
- 🟡 **STM32CubeIDE 不要装 C 盘**，建议 `D:\ST\STM32CubeIDE`

---

## ❓ 常见问题 FAQ

**Q: CAN 完全无通信？**
A: 检查顺序：① TJA1050 GND 是否与 OBD Pin4/5 连通 → ② CAN_H/CAN_L 是否接反 → ③ 车辆是否支持 CAN-OBD（2008+ 基本都支持）→ ④ 波特率（默认 500K，老车试 250K）

**Q: 蓝牙连上了但没数据？**
A: ① 确认 HC-05 波特率 9600 → ② STM32 USART2 波特率也是 9600 → ③ 用串口助手直接连 HC-05 看是否有输出

**Q: 上车后 STM32 不断重启？**
A: 供电不足。检查 12V→5V 模块输出电流是否 ≥500mA。

**Q: 读取故障码返回全零？**
A: 正常！说明当前没有故障码存储。

**Q: 能用于专业汽修吗？**
A: 本项目为 **DIY 学习用途**，不建议替代专业设备。

> 📖 更多排查内容见 [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)

---

## 🤝 贡献

欢迎 Issue & PR！提交前请：

1. Fork 本仓库
2. 新建分支 `git checkout -b feature/xxx`
3. 提交 `git commit -m "feat: xxx"`
4. 推送 `git push origin feature/xxx`
5. 发起 Pull Request

---

## 📄 License
MIT License

Copyright (c) 2025 daneang515-ai

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

> ⚠️ **免责声明**：本项目仅供学习研究使用，作者不对因使用本项目造成的任何车辆损坏、人身安全等问题负责。
