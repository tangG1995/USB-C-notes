# 阶段二：USB PD 协议栈架构与消息格式

> 学习目标：掌握 USB PD 的消息结构、状态机走向、SOP 通信路径，以及 VDM/Alt Mode 的工作原理。

---

## 2.1 协议栈概览——五层结构

USB PD 不是一条消息走天下，它是分层协议：

```
┌──────────────────────────────────────────────────────┐
│  应用层（Application）                                │
│   「我要 20V/5A」，「给 100W」                        │
│                      ↓ 上层请求                       │
│  ───────────────────────────────────────────────────│
│  策略层（Policy Engine）                              │
│   决定什么时候发什么消息，什么时候切换角色              │
│                      ↓                                │
│  ───────────────────────────────────────────────────│
│  协议层（Protocol Layer）                            │
│   打包成 PD 消息 → BMC 编码 → CC 引脚发出             │
│                      ↓                                │
│  ───────────────────────────────────────────────────│
│  物理层（PHY）                                        │
│   BMC 编解码，300kHz，单线半双工                      │
│                      ↓                                │
│  ───────────────────────────────────────────────────│
│  硬件层（CC 引脚 + VBUS）                            │
│   Rp/Rd 电阻检测 → BMC 信号传输 → VBUS 实际供电       │
└──────────────────────────────────────────────────────┘
```

**为什么分层重要？**
- 应用层想「充 100W」，不需要知道 BMC 怎么编码
- 物理层出错了，协议层负责重传，应用层只管等待
- 策略层决定「这时候该不该发起 PR_Swap」，隔离了业务逻辑和通信细节

---

## 2.2 三条通信路径：SOP / SOP' / SOP''

这是 USB PD 里最容易搞混的概念之一。

USB-C 有**三组不同的 CC 通信对象**，每组走的路径不同：

```
Source ◀─── SOP ───▶ Sink
   │                    │
   │◀── SOP' ──▶│ 线缆插头（Cable Plug）
   │              │
   │◀─ SOP'' ──▶│ 扩展电缆（Extended Cable）（用 eMarker）
   └────────────┘
```

| 路径 | 通信双方 | 用途 |
|------|---------|------|
| **SOP** | Source ↔ Sink（直接相连的两端） | 所有功率协商、标准 PD 消息 |
| **SOP'** | Source/Sink ↔ 线缆插头 | 线缆认证（这条线缆是否支持对应功率） |
| **SOP''** | Source/Sink ↔ 扩展电缆远端（eMarker） | VCONN 供电、Cable 能力查询 |

### 线缆里的 eMarker 芯片

每根「USB-C 认证线缆」里都有一个 tiny 的 eMarker 芯片：
- 记录线缆的「能力清单」：最高电压、最高电流、支持哪些协议
- 当 Source 发出 **SOP' Discover Identity** 时，eMarker 响应：「我是 100W 线缆，最高支持 20V/5A」
- 如果线缆能力不够（比如插了一根 60W 线缆却要 100W），Source 会降档协商

> 💡 **这就是为什么有些充电头 + 线缆组合跑不到 100W**：问题往往在线缆，不在充电头或设备。eMarker 的 SOP' 协商失败，Source 自动降档。

### SOP 消息的方向性

SOP 消息里的 Header 里有一个 **Data Role** 字段，决定消息方向：
- `DFP → UFP`：消息从 Source/Host 发往 Device/Sink
- `UFP → DFP`：消息从 Sink/Device 发往 Source/Host

Power Role（Source/Sink）和 Data Role（DFP/UFP）独立存在。

---

## 2.3 PD 消息格式（核心内容）

### 2.3.1 消息结构总览

一条 USB PD 消息 = **1 个 Header + N 个 Data Object**

```
┌──────────┬──────────┬──────────┬──────────┐
│ Header   │ Obj 0    │ Obj 1    │ ... Obj N │
│ (16 bits)│ (32 bits)│ (32 bits)│           │
└──────────┴──────────┴──────────┴──────────┘
 ↑ 必有     ↑ Control 消息：0 个对象
            ↑ Data 消息：1～7 个对象
```

### 2.3.2 Header（16 bits）—— 消息的身份证

