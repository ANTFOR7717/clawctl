# Audit 2: Mastra Gateway Integration — Deep-Dive on 7 Critical Blockers

**Audited plan:** [`plans/mastra-gateway-integration.md`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/plans/mastra-gateway-integration.md)
**Prior audit:** [`plans/mastra-gateway-audit.md`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/plans/mastra-gateway-audit.md)
**Audit date:** 2026-02-27
**Scope:** Expanded technical analysis of each of the 7 critical blockers identified in Audit 1, with concrete resolution specifications for each.

---

## Purpose of This Document

Audit 1 correctly identified 7 critical blockers. This document expands each blocker with:

1. **Exact code evidence** — the specific lines that prove the blocker is real
2. **Why the plan's proposed fix is insufficient** — what the plan says vs. what the code actually requires
3. **A concrete resolution specification** — the exact interface, function signature, or architectural change needed to unblock implementation

---

## Blocker 1 — `subscribeEmbeddedPiSession` Cannot Be Reused Unchanged

### What the plan claims

Section 8 of the integration plan states:

> "The Mastra path emits the same events as the pi path so `subscribeEmbeddedPiSession` and all its callers are **unchanged**."

Section 4.5 describes `stream-subscriber.ts` as translating Mastra `fullStream` chunks into `AgentEvent` objects, implying `subscribeEmbeddedPiSession` consumes those events.

### What the code actually does

[`src/agents/pi-embedded-subscribe.ts:630`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-subscribe.ts:630):

```typescript
const sessionUnsubscribe = params.session.subscribe(createEmbeddedPiSessionEventHandler(ctx));
```

[`src/agents/pi-embedded-subscribe.ts:654-662`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-subscribe.ts:654):

```typescript
if (params.session.isCompacting) {
  log.debug(`unsubscribe: aborting in-flight compaction runId=${params.runId}`);
  try {
    params.session.abortCompaction();
  } catch (err) { ... }
}
sessionUnsubscribe();
```

[`src/agents/pi-embedded-runner/run/attempt.ts:989-993`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/run/attempt.ts:989):

```typescript
queueMessage: async (text: string) => {
  await activeSession.steer(text);
},
isStreaming: () => activeSession.isStreaming,
isCompacting: () => subscription.isCompacting(),
```

[`src/agents/pi-embedded-runner/run/attempt.ts:1199-1202`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/run/attempt.ts:1199):

```typescript
await abortable(activeSession.prompt(effectivePrompt, { images: imageResult.images }));
// OR
await abortable(activeSession.prompt(effectivePrompt));
```

[`src/agents/pi-embedded-subscribe.types.ts:11`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-subscribe.types.ts:11):

```typescript
export type SubscribeEmbeddedPiSessionParams = {
  session: AgentSession;  // from @mariozechner/pi-coding-agent
  ...
};
```

### The full `AgentSession` surface used by `subscribeEmbeddedPiSession`

The function uses these `AgentSession` members directly:

| Member | Usage location | Purpose |
|---|---|---|
| `session.subscribe(handler)` | `pi-embedded-subscribe.ts:630` | Registers event handler for ALL session events |
| `session.isCompacting` | `pi-embedded-subscribe.ts:654` | Checked during unsubscribe to abort in-flight compaction |
| `session.abortCompaction()` | `pi-embedded-subscribe.ts:657` | Cancels in-flight compaction on unsubscribe |
| `session.isStreaming` | `attempt.ts:992` | Exposed via `queueHandle.isStreaming()` |
| `session.steer(text)` | `attempt.ts:990` | Mid-run message injection (queue handle) |
| `session.prompt(text, opts?)` | `attempt.ts:1199-1202` | Drives the actual LLM call |
| `session.messages` | `attempt.ts:1217` | Snapshot of conversation history |
| `session.abort()` | `attempt.ts:923` | Cancels the current run |
| `session.sessionId` | `attempt.ts:996` | Used for run tracking |
| `session.dispose()` | `compact.ts:732` | Cleanup after compaction |
| `session.agent.replaceMessages()` | `compact.ts:618` | Replaces history before compaction |

### Why the plan's `stream-subscriber.ts` approach is wrong

The plan describes `stream-subscriber.ts` as translating Mastra `fullStream` chunks into `AgentEvent` objects. But `subscribeEmbeddedPiSession` does **not** consume `AgentEvent` objects — it calls `params.session.subscribe(handler)` which registers a callback on the `AgentSession` event emitter. The session then drives the entire lifecycle: it calls `session.prompt()` to start the LLM call, receives events via the subscription, and manages compaction internally.

There is no way to feed Mastra stream chunks into `subscribeEmbeddedPiSession` without providing a full `AgentSession`-compatible object.

### Concrete resolution

**Option A (recommended): Write `subscribeMastraSession()`**

A new function with the same return contract as `subscribeEmbeddedPiSession`:

