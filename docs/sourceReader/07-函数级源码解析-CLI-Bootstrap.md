[上一篇](06-配置与数据流.md) · [总目录](README.md) · [下一篇](08-函数级源码解析-CLI-V4-CommandExecutor.md)

# CLI Bootstrap——`runZCodeProtocolAgent` + V4 Server + Transport

> **场景**：Agent CLI 作为 V4 服务端启动，从 stdio 接收 NDJSON 帧、分派命令、生成回执的全生命周期。
> **层级**：第四层——函数级源码解析
> **版本**：v1 (2026-09-23)
> **源码基准**：commit `872ad96`

---

## 函数：runZCodeProtocolAgent

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-entrypoint.ts`  
**所属：** module-level export function  
**职责：** Agent CLI 协议服务端的进程级入口，负责完整初始化链条的编排与 finally 清理  
**调用方：** `apps/zcode-cli/packages/cli/src/run.ts` (line 265)

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| options | `RunZCodeProtocolAgentOptions` | 可选配置项 |
| options.cwd | string | 当前工作目录 |
| options.env | NodeJS.ProcessEnv | 环境变量（用于 provider registry 等） |
| options.input | NodeJS.ReadableStream | stdin（NDJSON 输入流） |
| options.output | NodeJS.WritableStream | stdout（NDJSON 输出流） |
| options.version | string | CLI 版本号 |
| options.presentationSurface | PresentationSurface | "terminal" / "zcode_desktop" |
| options.lifecycle | AbortSignal | 生命周期信号（用于 abort/cleanup） |
| options.prepareStorageOnly | boolean | 仅准备存储模式（不启动完整服务器） |

### 返回值

`Promise<void>` — 阻塞到连接关闭或 error throw。

### 执行流程

```text
├── 1. prepare storage only mode (if options.prepareStorageOnly)
│      → createConfig()
│      → prepareProtocolStartupStorage({ dbPath, input, output })
│      → return
│
├── 2. startup timing & logging
│      → startupStartedAt = Date.now()
│      → loggerFactory.createLogger("zcode")
│      → traceContext = createRootTraceContext()
│      → installZCodeProtocolAiSdkWarningLogger(logger)
│
├── 3. SQLite database open
│      → acquireProtocolStartupResource({
│          create: () => openProtocolStartupStorage({ dbPath, ... })
│          disposeLate: closeSessionStore
│        })
│      → sessionStore = result
│
├── 4. Provider Registry runtime start
│      → startProcessProviderRegistryRuntime(runtimeEnv)
│      → providerRegistryRuntime = result
│      → registryService.ready
│
├── 5. Telemetry env preparation
│      → prepareZCodeTelemetryEnv(runtimeEnv, { cliVersion, productVersion })
│      → telemetryDeviceMid
│
├── 6. MCP telemetry tracker creation (if mcp feature enabled)
│      → createMcpTelemetryTracker({ onEvent, onResourceSamples })
│      → mcpTelemetryTracker
│
├── 7. Official MCP auth headers port (before server)
│      → createOfficialMcpAuthHeadersPort({ resolveContext })
│      → officialMcpAuth = { authHeadersPort, trustedOrigins, resolveZCodeApiOrigin }
│      → workspaceKey resolution (identity || path)
│
├── 8. MCP connection pool creation
│      → createMcpAdapterConnectionPool({ clientVersion, env, network, officialMcpAuth, telemetry })
│      → mcpPort = connectionPool.acquireLease({ leaseId: "protocol-settings" })
│
├── 9. Model selection facade
│      → createNodeModelSelectionFacade(registryService)
│      → modelSelectionFacade.getView(selection)
│
├── 10. ZCodeProtocolAgentServer construction
│      → new ZCodeProtocolAgentServer({
│            createZCodeApp: (appOpts) => createZCodeApp(appOptions),
│            cwd, env, loggerFactory,
│            mcpPort, mcpTelemetry, sessionStore,
│            syncAccountProviderConfig, refreshProviderRegistry, version
│         })
│      → server.officialMcpAuthRequestContext (backward fill)
│
├── 11. Node REPL browser broker (if mcp enabled)
│      → createNodeReplBrowserBroker({ browserControlPort, logger, platform })
│      → await broker.ready
│
├── 12. Connection lifecycle setup
│      → const connection = new ZCodeProtocolNdjsonConnection({
│            signal, clearPostResponseMessages, handleMessage,
│            input, onTransportClosed, output, takePostResponseBatch
│         })
│      → connection.start()
│      → server.setNotificationSink((notification) => connection.send(notification))
│
├── 13. Resource sampler start
│      → startProtocolResourceSampler(server, sendFn, logger)
│
├── 14. waitForClose() — block until transport closed
│      → await connection.waitForClose()
│
└── 15. finally cleanup
       → cleanupProtocolRuntime({ logger, deadlineAt, server, processResourceSampler,
                                  mcpTelemetryTracker, nodeReplBrowserBroker, mcpPort,
                                  mcpConnectionPool, sessionStore, providerRegistryRuntime })
       → shutdown completed log
