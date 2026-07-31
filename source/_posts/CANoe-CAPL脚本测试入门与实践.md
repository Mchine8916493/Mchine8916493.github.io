---
title: CANoe CAPL脚本测试入门与实践
date: 2026-07-31 18:04:53
tags:
  - 汽车电子
  - CANoe
  - CAPL
  - 测试
---

# CANoe CAPL 脚本测试入门与实践

在汽车电子开发中，CANoe 是 Vector 公司出品的总线仿真与测试工具，堪称 ECU 开发、集成测试与诊断的"瑞士军刀"。而 **CAPL**（Communication Access Programming Language）是 CANoe 内置的类 C 脚本语言，用于编写总线仿真、自动测试与报文处理逻辑。本文从零开始，带你掌握 CAPL 脚本测试的核心技能。

## 一、CAPL 是什么？

CAPL 是一种**事件驱动**的语言：平时脚本处于等待状态，一旦绑定的"事件"发生（收到报文、定时器到期、按键按下），对应的处理函数就被自动调用。

### 1.1 CAPL 与 C 的异同

| 特点 | CAPL | C |
|---|---|---|
| 语言风格 | 类 C | 标准 C |
| 事件驱动 | 是（核心） | 否 |
| 内存管理 | 自动管理 | 手动 |
| 指针 | 部分支持（受限） | 完整支持 |
| 编译运行 | CANoe 内置编译器 | 独立编译 |
| 适用场景 | 总线仿真/测试 | 通用编程 |

### 1.2 主要事件类型

```c
// 报文接收事件：收到 ID 为 0x123 的报文时执行
on message 0x123
{
    // ...
}

// 定时事件
on timer MyTimer
{
    // 定时器到期
}

// 按键事件
on key 's'
{
    // 按下键盘 s 键
}

// 系统启动事件
on start
{
    // 仿真启动
}

// 错误帧事件
on errorFrame
{
    // 总线错误帧
}

// 诊断请求事件
on diagRequest *
{
    // ...
}
```

## 二、CAPL 基本语法

### 2.1 数据类型

```c
// 基础类型
int    i;        // 32 位整数
word   w;        // 16 位无符号
dword  dw;       // 32 位无符号
float  f;        // 浮点
char   c[32];    // 字符串

// 枚举与常量
enum { OFF = 0, ON = 1 };
const int MAX_RETRY = 3;

// 报文与信号类型
message 0x123 msg123;      // CAN 报文
msTimer  myTimer;          // 毫秒定时器
```

### 2.2 信号访问

访问报文信号时，需要先将信号拖入"系统变量"或用 `message` 变量：

```c
on message 0x123
{
    // 方式一：通过 dbc 信号名直接访问（需关联数据库）
    if (this.Speed > 50)   // this 指向当前报文
    {
        write("车速超过 50 km/h");
    }

    // 方式二：使用消息选择器
    msg123 = this;
    int rpm = msg123.EngineSpeed;   // 读取 dbc 中的信号
    msg123.TargetSpeed = 80;        // 写入信号（发送前）
    output(msg123);                 // 发送到总线
}
```

### 2.3 定时器

```c
msTimer  cycleTimer;
char     cnt = 0;

on start
{
    // 每 100ms 触发一次
    setTimer(cycleTimer, 100);
}

on timer cycleTimer
{
    // 周期性发送报文
    message 0x123 txMsg;
    txMsg.Byte(0) = cnt;
    output(txMsg);
    cnt++;
    setTimer(cycleTimer, 100);   // 重新启动定时器
}
```

### 2.4 常用内置函数

| 函数 | 功能 |
|---|---|
| `output(msg)` | 发送报文到总线 |
| `setTimer(timer, ms)` | 启动定时器 |
| `cancelTimer(timer)` | 取消定时器 |
| `write()` | 输出到 Write 窗口 |
| `testWaitForMessage()` | 测试等待接收报文 |
| `testGetSignal()` / `testSetSignal()` | 读写信号值 |
| `getMsgChannel()` | 获取通道号 |
| `setSignal()` / `getSignal()` | 全局信号访问 |
| `getBusLoad()` | 读取总线负载 |

## 三、CANoe 测试功能开发（Test Module）

CAPL 除了仿真外，最重要的用途是构建**自动化测试**。CANoe 的 Test Module（测试节点）支持 `test_*` 系列函数，可以编写标准的测试用例。

### 3.1 基本测试框架

