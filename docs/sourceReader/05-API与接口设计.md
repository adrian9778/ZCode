[上一篇](04-核心模块与类关系.md) · [总目录](README.md) · [下一篇](06-配置与数据流.md)

# API 与接口设计——V4 协议 + 旧协议方法表 + 类型契约全景

> **场景**：提供所有跨进程接口的完整方法表、参数类型、返回值 schema。
> **层级**：第四层——接口级规范文档
> **版本**：v1 (2026-09-23)
> **源码基准**：commit `872ad96`

---

## 1. V4 Protocol Methods

**文件：** `packages/shared/src/zcode-protocol-v4/transport.ts`  
**偏移：** +307～+426

### Const：V4_METHODS

```typescript
export const V4_METHODS = {
  // Connection lifecycle
  connectionFlow: "v4/connection/flow",
  controllerSubscribe: "v4/controller/subscribe",
  controllerResync: "v4/controller/resync",
  controllerUnsubscribe: "v4/controller/unsubscribe",
  
  // Conversation subscription (realtime event stream)
  conversationSubscribe: "v4/conversation/subscribe",
  conversationResync: "v4/conversation/resync",
  conversationUnsubscribe: "v4/conversation/unsubscribe",
  
  // Conversation query (readonly, stateless)
  conversationRowsRange: "v4/conversation/rowsRange",          // line-based pagination
  conversationPlans: "v4/conversation/plans",                  // active branch exit plans
  conversationFileChanges: "v4/conversation/fileChanges",      // file diff tracking
  backgroundBashOutput: "v4/conversation/backgroundBashOutput", // bash output stream
  conversationFileRewindPreview: "v4/conversation/fileRewindPreview",
  
  // Workflow run event logs (audit trail)
  conversationWorkflowRunEvents: "v4/conversation/workflowRunEvents",
  conversationWorkflowRuns: "v4/conversation/workflowRuns",
  conversationWorkflowRunArtifacts: "v4/conversation/workflowRunArtifacts",
  conversationWorkflowRunArtifactData: "v4/conversation/workflowRunArtifactData",
  conversationWorkflowRunArtifactRead: "v4/conversation/workflowRunArtifactRead",
  conversationWorkflowRunWorkspace: "v4/conversation/workflowRunWorkspace",
  conversationWorkflowRunNodeResult: "v4/conversation/workflowRunNodeResult",
  
  // Usage stats (query-only, timeout-retry safe)
  usageStats: "v4/usage/stats",
  conversationUsage: "v4/conversation/usage",
  
  // Attachment streaming (chunk ≤ 512 KiB)
  attachmentBegin: "v4/attachment/begin",
  attachmentChunk: "v4/attachment/chunk",
  attachmentCommit: "v4/attachment/commit",
  attachmentAbort: "v4/attachment/abort",
  attachmentRead: "v4/attachment/read",
  conversationAttachmentRead: "v4/conversation/attachmentRead",
  conversationAttachmentStat: "v4/conversation/attachmentStat",
  attachmentPreviewSource: "v4/attachment/previewSource",
  
  // Command execution & query (input routing, queue management)
  commandsQuery: "v4/commands/query",
  command: "v4/command",                                       // PRIMARY INPUT CHANNEL
} as const;
```

### Const：V4_NOTIFICATIONS

```typescript
export const V4_NOTIFICATIONS = {
  v4NotificationMethod: "v4/notification",                    // generic notification
  v4PostResponseMessages: "v4/post-response-messages",        // post-response outbox
  ...
}
```

### 命令载荷类型：CommandPayloadMap

**文件：** `packages/shared/src/zcode-protocol-v4/input-intent.ts`  

```typescript
// Key commands and their payload schemas
type CommandPayloadMap = {
  sendText: {
    text: string;
    modelSelection?: ModelSelection;
    modelExecution?: ModelExecutionScope;
    attachments?: AttachmentRef[];
    context_refs?: SharedContextRef[];
    toolDisallowlist?: readonly string[];
    requestedDelivery?: "startNow" | "queue" | "guide";
    heldQueueDisposition?: "drainAllAndSend" | "ignoreQueueAndSend";
    expectedHeldQueueItemIds?: readonly string[];
    browserAmbientContext?: BrowserAmbientContext;
  };
  stop: {
    expectedForegroundExecutionId?: string;
  };
  createSession: {
    modelSelection?: ModelSelection;
    firstInput?: Omit<sendText.payload, "heldQueueDisposition">;
  };
  resumeSession: {
    sessionId: string;
  };
  deleteSession: {
    sessionId: string;
  };
  compact: {
    sessionId: string;
    strategy?: "summary" | "microcompact";
  };
  switchModel: {
    selection: ModelSelection;
  };
  setThoughtLevel: {
    thoughtLevel: "none" | "brief" | "full";
  };
  forkAssistant: {
    sourceMessageId: MessageId;
    reason?: string;
  };
  editUserQuery: {
    messageId: MessageId;
    newText: string;
  };
  retryTurn: {
    turnId: TurnId;
  };
  pauseGoal: {
    sessionId: string;
  };
  resumeGoal: {
    sessionId: string;
  };
  discardSharedContext: {
    ref: SharedContextRef;
  };
};
```