```
  15  14  13  12  11  10  09  08  07  06  05  04  03  02  01  00
 ┌────┬────┬────┬────┬──────────┬────────┬────┬────┬────────┐
 │MsgType(5) │Fmt │  Spec   │PortData│MsgID │Num │Rev │ Ext  │
 │  消息类型  │Fmt │  版本    │角色    │MsgID │Objs│Pro │ Ext  │
 └─────────────────────────────────────────────────────────┘
```

| 字段 | 位数 | 含义 |
|------|------|------|
| **Message Type** | 5 bits | 消息类型编号（0=GoodCRC, 1=GoToMin, 2=Accept, 3=Reject...） |
| **Format (Fmt)** | 1 bit | 0=Control 消息（无 Data Object）；1=Data 消息（有 Data Object） |
| **Specification Revision** | 2 bits | PD 规范版本（00=1.0, 01=2.0, 10=3.1） |
| **Port Data Role** | 1 bit | 0=UFP, 1=DFP |
| **Message ID** | 3 bits | 消息序列号（0-7），用于重复检测和重传匹配 |
| **Num Data Objects** | 3 bits | Data Object 的数量（0-7） |
| **Extended** | 1 bit | 是否有扩展字段（PD 3.1 新增） |

> 🔑 **MsgID 是防重放攻击的关键**：发送方每条消息递增 MsgID（0→7→0），接收方用 MsgID 过滤掉重复消息。这个字段如果不同步，握手会卡死。

### 2.3.3 Data Object（32 bits × N）—— 消息的正文

USB PD 有两大类 Data Object：**PDO（Power Data Object）** 和 **RDO（Request Data Object）**

---

## 2.4 PDO——Source 的能力清单

**Source_Capabilities 消息**里，Source 告诉 Sink：「我能提供以下这些供电选项。」

每种 PDO 都是一个 32-bit 的 Data Object，但格式分三种类型：

### 类型 1：Fixed Supply PDO（最常见）

```
  31  30  29  28  27  26  25  24 ...  11  10 ...  00
 ┌────┬────┬────┬────┬──────────────────────────────┐
 │1(0)│ 00 │Dual│  Peak  │    Voltage (mV)    │I_max │
 │类型 │保留 │Role│ Current│   电压（固定）     │I_max │
 └───────────────────────────────────────────────────┘
  ────────  BITS [31:30] = 固定 00b
```

| 字段 | 位 | 含义 |
|------|----|------|
| **Max Current** | 10 bits | 最大电流（10mA 步进，例如 3A = 300） |
| **Voltage** | 10 bits | 固定电压值（50mV 步进） |
| **Peak Current** | 2 bits | 峰值电流（00=Max, 01=1.25x, 10=1.50x, 11=1.75x） |
| **Data Role Swap (DRS)** | 1 bit | 是否支持 DR_Swap |
| **USB Comm Capable** | 1 bit | 是否支持 USB 数据通信 |
| **Externally Powered** | 1 bit | 是否需要外接电源（=0 表示有 USB-PD 充电宝功能） |
| **Dual Role Power** | 1 bit | 是否支持 Source/Sink 切换 |
| **Supply Type** | 2 bits | 00b = Fixed Supply（固定电压） |

**常用 PDO 速查（PD 3.1 SPR）：**

| 电压 | 电流 | 功率 | PDO 示例（十六进制） |
|------|------|------|----------------------|
| 5V | 3A | 15W | 0x0001A0F4 |
| 9V | 3A | 27W | 0x0002A0F4 |
| 15V | 3A | 45W | 0x0003A0F4 |
| 20V | 5A | **100W** | 0x0003C3F4 |

> 💡 **快速读 PDO 的技巧**：拿到一个 PDO 十六进制值，判断类型看 bits[31:30]；如果是 00（Fixed），bit[29:28] 是 Dual Role Power 位，bits[20:10] 是电压值（乘 50mV），bits[9:0] 是电流值（乘 10mA）。

---

### 类型 2：Variable Supply PDO（老式设备用）

- 无固定电压，在某个范围内连续可调
- 电压由 bits[29:20] 的 Max Voltage 和 bits[19:10] 的 Min Voltage 定义
- 现在基本不用，仅做兼容

---

### 类型 3：Battery Supply PDO

- 表示 Sink 是一个电池，从电池取电
- 定义的是电池能吸收的功率范围，不是 Source 能给的
- 很少见，主要用于电动工具等场景

