---
title: UDS诊断协议详解
date: 2026-08-02 17:05:00
tags:
  - UDS
  - 诊断协议
  - ISO14229
  - 汽车电子
  - CAN
  - DTC
  - 底层原理
---

UDS（Unified Diagnostic Services，统一诊断服务）是汽车电子领域最核心的诊断协议，定义在 **ISO 14229** 标准中。它规定了诊断仪（Tester）与 ECU 之间通信的"语言"，广泛用于产线测试、售后诊断、ECU 刷写（Bootloader）和整车下线。本文系统讲解 UDS 的协议栈、服务、时序与工程实现。

# 一、UDS 概述

## 1.1 什么是 UDS

UDS 定义了一套**统一的诊断服务**，无论哪个 ECU、哪种控制器、哪个厂商，诊断仪都通过同样的服务号与其通信：

- 读故障码：`0x19`
- 读数据：`0x22`
- 写数据：`0x2E`
- 刷写固件：`0x34/0x36/0x37`
- ECU 复位：`0x11`
- 会话切换：`0x10`

## 1.2 标准体系

| 标准 | 内容 |
|---|---|
| **ISO 14229-1** | 诊断服务定义（UDS 核心） |
| ISO 14229-2 | 会话层服务 |
| ISO 14229-3 | 基于 CAN 的应用层 |
| ISO 15765-2 | CAN 传输层（多帧传输） |
| ISO 15765-3 | 基于 CAN 的 UDS 实现 |
| ISO 27145 / 14229-5 | 基于以太网（DoIP / UDSonIP） |

## 1.3 诊断通信模型

```
┌─────────────┐  请求(诊断请求)   ┌─────────────┐
│  诊断仪     │ ────────────────▶ │    ECU      │
│  (Tester)   │ ◀──────────────── │  (Server)   │
└─────────────┘  响应(诊断响应)   └─────────────┘
```

- **诊断仪（Tester）**：请求方（PC 诊断工具、车机、产线设备）
- **ECU（Server）**：响应方，实现诊断栈（Dcm）

---

# 二、UDS 协议分层

## 2.1 分层架构

```
┌─────────────────────────────┐
│ 应用层：诊断服务（0x10-0x3E）│
├─────────────────────────────┤
│ 传输层：ISO 15765-2 (CanTp)  │
│   - 单帧/首帧/连续帧分帧      │
├─────────────────────────────┤
│ 数据链路层：CAN 2.0 / CAN FD │
└─────────────────────────────┘
```

## 2.2 CAN 报文 ID 分配（11 位）

| 报文 | 方向 | 典型 ID |
|---|---|---|
| 物理请求 | 诊断仪 → ECU | 0x7E0 |
| 物理响应 | ECU → 诊断仪 | 0x7E8 |
| 功能请求 | 诊断仪 → 所有 ECU | 0x7DF |

## 2.3 传输层（ISO 15765-2）

当诊断数据超过单帧 8 字节（CAN 2.0）时，需要**分帧**：

| 帧类型 | PCI 头 | 说明 |
|---|---|---|
| **单帧 SF** | 0x0N | 1-7 字节数据直接发 |
| **首帧 FF** | 0x1N | 告知总长度，开始多帧传输 |
| **连续帧 CF** | 0x2N | 后续数据（带序号） |
| **流控帧 FC** | 0x3N | 接收方控制发送节奏 |

### 单帧示例（读数据 0x22）

```
请求：07 22 F1 90        ← PCI=0x07(长度7) 服务=0x22 参数=F190
响应：06 62 F1 90 04 00  ← 长度6 服务响应=0x62 数据=0400
```

### 多帧流程

```
Tester                            ECU
  │  FF: 0x10 总长8字节 ...        │
  │ ────────────────────────────▶ │
  │  FC: 0x30 流控（允许发送）     │
  │ ◀──────────────────────────── │
  │  CF: 0x21 第一段数据           │
  │ ────────────────────────────▶ │
  │  CF: 0x22 第二段数据           │
  │ ────────────────────────────▶ │
```

---

# 三、诊断会话与安全

## 3.1 诊断会话（服务 0x10）

ECU 有不同权限级别的诊断会话：

| 会话 | 会话号 | 说明 |
|---|---|---|
| **默认会话** | 0x01 | 只读基本诊断（上电默认） |
| **编程会话** | 0x02 | 允许刷写固件 |
| **扩展会话** | 0x03 | 允许写数据、例程等 |
| 安全扩展会话 | 0x04 | 更高的安全级别（部分厂商） |