```

### 调用关系

```text
run.ts:265 → runZCodeProtocolAgent(options)
              ↓
          ZCodeProtocolAgentServer constructor (line 249)
              ↓
          ZCodeProtocolNdjsonConnection constructor + start (line 310-334)
              ↓
          startProtocolResourceSampler (line 336-340)
              ↓
          connection.waitForClose() (line 346) ← blocks here
              ↓
          finally → cleanupProtocolRuntime()
```

### 关键数据结构

```typescript
// 全局变量声明 (line 124-135)
let sessionStore: SqliteSessionStore | undefined;
let serverForCleanup: ZCodeProtocolAgentServer | undefined;
let nodeReplBrowserBroker: NodeReplBrowserBroker | undefined;
let mcpConnectionPool: McpConnectionPool | undefined;
let mcpPort: McpPort | undefined;
let mcpTelemetryTracker: McpTelemetryTracker | undefined;
let mcpResourceSink: ((samples) => void) | undefined;
let mcpTelemetrySink: ((event) => void) | undefined;
let processResourceSampler: ZCodeProcessResourceSampler | undefined;
let providerRegistryRuntime: ... | undefined;
```

### 错误处理

```text
catch (error) {
  options.lifecycle?.requestShutdown(error)
  startupTimer.fail(...)
  throw error   // rethrow to caller
}

finally {
  options.lifecycle?.requestShutdown()
  await cleanupProtocolRuntime({ ... })  // graceful shutdown
}
```

### 设计意图

这个函数的核心设计思想是 **"acquire + disposeLate" pattern**：每个子资源都通过 `acquireProtocolStartupResource()` 创建，在 finally 中按依赖反向顺序销毁。这确保了：
1. 任何步骤失败都不会泄漏资源
2. 清理顺序与初始化顺序完全镜像
3. 所有 async cleanup 都被 await

### 关键代码位置

| 代码段 | 偏移 |
|--------|------|
| Function definition | +82～+84 |
| prepareStorageOnly early return | +85～+93 |
| Logger + Trace init | +98～+108 |
| Database open | +139～+153 |
| Provider Registry | +156～+169 |
| MCP pool creation | +227～+243 |
| Server construction | +249～+295 |
| Browser Broker | +297～+309 |
| Connection init | +310～+319 |
| connection.start | +334 |
| Resource sampler | +336～+340 |
| waitForClose block | +346 |
| Catch handler | +347～+356 |
| Finally cleanup | +357～+376 |

---

## 类：ZCodeProtocolAgentServer

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts`  
**所属：** class  
**职责：** V4 命令路由表持有者，负责接收请求、分发、回执生成  
**构造函数参数：** 见 `constructor(params)` at line ~+110

### 方法：handleMessage

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts`  
**函数：** handleMessage  
**偏移：** +406 ～ +462

**职责：** 解析 incoming NDJSON message → dispatchRequest → send response

**参数：**
```typescript
message: ZCodeProtocolMessage {
  id?: string, method: string, params: any, result?: any
}
```

**返回值：**
```typescript
Promise<ZCodeProtocolOutgoingMessage | undefined>
```

**内部逻辑：**

```text
1. Validate message schema → zcodeProtocolMessageSchema.parse(message)
   
2. if isNotification(message):
      handleNotification(message)
      return undefined
   