---

### 类型 4：Programmable Power Supply PDO（PPS）—— PD 3.0 新增

这是目前快充协议里最常用的格式，各家私有协议（TURBOCHARGE、SuperCharge 等）大多基于 PPS。

```
  31  30  29  28 ... 17  16 ...  07  06  05  04  03  02  01  00
 ┌────┬────┬────┬──────────────────────┬──────────────────────────┐
 │1(1)│ PPR│PPS │  Max Voltage (mV)   │   Min Voltage (mV)     │
 │00b │    │    │  最大电压             │   最小电压             │
 │    │    │    │                      │                    │ I_max│
 └─────────────────────────────────────────────────────────────┘
  ─── bits[31:30] = 01b → PPS 类型
```

**PPS 的关键特性：**
- 电压可以**连续调节**（步进 20mV），不再限于固定档位
- 例如：手机告诉充电头「我要 10V/2A」，充电头输出 10V，然后微调电压让电流刚好 2A
- 电流上限由 bits[3:0] 定义，最大 5A

> ⚠️ **PPS 是发热控制的博弈**：充电头在 PPS 模式下需要实时调整输出电压/电流，闭环控制做不好就会输出震荡。这也是为什么某些 PPS 充电器充特定手机时发热严重。

---

### 类型 5：EPR Mode PDO（PD 3.1 EPR 新增）

- 专门用于 **48V EPR（扩展功率范围）** 档位
- 电压范围：28V / 36V / 48V
- 电流固定 5A，最高 240W
- Source 发出 EPR PDO 前，必须先进入 EPR Mode

---

## 2.5 RDO——Sink 的需求清单

当 Sink 收到 Source_Capabilities（包含 N 个 PDO）后，选择其中一个 PDO，发出 **Request 消息**：

```
  31  30  29  28  27  26  25  24 ...  11  10 ...  00
 ┌────┬──────────────────────────┬──────────────────────────┐
 │ Obj│    Operating Current     │    Max Current           │
 │ Pos│    (@ 期望电压, 10mA/bit) │    (不可超过 PDO 限制)   │
 │    │    工作电流               │    最大电流               │
 └──────────────────────────────────────────────────────────┘
  ↑ Object Position = 选了第几个 PDO（1-N）
```

| 字段 | 含义 |
|------|------|
| **Object Position** | 选中的 PDO 编号（1, 2, 3...） |
| **Operating Current** | Sink 期望的工作电流（可小于 PDO 上限） |
| **Max Current** | Sink 能接受的最大电流 |
| **Give Back Flag (GB)** | 置 1 = Sink 允许 Source 降低到 Give Back Current 以下（节电模式） |
| **No USB Suspend** | 置 1 = 请求保持 USB 数据通信（不让 Source 降低功率进入 suspend） |
| **USB Comm Capable** | Sink 是否支持 USB 数据（置 1 表示 Sink 有 USB 控制器） |
| **Capability Mismatch** | 置 1 = Sink 认为 Source 的能力不满足需求（Source 可以 Accept 但实际降档供电） |
| **Object Position** | 同上 |
| **Reserved** | 保留 |

**一个 RDO 例子：**

```
选择 PDO #3（20V/5A），期望 20V/4.5A，最大接受 5A

  Obj Pos = 3
  Operating Current = 4.5A = 450 (bits[19:10])
  Max Current = 5A = 500 (bits[9:0])
  Give Back = 0, No USB Suspend = 1, USB Comm = 1, Mismatch = 0

  RDO = 0x1C 0A E1 48
        (二进制逐字段对应)
```

---

## 2.6 Source 对 RDO 的响应流程

Sink 发完 Request 后，不是想充就能充的——Source 必须逐条确认：

```
Sink → Source : Request（RDO）              「我要 20V/4.5A」
Source → Sink : Accept                      「可以」
         [Source 开始调整 VBUS 到 20V]      
Source → Sink : PS_RDY                      「20V 准备好了」
         ✅ Power Contract 建立完成！
```

**Source 可能拒绝的情况：**

| Source 回复 | 含义 |
|-------------|------|
| **Accept** | 接受，开始调压调流 |
| **Reject** | 完全拒绝（功率不够，或不支持这个 PDO） |
| **Not Supported** | PD 版本不支持该请求 |
| **Wait** | 当前忙，稍后重试（不能连续发超过 2 次） |

