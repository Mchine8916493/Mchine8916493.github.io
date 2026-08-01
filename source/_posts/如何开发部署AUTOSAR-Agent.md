---
title: 如何开发部署AUTOSAR-Agent
date: 2026-08-01 12:36:54
tags:
  - AUTOSAR
  - opencode
  - AI Agent
  - 开发
  - 部署
  - 效率工具
---

# 如何开发部署一个 AUTOSAR Agent

> 需求：让 AI 能真正读懂 ARXML、管理 SWC、配置 RTE 和 BSW，成为汽车电子工程师的「AUTOSAR 架构专家」队友。这篇文章完整记录基于 opencode 从零开发到一键部署 AUTOSAR Agent 的全过程。

## 一、整体思路

### 为什么选 opencode 作为底座

AUTOSAR 生态没有现成的 AI 工具链，但可以基于通用 AI Agent 框架搭建。opencode 提供了一套轻量、可裁剪的 Agent 定义机制：

| 能力 | 说明 |
|---|---|
| Agent 定义 | `agent.md` 文件即可定义一个领域专家，支持 front-matter 声明元数据 |
| Skill 机制 | 每个 skill 是一个 `SKILL.md`，随对话按需加载，注入专业工作流 |
| Command 机制 | `/` 斜杠命令，快速触发固定任务 |
| 模型可配 | front-matter 中指定 `model`，可切换任意 Provider |
| 权限管控 | `permission.skill` 声明允许自动执行的 skill |

这套机制非常适合「领域知识专家型 Agent」：领域知识沉淀在 skill 里，通用推理交给模型，各司其职。

### 架构总览

```
用户终端
   │  执行 `autosar` 命令
   ▼
scripts/autosar.sh          ← 启动器：git pull 更新 + 环境变量 + 启动 agent
   │
   ▼
opencode --agent autosar    ← 加载 .opencode/agents/autosar/agent.md
   │
   ├── .opencode/skills/    ← 领域技能（按需注入）
   │     ├── arxml-validate    ARXML 验证
   │     ├── swc-manager        SWC 组件管理
   │     ├── rte-config         RTE 配置
   │     ├── ecu-extract        ECU Extract 管理
   │     ├── bsw-config         BSW 模块配置
   │     ├── autosar-code-gen   RTE/SWC 代码生成
   │     ├── diagram-draw       Graphviz 架构图
   │     └── pdf-parse          PDF 解析
   │
   └── .opencode/commands/  ← 斜杠命令
         ├── autosar-init      初始化项目结构
         ├── autosar-skill-list 列出技能
         └── code-review        代码审查
```

## 二、开发过程

### 1. 定义 Agent（agent.md）

一个 Agent 的核心就是一个 Markdown 文件，front-matter 声明元数据，正文是系统提示词：

```yaml
---
name: autosar
description: AUTOSAR 架构专家，处理 ARXML、SWC、RTE、BSW、ECU Extract 等任务
model: opencode/deepseek-v4-flash-free
permission:
  skill:
    arxml-validate: allow
    swc-manager: allow
    ...
---
```

正文部分要写清楚三件事：

- **核心能力**：ARXML 解析/验证/生成、SWC 设计与配置、RTE 配置、ECU Extract 管理、BSW 配置、文档与代码生成
- **工作流程**：启动时切回用户原始工作目录（通过环境变量 `AUTOSAR_ORIG_DIR`）、先理解已有 ARXML 结构、再调用对应 skill、变更必须兼容 AUTOSAR 标准、用 Git 管理版本
- **约束**：严格遵守 AUTOSAR 命名规范（CamelCase 模块名、大写包名）、优先复用已有 SWC 与数据类型定义、ECU Extract 不覆盖系统级配置

### 2. 开发 Skill（SKILL.md）

Skill 是领域知识的最小单元。每个 skill 目录下放一个 `SKILL.md`，同样用 front-matter 声明元数据：

```yaml
---
name: swc-manager
description: AUTOSAR Software Component 应用层组件的创建、配置与管理
license: MIT
compatibility: opencode
metadata:
  domain: autosar
---
```

正文则描述该 skill 的**职责**、**工作方式**和**约束**。比如 `swc-manager` 负责创建 Application / SensorActuator / Composition 三类 SWC、配置 P-Port / R-Port、配置端口接口与数据类型等，并明确约束：Composition 不含 Runables、端口接口引用必须已存在于 `AR-PACKAGES`。

这套设计的价值在于：**模型不需要把 AUTOSAR 全量知识背在上下文里**，用到什么 skill 再加载，省 token 且更精准。

### 3. 开发 Command（commands/）

Command 用于快速触发固定任务，同样是一个 Markdown 文件加 front-matter：

```yaml
---
description: 初始化 AUTOSAR 项目目录结构和骨架 ARXML
---
```

`autosar-init` 会创建 `system/`、`ecu/`、`swc/`、`bsw/`、`generate/`、`tools/` 目录结构、骨架 ARXML 和 `.gitignore`，并自动 `git init` 提交初始结构。这样成员在新项目中第一次使用即可一键初始化工程骨架。