```
请求：02 10 03        ← 请求切换到扩展会话
响应：06 50 03 00 32 01 F4  ← 0x50=响应 03=扩展 P2=0x32 P2*=0x01F4
```

- 会话有**超时机制**（P2/P2*），超时自动回到默认会话
- 编程/扩展会话是刷写、标定等高级操作的前提

## 3.2 安全访问（服务 0x27）

敏感操作（刷写、写数据）前需要**解锁**：

```
① Tester: 27 01        ← 请求种子（Seed） 
② ECU:    67 01 5A3F    ← 返回种子 Seed = 0x5A3F
③ Tester: 27 02 8C41    ← 发送计算出的密钥 Key = 0x8C41
④ ECU:    67 02          ← 验证通过，解锁成功
```

- Seed-Key 算法由各 OEM/供应商保密
- 连续错误尝试会**锁定**（防暴力破解），延迟响应

## 3.3 超时参数

| 参数 | 说明 |
|---|---|
| **P2** | 默认响应时间（≤50ms 通常） |
| **P2\*** | 扩展响应时间（长任务） |
| S3 | 会话超时（通常 5s，超过回默认会话） |

---

# 四、常用诊断服务详解

## 4.1 会话控制 0x10

已在上文讲解。

## 4.2 ECU 复位 0x11

```
请求：02 11 01     ← 硬复位（0x01 硬复位/0x03 软复位/0x04 快速上电）
响应：06 51 01 00 32 01 F4
```

- 刷写完成后复位，激活新固件
- 复位后 ECU 回到默认会话

## 4.3 读取数据 0x22

```
请求：03 22 F1 90           ← DID = 0xF190（如电瓶电压）
响应：06 62 F1 90 04 00     ← 数据 = 0x0400（10.24V，分辨率0.01V）
```

- **DID**（Data Identifier）标识要读的数据
- 常用 DID：F190（电源电压）、F191（点火状态）、F18F（VIN）等

## 4.4 写入数据 0x2E

```
请求：06 2E F1 90 04 00     ← 写入 DID F190 = 0x0400
响应：06 6E F1 90 04 00     ← 写入成功，回显
```

- 写**标定值、配置、学习值**等
- 通常需要进入扩展会话并安全访问

## 4.5 读故障码 0x19

### 子功能

| 子功能 | 说明 |
|---|---|
| 0x01 | 按状态读 DTC |
| 0x02 | 按严重等级读 DTC |
| 0x04 | 读快照数据（冻结帧） |
| 0x06 | 读扩展数据 |
| 0x0A | 读支持的 DTC 列表 |

```
请求：03 19 02 01     ← 读当前存储的故障码
响应：04 59 02 01 05  ← 有 5 个故障码
```

### 故障码 DTC 格式

```
DTC = 3 字节（如 0x123456）
 ├─ 0x12：故障所属系统（0x1=动力，0xC=底盘...）
 ├─ 0x34：故障位置
 └─ 0x56：故障类型
```

## 4.6 清除故障码 0x14

```
请求：04 14 FF FF FF     ← 清除所有 DTC（组号 FF 表示全部）
响应：01 54
```

## 4.7 例程控制 0x31

执行 ECU 内的**特定程序**（标定学习、自检、复位学习值）：

```
请求：04 31 01 0203     ← 开始例程 0x0203
响应：06 71 01 0203 00 00
```

- 子功能：01 开始 / 02 停止 / 03 查询结果

## 4.8 输入输出控制 0x2F

诊断仪直接控制 ECU 输出（执行器测试）：

```
请求：04 2F 0101 03     ← 控制 IO 0101 为状态 0x03
响应：06 6F 0101 03 00 00
```

## 4.9 刷写相关服务

| 服务 | 功能 |
|---|---|
| **0x34** 请求下载 | 协商下载地址、数据长度 |
| **0x36** 传输数据 | 传输固件块 |
| **0x37** 请求退出传输 | 完成/取消传输 |
| **0x38** 请求文件传输 | 文件级下载 |

### 刷写流程

```
① 10 02          进入编程会话
② 27 01/02       安全解锁
③ 31 01 0202     预编程（关闭DTC记录等）
④ 34 00 44 8000 0000 1000   请求下载 0x1000 字节到 0x8000
⑤ 36 01 数据...  传输数据（可多帧）
⑥ 37 00          退出传输
⑦ 31 01 FF00     编程校验（CRC）
⑧ 11 01          ECU 复位激活
```