> 💡 **Wait 消息是互操作性测试的常见 fail 点**：有些 Sink 收到 Wait 就无限重试，实际上应该重新发起 Source_Capabilities 协商。

---

## 2.7 PD 状态机——从连接到握手

这是 USB PD 的精髓，也是实际调试时必须烂熟于心的东西。

### Sink 侧状态机（USB Type-C™ Port Manager）

```
  ①                            ②                    ③
 ┌──────────┐    CC有Rd下拉    ┌──────────┐  PD通信开始 ┌──────────┐
 │Unattached│ ──────────────▶ │AttachWait│ ──────────▶│ Attached │
 │  .SNK    │   VBUS 检测到    │  .SNK    │            │   .SNK   │
 └──────────┘                 └──────────┘            └──────────┘
                                                          │
                                                          │ 收到 Hard Reset
                                                          ▼
                                                    ┌──────────┐
                                                    │HardReset │
                                                    │  .Receiv │
                                                    └──────────┘
                                                          │
                              VBUS 消失 → 回到 ①
```

### Source 侧状态机

```
  ①                            ②                    ③                    ④
 ┌──────────┐    CC被Rd拉低    ┌──────────┐  VBUS 开启  ┌──────────┐  PD 协商完成 ┌──────────┐
 │Unattached│ ──────────────▶ │AttachWait│ ──────────▶│ Attachd  │ ──────────▶│Transiton│
 │  .SRC    │                 │   .SRC   │            │   .SRC   │            │ .Supply  │
 └──────────┘                 └──────────┘            └──────────┘            └──────────┘
                                                                                      │
                                                                                      ▼
                                                                               ┌──────────┐
                                                                               │  USB PD  │
                                                                               │Contract  │
                                                                               └──────────┘
```

### PD 通信状态流转（详细版）

```
  Source                      Sink
    │                          │
    │── Source_Capabilities ──▶│  ① Source 广播能力清单（N 个 PDO）
    │◀── GoodCRC ──────────────│  ② 必须回复 GoodCRC，否则重传
    │                          │
    │◀── Request (RDO) ────────│  ③ Sink 选一个 PDO，提出需求
    │── GoodCRC ──────────────▶│  ④ 必须回复 GoodCRC
    │                          │
    │── [Accept / Reject] ────▶│  ⑤ Source 决定是否接受
    │◀── GoodCRC ──────────────│
    │                          │
    │  [开始调整 VBUS 电压]     │
    │                          │
    │── PS_RDY ───────────────▶│  ⑥ Source 通知「电压稳定了」
    │◀── GoodCRC ──────────────│
    │                          │
    │   ✅ Power Contract 建立  │  ← 双方进入稳定供电状态
    │                          │
    │  [可随时发起新协商]       │
    │── Source_Capabilities ──▶│  ⑦ Source 发出新能力清单
    │    (进入新协商流程)       │
```

### Power Contract（功率契约）

> 这是 USB PD 最核心的概念：**一旦 Source_Capabilities + Request + Accept + PS_RDY 完成，双方就进入了一个「功率契约」状态。**
>
> 在契约有效期内，Source 必须按约定电压/电流供电；Sink 必须按约定功率取电。
> 任何一方想改变契约，必须重新发起 Source_Capabilities 协商。
>
> 这就是为什么叫 **Power Delivery** —— 不只是充电，是一份写在协议里的「供电合同」。

---

## 2.8 Hard Reset——最暴力的复位手段

当正常协商流程走不下去了，Source 可以发出 **Hard Reset**：

```
Source：关断 VBUS  → 等 tSrcRecover → 重新开启 VBUS → 重新发起协商
Sink  ：检测 VBUS 消失 → 进入 Unattached.SNK → 等待重新 Attach
```

**Hard Reset 触发条件（常见）：**
- 重传 5 次同一消息后仍无 GoodCRC 回复
- 协议层检测到无法恢复的错误
- Source 发现输出过流/过热，需要紧急降载

**Hard Reset 的破坏力：**
- VBUS 直接中断，所有正在充电的设备瞬间断电
- USB 数据连接也会中断（对于 DFP 来说相当于重新枚举设备）
- 这就是为什么很多设备在充电时拔插突然重启——Hard Reset 触发了