```c
// 测试用例：验证车速信号的限幅逻辑
testcase TC_Check_Speed_Clamp()
{
    int speedIn, speedOut;

    // 1. 测试开始
    testCaseStart("TC_Check_Speed_Clamp", "车速信号限幅验证");

    // 2. 设置激励：注入 120 km/h
    testSetSignal(EngineSpeed, 120);

    // 3. 等待 ECU 响应（1s 超时）
    if (testWaitForSignalMatch(DisplayedSpeed, 100, 1000))
    {
        speedOut = testGetSignal(DisplayedSpeed);
        // 4. 断言：显示值不应超过 100
        if (speedOut > 100)
        {
            testStepFail("限幅失效", "期望<=100，实际=%d", speedOut);
        }
        else
        {
            testStepPass("限幅正常", "输出=%d", speedOut);
        }
    }
    else
    {
        testStepFail("超时", "1s 内未收到响应");
    }

    testCaseEnd();
}

// 测试主入口
void MainTest()
{
    TestGroupBegin("速度功能测试", "Speed Function");
    TC_Check_Speed_Clamp();
    TestGroupEnd();
}
```

### 3.2 常用测试函数

| 函数 | 功能 |
|---|---|
| `testCaseStart() / testCaseEnd()` | 用例开始/结束 |
| `testStepPass() / testStepFail()` | 记录通过/失败步骤 |
| `testWaitForMessage(id, timeout)` | 等待指定报文 |
| `testWaitForSignalMatch(sig, val, timeout)` | 等待信号达到目标值 |
| `testWaitForTimeout(ms)` | 固定延时等待 |
| `testReportWrite()` | 写测试报告 |
| `TestGroupBegin() / TestGroupEnd()` | 用例分组 |
| `testAssert()` | 通用断言 |

### 3.3 测试报告

用例执行后 CANoe 自动生成测试报告（HTML/XML），包含：

- 每条用例的执行结果（Pass/Fail/Error）
- 每个测试步骤的详细记录（值、时间戳）
- 失败时的错误信息与总线日志关联

## 四、实战：发动机转速超速报警测试

以"发动机超速报警"功能为例，完整演示 CAPL 测试用例开发。

### 4.1 测试需求

| 项目 | 内容 |
|---|---|
| 测试对象 | 仪表盘 ECU（显示车速） |
| 输入信号 | `EngineSpeed`（发动机转速） |
| 输出信号 | `SpeedAlarm`（报警标志，1=报警） |
| 报警阈值 | 6000 rpm |
| 判定条件 | 转速 > 6000 时 200ms 内报警置 1 |

### 4.2 测试用例实现

```c
/* 超速报警测试 */
testcase TC_Overspeed_Alarm()
{
    const int THRESHOLD = 6000;
    int result;

    testCaseStart("TC_Overspeed_Alarm", "发动机超速报警");

    // 步骤 1：正常转速，不应报警
    testSetSignal(EngineSpeed, 3000);
    testWaitForTimeout(200);
    result = testGetSignal(SpeedAlarm);
    testStep(result == 0 ? pass : fail, "正常转速不报警", "SpeedAlarm=%d", result);

    // 步骤 2：超过阈值，应报警
    testSetSignal(EngineSpeed, 6500);
    if (testWaitForSignalMatch(SpeedAlarm, 1, 200))
    {
        testStepPass("超速报警", "6500rpm 时 200ms 内触发报警");
    }
    else
    {
        testStepFail("超速报警", "6500rpm 未在 200ms 内报警");
    }

    // 步骤 3：恢复转速，报警应解除
    testSetSignal(EngineSpeed, 3000);
    if (testWaitForSignalMatch(SpeedAlarm, 0, 500))
    {
        testStepPass("报警解除", "转速恢复后报警清除");
    }
    else
    {
        testStepFail("报警解除", "报警未清除");
    }

    testCaseEnd();
}
```

### 4.3 配置测试环境

1. 在 Simulation Setup 中新建 **Test Module** 节点；
2. 双击测试节点，打开 CAPL Browser 编写脚本；
3. 添加测试用例到 `MainTest()`；
4. 关联 DBC 文件，使信号名可用；
5. 点击运行，自动执行并生成报告。

## 五、OSEK TP DLL 库函数调用（CAN TP 传输层）