### 回执类型：CommandAck

**偏移：** shared/zcode-protocol-v4/transport.ts ~+280

```typescript
export const commandAckSchema = z.object({
  status: z.enum(["success", "error", "pending"]),
  delivery: z.enum(["startNow", "queue"]).optional(),
  inputId: z.string().uuid().optional(),
  reasonCode: z.string().optional(),           // error reason
  message: z.string().optional(),              // human-readable error
  ttft: z.number().int().positive().optional(), // time-to-first-token (ms)
  memoryEnabled: z.boolean().default(false).optional(),
});
```

---

## 2. Legacy Protocol Methods

**文件：** `packages/shared/src/zcode-protocol/index.ts`  
**偏移：** +3560～+3664

### Const：zcodeProtocolMethods

```typescript
export const zcodeProtocolMethods = {
  runtimeCapabilities: "runtime/capabilities",
  computerUseOperationEvent: "computer-use/operation-event",
  
  // Session CRUD
  sessionCreate: "session/create",
  sessionResume: "session/resume",
  sessionList: "session/list",
  sessionSubagents: "session/subagents",
  sessionRequestRuntimePreferences: "session/requestRuntimePreferences",
  sessionRead: "session/read",
  sessionMessages: "session/messages",
  sessionEvents: "session/events",
  sessionDebug: "session/debug",
  sessionSubscribe: "session/subscribe",
  sessionSend: "session/send",                              // @deprecated → v4/sendText
  sessionStop: "session/stop",                              // @deprecated → v4/stop
  sessionCancelBackgroundTask: "session/cancelBackgroundTask",
  sessionFork: "session/fork",
  sessionCompact: "session/compact",
  sessionGoal: "session/goal",
  sessionClose: "session/close",
  sessionSetModel: "session/setModel",                      // @deprecated (partially)
  sessionSetThoughtLevel: "session/setThoughtLevel",        // @deprecated (partially)
  sessionSetMode: "session/setMode",                        // @deprecated (partially)
  
  // Workspace settings
  workspaceReadPresentation: "workspace/readPresentation",
  workspaceHookTrustGrant: "workspace/hooks/trustGrant",
  providerUpdateAccountConfig: "provider/updateAccountConfig",
  workspaceUpdateInteractionPreferences: "workspace/updateInteractionPreferences",
  workspaceUpdateModelIoPreferences: "workspace/updateModelIoPreferences",
  workspaceUpdateOffPeakToolPolicy: "workspace/updateOffPeakToolPolicy",
  workspaceUpdateDynamicWorkflowPolicy: "workspace/updateDynamicWorkflowPolicy",
  workspaceGenerateText: "workspace/generateText",
  workspaceCancelGenerateText: "workspace/cancelGenerateText",
  
  // Provider connectivity
  providerTestModelConnectivity: "provider/testModelConnectivity",
  
  // MCP
  mcpList: "mcp/list",
  
  // Plugins & Skills
  pluginsList: "plugins/list",
  pluginsReferenceCatalog: "plugins/referenceCatalog",
  pluginsReferenceCatalogWithCategory: "plugins/referenceCatalogWithCategory",
  skillsReferenceCatalog: "skills/referenceCatalog",
  
  // Workflows
  workflowsList: "workflows/list",
  workflowsGet: "workflows/get",
  workflowsUpdateMeta: "workflows/updateMeta",
  workflowsDelete: "workflows/delete",
  workflowsRuns: "workflows/runs",
  workflowsMove: "workflows/move",
  pluginsResolveSuggestedReference: "plugins/resolveSuggestedReference",
  pluginsSetEnabled: "plugins/setEnabled",
  pluginsOverview: "plugins/overview",
  pluginsMarketplaceAdd: "plugins/marketplace/add",
  pluginsMarketplaceRemove: "plugins/marketplace/remove",
  pluginsMarketplaceUpdate: "plugins/marketplace/update",
  pluginsInstall: "plugins/install",
  pluginsCancelOperation: "plugins/cancelOperation",
  pluginsUninstall: "plugins/uninstall",
  pluginsUpdate: "plugins/update",
  pluginsRestoreBuiltin: "plugins/restoreBuiltin",
  pluginsConfigure: "plugins/configure",
  pluginsResetConfig: "plugins/resetConfig",
  pluginsValidate: "plugins/validate",
  pluginsDescribe: "plugins/describe",
  
  // Automation
  automationCreate: "automation/create",
  automationUpdate: "automation/update",
  automationCheckTaskBinding: "automation/checkTaskBinding",
  automationList: "automation/list",
  automationDelete: "automation/delete",
  
  // Off-Peak tasks
  offPeakCreate: "offPeak/create",
  offPeakList: "offPeak/list",
  
  // Deprecated/legacy
  usageStats: "usage/stats",                                // @deprecated
  sessionUsage: "session/usage",                            // @deprecated
  
  // Resource management
  processChildProcesses: "process/childProcesses",
  
  // Interaction broker
  interactionRequestPermission: "interaction/requestPermission",
  interactionRequestUserInput: "interaction/requestUserInput",
  interactionRequestProviderRuntimeHeaders: "interaction/requestProviderRuntimeHeaders",
  interactionRequestOfficialMcpAuthHeaders: "interaction/requestOfficialMcpAuthHeaders",
  interactionBrowserList: "interaction/browserList",
  interactionBrowserExecute: "interaction/browserExecute",
} as const;
```