---

# 五、响应格式与错误处理

## 5.1 响应格式

**肯定响应**：服务号 + 0x40

```
请求：0x22 → 响应：0x62
请求：0x10 → 响应：0x50
```

**否定响应**：服务号 0x7F + 请求服务号 + NRC

```
响应：03 7F 22 31     ← 服务 0x22 返回 NRC=0x31
```

## 5.2 否定响应码（NRC）完整解析

当 ECU 无法肯定响应时，返回否定响应：`0x7F + 请求服务号 + NRC`。

```
响应：03 7F 22 31     ← 服务 0x22 返回 NRC=0x31
```

NRC（Negative Response Code，否定响应码）按 ISO 14229-1 定义，配合请求服务号指明失败原因。完整含义如下：

### 通用类（0x1x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x10 | generalReject | **一般拒绝**：请求在接收时被错误处理，无更具体原因 |
| 0x11 | serviceNotSupported | **服务不支持**：请求的服务 ID 不存在（如未实现 0x2E） |
| 0x12 | subFunctionNotSupported | **子功能不支持**：服务存在但该子功能不支持（如 0x19 02） |
| 0x13 | incorrectMessageLengthOrInvalidFormat | **消息长度错误或格式无效**：长度与规定不符、无效字节 |
| 0x14 | responseTooLong | **响应信息过长**：响应数据超出一帧/缓冲区容量 |
| 0x15 | busyRepeatRequest | **忙，重复请求**：ECU 忙，请稍后重试 |
| 0x16 | incorrectInactiveSession | **会话错误**：请求在不正确的会话中发出 |
| 0x17 | requestNotCorrectlyReceived | **请求未正确接收**：接收时校验失败 |

### 条件与序列类（0x2x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x21 | busyRepeatRequest | **忙碌，重复请求**：ECU 正在处理其他操作 |
| 0x22 | conditionsNotCorrect | **条件不正确**：前置条件未满足（如未安全解锁就写数据） |
| 0x23 | badSupportedFunction | **不支持/无效功能组合**：请求的功能不被当前配置支持 |
| 0x24 | requestSequenceError | **请求序列错误**：违反规定的时序（如未 0x34 就发 0x36） |
| 0x25 | noResponseFromSubnetComponent | **子网组件无响应**：子网内的组件没有响应 |
| 0x26 | failurePreventsExecutionOfRequestedAction | **故障阻止执行**：因某故障无法执行请求动作 |
| 0x27 | busyRequestedAction | **动作忙**：请求的动作当前正忙 |
| 0x28 | conditionsNotCorrectForRequestedAction | **请求动作条件不满足** |
| 0x29 | requestSequenceErrorForSubnetComponent | **子网组件请求序列错误** |
| 0x2A | actionNotSupportedForClient | **不支持客户端动作** |
| 0x2B | maxNumberOfReceivedPacketsExceeded | **接收包数超过上限** |
| 0x2C | invalidFrameNumber | **无效帧号** |

### 数据与范围类（0x3x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x31 | requestOutOfRange | **请求超出范围**：DID/DTC/参数超出有效范围（最常见错误之一） |
| 0x32 | requestOutOfRangeSubnetComponent | **子网组件参数超出范围** |
| 0x33 | securityAccessDenied | **安全访问被拒绝**：未解锁就执行需要安全的操作 |
| 0x34 | authenticationFailed | **认证失败**：身份验证不通过 |
| 0x35 | invalidKey | **密钥无效**：0x27 解锁回传的 Key 计算错误 |
| 0x36 | exceedNumberOfAttempts | **超出尝试次数**：解锁尝试次数过多被锁定 |
| 0x37 | requiredTimeDelayNotExpired | **时间延迟未过**：连续请求太频繁，需等待 |
| 0x38 | secureDataTransmissionRequired | **需要安全传输**：要求加密/安全数据连接 |
| 0x39 | secureDataTransmissionNotSupported | **不支持安全传输** |
| 0x3A | secureDataTransmissionNotAllowed | **不允许安全传输** |
| 0x3B | certificateVerificationFailed | **证书验证失败** |
| 0x3C | certificateExpired | **证书已过期** |
| 0x3D | certificateInvalid | **证书无效** |
| 0x3E | certificateCouldNotBeUsed | **证书无法使用** |
| 0x3F | certificateCouldNotBeFound | **找不到证书** |