CAN 单帧数据只有 8 字节（经典 CAN）或 64 字节（CAN FD），当需要传输超过单帧容量的大块数据（如 UDS 诊断请求/响应、Bootloader 刷写数据）时，必须借助**传输层协议**将数据分段发送、重组接收。Vector 官方提供的 **OSEK TP DLL**（`osek_tp.dll`）在 CANoe 中实现了基于 ISO 15765-2 的 CAN TP 传输层，并封装为一组 CAPL 可直接调用的库函数，是 CANoe 诊断测试与 ECU 刷写仿真的利器。

### 5.1 为什么需要 TP 层

ISO 15765-2 定义了四种帧类型：

| 帧类型 | 缩写 | 作用 |
|---|---|---|
| 单帧 | SF | 单帧即可装下的数据（≤7 字节） |
| 首帧 | FF | 多帧传输的第一帧，声明总数据长度 |
| 连续帧 | CF | 后续分段数据 |
| 流控帧 | FC | 接收方告知发送方：块大小（BlockSize）与最小间隔时间（STmin） |

> 以 4096 字节的 UDS 响应为例：若不使用 TP 层，需手动拆成数百帧并自行管理时序、超时与重组，极易出错；使用 `osek_tp.dll` 后只需一行 `CanTpSendData()`，拆包、流控、重组全部自动完成。

### 5.2 加载 osek_tp.dll

1. 在 Simulation Setup 中**右键目标节点**（仿真节点或 Test Module 测试节点）；
2. 选择 **Configuration → Components**；
3. 点击 **Add**，选择 `osek_tp.dll`（Vector 安装目录下自带）；
4. **重新打开 CAPL Browser**，即可在函数列表中找到导出的 `CanTp_*` 系列函数。

> 注意：DLL 是节点级组件，**每个需要 TP 能力的节点都要单独添加**；加载后必须重开 CAPL Browser 才能识别新函数。

### 5.3 常用 API 一览

| 分类 | 函数 | 说明 |
|---|---|---|
| 连接管理 | `CanTpCreateConnection(mode)` | 创建 TP 连接并返回句柄（0 = 普通模式） |
| | `CanTpDeleteConnection(handle)` | 删除连接 |
| 地址配置 | `CanTpSetTxIdentifier(handle, id)` | 设置发送 CAN ID（如 0x710） |
| | `CanTpSetRxIdentifier(handle, id)` | 设置接收 CAN ID（如 0x718） |
| 流控参数 | `CanTpSetBlockSize(handle, n)` | 流控块大小（0 = 不限制） |
| | `CanTpSetSTmin(handle, ms)` | 连续帧最小间隔（0~127ms） |
| | `CanTpSetFlowControlDelay(handle, ms)` | 接收方发出流控帧前的延迟 |
| 超时参数 | `CanTpSetTimeoutAr/As/Bs/Cr(handle, ms)` | AR/AS/BS/CR 各阶段超时 |
| 填充配置 | `CanTpSetPadding(handle, byte)` | 空闲字节填充值（如 0xAA） |
| 数据收发 | `CanTpSendData(handle, data[], len)` | 发送大块数据（自动分段） |
| 回调函数 | `CanTp_ReceptionInd(handle, data[])` | 收到完整消息时回调（自动重组） |
| | `CanTp_ErrorInd(handle, errCode)` | TP 层错误回调 |

### 5.4 建立 TP 连接

```c
variables
{
    long    g_CanTpHandle = 0;
    message 0x710 txPhyMsg;      // 物理寻址：诊断请求
    message 0x718 rxResMsg;      // ECU 诊断响应
}

/* 初始化 TP 连接（on start 或发送前调用一次） */
void CanTP_Init(void)
{
    g_CanTpHandle = CanTpCreateConnection(0);   // 0 = 普通模式

    CanTpSetTxIdentifier(g_CanTpHandle, 0x710);
    CanTpSetRxIdentifier(g_CanTpHandle, 0x718);

    // 流控参数：块大小 0（不限制），STmin = 20ms
    CanTpSetBlockSize(g_CanTpHandle, 0);
    CanTpSetSTmin(g_CanTpHandle, 20);
    CanTpSetFlowControlDelay(g_CanTpHandle, 15);

    // 超时参数（单位 ms）
    CanTpSetTimeoutAr(g_CanTpHandle, 250);
    CanTpSetTimeoutAs(g_CanTpHandle, 250);
    CanTpSetTimeoutBs(g_CanTpHandle, 100);
    CanTpSetTimeoutCr(g_CanTpHandle, 250);

    // 空闲字节填充
    CanTpSetPadding(g_CanTpHandle, 0xAA);

    write("OSEK TP 连接已建立，Handle=%d", g_CanTpHandle);
}
```