### Contract Schemas (示例)

```typescript
export const zcodeProtocolSessionMethodContracts = {
  [zcodeProtocolMethods.workspaceHookTrustGrant]: {
    params: zcodeWorkspaceHookTrustGrantParamsSchema,
    result: zcodeWorkspaceHookTrustGrantResultSchema,
  },
  [zcodeProtocolMethods.mcpList]: {
    params: zcodeMcpListParamsSchema,
    result: zcodeMcpListResultSchema,
  },
  [zcodeProtocolMethods.interactionBrowserList]: {
    params: zcodeBrowserListParamsSchema,
    result: zcodeBrowserListResultSchema,
  },
  [zcodeProtocolMethods.interactionBrowserExecute]: {
    params: zcodeBrowserExecuteParamsSchema,
    result: zcodeBrowserExecuteResultSchema,
  },
} as const satisfies Partial<...>;
```

---

## 3. Host ↔ Agent CLI 通信协议

### NDJSON Frame Format

每个帧是一行 JSON，格式如下：

**Request（Host → Agent）:**
```json
{"id":"req-123","method":"v4/command","params":{"envelope":{...}},"result":null}
```

**Response（Agent → Host）:**
```json
{"id":"req-123","result":{"status":"success","delivery":"startNow"},"result":null}
```

**Notification（bidirectional push）:**
```json
{"method":"v4/notification","params":{"sessionId":"sess-1","event":{"type":"turn-started"}}}
```

### 协议版本标识

| 版本 | 前缀 | 状态 |
|------|------|------|
| V4 | `v4/*` | 主推 |
| Legacy session | `session/*` | @deprecated |
| Legacy workspace | `workspace/*` | 部分保留 |
| Legacy plugin/automation | `plugins/*`, `automation/*` | 保留（非会话路径） |

---

## 4. ZCodeApp Interface (bootstrap ↔ core bridge)

**文件：** `apps/zcode-cli/packages/bootstrap/src/app/types.ts`  
**偏移：** +600+

```typescript
export interface ZCodeAppOptions {
  sessionId: string;
  cwd: string;
  env: NodeJS.ProcessEnv;
  providerRegistry: ProviderRegistry;
  presentationSurface?: PresentationSurface;
  mcpPortFactory?: () => McpPort;
  resolveEffectiveModelSelection?: (selection: ModelSelection) => SelectionResult;
  onToolExecResource?: (params: ToolExecResourceParams) => void;
  // ... more fields
}

export interface ZCodeApp {
  runtime: AgentRuntime;
  sendInput(input: PromptInput, options?: SendInputOptions): Promise<SendInputResult>;
  subscribeEvents(handler: EventSubscriptionHandler): UnsubscribeFn;
  continueActiveTargetLoop(options: ContinueActiveTargetOptions): Promise<void>;
  stopActiveForegroundExecution(options: StopForegroundExecutionOptions): StopExecutionResult;
  enqueueDeferredInput(params: EnqueueDeferredInputParams): EnqueueResult;
}
```

---

## 5. ConversationTransport Interface (UI ↔ Service)

**文件：** `packages/ui/src/v4/transport.ts`  
**偏移：** +58+