---

## 2.9 VDM 消息与 Alt Mode（扩展模式）

USB-C 的强大之处：不只能传 USB，还可以切换成「显示模式」「雷电模式」等，这就是 **Alt Mode（替代模式）**。

### VDM 是什么

**Vendor Defined Message（厂商定义消息）** 是 PD 协议里的「插件」，让厂商可以传输任何自定义内容。

VDM 消息格式：
```
Header: Message Type = VDM (0x0F)
VDM Header:
  ├─ SVID（16 bits）：Vendor ID（USB-IF 分配）
  │                   0xFF00 = USB-IF 通用 VDM
  │                   其他 = 厂商私有 VDM
  ├─ VDM Type（1 bit）：0 = Unstructured VDM（无结构）
  │                     1 = Structured VDM（有标准结构）
  └─ Command（5 bits）：Discover / Enter / Exit / Attention 等

Data Objects（N 个）：
  └─ 内容由厂商定义
```

### Structured VDM 状态机（标准 Alt Mode 流程）

```
Discover Identity（发现设备）：
  Sink → Source : Discover Identity（请求：我是谁）
  Source → Sink : ACK + 设备信息（Product Type, TID 认证码, 是否支持 Alt Mode）

Discover Modes（发现支持的模式）：
  Sink → Source : Discover Modes
  Source → Sink : ACK + [DisplayPort, HDMI, Thunderbolt...] ← 支持的 Mode 列表

Enter Mode（进入模式）：
  Sink → Source : Enter Mode（请求进入某个 Mode）
  Source → Sink : ACK（确认进入）

  [此时 USB-C 的 TX/RX/SBU 复用为 DP/HDMI/雷电信号]
  [VBUS 保持 PD 协商的电压]

Exit Mode（退出模式）：
  Sink → Source : Exit Mode
  Source → Sink : ACK
  [恢复为标准 USB-C 模式]
```

### 最常见的 Alt Mode：DisplayPort（DP）

**DisplayPort Alt Mode（DP over USB-C）** 是最广泛部署的替代模式：

```
USB-C 连接器在 DP Alt Mode 下的引脚分配：

TX1+/TX1- / RX1+/RX1-  → 复用为 DP 通道
  ├─ 2-lane 模式：使用 TX1+TX1- 和 RX1+RX1- 中的 2 条
  │              最高 4K@60Hz（HBR2）
  └─ 4-lane 模式：使用全部 4 条
                  最高 8K@60Hz 或 4K@120Hz（HBR3）

SBU1/SBU2  → 复用为 DP AUX 通道（Sideband）

D+/D-      → 保持 USB 2.0 数据（不一定激活）

VBUS       → 保持 PD 协商的功率
```

**DP Alt Mode + PD 的组合场景：**

| 场景 | VBUS | TX/RX | 数据 |
|------|------|-------|------|
| 纯充电 | 按 PD 协商 | 空闲 | 无 |
| 充电 + USB 数据 | 按 PD 协商 | USB SuperSpeed | USB 2.0 |
| 充电 + DP 视频 | 按 PD 协商 | 复用为 DP 通道 | 无 USB 数据 |
| 充电 + DP 视频 + USB 数据 | 按 PD 协商 | DP 4-lane | USB 2.0（走 D+/D-）|

> 💡 **这就是为什么有些 USB-C 扩展坞接笔记本后，视频输出正常但 USB 设备不识别**——扩展坞进入了 DP Alt Mode，但 D+/D- 的 USB 数据路由没有正确建立，需要检查 DR_Swap 是否成功。

### Thunderbolt 3/4（雷电模式）

Thunderbolt 是 Intel 的私有 Alt Mode，基于 USB-C 物理层，但有完全独立的认证体系：

- 使用 SOP' 和 SOP'' 深度协商（比标准 USB PD 更复杂的认证流程）
- 需要 Intel 的 TB 认证芯片（笔记本端和扩展坞端都要有）
- 支持 PCIe 数据隧道（不只是 USB/DP，还能传 PCIe）
- 主动线缆（Active Cable）vs 被动线缆：主动线缆内部有信号放大芯片

---

## 2.10 实用调试知识——PD 抓包分析

### 如何抓取 PD 消息