### 5.5 发送大数据（自动分段）

```c
/* 发送 UDS 22 服务：读取 DID 0xF190 */
void SendUdsReadDataByID(void)
{
    byte reqData[8];
    long  len = 0;

    reqData[len++] = 0x22;       // UDS SID：ReadDataByIdentifier
    reqData[len++] = 0xF1;       // DID 高字节
    reqData[len++] = 0x90;       // DID 低字节

    // TP 层自动完成：单帧/多帧判断、FF+CF 分段、等待流控帧
    CanTpSendData(g_CanTpHandle, reqData, len);
}
```

发送 3 字节请求时 TP 层会自动使用单帧（SF）；当数据超过单帧容量时自动切换为首帧（FF）+ 连续帧（CF），并依据接收方的流控帧（FC）管理发送节奏——**拆包与组包逻辑全部由 DLL 完成**。

### 5.6 接收回调（自动重组）

ECU 返回多帧响应时，DLL 会自动重组数据并调用回调函数：

```c
/* 回调：收到完整 TP 消息（已自动重组，data[] 不含长度字节） */
void CanTp_ReceptionInd(long connHandle, byte data[])
{
    int  len = elcount(data);
    byte i;

    write("收到 TP 数据，连接=%d，长度=%d", connHandle, len);
    for (i = 0; i < len; i++)
    {
        write("  data[%d] = 0x%02x", i, data[i]);
    }
}
```

### 5.7 其他回调函数

| 回调函数 | 触发时机 | 典型用途 |
|---|---|---|
| `CanTp_FirstFrameInd(handle, len)` | 收到对端首帧 | 获知对方即将发送的大块数据长度 |
| `CanTp_PreSend(handle)` | 每次发送数据前 | 更新数据 / 记录发送时刻 |
| `CanTp_SendCon(handle)` | 发送完成确认 | 确认发送成功，驱动下一步流程 |
| `CanTp_ErrorInd(handle, errCode)` | TP 层错误（超时等） | 错误上报与重发策略 |
| `CanTp_ChannelModeChangeInd(handle)` | 通道模式切换 | 诊断会话切换联动 |

### 5.8 使用要点

- 初始化顺序：**先 `CanTpCreateConnection`，再 Set 各项参数**，最后才能 `CanTpSendData`；
- 句柄要保存为全局变量，回调函数通过 `connHandle` 区分多路连接；
- 发送前确认节点 CAN ID 与寻址类型匹配（物理寻址 0x710/0x718，功能寻址 0x7DF）；
- 结合诊断测试时，可封装 `SendUdsRequest(sid, data, len)` + `CanTp_ReceptionInd` 形成完整的 UDS 收发闭环；
- 64 位工程注意选用匹配位数的 DLL（Exec32/Exec64 目录）。

## 六、LIN TP DLL 库函数调用（LIN TP 传输层）

与 CAN 类似，LIN 总线也有一套自己的传输层协议 **LIN TP**（定义于 LIN 2.x 规范，参考 ISO 15765-2 设计），用于在 8 字节的 LIN 帧上传输超过单帧容量的诊断数据。Vector 提供的 **LINtp.dll** 在 CANoe 中封装了 LIN TP 协议，即使不使用 CDD/PDX 诊断数据库，也能通过 CAPL 直接实现 LIN 诊断通信。

### 6.1 LIN TP 与 CAN TP 的差异

| 对比项 | CAN TP（osek_tp.dll） | LIN TP（LINtp.dll） |
|---|---|---|
| 物理层 | CAN / CAN FD | LIN（单线，12V） |
| 拓扑 | 多主/多从，总线仲裁 | 主从结构，无仲裁 |
| 诊断寻址 | CAN ID（0x710/0x718 等） | **NAD**（节点诊断地址） |
| 诊断帧 | 任意诊断 ID | 固定 0x3C（MasterReq）/ 0x3D（DiagResp） |
| 发送时机 | 任意时刻主动发送 | 由**主机调度表**控制帧时隙 |
| 数据帧格式 | CAN ID + PCI + 数据 | NAD + PCI + 数据 |
| 单帧数据量 | SF ≤ 7 字节 | SF ≤ 6 字节 |

LIN 的诊断数据帧格式为 `NAD + PCI + Data`：

- **NAD**：目标从节点的诊断地址（如 0x55）；
- **PCI**：协议控制信息，标明帧类型（SF=0 / FF=1 / CF=2 / FC=3）与帧序号；
- **Data**：有效数据，单帧（SF）最多 6 字节，连续帧（CF）最多 7 字节。