### 4. 编写项目文档（AGENTS.md）

仓库根目录的 `AGENTS.md` 是团队协作的「说明书」，包含：

- 项目定位（个人笔记/参考目录，非软件项目）
- 安装/启动命令（`npm i -g opencode-ai`、`opencode serve`）
- 项目结构说明
- 各 Agent 的能力清单
- **发布安全强制规则**：任何发布到公网的操作必须先执行 `sensitive-info-review` skill

### 5. 版本管理

整个仓库用 Git 管理，提交信息遵循 Conventional Commits：

```bash
git add .
git commit -m "feat: add swc-manager skill for SWC component management"
git push origin master
```

## 三、部署过程

### 1. 源码托管

源码托管在 Gitee 私有仓库，作为团队的安装源和更新源。

### 2. 一键安装脚本（scripts/install.sh）

团队成员安装只需一条命令：

```bash
curl -fsSL https://gitee.com/<org>/<repo>/raw/main/scripts/install.sh | bash
```

`install.sh` 自动完成 5 件事：

1. **检查依赖**：缺 git 报错；缺 opencode 自动安装（`curl -fsSL https://opencode.ai/install | bash`）
2. **克隆/更新仓库**：已存在则 `git pull --rebase --autostash`，不存在则 `git clone` 到 `~/.autosar-agent`
3. **安装启动命令**：在 `~/.local/bin/autosar` 写入一个包装脚本，转发到 `scripts/autosar.sh`
4. **配置 PATH**：自动检测 shell（zsh/bash），把 `~/.local/bin` 追加到 `.zshrc` / `.bashrc`
5. **提示配置模型**：首次使用需执行 `opencode connect` 配置 Provider

### 3. 启动器（scripts/autosar.sh）

`autosar` 命令启动时的行为：

```bash
AUTOSAR_HOME="${AUTOSAR_HOME:-$HOME/.autosar-agent}"
ORIG_DIR="$(pwd)"                       # 记录用户当前目录
cd "$AUTOSAR_HOME" && git pull --rebase --autostash   # 自动更新到最新版
export AUTOSAR_ORIG_DIR="$ORIG_DIR"     # 让 agent 能切回用户工作目录
exec opencode "$ORIG_DIR" --agent autosar
```

几个关键设计：

- **自动更新**：每次启动先 `git pull`，成员永远用最新技能，无需手动升级
- **工作目录保真**：通过 `AUTOSAR_ORIG_DIR` 环境变量把用户启动时的目录传给 agent，agent 在 `agent.md` 里声明启动时 `cd "$AUTOSAR_ORIG_DIR"`，解决「在 `~/.autosar-agent` 下启动导致工作目录错乱」的问题
- **可选飞书桥接**：检测到 `feishu/.env` 配置时自动拉起 `opencode serve` 和飞书桥接服务，实现手机远程操控（详见本站另一篇《用飞书连接 AUTOSAR-Agent 手机远程操控指南》）

### 4. 知识文档发布（本博客）

Agent 的能力说明和实战指南通过 Hexo 博客发布，与源码仓库解耦，方便成员在浏览器里查阅。发布流程：

```bash
cd ~/mchine8916493.github.io
npx hexo new "文章标题"          # 创建文章
# 编辑 source/_posts/文章标题.md
npx hexo clean && npx hexo generate && npx hexo deploy   # 构建并部署到 gh-pages
git add -A && git commit -m "新增文章：文章标题" && git push origin master  # 备份源码
```

> ⚠️ 强制规则：**任何发布到公网的操作**（git push、hexo deploy、上传等）之前，必须先运行 `sensitive-info-review` skill 做敏感信息审查，确认无 API 密钥、口令、私钥等泄露后才能发布。

## 四、持续迭代的实践经验

1. **沉淀优先于重复**：每次手工解答用户问题后，如果发现是重复性任务，就把它固化成 skill 或 command，形成知识飞轮
2. **一次只改一个点**：开发新 skill 时用 Git 分支，验证通过再合并，避免破坏在用的 agent
3. **文档与代码同仓**：`AGENTS.md`、skill、脚本、安装器放在同一个仓库，成员 clone 一次全部拿到
4. **发布前必做安全审查**：扫描密钥/口令/私钥模式，检查 `.env`、`*.log`、`state.json` 等运行时产物是否被意外入库

## 五、总结

基于 opencode 搭建领域专家型 Agent 的完整链路是：

> **写 agent.md 定义角色 → 写 SKILL.md 沉淀领域知识 → 写 command 固化高频操作 → Git 管理 → 一键安装脚本分发 → 启动器自动更新 → 博客发布知识文档**

这套模式的精髓在于「**配置即代码**」：整个 Agent 就是一个可版本化、可分发、可一键部署的 Git 仓库，团队成员既能直接使用，也能 fork 后按自己的领域裁剪。AUTOSAR Agent 只是第一个例子——同样的方法可以复制出 LIN/以太网协议栈专家、ISO 26262 功能安全专家、CANoe 脚本专家等更多领域队友。