3. if isRequest(message):
      const result = await this.dispatchRequest(request)
      → build response frame
      → queuePostResponseMessages()
      
4. Take batch via takePostResponseBatch(requestId) for ordering
```

### 方法：dispatchRequest

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/server.ts`  
**函数：** dispatchRequest  
**偏移：** +458 ～ +570

**参数：**
```typescript
request: ZCodeProtocolRequest
```

**返回值：**
```typescript
Promise<any>  // 根据 method type 返回不同结果
```

**内部逻辑（V4 METHOD routing）：**

```text
switch (request.method) {
  // --- V4 Connection methods ---
  case V4_METHODS.connectionFlow:
    → handleConnectionFlow(envelope.params.envelope)
    
  // --- V4 Conversation subscribe/resync/unsubscribe ---
  case V4_METHODS.conversationSubscribe:
    → gateway.subscribeSessionsIndexReserved(params)
    → gateway.subscribeWorkspaceConfigReserved(params)
    → gateway.subscribeReserved(params)
    → commit initialWires
    
  case V4_METHODS.conversationResync:
    → requireV4Gateway().resyncReserved(params)
    
  case V4_METHODS.conversationUnsubscribe:
    → gateways.unsubscribeAll(subscriptionId)
    
  // --- V4 Conversation query methods (readonly) ---
  case V4_METHODS.conversationRowsRange:
  case V4_METHODS.conversationPlans:
  case V4_METHODS.backgroundBashOutput:
  case V4_METHODS.conversationFileChanges:
  case V4_METHODS.conversationWorkflowRunEvents:
  case V4_METHODS.conversationUsage:
    → route to gateway.query*() or read from projection
  
  // --- V4 Attachment streaming ---
  case V4_METHODS.attachmentBegin:
  case V4_METHODS.attachmentChunk:
  case V4_METHODS.attachmentCommit:
  case V4_METHODS.attachmentRead:
    → handleAttachmentOps(sessionId, ...)
    
  // --- Legacy protocol fallbacks ---
  case zcodeProtocolMethods.sessionCreate:
    → legacyHandler.sessionCreate(params)
  case zcodeProtocolMethods.sessionResume:
    → legacyHandler.sessionResume(params)
  case zcodeProtocolMethods.sessionSend:
    → legacyHandler.sessionSend(params)
  ...
}
```

**错误传播：**

```text
Method not found → ProtocolError.methodNotFound
Internal error → ProtocolError.internalServerError
Validation fail → ProtocolError.invalidParams
```

### 关键私有字段

```typescript
private readonly gateway: ConversationV4Gateway;   // V4 main dispatcher
private readonly sessionsManager: SessionsManager;   // session CRUD
private readonly commandHandlers: CommandHandlerMap; // legacy command table
private readonly notificationQueue: MessageQueue;     // outbound batching
```

### 调用关系

```text
dispatchRequest(request)
  ↓ (V4)
ConversationV4Gateway.handleV4Command(envelope)
  ↓ (handler lookup)
NATIVE_HANDLERS[type]  (from handlers/index.ts)
  ↓
handler(host, envelope) → Promise<CommandResult|undefined>
```

---

## 类：ZCodeProtocolNdjsonConnection

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/transport.ts`  
**所属：** class  
**职责：** stdin/stdout NDJSON 帧的 parse → encode 双向管道  
**构造函数参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| signal | AbortSignal | 生命周期信号 |
| handleMessage | ZCodeProtocolMessageHandler | 消息处理器回调 |
| input | NodeJS.ReadableStream | stdin |
| output | NodeJS.WritableStream | stdout |
| clearPostResponseMessages | () ⇒ void | post-response 消息清理 |
| takePostResponseBatch | (requestId) ⇒ ... | post-response 批量提取 |
| onTransportClosed | (error) ⇒ void | 连接关闭回调 |

### 方法：start

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/transport.ts`  
**函数：** start  
**偏移：** +110 ～ +180 (estimated)

**职责：** 启动 stdin→output 读取 loop，注册 stdout write buffer

**内部逻辑：**