### 6.2 加载 LINtp.dll 与工程准备

1. LIN 工程需先加载 **LDF 文件**，其中必须包含 0x3C 诊断请求帧、0x3D 诊断响应帧，以及**主机调度表**（调度表中至少要有一条包含诊断帧的表项，如 AppSend / MasterReq / DiagResp）；
2. 在 **LIN 主节点**上右键 → Configuration → Components，添加 `LINtp.dll`（LIN 诊断发送只能由主节点发起）；
3. 重新打开 CAPL Browser 即可看到 `LINtp_*` 系列函数。

> 提示：LIN 从节点可以使用真实 ECU，也可以用 CAPL 仿真从节点接收诊断请求并回复。

### 6.3 常用 API 一览

| 分类 | 函数 | 说明 |
|---|---|---|
| 数据发送 | `LINtp_DataReq(data[], count, nad)` | 发送诊断数据：数据缓冲、字节数、目标 NAD |
| 数据接收 | `LINtp_DataInd(count, nad)` | 回调：收到完整数据（已自动重组） |
| | `LINtp_GetRxData(buffer[], count)` | 在回调中取出接收数据 |
| 发送确认 | `LINtp_DataCon(count)` | 回调：发送完成，可获取已发送字节数 |
| 错误处理 | `LINtp_ErrorInd(error)` | 回调：TP 层错误上报 |
| 调度配合 | `linChangeSchedTable(table)` | 标准 LIN CAPL 函数，切换调度表 |

> `linChangeSchedTable()` 并非 LINtp.dll 导出，而是 LIN 节点自带的标准函数，但它是 LIN 诊断收发**必不可少**的配合函数——只有调度表运行到 0x3C/0x3D 帧的时隙时，诊断帧才会真正出现在总线上。

### 6.4 发送单帧诊断请求

```c
variables
{
    byte req_data[4095];
    byte rxBuffer[4096];
    byte NAD = 0x55;             // 目标从节点诊断地址

    enum eSchedTables
    {
        AppSend    = 0,          // 应用调度表
        MasterReq  = 1,          // 0x3C 诊断请求调度表
        DiagResp   = 2           // 0x3D 诊断响应调度表
    };
}

on key 'a'
{
    // 1. 切换到 0x3C 诊断请求调度表
    linChangeSchedTable(MasterReq);

    // 2. 填充诊断数据（无需自行添加长度，TP 层自动处理）
    req_data[0] = 0x10;          // UDS SID：DiagnosticSessionControl
    req_data[1] = 0x01;          // SubFunction：DefaultSession

    // 3. 发送 2 字节请求 → 自动封装为单帧（SF）
    LINtp_DataReq(req_data, 2, NAD);
}
```

### 6.5 发送多帧诊断请求

发送逻辑与单帧完全相同，**只要数据超过单帧容量（>6 字节），DLL 自动按 FF + CF 分段**：

```c
on key 'b'
{
    byte i;
    linChangeSchedTable(MasterReq);

    // 构造 110 字节的写入请求（UDS 2E 服务：WriteDataByIdentifier）
    req_data[0] = 0x2E;
    req_data[1] = 0xF1;
    req_data[2] = 0x90;
    for (i = 3; i < 110; i++)
    {
        req_data[i] = i;         // 填充业务数据
    }

    LINtp_DataReq(req_data, 110, NAD);   // 自动多帧发送
}
```

### 6.6 接收响应回调（自动重组）

```c
/* 回调：收到从节点响应（DLL 已完成重组） */
void LINtp_DataInd(long count, DWORD nad)
{
    long i;
    LINtp_GetRxData(rxBuffer, count);    // 取出完整数据

    write("从节点 0x%X 收到 %d 字节响应", nad, count);
    for (i = 0; i < count; i++)
    {
        write("  rxBuffer[%d] = 0x%X", i, rxBuffer[i]);
    }

    // 响应接收完毕，切回应用调度表
    linChangeSchedTable(AppSend);
}

/* 回调：发送完成确认 */
void LINtp_DataCon(long count)
{
    write("成功发送 %d 字节，切换到 0x3D 调度表等待响应", count);
    linChangeSchedTable(DiagResp);
}

/* 回调：TP 层错误 */
void LINtp_ErrorInd(int error)
{
    write("LIN TP 错误，错误码 = %d", error);
}
```

