# 12 · 函数级源码解析：UI SessionPane（命令构造与下发）

> 上一篇：[11 · 函数级源码解析：Desktop Main + Host](./11-函数级源码解析-Desktop-Main-Host.md)
> 总目录：[README.md](./README.md)
> 下一篇：[13 · 函数级源码解析：其他关键模块](./13-函数级源码解析-其他关键模块.md)

本篇覆盖 **Renderer 侧用户点击"发送"后，命令信封的构造与下发**——主链路的起点（Layer 2/3 的 UI 端）：

```
Composer 输入
  → SessionPane.dispatchCommand   (renderer)   ◀── 本篇入口
    → createCommandEnvelope        (commandFactory.ts)
    → pendingCommandRegistry.record / lease.store.markCommandPending
    → sendCommand(envelope)        (agentConversationTransport.ts)
      → ensureHandshake → agentService.sendConversationCommandV4  (host, 见 doc 10)
```

涉及文件：

| 文件 | 角色 |
| --- | --- |
| `packages/ui/src/v4/SessionPane.tsx` | 会话面板，命令派发入口 `dispatchCommand` |
| `packages/ui/src/v4/commandFactory.ts` | renderer 侧 `createCommandEnvelope` 工厂 |
| `packages/ui/src/v4/agentConversationTransport.ts` | `ConversationTransport.sendCommand`（RPC 桥） |

---

## 12.1 `dispatchCommand`（SessionPane）

**符号定位**：`packages/ui/src/v4/SessionPane.tsx`
`dispatchCommand` 用 `useCallback` 定义于 **+1395**，签名 +1396~+1405，主体 +1406~+1480+（本篇分析至 catch 段 +1466~+1480+）。

**签名**：

```typescript
const dispatchCommand = useCallback(
  async (
    type: CommandType,
    payload: Record<string, unknown>,
    targetSessionId: string | null,
    baseRevision?: number,
    baseLogEpoch?: string,
    telemetrySeed?: ConversationPromptTelemetrySeed,
    onEnvelopeCreated?: (envelope: CommandEnvelope) => void,
    sessionCreateSource?: SessionCreateSource,
  ): Promise<CommandAck> => { /* ... */ },
  [onSessionCreated, sessionId],
);
```

**参数 / 返回值**：

- `type` / `payload`：V4 命令类型与载荷（如 `sendText` / `createSession`）。
- `targetSessionId`：目标 session（createSession 时为 null）。
- `baseRevision` / `baseLogEpoch`：CAS / 行定位幂等键（来自 UI 本地投影）。
- `telemetrySeed`：本地 TTFT 观测种子（仅 `localTtft` 非空且非远端时启用）。
- 返回 `Promise<CommandAck>`。

**关键内部逻辑（逐段）**：

1. **提交态采集**（见 +1406~+1409）：
   - `const submission = submissionConfigFromCommand(type, payload)`：从命令提取模型/权限等提交配置。
   - `captureComposerRecentSubmission(workspacePath, submission, workspaceIdentity)`：记录 Composer 最近提交（草稿恢复用）。
2. **选中模型采集**（见 +1410~+1413）：`captureAcceptedModelSelection(submission.modelSelection)`——仅当 `submission` 存在且目标 session 是当前 session 时。
3. **构造信封**（见 +1414~+1420）：
   ```typescript
   const envelope = createCommandEnvelope({
     type,
     sessionId: targetSessionId,
     payload: payload as never,
     ...(baseRevision !== undefined ? { baseRevision } : {}),
     ...(baseLogEpoch ? { baseLogEpoch } : {}),
   });
   ```
   `createCommandEnvelope` 见 12.2。
4. **信封回调**（见 +1421）：`onEnvelopeCreated?.(envelope)`——必须早于第一次上行，供调用方在 transport error / renderer refresh 后仍可查询。
5. **pending 注册**（见 +1423~+1443）：
   - `createSession` 时取 `groupedDraftTask`（见 +1423~+1427）随 `pendingCommandRegistry.record(envelope, {...})` 记录（见 +1428），用于恢复账本。
6. **挂起命令标记**（见 +1444~+1450）：若 `lease?.store` 存在且非 createSession，`lease.store.markCommandPending({ commandId, type, issuedAt })`——UI 恢复账本标记该命令 pending。
7. **TTFT 观测注入**（见 +1452~+1460）：若 `telemetrySeed?.localTtft` 且非远端 workspace，调 `getLocalTtftObserver().dispatch(...)` 把观测挂到 envelope。
8. **下发并收 ACK**（见 +1461~+1465）：
   ```typescript
   ack = await sendCommand(envelope);
   if (telemetrySeed?.localTtft && ack.reasonCode === "guard.heldQueueConfirmationStale")
     getLocalTtftObserver()?.confirmationRetry(...);
   else if (telemetrySeed?.localTtft)
     getLocalTtftObserver()?.ack(telemetrySeed.localTtft, ack.status, ack.ttftExcluded);
   ```
   `sendCommand` 是本地 `agentConversationTransport` 的 `sendCommand`（见 12.3）。
9. **失败处理**（见 +1466~+1480+）：
   - TTFT `exclude("failed")`（见 +1467~+1468）。
   - 若 `lease?.store`：`lease.store.settleCommand(envelope.commandId)`（见 +1469~+1471）。
   - **provider_not_ready 确定性拒绝**（见 +1472~+1477）：Host `getClient` 前拒绝，CLI 不可能已 admission；调 `pendingCommandRegistry.settle(envelope.sessionId, envelope.commandId)` 清恢复记录（即使 unknown 现静默清账，也不留无效记录）。
   - `recordV4CommandAck({ type, ... })`（见 +1478+）上报。

**调用关系**：

