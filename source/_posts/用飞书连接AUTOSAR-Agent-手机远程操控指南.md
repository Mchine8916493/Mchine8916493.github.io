---
title: 用飞书连接AUTOSAR-Agent-手机远程操控指南
date: 2026-07-31 19:02:05
tags:
  - AUTOSAR
  - 飞书
  - opencode
  - AI Agent
  - 效率工具
---

# 用飞书连接 AUTOSAR Agent：手机远程操控指南

> 需求：电脑上跑着一个 AUTOSAR 架构专家 Agent（基于 opencode），想用手机飞书和它对话——在地铁上让它写个 SWC 骨架、验证 ARXML、查 BSW 配置。

这篇文章记录完整的接入方案：**飞书长连接 + 本地桥接服务 + opencode HTTP API**，以及过程中踩过的坑。

## 一、方案选型

### 核心约束

家用电脑没有公网 IP，也懒得搞域名和 HTTPS 证书。很多机器人接入方案要求配置「事件回调 URL」，直接卡死。

### 解决方案：飞书「长连接」模式

飞书开放平台的事件订阅支持 **WebSocket 长连接**：飞书服务器主动把事件推到你的电脑，**不需要公网 IP、不需要域名、不需要内网穿透**，手机和电脑在任意网络下都可用。这是本方案能成立的关键。

```
手机飞书 ──▶ 飞书开放平台 ──WebSocket长连接──▶ 本机桥接服务(Python)
    ▲                                            │  HTTP
    └──────────── 飞书API 回复消息 ◀─────────────┘
                                          opencode serve :4096
                                              └─▶ autosar agent
```

- 每个飞书会话（chat_id）映射一个 opencode session → **多轮对话上下文连续**
- `/new` 命令强制开启新 session → 清空上下文
- 回复超长自动分段（飞书单条消息有长度限制）
- 长连接断开后 SDK 自动重连

## 二、opencode serve 驱动的关键发现

### 正确的 API 路由

opencode 的 `serve` 命令启动 headless HTTP 服务（默认 127.0.0.1:0，可指定端口），暴露 OpenAPI 文档。驱动 Agent 的核心是 **v1 同步执行路由**：

| 操作 | 请求 | 说明 |
|---|---|---|
| 建会话 | `POST /session` | body: `{"agent":"autosar","title":"..."}` |
| 发消息 | `POST /session/{id}/message` | body: `{"parts":[{"type":"text","text":"..."}],"agent":"autosar"}`，**同步阻塞直到一轮推理完成** |
| 读记录 | `GET /session/{id}/message` | 获取会话消息列表 |

⚠️ 注意：文档里 `/api/session/{id}/prompt` 那条路径是「入队」路由，不会真的执行模型推理；真正干活的是不带 `/api` 前缀的 `/session/{id}/message`。

### 访问鉴权

启动时设置环境变量 `OPENCODE_SERVER_PASSWORD` 即可启用 Basic Auth（用户名默认 `opencode`）。桥接服务请求时带上 `Authorization: Basic ...` 头。没有密码直接访问会返回 `401`。

### 排坑：Agent 配置的模型失效

桥接服务第一次调试时，发消息始终返回 500，查日志发现：

```
ProviderModelNotFoundError: Model not found: anthropic/claude-sonnet-4
```

原来 agent 定义（`.opencode/agents/autosar/agent.md`）里写的模型 ID 在 models.dev 目录里已经下架了。查询当前可用的模型列表：

```bash
curl http://127.0.0.1:4096/api/model | python3 -m json.tool
```

把 `model:` 改为当前 `status=active` 的模型（示例用了 `opencode/deepseek-v4-flash-free`）后恢复正常。**教训：模型 ID 会过期，Agent 报 500 时先查模型目录。**

## 三、桥接服务实现要点

### 技术栈

- Python 3 + `lark-oapi`（飞书官方 SDK，支持 WS 长连接客户端）
- 标准库 `urllib` 调 opencode API（零额外依赖）

### 核心流程