### 6.7 完整收发时序

以"主节点发 22 服务读 DID、从节点回数据"为例：

```
[主节点]                         [总线]                       [从节点]
  LINtp_DataReq(0x22...)  →  0x3C 帧(SF)      →   收到请求
  linChangeSchedTable      ↕                    处理数据
  LINtp_DataCon(count)                          （0x3D 帧时隙）
  LINtp_DataInd(count)   ←  0x3D 帧(FF+CF…)  ←   响应多帧
  LINtp_GetRxData(buf,count) → 得到完整响应
```

### 6.8 使用要点

- LIN 诊断由**主节点**发起，LINtp.dll 加在主节点上；从节点端可用真实 ECU 或 CAPL 仿真；
- **NAD 必须与 LDF 中定义的从节点 NAD 一致**，否则从节点不响应；
- 发送前先 `linChangeSchedTable(MasterReq)`，发送完成在 `LINtp_DataCon` 中切到 `DiagResp`，接收完毕切回应用表——调度表切换是 LIN 诊断的关键节奏；
- 接收数据务必在 `LINtp_DataInd` 回调中调用 `LINtp_GetRxData()` 取出，回调外取不到；
- 多帧数据由 DLL 自动分段，无需关心 FF/CF 的 PCI 细节。

## 七、DoIP DLL 库函数调用（以太网诊断传输层）

**答案是：有。** 以太网对应的传输层建模库是 **DoIP.dll**（新版 CANoe 中为 **DoIP.vmodule**）。与 CAN/LIN 的 TP 层不同，以太网诊断不需要在数据链路层拆帧——**TCP 协议本身已承担分段与重组**，以太网的"传输层"封装的是更高一层的 **DoIP（Diagnostics over IP，ISO 13400-2）** 协议。

### 7.1 三种 TP DLL 对比

| DLL | 总线 | 协议 | 寻址方式 | 分段方式 |
|---|---|---|---|---|
| `osek_tp.dll` | CAN | ISO 15765-2 | CAN ID（0x710/0x718） | TP 层拆包（SF/FF/CF/FC） |
| `LINtp.dll` | LIN | LIN 2.x TP | NAD（节点诊断地址） | TP 层拆包（0x3C/0x3D） |
| `DoIP.dll` | Ethernet | ISO 13400-2 | **逻辑地址**（Tester/ECU） | **TCP 协议自动分段** |

### 7.2 DoIP 协议要点

- **双通道**：UDP 端口 13400 用于车辆发现（Vehicle Identification）；TCP 端口 13400 用于路由激活与诊断消息传输；
- **DoIP 报文头**：`版本号(1B) + 逆版本号(1B) + 负载类型(2B) + 负载长度(4B) + 负载`；
- **逻辑地址寻址**：诊断消息靠逻辑地址（非 IP）寻址，如 Tester = 0x0E00/0x1000，ECU = 0x0001~0x0DFF；
- **通信流程**：车辆发现 → 建立 TCP 连接 → 发送**路由激活请求**（响应码 0x10 = 激活成功）→ 传输诊断消息（UDS 负载）→ 关闭连接；
- **协议变体**：CANoe 还支持 DoSoAd 与 OEM 专用的 HSFZ 协议，可在 Diagnostics/ISO TP 配置中选择。

### 7.3 加载 DoIP.dll / DoIP.vmodule

1. 右键网络节点（或测试节点）→ **Configuration → Components**；
2. 点击 **Add**，在 CANoe 安装目录的 `Exec32/Exec64` 文件夹中选择 `DoIP.dll`（或新版 `DoIP.vmodule`）；
3. 重新打开 CAPL Browser，即可看到 `DoIP_*` 系列函数。

> 注意：标准 CANoe 即自带 DoIP.dll（Vector KB0011588）。若需要在 Trace 中解析以太网层/DoIP 层报文，则需要额外的 **CANoe.Ethernet** 选项。

### 7.4 常用 API 一览

