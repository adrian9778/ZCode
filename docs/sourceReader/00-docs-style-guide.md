[总目录](README.md)

# 定位约定与源码引用格式

> **场景**：统一本套文档的叙述方式、源码定位规则和图表表达惯例。
> **版本**：v1 (2026-09-23)
> **源码基准**：commit `872ad96`（v3.14.0, `feat: open source`, 2026-09-21）

## 1. 源码定位规则

### 1.1 函数级定位（强制）

所有对源码的定位采用「三段式」：

```text
文件：packages/ui/src/v4/SessionPane.tsx
函数：dispatchCommand
偏移：+1395 ～ +1461
```

含义：
- `文件` — 仓库内绝对路径（从仓库根 `ZCode/` 起算）。
- `函数` — 模块名 / 类名（如 `impl SessionPane` 或 `module v4`）/ 函数名。
- `偏移` — 相对该函数定义行的行号偏移：`+0` = 函数定义行，`+1` = 下一行。不依赖 IDE 的绝对行号，随源码漂移自动适配。

调用关系同时标注双方：

```text
调用方：packages/ui/src/v4/SessionPane.tsx
    函数：dispatchCommand 偏移 +1461
    ↓
被调用方：packages/ui/src/v4/agentConversationTransport.ts
    函数：sendCommand 偏移 +350
```

### 1.2 非函数代码（稳定符号）

对于 Struct / Class / Enum / Trait / Const / Module 等不使用函数偏移：

```text
文件：apps/zcode-cli/packages/bootstrap/src/app/input-facade.ts
Struct：SendInputOptions
字段：delivery

文件：packages/shared/src/zcode-protocol/index.ts
Const：zcodeProtocolMethods.sessionCreate
常量值："session/create"
```

### 1.3 第三方依赖

标明库名和语义角色，不臆测实现细节：

```text
依赖：@ai-sdk/openai-compatible — 兼容 OpenAI 格式的 LLM SDK
依赖：electron — UtilityProcess.fork() 创建 Host 子进程
```

## 2. 图表表达惯例

| 图表类型 | 用途 | 推荐工具 |
|----------|------|----------|
| ASCII 流程图 | 端到端流程、网络协议帧结构、请求/响应交换 | 文本框 |
| Mermaid 时序图 | 模块间事件顺序、跨进程消息流 | `flowchart TD` / `sequenceDiagram` |
| Mermaid 架构图 | 系统层级、模块归属、部署拓扑 | `graph TD` |
| Mermaid 关系图 | Struct / Class / Trait 之间的继承、实现、组合关系 | `classDiagram` |

原则：**ASCII 负责表达阅读流程和数据流，Mermaid 负责表达结构和关系。**

## 3. 叙述风格

- **禁止空泛描述**。每句结论必须有源码依据。无法从源码证实的结论必须写「当前源码无法确定」。
- **禁止「同上」「略」「以此类推」**。核心逻辑必须逐个展开。
- **禁止第四层提前出现在第一/二层**。分层严格执行。
- **每一篇都带导航链接**：上一篇 / 总目录 / 下一篇。

## 4. 术语约定

| 术语 | 含义 |
|------|------|
| Host Process | Electron UtilityProcess fork 出的 Node.js 子进程，每个窗口一个 |
| Agent CLI | `@zcode/cli`，运行 Agent runtime 的 Node.js 进程，通过 stdio NDJSON 与 Host 通信 |
| V4 协议 | `zcode-protocol-v4`，新的命令/事件流协议框架 |
| 旧协议 | `zcode-protocol/index.ts` 中 `session/*`、`workspace/*` 等方法族 |
| Core | `@zcode/core`，Agent 的核心运行时逻辑（turn loop、tool execution、event reduction） |
| Adapter | `@zcode/adapters`，Core 对外的环境抽象层（fs、exec、mcp、model 等） |
| Service | `@zcode/services`，Host 进程中的业务服务（session service、agent service、task adapter 等） |

[总目录](README.md)