```typescript
// src/agents/mastra/subscribe-mastra-session.ts

export type SubscribeMastraSessionParams = Omit<SubscribeEmbeddedPiSessionParams, "session"> & {
  // Mastra-specific: the agent runner provides these callbacks instead of AgentSession
  onPrompt: (text: string, opts?: { images?: unknown[] }) => Promise<void>;
  onSteer: (text: string) => Promise<void>;
  onAbort: () => void;
  getIsStreaming: () => boolean;
  getMessages: () => AgentMessage[];
  // Event feed: caller pushes events into the subscriber
  eventFeed: MastraEventFeed;
};

export type MastraEventFeed = {
  push: (event: MastraSessionEvent) => void;
  complete: () => void;
  error: (err: unknown) => void;
};

export type MastraSessionEvent =
  | { type: "text_delta"; text: string }
  | { type: "text_end"; text: string }
  | { type: "message_end"; usage?: UsageLike }
  | { type: "tool_execution_start"; toolName: string; toolCallId: string; input: unknown }
  | { type: "tool_execution_end"; toolName: string; toolCallId: string; result: unknown; meta?: string }
  | { type: "compaction_start" }
  | { type: "compaction_end"; retrying: boolean }
  | { type: "error"; error: unknown };

export function subscribeMastraSession(
  params: SubscribeMastraSessionParams,
): ReturnType<typeof subscribeEmbeddedPiSession> {
  // Implements the same state machine as subscribeEmbeddedPiSession
  // but driven by MastraSessionEvent pushes instead of AgentSession.subscribe()
  // Returns the same shape: { assistantTexts, toolMetas, unsubscribe, isCompacting,
  //   waitForCompactionRetry, getMessagingToolSentTexts, ... }
}
```

The `agent-runner.ts` pushes events into `eventFeed` as it consumes `output.fullStream`. The `unsubscribe()` implementation calls `onAbort()` instead of `session.abortCompaction()`.

**Option B (not recommended): `AgentSession` shim**

Wrap Mastra's `Agent` in an object that satisfies the `AgentSession` interface. This is fragile because `AgentSession` is a concrete class with internal state (event emitter, compaction lock, etc.) that cannot be faithfully shimmed without reimplementing the entire pi-coding-agent session lifecycle.

**Decision required in the plan:** Choose Option A and specify the full `MastraSessionEvent` type mapping from Mastra `fullStream` chunk types.

---

## Blocker 2 — `EmbeddedRunAttemptResult.lastAssistant` Is Typed as `AssistantMessage` from `@mariozechner/pi-ai`

### What the plan claims

The plan is silent on this field. Section 4.5 (`agent-runner.ts`) returns `{ text, toolCalls, toolResults }` — no `lastAssistant`.

### What the code actually requires

[`src/agents/pi-embedded-runner/run/types.ts:37`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/run/types.ts:37):

```typescript
import type { AssistantMessage } from "@mariozechner/pi-ai";

export type EmbeddedRunAttemptResult = {
  // ...
  lastAssistant: AssistantMessage | undefined;
  // ...
};
```

`AssistantMessage` from `@mariozechner/pi-ai` is a typed content-block message:

```typescript
// From @mariozechner/pi-ai (inferred from usage)
type AssistantMessage = {
  role: "assistant";
  content: Array<
    | { type: "text"; text: string }
    | { type: "tool_use"; id: string; name: string; input: unknown }
    | { type: "thinking"; thinking: string }
  >;
};
```

### Where `lastAssistant` is consumed

The field is used by callers of `runEmbeddedAttempt` for:
1. **Error classification** — detecting `isCloudCodeAssistFormatError` from the last assistant message content
2. **Retry logic** — checking if the last response was a tool call or text to decide retry strategy
3. **Transcript repair** — `sanitizeToolUseResultPairing` uses message content shapes

### Why this is a TypeScript compilation blocker

The Mastra path in `attempt.ts` must return an `EmbeddedRunAttemptResult`. If `lastAssistant` is not populated with a valid `AssistantMessage`, the TypeScript compiler will reject the return statement. The plan's `MastraRunResult` (`{ text, toolCalls, toolResults }`) has no `AssistantMessage` field.

### Concrete resolution

**Option A (recommended): Construct a compatible `AssistantMessage` from Mastra output**

```typescript
// In agent-runner.ts, after stream completes:
function buildLastAssistantFromMastra(params: {
  text: string;
  toolCalls: ToolCallChunk[];
}): AssistantMessage {
  const content: AssistantMessage["content"] = [];
  if (params.text) {
    content.push({ type: "text", text: params.text });
  }
  for (const tc of params.toolCalls) {
    content.push({
      type: "tool_use",
      id: tc.toolCallId,
      name: tc.toolName,
      input: tc.args,
    });
  }
  return { role: "assistant", content };
}
```