- 调用方：Composer（见 `packages/ui/src/v4/ConversationComposer.tsx`）及 SessionPane 内各交互（stop / resolveInteraction / CAS 配置写等）。
- 被调：`createCommandEnvelope`（12.2）、`submissionConfigFromCommand`、`captureComposerRecentSubmission`、`captureAcceptedModelSelection`、`pendingCommandRegistry.record`、`sendCommand`（12.3）。

---

## 12.2 `createCommandEnvelope`（renderer 工厂）

**符号定位**：`packages/ui/src/v4/commandFactory.ts` +51。
（与 host 侧 `createHostCommandEnvelope` 平行，见 doc 10 §10.4。）

**签名**：

```typescript
export function createCommandEnvelope<T extends CommandType>(
  input: CreateCommandEnvelopeInput<T>,
): CommandEnvelope
```

`CreateCommandEnvelopeInput<T>`（见 +41~+48）：`type`、`payload`、`sessionId`（createSession 为 null）、`baseRevision?`、`baseLogEpoch?`。

**关键内部逻辑**：

1. **clientId 稳定化**（见 `getV4ClientId` +21~+39）：优先 `localStorage["zcode-v4-client-id:v1"]` 持久化；无 storage（incognito / node 测试）退化为进程内 `cachedClientId`。刷新后 clientId 不变 → 幂等表跨刷新识别重试。
2. **CAS 守卫**（见 +54~+56）：`COMMANDS_REQUIRING_BASE_REVISION` 缺 `baseRevision` 直接抛（客户端编程错误就地暴露）。
3. **行定位守卫**（见 +57~+59）：`ROW_TARGETING_COMMANDS` 缺 `baseLogEpoch` 直接抛。
4. **组装**（见 +60~+69）：`commandId: uuidv7()`（时间有序，重试不变）、`clientId: getV4ClientId()`、`issuedAt: Date.now()`。

**与 host 工厂差异**：renderer 走 `localStorage` 持久化 clientId；host 走进程内稳定 id（host 不能被 ui 反向依赖，故两份实现，见 `zcodeV4HostCommand.ts` +1~+5 注释）。

---

## 12.3 `ConversationTransport.sendCommand`（RPC 桥）

**符号定位**：`packages/ui/src/v4/agentConversationTransport.ts`
`sendCommand` 方法定义于 **+350**，主体 +350~+367。

**签名**：

```typescript
async sendCommand(envelope: CommandEnvelope): Promise<CommandAck>
```

**关键内部逻辑**：

1. **握手**（见 +351）：`const hello = await ensureHandshake()`——确保与 Host 的 RPC 通道已建立（首次调用触发 handshake，缓存后续复用）。
2. **Plan 能力探测**（见 +352~+363）：
   ```typescript
   const payload = envelope.payload as { planEnabled?; config?; firstInput? };
   if (
     hello.capabilities.independentPlanState !== true &&
     (payload.planEnabled || payload.config?.planEnabled || payload.firstInput?.planEnabled)
   ) {
     throw new Error("proto.independentPlanUnsupported");
   }
   ```
   旧 Host 会剥未知字段，不能把 yolo + Plan 错发成完全访问执行——能力不满足直接抛。
3. **下发**（见 +364~+366）：
   ```typescript
   return sendWithConversationDelayE2E(() =>
     agentService.sendConversationCommandV4({ ...workspace, envelope }),
   );
   ```
   `agentService` 是注入的 `IZCodeAgentService` 代理（renderer→Host RPC）；`sendWithConversationDelayE2E` 包裹端到端延迟观测。`workspace` 含 `workspacePath` / `workspaceIdentity?` / `remoteSessionId?`，合并进调用参数。

**调用关系**：

- 调用方：`SessionPane.dispatchCommand`（12.1）、`queryCommands`（+368）、`rowsRange`（+375）、`plans`（+384）等同一 transport 的只读方法。
- 被调：`ensureHandshake`、`agentService.sendConversationCommandV4`（→ host，见 doc 10 §10.3）。

---

## 12.4 数据流小结（一次 UI sendText 的字段行程）

```
Composer 提交
  → dispatchCommand(type="sendText", payload={text,...})
    submissionConfigFromCommand → captureComposerRecentSubmission / captureAcceptedModelSelection
    createCommandEnvelope
      commandId = uuidv7()
      clientId = getV4ClientId()  (localStorage 持久化)
      sessionId = targetSessionId
      payload = {text, modelSelection, toolDisallowlist? ...}
    pendingCommandRegistry.record(envelope)        // 恢复账本
    lease.store.markCommandPending(commandId)      // UI pending 标记
    ── sendCommand(envelope) ──
      ensureHandshake → hello.capabilities.independentPlanState 检查
      agentService.sendConversationCommandV4({ ...workspace, envelope })
        → host: buildConversationCommandEnvelope → client.request(V4_METHODS.command)
    ← CommandAck
    lease.store.settleCommand(commandId) / pendingCommandRegistry.settle
```

**诚实声明**：`ensureHandshake` 的具体 handshake 协议（hello capabilities 来源、`sendWithConversationDelayE2E` 的延迟观测实现）当前未在本篇逐行核对，需另读 `agentConversationTransport.ts` 的 handshake 段与 `conversationDelayE2E` 包装确认。`pendingCommandRegistry` / `lease.store` 的账本与 message store 收口细节见 doc 04（核心模块与类关系）相关 store 描述。

---

> 上一篇：[11 · 函数级源码解析：Desktop Main + Host](./11-函数级源码解析-Desktop-Main-Host.md)
> 总目录：[README.md](./README.md)
> 下一篇：[13 · 函数级源码解析：其他关键模块](./13-函数级源码解析-其他关键模块.md)