| 分类 | 函数 | 说明 |
|---|---|---|
| 模式初始化 | `DoIP_InitAsTester()` | DLL 默认按 ECU 仿真模式运行；测试节点需在 `on preStart` 中调用切换为 Tester 模式 |
| 逻辑地址 | `DoIP_SetTesterLogicalAddress(addr)` | 设置 Tester 逻辑地址（发送时使用） |
| | `DoIP_SetVehicleLogicalAddress(addr)` | 设置 ECU/车辆逻辑地址 |
| 连接管理 | `DoIP_ConnectToVehicle()` | 连接配置的车辆（必要时先执行车辆发现） |
| | `DoIP_CloseConnection()` | 关闭 TCP 连接 |
| 数据收发 | `DoIP_DataReq(buf[], count, ecuAddr, testerAddr)` | 发送 DoIP 诊断消息 |
| | `DoIP_DataInd(buf[], count, ecuAddr, testerAddr)` | 接收回调（收到诊断数据） |
| 超时配置 | `DoIP_GetSetDiagnosticMessageTimeout(ms)` | 诊断消息超时 |
| | `DoIP_GetSetVehicleDiscoveryTimeout(ms)` | 车辆发现超时 |
| | `DoIP_GetSetAliveCheckTimeout(ms)` | 在线保持超时 |
| 回调 | `_DoIP_VehicleIdentificationCompleteInd()` | 车辆识别阶段完成 |
| | `_DoIP_PeriodicDataInd()` | 收到周期数据（负载类型 0x8004） |
| | `_DoIP_TransportLayerStatusInd()` | 传输层状态上报（0x9001） |

### 7.5 Tester 端最小示例：发送 UDS 22 服务

```c
includes
{
    /* 若使用 CCI（CAPL 回调接口）需包含参考实现 */
    /* #include "Diagnostics\\CCI_DoIP.cin" */
}

variables
{
    dword gEcuAddr  = 0x0001;    // ECU 逻辑地址
    dword gTestAddr = 0x1000;    // Tester 逻辑地址
    byte  reqData[8];
}

on preStart
{
    // 切换为 Tester 模式（默认是 ECU 仿真模式）
    DoIP_InitAsTester();
}

on start
{
    // 设置逻辑地址并连接车辆
    DoIP_SetTesterLogicalAddress(gTestAddr);
    DoIP_ConnectToVehicle();
}

on key 'r'
{
    // 发送 UDS 22 服务：读取 DID 0xF190
    reqData[0] = 0x22;
    reqData[1] = 0xF1;
    reqData[2] = 0x90;

    DoIP_DataReq(reqData, 3, gEcuAddr, gTestAddr);
    write("已发送 DoIP 诊断请求");
}
```

### 7.6 接收响应回调

```c
/* 回调：收到 ECU 的 DoIP 诊断响应（UDS 负载，TCP 已自动重组） */
void DoIP_DataInd(byte buffer[], dword count,
                  dword ecuAddress, dword testerAddress)
{
    long i;
    write("收到 ECU 0x%X 响应 %d 字节", ecuAddress, count);
    for (i = 0; i < count; i++)
    {
        write("  data[%d] = 0x%02x", i, buffer[i]);
    }
}
```

### 7.7 结合 CCI 做故障注入与网关

内置诊断通道严格遵循诊断规范，无法注入协议级错误。需要**违反 DoIP 协议**的测试（如错误负载类型、网关仿真）时，必须使用 **CCI（CAPL Callback Interface）**：

1. 节点 Components 添加 DoIP.vmodule；
2. CAPL 中 `#include "Diagnostics\CCI_DoIP.cin"`；
3. 用 `DoIP_TCPSend()` 发送自定义负载类型的数据，配合 `_DoIP_PeriodicDataInd()` / `_DoIP_TransportLayerStatusInd()` 回调处理 0x8004/0x9001 扩展负载类型；
4. 可在诊断配置对话框中**取消勾选 "Simulation by"**，改用 CCI 实现 ECU 仿真。

### 7.8 使用要点

- DoIP 数据分段由 **TCP 完成**，无需像 CAN/LIN TP 那样手动关注 FF/CF/FC；
- **逻辑地址**是 DoIP 寻址核心（而非 IP），发送前务必 `DoIP_SetTesterLogicalAddress` / `DoIP_SetVehicleLogicalAddress`；
- 测试节点务必在 `on preStart` 调用 `DoIP_InitAsTester()`，否则 DLL 按 ECU 仿真运行；
- 车辆发现依赖 UDP 广播/组播（A_DoIP_Ctrl 约 2s 超时），连接不上先查防火墙与 TCP 13400 端口；
- 官方完整函数清单见 CANoe Help 索引中的 **DoIP_** 前缀条目；示例工程在 **Help | Sample Configurations | Diagnostics | Sample Configurations Ethernet**。

## 八、CAPL 调试技巧

### 8.1 断点与单步

CAPL Browser 支持设置断点、单步执行和变量监视，适合定位逻辑错误。