This requires importing `AssistantMessage` from `@mariozechner/pi-ai` in the Mastra adapter — which means `@mariozechner/pi-ai` cannot be removed in Phase 2 until `lastAssistant` is either removed from the return type or the type is changed.

**Option B: Change the return type**

```typescript
// In types.ts:
import type { AssistantMessage } from "@mariozechner/pi-ai";

export type EmbeddedRunAttemptResult = {
  // ...
  lastAssistant: AssistantMessage | MastraAssistantMessage | undefined;
  // ...
};

type MastraAssistantMessage = {
  role: "assistant";
  content: Array<{ type: "text"; text: string } | { type: "tool_use"; id: string; name: string; input: unknown }>;
  _source: "mastra";
};
```

All consumers of `lastAssistant` must then be updated to handle the union type. This is a larger change but enables full pi-ai removal in Phase 2.

**Decision required in the plan:** Choose Option A for Phase 1 (minimal change, keeps pi-ai as a type dependency), plan Option B for Phase 4 (pi-ai removal).

---

## Blocker 3 — `EmbeddedRunAttemptParams.model` Is Typed as `Model<Api>` from `@mariozechner/pi-ai`

### What the plan claims

Section 4.4 shows `toMastraModelConfig()` taking `modelApi`, `baseUrl`, `apiKey` as separate parameters. Section 6.2 shows the Mastra branch receiving `params.model` — but does not show how to extract these fields from `Model<Api>`.

### What the code actually requires

[`src/agents/pi-embedded-runner/run/types.ts:19`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/run/types.ts:19):

```typescript
import type { Api, Model } from "@mariozechner/pi-ai";

export type EmbeddedRunAttemptParams = EmbeddedRunAttemptBase & {
  provider: string;
  modelId: string;
  model: Model<Api>;          // ← pi-ai internal type
  authStorage: AuthStorage;   // ← pi-coding-agent type
  modelRegistry: ModelRegistry; // ← pi-coding-agent type
  thinkLevel: ThinkLevel;
  // ...
};
```

### The `Model<Api>` structure

`Model<Api>` from `@mariozechner/pi-ai` is a generic type parameterized by `Api` (the API format enum). Its internal structure (inferred from usage in `attempt.ts` and `compact.ts`) includes:

```typescript
// Inferred from usage in attempt.ts and extensions.ts
type Model<A extends Api> = {
  api: A;                    // e.g. "anthropic-messages", "openai-completions"
  contextWindow?: number;    // used in extensions.ts:25
  // Additional fields: baseUrl, apiKey, headers — not directly accessed in attempt.ts
  // These are accessed via authStorage/modelRegistry, not directly from model
};
```

### The critical insight the plan misses

In `attempt.ts`, the `model` object is **not** the source of `baseUrl` and `apiKey`. Those come from `authStorage` (an `AuthStorage` from pi-coding-agent) and are resolved via `resolveModelAuthMode()`. The `model` object provides `model.api` (the API format) and `model.contextWindow` (for extension factories).

The plan's `toMastraModelConfig()` signature assumes `baseUrl` and `apiKey` are passed in — but the Mastra branch in `attempt.ts` must resolve them from `authStorage` first, using the same `resolveModelAuthMode()` / `getApiKeyForModel()` logic that the pi path uses.

### Concrete resolution

The Mastra branch in `attempt.ts` must:

1. **Extract `model.api`** for the provider format switch in `toMastraModelConfig()`
2. **Resolve auth** using the existing `resolveModelAuthMode()` and `getApiKeyForModel()` functions (already in `src/agents/model-auth.ts`) — these take `authStorage` and return `{ apiKey, headers }`
3. **Extract `model.contextWindow`** for passing to `buildEmbeddedExtensionFactories()` (or its Mastra equivalent)

```typescript
// In attempt.ts, Mastra branch:
const usesMastra = params.config?.agents?.defaults?.gateway === "mastra";

if (usesMastra) {
  // Resolve auth the same way the pi path does
  const authMode = resolveModelAuthMode(params.provider, params.model, params.authStorage);
  const apiKey = await getApiKeyForModel(params.model, params.authStorage, authMode);
  const extraHeaders = resolveExtraHeaders(params.provider, authMode);

  return runMastraAgent({
    provider: params.provider,
    modelId: params.modelId,
    modelApi: params.model.api,          // ← extracted from Model<Api>
    contextWindow: params.model.contextWindow,
    baseUrl: resolveProviderBaseUrl(params.model.api, params.config),
    apiKey,
    headers: extraHeaders,
    // ...
  });
}
```