```typescript
export interface ConversationTransport {
  /** Connect and handshake with V4 server */
  connect(): Promise<void>;
  
  /** Subscribe to conversation event stream */
  subscribeConversation(subscriptionId: string, topic: Topic): Promise<SubscribeAck>;
  
  /** Unsubscribe from conversation events */
  unsubscribe(subscriptionId: string): Promise<void>;
  
  /** Send a command (createSession, sendText, etc.) */
  sendCommand(envelope: CommandEnvelope): Promise<CommandAck>;
  
  /** Query pending/queued commands */
  queryCommands(params: CommandsQueryParams): Promise<CommandsQueryResult>;
  
  /** Fetch conversation rows (pagination) */
  rowsRange(params: V4ConversationRowsRangeParams): Promise<V4ConversationRowsRangeResult>;
  
  /** Get valid plan options for current branch */
  plans(params: V4ConversationPlansParams): Promise<V4ConversationPlansResult>;
  
  /** File changes in active turn */
  fileChanges(params: V4ConversationFileChangesParams): Promise<V4ConversationFileChangesResult>;
  
  // Plus workflow-related query methods...
}
```

---

## 6. V4CommandCoreHost Interface (handler ↔ core bridge)

**文件：** `apps/zcode-cli/packages/bootstrap/src/zcode-protocol-v4/commands/types.ts`  
**偏移：** +40～+120

```typescript
export interface V4CommandCoreHost {
  getRecord(sessionId: string): V4SessionRecordView | undefined;
  ensureModelReady?(record: V4SessionRecordView): Promise<void>;
  hasUsableRuntimeModelTarget?(record: V4SessionRecordView): boolean;
  afterLegacyStateMutation?(record: V4SessionRecordView, mutationReason: string): Promise<void>;
  getInputRoutingMode?(sessionId: string): RoutingMode | null;
  logger?: LoggerInterface;
}
```

---

## 7. ZCodeAgentService Interface (Host service API)

**文件：** `packages/services/src/zcode-agent/zcodeAgent.ts`  
**偏移：** +600~+800

```typescript
export interface IZCodeAgentService {
  // Core operations
  sendPrompt(target: TaskKeyTarget, params: SendPromptParams): Promise<void>;
  sendConversationCommandV4(params: ZCodeAgentConversationCommandParams): Promise<CommandAck>;
  queryConversationCommandsV4(params: ZCodeAgentCommandsQueryParams): Promise<CommandsQueryResult>;
  
  // Subscription ops
  subscribeConversationV4(params: SubscribeV4Params): Promise<SubscribeAck>;
  unsubscribeConversationV4(params: UnsubscribeV4Params): Promise<void>;
  
  // Query ops
  conversationRowsRangeV4(params: RowsRangeV4Params): Promise<RowsRangeResult>;
  conversationPlansV4(params: PlansV4Params): Promise<PlansResult>;
  conversationFileChangesV4(params: FileChangesV4Params): Promise<FileChangesResult>;
  
  // Status / config
  getV4Capability(hostProcessLabel: string): Promise<V4Capabilities>;
  
  // Lifecycle
  disconnect(target: Target): Promise<void>;
  
  // Plugin management
  pluginsList(...): Promise<PluginsListResult>;
  pluginsInstall(...): Promise<PluginInstallResult>;
  // ... many more plugin/automation/MCP methods
}
```

---

## 8. Controller Topics (UI ↔ Desktop Bridge)

**文件：** `packages/shared/src/zcode-protocol-v4/controller.ts`  
**偏移：** +8~+15

```typescript
export const CONTROLLER_WORKSPACES_TOPIC = "controller/workspaces" as const;
export const CONTROLLER_TASKS_INDEX_TOPIC = "controller/tasks-index" as const;

export type WindowHostControllerTopic = 
  | typeof CONTROLLER_WORKSPACES_TOPIC
  | typeof CONTROLLER_TASKS_INDEX_TOPIC;
```

---

## 9. Error Reason Codes Registry

### V4 Input Admission Rejections

| reasonCode | Source | 含义 |
|------------|--------|------|
| `proto.invalidPayload` | sendText handler | 输入为空 |
| `fault.command.inputRejected` | sendText handler | 前台调度繁忙 |
| `activePrompt` | prompt-turn | admission rejected by Core |
| `restoreWarning` | prompt-turn | 模型恢复警告未清除 |
| `guard.heldQueueConfirmationStale` | UI pendingCommandReplay | 队列项已过期 |
| `guard.selectionSideChatRestrictedCommand` | executor.ts | selection_side_chat 不允许该操作 |
| `proto.independentPlanUnsupported` | agentConversationTransport | 不支持 Plan 独立模式 |

---

[上一篇](04-核心模块与类关系.md) · [总目录](README.md) · [下一篇](06-配置与数据流.md)