```python
# 飞书长连接客户端
from lark_oapi.ws import Client as WSClient
from lark_oapi.event.dispatcher_handler import EventDispatcherHandler

handler = (EventDispatcherHandler.builder("", verification_token)
    .register_p2_im_message_receive_v1(on_message)   # 订阅消息事件
    .build())
ws_client = WSClient(app_id, app_secret, event_handler=handler)
ws_client.start()   # 阻塞，自动重连
```

收到消息后的处理：

1. **回执**：立即回复「⏳ 已收到，agent 处理中…」（模型推理可能要几十秒，先给用户反馈）
2. **异步执行**：起后台线程调用 opencode API，`POST /session/{id}/message` 同步等待
3. **回复**：用 `im/v1/messages/{message_id}/reply` 接口把结果作为「回复消息」发回去

### 细节处理

- **群聊 @ 识别**：只有被 @ 才响应。注意飞书事件里 `content` 是 JSON 字符串，内部引号是转义的（`\"`），要**先 JSON 反序列化再匹配** `<at user_id="...">`，否则正则永远匹配不到（踩过的坑）。
- **消息类型过滤**：目前只处理 `text`，图片/富文本回提示。
- **白名单**：按 `open_id`/`union_id` 过滤可用用户，首次使用从日志里取 sender 值。
- **会话持久化**：`chat_id → session_id` 映射存 JSON 文件，桥接服务重启后对话上下文不丢。
- **长文本分段**：单条 >3500 字符按行切分，超长单行硬切。

## 四、配置步骤（概览）

### 第 1 步：飞书开放平台（网页操作，约 10 分钟）

1. [open.feishu.cn/app](https://open.feishu.cn/app) 创建**企业自建应用**（个人用可选「仅自己可用」免审批）
2. 添加**机器人**能力
3. **权限管理**开通：`im:message`（读取单聊消息）+ `im:message:send_as_bot`（以机器人身份发消息）
4. **事件与回调** → 选「**使用长连接接收事件**」→ 订阅 `im.message.receive_v1`
5. **版本管理与发布** → 创建版本 → 发布
6. 手机飞书搜索应用名即可找到机器人

### 第 2 步：本机配置

```bash
# 1. 启动 opencode headless 服务（带鉴权）
export OPENCODE_SERVER_PASSWORD='你的密码'
nohup opencode serve --hostname 127.0.0.1 --port 4096 &

# 2. 配置桥接服务
cd ~/opencode-project/feishu
cp .env.example .env        # 填入 App ID / App Secret / 密码
./start.sh --daemon         # 后台启动

# 3. 查看状态
./start.sh status           # stop 停止
```

启动成功后日志里能看到两条关键信息：

```
opencode 服务连通: HTTP 200
connected to wss://msg-frontier.feishu.cn/ws/v2?...
```

第二条说明长连接已建立，链路打通。

### 日常使用

- 发普通消息 = 和 Agent 对话（ARXML 验证、SWC 设计、RTE/BSW 配置、代码生成等）
- `/status` 查看桥接状态
- `/new` 清空上下文

## 五、踩坑清单

| 现象 | 原因 | 解决 |
|---|---|---|
| Agent 返回 500 | agent.md 里模型 ID 已下架 | 查 `/api/model`，改有效模型 |
| 群聊不响应 | 事件 content 是转义 JSON，正则匹配失败 | 先反序列化再匹配 @ |
| 桥接启动自检 401 | 健康检查没带 Basic Auth 头 | 自检请求也带上鉴权头 |
| `POST /session` 报错 | 请求带了 schema 之外的字段 | 严格按 OpenAPI 文档构造 body |

## 六、总结

- **飞书长连接**让家用电脑免公网 IP 接入机器人，体验接近微信机器人
- **opencode serve** 提供完整 HTTP API，任何语言都能驱动 Agent，是头less 化的标准入口
- 桥接层 200 行 Python 搞定，核心是「事件接入 + 会话映射 + 异步回执」

从此 AUTOSAR 架构师随身携带，手机上发条消息就能让它干活。

---

*本文是实操记录，代码与脚本仅作示意。部署前请务必做好鉴权与白名单配置。*