```text
1. input.on('data', onData)
   → chunks accumulate into private buffer
   
2. processDataChunks(chunk)
   → split by newline ("\n")
   → JSON.parse each line
   → validate with zcodeProtocolMessageSchema
   → call handleMessage(parsedMessage)
   
3. output.write(responseFrame)
   → serialize response as NDJSON frame
   → flush after batch
```

### 方法：send

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/transport.ts`  
**函数：** send  
**偏移：** ~+180～+200

**参数：**
```typescript
message: ZCodeProtocolOutgoingMessage
```

**内部逻辑：**

```text
1. Build NDJSON frame:
   ```json
   {"id":"req-123","method":"...","result":{"status":"success"},null}
   ```
   
2. output.write(frame + "\n")
   
3. queueFlush()
```

### 方法：waitForClose

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol/transport.ts`  
**函数：** waitForClose  
**偏移：** ~+220～+240

**返回值：**
```typescript
Promise<void>  // resolves when stdin readable ended
```

**内部逻辑：**

```text
1. this.closedPromise = new Promise(resolve => { this.resolveClosed = resolve })
   
2. input.on('end', () => resolveClosed())
   
3. input.on('error', (error) => rejectClosed(error))
   
4. return this.closedPromise
```

### 关键私有字段

```typescript
private buffer = "";                  // partial lines accumulator
private processing: Promise<void>;    // sequential message processor
private lastQueuedMessageStarted: Promise<void>;  // backpressure control
private readonly closedPromise: Promise<void>;    // waitable promise
private terminal = false;             // once true → no more writes
private transportCloseNotified = false;
private draining = false;             // EOF drain state
```

### 数据格式规范

```
Incoming (stdin):
{"id":"1","method":"v4/command","params":{"envelope":{...}},"result":null}\n
{"id":"2","method":"v4/conversation/subscribe",...}\n

Outgoing (stdout):
{"id":"1","result":{"status":"success"},"null"}\n
{"method":"v4/notification","params":{...},...}\n  # notifications have no id
```

### 错误处理

```text
parse fail → skip line, log warning
schema validation fail → skip line, log warning
handleMessage throw → catch in processing chain, notify onTransportClosed
stdin EOF → resolve closedPromise → trigger cleanup
stdin error → reject closedPromise → propagate to caller
```

### 背压控制

```text
processing: Promise 保证消息串行处理（不会并发 handleMessage）
lastQueuedMessageStarted: 确保下一个 parse 只在上一批消息处理完后开始
draining mode: EOF 后不再接受新消息
```

---

## 函数：createZCodeApp (bootstrap factory)

**文件：** `apps/zcode-cli/packages/bootstrap/src/app/create-app.ts`  
**函数：** createZCodeApp  
**偏移：** ~+50～+200

**职责：** 构造 App instance（包含 Core Runtime、Input Facade 等），作为 ZCodeProtocolAgentServer 的参数注入

**参数：**
```typescript
appOptions: ZCodeAppOptions {
  sessionId: string,
  cwd: string,
  env: NodeJS.ProcessEnv,
  providerRegistry: ProviderRegistry,
  presentationSurface: PresentationSurface,
  mcpPortFactory?: () => McpPort,
  resolveEffectiveModelSelection?: (selection) => ...,
  ...
}
```

**返回值：**
```typescript
ZCodeApp {
  runtime: AgentRuntime,
  sendInput: InputFacade['sendInput'],
  subscribeEvents: EventHub.subscribe,
  ...
}
```

**内部逻辑：**

```text
1. Create ProviderRegistry (from options.providerRegistry)
2. Create ModelSelectionFacade
3. Create AgentRuntime (via core.RuntimeFactory.create)
4. Create InputFacade (wraps runtime methods)
5. Create SessionMapper (for legacy protocol compatibility)
6. Return { runtime, sendInput, subscribeEvents, ... }
```

### 调用关系

```text
createZCodeApp(opts)
  ↓
  RuntimeFactory.create(config)
      ↓
      AgentRuntime.constructor(deps)
  ↓
  InputFacade.constructor(runtime, sessionId, deps)
  ↓
  SessionMapper.constructor(store, runtime)
  ↓
  return app instance
```

---

[上一篇](06-配置与数据流.md) · [总目录](README.md) · [下一篇](08-函数级源码解析-CLI-V4-CommandExecutor.md)