**The plan must specify:**
- Which auth resolution functions are called in the Mastra branch
- How `model.api` is extracted (it's a direct property access: `params.model.api`)
- That `authStorage` and `modelRegistry` remain as `EmbeddedRunAttemptParams` fields even in the Mastra path (they are needed for auth resolution)

---

## Blocker 4 — Compaction Path Is Severely Underspecified

### What the plan claims

Section 4.7 describes `mastraGenerateSummary()` as a replacement for `generateSummary`. Section 9.2 says `compact.ts` gets "Add Mastra branch for compaction using `mastraGenerateSummary`."

### What `compact.ts` actually does (~700 lines)

The compaction path in [`src/agents/pi-embedded-runner/compact.ts`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/compact.ts) is not a simple `generateSummary` call. It is a full agent run with these phases:

**Phase 1 — Pre-compaction setup (lines 1–550)**
- Resolves workspace, agent dir, session file
- Acquires session write lock (`acquireSessionWriteLock`)
- Loads skills, applies env overrides
- Resolves model, auth, provider
- Builds system prompt (full system prompt, not just compaction instructions)
- Builds extension factories (`buildEmbeddedExtensionFactories`)
- Creates `DefaultResourceLoader` with extension factories
- Calls `createAgentSession()` — a full `AgentSession` with all tools

**Phase 2 — History preparation (lines 550–620)**
- Calls `sanitizeSessionHistory()` — removes invalid messages, repairs tool use/result pairing
- Calls `validateGeminiTurns()` / `validateAnthropicTurns()` — provider-specific validation
- Calls `limitHistoryTurns()` — truncates to DM history limit
- Calls `sanitizeToolUseResultPairing()` again after truncation
- Calls `session.agent.replaceMessages(limited)` — injects prepared history into session

**Phase 3 — Compaction LLM call (lines 620–730)**
- Fires `before_compaction` hooks (fire-and-forget)
- Calls `session.compact(params.customInstructions)` — this is the actual LLM call
- Calls `estimateTokens()` on remaining messages
- Fires `after_compaction` hooks (fire-and-forget)
- Returns `{ ok: true, compacted: true, result: { summary, firstKeptEntryId, tokensBefore, tokensAfter } }`

**Phase 4 — Cleanup (lines 727–744)**
- Calls `flushPendingToolResultsAfterIdle()` — drains pending tool results
- Calls `session.dispose()` — releases session resources
- Releases session write lock
- Restores skill env, restores cwd

### Why `mastraGenerateSummary` is not a drop-in replacement

`session.compact()` does not just call an LLM with a summary prompt. It:
1. Uses the session's existing message history (already loaded into `session.agent`)
2. Runs a structured compaction algorithm that produces `{ summary, firstKeptEntryId, tokensBefore }`
3. Writes the compacted history back to the JSONL session file via `SessionManager`
4. Returns a structured result with `firstKeptEntryId` (used for JSONL branching)

`mastraGenerateSummary` using `agent.generate()` produces only a text summary. It does not:
- Know which messages to keep (`firstKeptEntryId`)
- Write back to the JSONL file
- Produce `tokensBefore` / `tokensAfter` estimates
- Handle the session write lock

### Concrete resolution

The Mastra compaction path requires a full implementation, not a one-liner replacement. The plan must specify:

**Step 1: Define the compaction prompt and output contract**

```typescript
// src/agents/mastra/compaction.ts

export type MastraCompactionResult = {
  summary: string;
  // The index of the first message to keep after compaction
  // (messages before this index are replaced by the summary)
  firstKeptMessageIndex: number;
  tokensBefore: number;
  tokensAfter: number;
};

export async function mastraCompact(params: {
  messages: AgentMessage[];
  model: MastraModelConfig;
  customInstructions?: string;
  systemPrompt: string;
}): Promise<MastraCompactionResult> {
  // 1. Estimate tokens before
  const tokensBefore = estimateTokensForMessages(params.messages);

  // 2. Build compaction prompt (same prompt structure as pi-coding-agent uses)
  const compactionPrompt = buildCompactionPrompt(params.messages, params.customInstructions);

  // 3. Call LLM for summary
  const agent = new Agent({ id: "openclaw-compaction", model: params.model, instructions: params.systemPrompt, tools: {} });
  const result = await agent.generate([{ role: "user", content: compactionPrompt }]);

  // 4. Determine firstKeptMessageIndex
  // Convention: keep the last N messages (same as pi-coding-agent's default)
  const firstKeptMessageIndex = resolveFirstKeptIndex(params.messages);

  // 5. Estimate tokens after
  const keptMessages = params.messages.slice(firstKeptMessageIndex);
  const summaryTokens = estimateTokensForText(result.text);
  const tokensAfter = summaryTokens + estimateTokensForMessages(keptMessages);

  return { summary: result.text, firstKeptMessageIndex, tokensBefore, tokensAfter };
}
```

**Step 2: Specify JSONL write-back**

After `mastraCompact()` returns, the Mastra branch in `compact.ts` must:
1. Build the new message array: `[summaryMessage, ...messages.slice(firstKeptMessageIndex)]`
2. Write the new messages to the JSONL session file using the same atomic write pattern
3. Return `{ firstKeptEntryId }` compatible with the existing `EmbeddedPiCompactResult` type

**Step 3: Specify session write lock handling**

The Mastra compaction branch must acquire and release `acquireSessionWriteLock` — the same lock used by the pi path. This prevents concurrent compaction and run attempts from corrupting the JSONL file.

**The plan must add these to the modified files table:**
- `src/agents/mastra/compaction.ts` — full compaction implementation (not just `mastraGenerateSummary`)
- `src/agents/pi-embedded-runner/compact.ts` — Mastra branch that calls `mastraCompact()` and handles JSONL write-back

---

## Blocker 5 — No Context Overflow / Compaction Safeguard for Mastra Path

### What the plan claims

Section 15.3 recommends `maxSteps: 50`. The plan does not address what happens when the context window fills during a Mastra run.

### What the pi path does

[`src/agents/pi-embedded-runner/extensions.ts:64-95`](../sessions/agent_3acb03c6-b68d-49a5-8781-feff13385a57/src/agents/pi-embedded-runner/extensions.ts:64):

```typescript
export function buildEmbeddedExtensionFactories(params: {
  cfg: OpenClawConfig | undefined;
  sessionManager: SessionManager;
  provider: string;
  modelId: string;
  model: Model<Api> | undefined;
}): ExtensionFactory[] {
  const factories: ExtensionFactory[] = [];
  if (resolveCompactionMode(params.cfg) === "safeguard") {
    // Registers compaction safeguard extension
    setCompactionSafeguardRuntime(params.sessionManager, { ... });
    factories.push(compactionSafeguardExtension);
  }
  const pruningFactory = buildContextPruningFactory(params);
  if (pruningFactory) {
    factories.push(pruningFactory);  // cache-TTL context pruning
  }
  return factories;
}
```

The `compactionSafeguardExtension` is a pi-coding-agent `ExtensionFactory` that:
1. Monitors token usage after each LLM response
2. When usage exceeds the configured threshold, triggers compaction before the next prompt
3. Prevents the agent from running out of context mid-task

The `contextPruningExtension` with `mode = "cache-ttl"`:
1. Prunes old messages from the context window based on cache TTL timestamps
2. Reduces token usage without full compaction

### The failure mode with `maxSteps: 50`

With `maxSteps: 50` and no compaction safeguard:

1. Agent starts a long task (e.g., refactoring a large codebase)
2. After ~30 tool calls, the context window is 80% full
3. The pi path would trigger compaction here via the safeguard extension
4. The Mastra path has no equivalent — it continues until either:
   - The provider returns a context overflow error (hard failure)
   - `maxSteps: 50` is reached (silent truncation — the agent stops mid-task)

Users with `compaction.mode = "safeguard"` in their config will silently get broken behavior. The config is read but has no effect in the Mastra path.

### Concrete resolution

The plan must choose one of these options and specify it explicitly:

**Option A (recommended for Phase 1): Config validation error**

```typescript
// In attempt.ts, Mastra branch setup:
if (usesMastra) {
  const compactionMode = params.config?.agents?.defaults?.compaction?.mode;
  const contextPruningMode = params.config?.agents?.defaults?.contextPruning?.mode;

  if (compactionMode === "safeguard") {
    throw new ConfigurationError(
      "agents.defaults.compaction.mode = 'safeguard' is not supported with gateway = 'mastra'. " +
      "Use gateway = 'pi' for compaction safeguard support, or set compaction.mode = 'default'."
    );
  }
  if (contextPruningMode === "cache-ttl") {
    throw new ConfigurationError(
      "agents.defaults.contextPruning.mode = 'cache-ttl' is not supported with gateway = 'mastra'."
    );
  }
}
```

**Option B: Mastra-native context overflow handler**

Implement a token-counting wrapper around `agent.stream()` that:
1. After each `finish` chunk, checks `usage.totalTokens` against the configured context window
2. If usage exceeds threshold, calls `mastraCompact()` and retries the prompt
3. Exposes `isCompacting()` and `waitForCompactionRetry()` compatible with the existing queue handle interface

This is the correct long-term solution but requires significant implementation work. It should be specified as a Phase 2 item.

**Option C: Document the limitation**

Add to the plan's risk table:
> `gateway = "mastra"` does not support `compaction.mode = "safeguard"` or `contextPruning.mode = "cache-ttl"`. Long-running agents may hit context overflow. Mitigation: use `gateway = "pi"` for agents that require compaction.

**The plan must explicitly choose one of these options.** Currently it is silent, which means the implementation will silently break these config options.

---

## Blocker 6 — Anthropic Is NOT OpenAI-Compatible

### What the plan claims

Section 5 (Provider Support Matrix):

> `anthropic-messages` | `OpenAICompatibleConfig` with Anthropic base URL | Anthropic API is OpenAI-compatible for basic calls

Section 15.1:

> `OpenAICompatibleConfig` sets `Authorization: Bearer <apiKey>`. Passing `sk-ant-oat-*` as `apiKey` sends the correct Bearer header that Anthropic's OAuth API expects. **No `@ai-sdk/anthropic` package is needed for OAuth tokens.**

### Why this is wrong

The plan conflates two separate issues:

**Issue A: Auth header format** — Section 15.1 is correct for OAuth tokens (`sk-ant-oat-*`). These tokens use `Authorization: Bearer` which `OpenAICompatibleConfig` sends correctly.

**Issue B: API request/response format** — This is where the plan is wrong. Anthropic's API uses a completely different wire format from OpenAI:

| Aspect | OpenAI format | Anthropic format |
|---|---|---|
| Request body | `{ model, messages: [{role, content: string}], tools: [{type: "function", function: {name, parameters}}] }` | `{ model, messages: [{role, content: [{type: "text", text}]}], tools: [{name, description, input_schema}] }` |
| Tool call in response | `{ tool_calls: [{id, type: "function", function: {name, arguments: string}}] }` | `{ content: [{type: "tool_use", id, name, input: object}] }` |
| Tool result in request | `{ role: "tool", tool_call_id, content: string }` | `{ role: "user", content: [{type: "tool_result", tool_use_id, content}] }` |
| Required headers | `Authorization: Bearer` | `x-api-key: <key>` AND `anthropic-version: 2023-06-01` |
| Streaming format | `data: {"choices": [{"delta": {"content": "..."}}]}` | `data: {"type": "content_block_delta", "delta": {"type": "text_delta", "text": "..."}}` |

`createOpenAICompatible` in Mastra sends OpenAI-format requests. Sending these to `api.anthropic.com/v1/messages` will return HTTP 400 errors because:
1. The `tools` array format is wrong (OpenAI uses `function` wrapper, Anthropic uses flat `name`/`input_schema`)
2. The `content` format is wrong (OpenAI uses strings, Anthropic uses typed content blocks)
3. The `anthropic-version` header is missing (required by Anthropic's API)

### The OAuth token exception

Section 15.1's analysis is correct **only** for the auth header. Anthropic's OAuth API (`sk-ant-oat-*` tokens) does accept `Authorization: Bearer` — but it still uses Anthropic's wire format, not OpenAI's. The auth header fix does not make the API format compatible.

### Concrete resolution

**Add `@ai-sdk/anthropic` to dependencies** and use it for `anthropic-messages`:

```typescript
// src/agents/mastra/model-config.ts

import { anthropic } from "@ai-sdk/anthropic";
import { createAnthropic } from "@ai-sdk/anthropic";

export function toMastraModelConfig(params: {
  provider: string;
  modelId: string;
  modelApi: ModelApi;
  baseUrl?: string;
  apiKey?: string;
  headers?: Record<string, string>;
}): MastraModelConfig {
  switch (params.modelApi) {
    case "anthropic-messages": {
      // Use @ai-sdk/anthropic for correct wire format
      // For OAuth tokens (sk-ant-oat-*), pass as apiKey — @ai-sdk/anthropic
      // handles both x-api-key and Bearer auth depending on token type
      const provider = createAnthropic({
        apiKey: params.apiKey,
        baseURL: params.baseUrl,
        headers: params.headers,
      });
      return provider(params.modelId);  // Returns LanguageModelV1
    }

    case "openai-completions":
    case "openai-responses":
    case "openai-codex-responses":
    case "ollama":
    case "github-copilot": {
      // These are genuinely OpenAI-compatible
      return {
        id: `${params.provider}/${params.modelId}`,
        url: resolveProviderBaseUrl(params.modelApi, params.baseUrl),
        apiKey: params.apiKey,
        headers: params.headers,
      } satisfies OpenAICompatibleConfig;
    }

    case "google-generative-ai": {
      // Already in plan: use @ai-sdk/google
      const { google } = await import("@ai-sdk/google");
      return google(params.modelId);
    }

    case "bedrock-converse-stream": {
      // Already in plan: use @ai-sdk/amazon-bedrock
      const { bedrock } = await import("@ai-sdk/amazon-bedrock");
      return bedrock(params.modelId);
    }
  }
}
```

**Updated dependency list:**

```json
{
  "@mastra/core": "1.8.0",
  "@ai-sdk/anthropic": "^1.2.0",
  "@ai-sdk/google": "^1.2.0",
  "@ai-sdk/amazon-bedrock": "^1.2.0",
  "ai": "^4.3.0"
}
```

**Updated provider support matrix:**

| `ModelApi` | Mastra approach | Notes |
|---|---|---|
| `anthropic-messages` | `@ai-sdk/anthropic` `LanguageModelV1` | **NOT OpenAI-compatible** — requires native SDK |
| `openai-completions` | `OpenAICompatibleConfig` | Direct OpenAI endpoint |
| `openai-responses` | `OpenAICompatibleConfig` | Responses API endpoint |
| `openai-codex-responses` | `OpenAICompatibleConfig` | Codex endpoint |
| `google-generative-ai` | `@ai-sdk/google` `LanguageModelV1` | Google requires native SDK for auth |
| `bedrock-converse-stream` | `@ai-sdk/amazon-bedrock` `LanguageModelV1` | AWS SigV4 requires native SDK |
| `ollama` | `OpenAICompatibleConfig` with Ollama base URL | Ollama exposes OpenAI-compatible API |
| `github-copilot` | `OpenAICompatibleConfig` with Copilot base URL | Token-based auth via headers |

**Note on Anthropic OAuth tokens:** `@ai-sdk/anthropic` v1.x supports both `x-api-key` (standard API keys) and `Authorization: Bearer` (OAuth tokens). The `createAnthropic({ apiKey })` constructor passes the key as `x-api-key` by default. For OAuth tokens, the `headers` option must be used:

```typescript
const provider = createAnthropic({
  headers: {
    "Authorization": `Bearer ${oauthToken}`,
    "anthropic-version": "2023-06-01",
  },
});
```

The `model-config.ts` must detect OAuth tokens (`sk-ant-oat-*` prefix) and switch to Bearer auth.

---

## Blocker 7 — `maxSteps: 50` Cuts Off Agents That Need Mid-Run Compaction

### What the plan claims

Section 15.3:

> Recommended default: `50` (matches typical pi-coding-agent behavior for complex tasks). Configurable via `agents.defaults.maxSteps` in the OpenClaw config.

### Why this is insufficient

The pi-coding-agent loop has **no hard step limit**. It runs until the model stops calling tools. The compaction safeguard extension (Blocker 5) handles context overflow by triggering compaction mid-run and continuing. The agent can run for hundreds of tool calls on complex tasks.

`maxSteps: 50` means:
- An agent that needs 60 tool calls to complete a task will be silently truncated at step 50
- The agent will return a partial result with no error indication
- The user sees an incomplete response with no explanation

More critically, the interaction between `maxSteps` and compaction is undefined:
- If compaction is triggered at step 40 (via the Mastra-native handler from Blocker 5 Option B), does the step counter reset?
- If not, the agent gets only 10 more steps after compaction — far fewer than needed

### The `maxSteps` value is not the real problem

The real problem is that `maxSteps` is a hard ceiling on a loop that the pi path runs without a ceiling. The plan's recommendation of `50` is arbitrary and will break agents that currently work fine with the pi path.

### Concrete resolution

**Step 1: Make `maxSteps` configurable with a high default**

```typescript
// In agent-runner.ts:
const output = await agent.stream(coreMessages, {
  maxSteps: params.config?.agents?.defaults?.maxSteps ?? 200,
  // 200 is a safety ceiling, not a target — most agents finish in <50 steps
  // but complex refactoring tasks can exceed 100
  providerOptions: toMastraProviderOptions(params.thinkLevel, params.provider),
});
```

**Step 2: Specify step counter behavior during compaction**

If Blocker 5 Option B (Mastra-native compaction handler) is implemented, the plan must specify:

```typescript
// Compaction resets the step counter by creating a new agent.stream() call
// with the compacted message history. The maxSteps applies per-stream-call,
// not per-run. This matches the pi path behavior where compaction creates
// a new session.prompt() call.

async function runMastraAgentWithCompaction(params: RunMastraAgentParams): Promise<MastraRunResult> {
  let messages = params.messages;
  let totalSteps = 0;
  const maxTotalSteps = params.maxSteps ?? 200;

  while (totalSteps < maxTotalSteps) {
    const stepsRemaining = maxTotalSteps - totalSteps;
    const output = await agent.stream(toCoreMessages(messages), {
      maxSteps: Math.min(stepsRemaining, 50),  // per-call limit
    });

    // Consume stream, count steps
    let stepsThisCall = 0;
    for await (const chunk of output.fullStream) {
      if (chunk.type === "step-finish") stepsThisCall++;
      // ... emit events
    }
    totalSteps += stepsThisCall;

    // Check if compaction is needed
    const usage = await output.usage;
    if (needsCompaction(usage, params.contextWindow)) {
      messages = await compactAndContinue(messages, params);
      continue;  // restart loop with compacted history
    }

    break;  // agent finished normally
  }
}
```

**Step 3: Document the behavioral difference**

The plan must explicitly state:

> `gateway = "mastra"` with `maxSteps` set to any finite value will truncate agents that exceed that step count. The pi path has no equivalent limit. For agents that require unlimited tool call loops (e.g., large codebase refactoring), use `gateway = "pi"` until Mastra-native compaction is implemented (Phase 2).

---

## Summary: Required Plan Additions

The following items must be added to the integration plan before implementation begins:

### New files required (additions to Section 9.1)

| File | Purpose |
|---|---|
| `src/agents/mastra/subscribe-mastra-session.ts` | Full `subscribeMastraSession()` implementation (Blocker 1) |
| `src/agents/mastra/mastra-event-feed.ts` | `MastraEventFeed` and `MastraSessionEvent` types (Blocker 1) |

### Modified files (corrections to Section 9.2)

| File | Required change |
|---|---|
| `src/agents/mastra/model-config.ts` | Add `@ai-sdk/anthropic` branch for `anthropic-messages` (Blocker 6) |
| `src/agents/mastra/compaction.ts` | Full compaction implementation with JSONL write-back, not just `mastraGenerateSummary` (Blocker 4) |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Auth resolution before Mastra branch; config validation for unsupported extensions (Blockers 3, 5) |
| `src/agents/pi-embedded-runner/run/types.ts` | `lastAssistant` type handling for Mastra path (Blocker 2) |

### New dependencies (corrections to Section 10.1)

```json
{
  "@mastra/core": "1.8.0",
  "@ai-sdk/anthropic": "^1.2.0",
  "@ai-sdk/google": "^1.2.0",
  "@ai-sdk/amazon-bedrock": "^1.2.0",
  "ai": "^4.3.0"
}
```

### New implementation steps (additions to Section 13)

Insert after step 7 (stream-subscriber.ts):

```
7a. Create src/agents/mastra/mastra-event-feed.ts — MastraEventFeed and MastraSessionEvent types
7b. Create src/agents/mastra/subscribe-mastra-session.ts — full subscribeMastraSession() implementation
7c. Write unit tests for subscribeMastraSession() covering: text streaming, tool calls, compaction events, abort, messaging tool tracking
```

Insert after step 9 (compaction.ts):

```
9a. Specify compaction prompt format and firstKeptMessageIndex resolution logic
9b. Specify JSONL write-back after mastraCompact() returns
9c. Verify session write lock is acquired/released in Mastra compaction branch
```

Insert after step 11 (index.ts):

```
11a. Add config validation in attempt.ts Mastra branch: error on compaction.mode = "safeguard" and contextPruning.mode = "cache-ttl"
11b. Document maxSteps behavioral difference in CHANGELOG and plan
```

---

## Architecture Diagram — Corrected for All 7 Blockers

```mermaid
graph TD
    A[Channel Message] --> B[gateway/server-methods/chat.ts]
    B --> C[pi-embedded-runner/run.ts]
    C --> D[pi-embedded-runner/run/attempt.ts]
    D --> E{agents.gateway config}
    E -->|pi - current default| F[createAgentSession - pi-coding-agent]
    E -->|mastra - new| G[mastra/agent-runner.ts runMastraAgent]
    F --> H[subscribeEmbeddedPiSession - uses AgentSession directly]
    G --> I[subscribeMastraSession - NEW - same output contract]
    G --> J[Auth resolution via resolveModelAuthMode + getApiKeyForModel]
    J --> K{modelApi}
    K -->|anthropic-messages| L[@ai-sdk/anthropic LanguageModelV1]
    K -->|openai-compatible| M[OpenAICompatibleConfig]
    K -->|google-generative-ai| N[@ai-sdk/google LanguageModelV1]
    K -->|bedrock-converse-stream| O[@ai-sdk/amazon-bedrock LanguageModelV1]
    H --> P[assistantTexts + toolMetas + compaction]
    I --> P
    P --> Q[Channel delivery - unchanged]
    D --> R[JSONL session file - SessionManager]
    G --> S[JSONL session file - mastraCompact writes back]
    G --> T{compaction.mode}
    T -->|safeguard| U[ConfigurationError - not supported in Mastra path]
    T -->|default| V[maxSteps ceiling - document behavioral difference]
    D --> W[lastAssistant - buildLastAssistantFromMastra constructs AssistantMessage]
```

---

## Verdict

The 7 critical blockers are real and each requires a concrete specification before implementation. The most impactful changes to the plan are:

1. **Blocker 1** — Write `subscribeMastraSession()` as a new function (not a shim). This is the largest implementation item.
2. **Blocker 6** — Add `@ai-sdk/anthropic` to dependencies. This is a one-line dependency change with a small code change in `model-config.ts`.
3. **Blocker 4** — Specify the full compaction path including JSONL write-back. The current plan's `mastraGenerateSummary` is ~5% of what's needed.
4. **Blocker 5** — Add config validation errors for unsupported extension modes. This prevents silent behavioral regressions.
5. **Blockers 2, 3, 7** — Smaller but required: `lastAssistant` construction, auth resolution from `Model<Api>`, and `maxSteps` documentation.

None of these blockers require abandoning the plan's architecture. The feature-flag approach, JSONL preservation, and TypeBox→Zod conversion are all correct. The blockers are gaps in specification, not fundamental design flaws.