### 内部与硬件类（0x4x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x40 | softwareError | **软件内部错误**：内部逻辑异常 |
| 0x41 | hardwareError | **硬件内部错误**：控制器硬件异常 |
| 0x42 | externalTestEquipmentError | **外部测试设备错误** |
| 0x43 | externalTestEquipmentOverloaded | **外部设备过载** |
| 0x44 | errorDetectedByExternalTestEquipment | **外部设备检测到错误** |
| 0x45 | targetNotPresent | **目标不存在** |
| 0x46 | subComponentMissingOrFailed | **子组件缺失或失效** |
| 0x47 | internalFlashError | **内部 Flash 错误** |
| 0x48 | dataNotAvailable | **数据不可用** |

### 编程/刷写类（0x7x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x70 | uploadDownloadNotAccepted | **上载/下载被拒绝**：刷写前置条件不满足 |
| 0x71 | transferDataSuspended | **传输数据挂起** |
| 0x72 | generalProgrammingFailure | **一般编程失败**：擦写 Flash 失败 |
| 0x73 | wrongBlockSequenceCounter | **块序列计数器错误**：0x36 序号不连续（丢帧） |
| 0x74 | requestCorrectlyReceived_ResponsePending | **已正确接收-响应待定**（处理中，稍后补发正式响应） |
| 0x75 | incorrectCombinationMethod | **组合方法错误**：子功能与服务组合无效 |
| 0x76 | sessionNotSupported | **会话不支持**：当前会话不允许该服务 |
| 0x77 | serviceNotSupportedInActiveSession | **服务在当前会话不支持**（如默认会话下禁止刷写） |
| 0x78 | rspPending | **响应待定/等待**：长任务进行中，需上位机等待并轮询 |
| 0x79 | moduleIdentificationNotSupported | **模块识别不支持** |
| 0x7A | missingIdentifier | **缺少标识符（ID）** |
| 0x7B | unexpectedIdentifier | **意外的标识符** |
| 0x7C | toMuchData | **数据过多**：超出允许范围 |
| 0x7D | invalidDiagnosticMessageType | **无效诊断消息类型** |
| 0x7E | subFunctionNotSupportedInActiveSession | **子功能在当前会话不支持** |
| 0x7F | serviceNotSupportedInActiveSession | **服务在当前会话不支持** |
| 0x80 | wrongModel | **型号错误** |

### 车辆制造商特定类（0x9x）

| NRC | 英文名 | 含义 |
|---|---|---|
| 0x90 | vehicleManufacturerSpecificStatusCodeNotAvailable | **状态码不可用**：整车厂定义的旧版状态 |
| 0x91 | vehicleManufacturerSpecificStatusCodeAvailable | **状态码可用** |
| 0x92 | vehicleManufacturerSpecificConditionsNotCorrect | **整车厂特定条件不正确**：请求的某个整车厂自定前提未满足（等价于"整车厂版的 0x22/0x31"） |
| 0x93 | vehicleManufacturerSpecificRequestSequenceError | **整车厂特定请求序列错误**：请求顺序不符合整车厂私下规定（等价于"整车厂版的 0x24"） |

> 0x90-0x9F 是 ISO 14229 保留给**整车厂自定义**的 NRC 区间，其中最常用的是 **0x92**（条件不满足）和 **0x93**（请求序列错误）。它们通常用来区分"标准条件"与"OEM 私有条件"——例如某整车的刷写流程要求在 0x34 前必须先发一个私有服务，否则就返回 0x93。**这两个码的含义由各 OEM 自行定义，手册里查不到统一标准**。
>
> 表中 0x7E / 0x7F 是"当前会话不支持"的两个常见变体，和 0x11 / 0x12（"根本不支持"）的区别在于：**前者换了会话就能用，后者任何会话都不能用**。

### NRC 使用要点

1. **优先返回最具体的原因**：如既能返回 0x22 又能返回 0x31，选更贴合实际的那个
2. **0x78 是"软响应"**：长任务（刷写、自检）用 0x78 先占位，避免上位机超时，完成后补发正常响应
3. **0x33/0x35** 常与 0x27 解锁相关，连续 0x35 过多会触发 0x36 锁定
4. **0x31 返回值排第一**：读/写不存在的 DID、参数超范围，几乎都会见到 0x31
5. **刷写常见组合**：0x70（没解锁/前置不满足）→ 0x24（0x34/0x36 顺序错）→ 0x73（丢帧序号错）