### 8.2 打印调试信息

```c
on message 0x456
{
    // 格式化输出报文数据
    write("ID=0x%x DLC=%d Data=0x%02x 0x%02x 0x%02x 0x%02x",
          this.id, this.dlc,
          this.byte(0), this.byte(1),
          this.byte(2), this.byte(3));
}
```

### 8.3 使用系统变量调试

通过 CANoe 的 **System Variables** 面板可以实时读写系统变量，配合 CAPL 里的 `@sysvar` 语法，方便在运行中调整参数：

```c
on message 0x123
{
    // 读取系统变量（可在面板上实时修改）
    int limit = @sysvar::Test::SpeedLimit;
    if (this.Speed > limit)
    {
        write("超过限速 %d", limit);
    }
}
```

## 九、常见问题与避坑指南

| 问题 | 原因 | 解决方案 |
|---|---|---|
| 信号名未定义 | 未关联 DBC | Simulation Setup → 右键节点 → 分配数据库 |
| 报文发不出去 | 节点未激活/通道错误 | 检查节点激活状态与 `getMsgChannel()` |
| 定时器不触发 | 忘记 `setTimer` 重启 | 周期任务需在回调里重新 setTimer |
| `this` 无法识别 | 事件块外使用 `this` | `this` 只在事件处理函数内有效 |
| 字符串比较失败 | 使用 `==` 比较字符数组 | 用 `strcmp()` 比较 |
| 测试用例没执行 | 未挂到 `MainTest()` | 将用例名写入 MainTest 调用链 |
| 浮点比较误差 | 直接比较 float 值 | 定义容差范围或取整比较 |
| `CanTp_*` 函数未定义 | 节点未加载 `osek_tp.dll` | 节点 Components 中添加 DLL 并重开 CAPL Browser |
| `CanTpSendData` 无响应 | 未先 `CanTpCreateConnection` 或参数未设置 | 按 5.4 节顺序初始化连接后再发送 |
| 多帧响应收不全 | 收发 CAN ID 或寻址类型配置错误 | 核对 `CanTpSetTx/RxIdentifier` 与物理/功能寻址 |
| `LINtp_*` 函数未定义 | 节点未加载 `LINtp.dll` | 在 LIN 主节点 Components 中添加 DLL |
| 从节点不响应 | NAD 与 LDF 配置不一致 | 核对 LDF 中从节点 NAD 与 `LINtp_DataReq` 第三参数 |
| 诊断帧没上总线 | 调度表未包含 0x3C/0x3D 或未切换 | LDF 添加诊断调度表，发送前 `linChangeSchedTable(MasterReq)` |
| 取不到接收数据 | 未在 `LINtp_DataInd` 内取数 | 回调内调用 `LINtp_GetRxData()` 读取 |
| `DoIP_*` 函数不可用 | 节点未添加 DoIP.dll/DoIP.vmodule | 节点 Components 中添加建模库 |
| 测试节点发不出诊断消息 | 未切换 Tester 模式 | `on preStart` 中调用 `DoIP_InitAsTester()` |
| 连不上 ECU / 无响应 | 防火墙、TCP 13400 端口、逻辑地址错误 | 核对 IP 连通性、端口与 `DoIP_Set*LogicalAddress` |

## 十、CAPL 进阶方向

- **诊断测试**：结合 ODX/CDD 文件，用 `diagRequest`/`diagResponse` 事件做 UDS 诊断自动化；
- **多总线协同**：CAN + LIN + FlexRay + Ethernet 混合仿真测试；
- **ECU 测试自动化**：配合 vTESTstudio 图形化生成测试用例后导出 CAPL；
- **回归测试**：与 Jenkins/CI 集成，无人值守跑全量用例；
- **HIL 测试**：CAPL 与 dSPACE/NI 硬件在环平台结合做实时闭环测试。

## 总结

CAPL 是汽车总线测试领域绕不开的核心技能，它的价值在于：

1. **仿真**：用脚本模拟节点行为，替代缺失的 ECU；
2. **测试**：自动化执行用例、自动判定、自动出报告；
3. **诊断**：与诊断协议栈结合实现 UDS 刷写与功能测试。

学习路径建议：先掌握 `on message`、定时器和报文收发，再学习 `test_*` 函数构建用例，最后结合 DBC、诊断与 CI 形成完整的自动化测试体系。**从一段能"发报文"的脚本，到一套能"自动验证功能"的测试工程，这就是 CAPL 的核心进阶之路。**