**工具选项：**
1. **USB-IF PD 认证分析仪**（专业工具，$2000+，贵但准）
2. **树莓派 + TCPC 芯片**（开源方案，需要硬件改装）
3. **示波器 + CC 引脚探头**（能看到 BMC 波形，但解析困难）
4. **赛普拉斯/瑞昱 USB-C 协议分析仪**（介于 1 和 2 之间）

### 常见 PD 握手失败原因速查

| 现象 | 最可能原因 |
|------|-----------|
| Source 不发 Source_Capabilities | CC 连接不稳定（Rp/Rd 接触不良） |
| Sink 不发 Request | Sink 未使能 PD（某些手机默认关闭） |
| 收到 Reject 后反复重试 | PDO 选择错误或功率不匹配 |
| Hard Reset 频繁触发 | VBUS 短路保护触发 / 线缆过载 |
| PPS 模式下电压震荡 | 充电头闭环响应差（环路带宽不够） |
| DP Alt Mode 进入失败 | SVID 不匹配 / 线缆不支持对应 Mode |

### 解析 Source_Capabilities 的 Python 小脚本（供参考）

```python
def parse_pdo(pdo_hex):
    pdo = int(pdo_hex, 16)
    supply_type = (pdo >> 30) & 0b11

    if supply_type == 0b00:  # Fixed Supply
        voltage_mv = ((pdo >> 10) & 0x3FF) * 50  # mV
        max_current_ma = (pdo & 0x3FF) * 10      # mA
        dual_role = bool((pdo >> 29) & 1)
        return f"Fixed {voltage_mv/1000:.1f}V / {max_current_ma}mA" \
               f" {'DRP' if dual_role else 'Source-only'}"
    elif supply_type == 0b11:  # PPS
        max_v_mv = ((pdo >> 17) & 0xFF) * 100   # mV
        min_v_mv = ((pdo >> 8) & 0xFF) * 100    # mV
        max_i_ma = (pdo & 0x7F) * 50            # mA
        return f"PPS {min_v_mv/1000:.1f}-{max_v_mv/1000:.1f}V " \
               f"max {max_i_ma}mA"
    else:
        return f"Unknown type {supply_type}"

# 示例
print(parse_pdo("0x0003C3F4"))  # → "Fixed 20.0V / 5000mA DRP"
print(parse_pdo("0x80029264"))  # → "PPS 3.3-21.0V max 3000mA"
```

---

## 2.11 本阶段核心认知总结

```
┌─────────────────────────────────────────────────────────────┐
│  PD 消息 = Header（16 bits）+ N × Data Object（32 bits）   │
│                                                             │
│  PDO = Source 的「能力菜单」（Fixed/PPS/EPR 三种）          │
│  RDO = Sink 的「点菜单」（选哪个 + 要多少电流）             │
│                                                             │
│  SOP = Source ↔ Sink（直接握手）                           │
│  SOP' = Source/Sink ↔ 线缆（查线缆能力）                    │
│  SOP'' = ↔ 扩展电缆远端（eMarker）                          │
│                                                             │
│  Power Contract = 供电合同，建立后双方都要遵守              │
│  Hard Reset = 掀桌子重来（VBUS 中断，杀伤力最大）           │
│                                                             │
│  VDM = Alt Mode 的入口钥匙，DP/雷电/TBT 都靠它协商         │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ 阶段二检验题

**检验 1：** Source 发出了 4 个 PDO（5V/3A、9V/3A、15V/3A、20V/5A），Sink 发出的 RDO 里 Object Position = 3，实际请求是什么？

**检验 2：** PPS PDO 和 Fixed PDO 的本质区别是什么？为什么 PPS 更适合手机快充？

**检验 3：** SOP 和 SOP' 分别用于哪些场景？线缆里的 eMarker 用哪个 SOP 通信？

**检验 4：** Hard Reset 触发了会发生什么？它和重新发起 Source_Capabilities 协商有什么区别？

---

## 下节预告：阶段三 —— PD 状态机深度与 PR_Swap/DR_Swap

下一节将深入：
- PR_Swap 完整时序图（6步，每步 t 时间窗口）
- DR_Swap 的前提条件和失败原因
- USB PD 互操作性测试（PD CTS/TD 4.8.x）重难点解析
- EPR Mode 进入流程与 48V 实测注意事项

准备好了继续阶段三？

---

*整理时间：2026-04-16*