---

# 六、诊断栈实现（AUTOSAR）

## 6.1 AUTOSAR 诊断模块

| 模块 | 功能 |
|---|---|
| **Dcm** | 诊断通信管理：会话、安全访问、服务调度 |
| **Dem** | 诊断事件管理：DTC 记录 |
| **Fim** | 功能抑制：按 DTC 状态抑制功能 |
| **CanTp** | CAN 传输层：分帧/重组 |
| **PduR** | PDU 路由 |

## 6.2 数据流

```
CAN 接收 → Can → CanIf → CanTp → PduR → Dcm
                                             ↓
                                    请求解析 → 处理
                                             ↓
CAN 发送 ← Can ← CanIf ← CanTp ← PduR ← Dcm
```

## 6.3 Dcm 关键配置

```c
// 服务表配置（示意）
Dcm_ServiceTable[] = {
    {0x10, Dcm_SessionControl, P2=50ms, S3=5000ms},
    {0x22, Dcm_ReadData,        ...},
    {0x19, Dcm_ReadDtc,         ...},
    {0x27, Dcm_SecurityAccess,  ...},
    {0x34, Dcm_RequestDownload, ...},
};
```

---

# 七、工程实现要点

## 7.1 状态机设计

```
默认会话 ──0x10 02──▶ 编程会话 ──0x10 03──▶ 扩展会话
    ▲                   │                     │
    └────超时/复位────────┴─────────────────────┘
```

## 7.2 开发中的关键细节

1. **时序严格**：P2 超时、S3 会话超时必须准确
2. **多帧正确性**：流控帧参数（BlockSize、STmin）合理设置
3. **安全防护**：连续错误锁定、刷写前置条件检查
4. **长任务响应**：用 0x78 NRC + 周期性响应保持连接
5. **DTC 快照**：冻结帧记录故障发生时刻的环境数据

## 7.3 测试工具

| 工具 | 用途 |
|---|---|
| **CANoe + CANdelaStudio** | 诊断仿真与自动化测试 |
| **诊断仪**（德科/元征等） | 实车诊断 |
| **Python 脚本**（python-can） | 自动化测试 |
| **Vector CANape** | 标定 + 诊断 |
| **总线记录仪** | 抓包分析 |

---

# 八、扩展：DoIP（以太网诊断）

面向车载以太网，UDS 运行在 TCP/IP 上：

| 特性 | CAN 诊断 | DoIP |
|---|---|---|
| 传输介质 | CAN/CAN FD | 以太网 |
| 最大数据 | 8/64 字节/帧 | 64KB+ 无分帧 |
| 应用场景 | 传统 ECU | 域控制器、OTA |
| 端口 | - | TCP 13400 |
| 发现 | 广播 | UDP 车辆发现 |

---

# 九、学习路径

1. **协议基础**：掌握 ISO 14229 服务定义 + ISO 15765-2 传输层
2. **工具实践**：CANoe 模拟诊断仪与 ECU，抓包分析
3. **实现诊断栈**：在 MCU 上实现最小 Dcm（0x10/0x22/0x19/0x27/0x14）
4. **刷写实战**：实现 UDS 刷写流程（结合 Bootloader）
5. **进阶**：Dem 故障管理、DoIP、UDS 自动化测试（CAPL/Python）

**推荐资源**：
- ISO 14229-1 标准文档
- Vector 诊断技术文档
- AUTOSAR SWS Dcm 规范
- 博客《CANoe-CAPL脚本测试入门与实践》《Bootloader与FOTA升级详解》

---

# 总结

UDS 诊断协议的知识主线：

> **分层（应用/传输/链路）→ 会话与安全（0x10/0x27）→ 数据读写（0x22/0x2E/0x2F）→ 故障管理（0x19/0x14）→ 刷写服务（0x34/0x36/0x37）→ 工程实现（Dcm/Dem）→ 扩展（DoIP）**

**核心思想**：UDS 是汽车 ECU 的"标准操作界面"——无论功能多复杂，诊断接口只有一套标准服务，让测试、刷写、故障排查都基于同一语言。

> **一句话记住**：UDS = "ECU 的 HTTP 协议"——0x22 是 GET 数据、0x2E 是 POST 数据、0x19 是查日志（DTC）、0x27 是登录鉴权、0x34/36 是上传固件，理解了这个类比，整套协议就通了。
