# AI chat — streamed multi-turn assistant with tool use & persistent history

Load this reference when adding a chat interface backed by Claude / OpenAI / Cerebras with multi-turn tool use over the app's records, persistent chat history, server-side context compaction, or model picker UIs. Skip it for one-shot LLM calls (use `integration.post('anthropic/chat-completion', ...)` or `createDeepSpaceAI` with `generateText` instead) or for tasks that don't need an in-app chat UI.

## What ships out of the box

The scaffold ships a complete chat backend at `src/ai/chat-routes.ts` (registered into the Hono app via `registerAiChatRoutes(app, resolveAuth)` in `worker.ts`). It exposes four endpoints:

| Endpoint | Purpose |
|---|---|
| `POST /api/ai/chats` | Create a chat row owned by the caller. Body: `{ title? }`. Returns `{ chat: ChatRow }`. |
| `PATCH /api/ai/chats/:id` | Rename / patch a chat. Body: `{ title? }`. Owner-checked (404 on cross-user). |
| `DELETE /api/ai/chats/:id` | Delete chat + cascade-delete its messages. Owner-checked. |
| `POST /api/ai/chat` | The streaming turn. Body: `{ chatId, userMessageId, content, modelId? }`. Returns `text/event-stream` (Vercel AI SDK v5 `UIMessage` chunks). Header `X-Asst-Id` carries the assistant row's id for client-side dedup. |

History persists in two pre-built collections — register them in `src/schemas.ts`:

- **`AI_CHATS_SCHEMA`** (`'ai-chats'`) — one row per chat. Columns: `userId`, `title`, `model`, `compactedSummary`, `compactedThroughId`. RBAC: members `read/update/delete: 'own'`, `create: false` (writes flow only through the worker).
- **`AI_MESSAGES_SCHEMA`** (`'ai-messages'`) — one row per turn. Columns: `chatId`, `userId`, `role`, `content`, `parts` (json — UI-shape tool invocations). Same RBAC.

Both schemas are exported from `'deepspace/worker'`. The `create: false` posture is intentional — direct WS creates would let a user PUT a forged `role: 'assistant'` row that the next turn's `loadMessages` would feed back to the LLM as if real. **Don't relax `create` to `'own'`.**

The streaming pipeline (`POST /api/ai/chat` handler):

1. Verify JWT, look up the chat (404 if missing or owned by another user).
2. Load history (`loadMessages`) without writing the new user row. Persistence starts only in `onFinish`; a transport failure or zero-step abort that skips it writes nothing. A post-step abort may finish with a partial paired turn.
3. Append the in-flight user message in memory; dedup consecutive user messages as defense in depth against malformed history.
4. Run `prepareMessagesWithCompaction`: truncate old tool results, apply cached summary if present, summarize the older half if still over budget, fall back to sliding window on summarizer error.
5. Build tools (`buildTools` from `src/ai/tools.ts`) and call `streamText({ model, system, messages, tools, stopWhen: stepCountIs(5), abortSignal: c.req.raw.signal, onFinish })`.
6. On `onFinish`: write user → assistant → metadata, in that order, with per-call retry. If user-write fails after retry, **abort** the assistant write (otherwise the next turn's history reads an assistant row with no preceding user).

## Reference implementation — `npx deepspace add ai-chat`

> **Copilot-template apps already ship this chat surface** — the same ChatPanel lives at `src/components/chat/ChatPanel.tsx`, embedded in the shell's chat dock (`src/components/shell/ChatDock.tsx`). Don't run `add ai-chat` there: the installer doesn't detect the template's copy (different path), so it would install a second, divergent ChatPanel against the same collections. `add ai-chat` is for adding the full-page assistant to starter-template apps.

Installs five files into the scaffolded app:

- `src/components/ChatPanel.tsx` — composes the message list, model picker, composer, abort/retry controls, `useQuery('ai-messages')`, and in-flight overlay.
- `src/components/ChatPanel.messages.tsx` — memoized message, Markdown, tool-status, empty, and thinking renderers.
- `src/components/ChatPanel.stream.ts` — stream transport plus the pure overlay reducer; owns auto-create, abort, retry, SSE errors, and tool-call lifecycle.
- `src/pages/(app)/(protected)/assistant.tsx` — protected full-page assistant with conversation history, create, rename, delete, and resize UI.
- `src/schemas/ai-chat-schema.ts` — exports `aiChatSchemas = [AI_CHATS_SCHEMA, AI_MESSAGES_SCHEMA]`; the installer spreads it into `src/schemas.ts`.

`feature.json` declares the scaffold dependencies (`react-markdown`, `remark-gfm`, `remark-breaks`, `rehype-highlight`, `highlight.js`) so `npx deepspace add ai-chat` runs `npm install` for them.

**`ChatPanel` is meant to be embedded.** Pass `chatId={null}` to auto-create on first send and `onChatCreated` to track the new id. Pass `disabled` while a parent-owned create is in flight so the panel doesn't kick off its own duplicate auto-create. Read its prop docstring at install time.

Persistent mutations arrive live through `useQuery`; the stream hook overlays the pending user/assistant turns immediately, then `ChatPanel` removes them as matching `recordId`s arrive. `X-Asst-Id` reconciles the pending assistant without relying on client clocks. Stop or transport failure removes an empty pending assistant but preserves partial text/tool rows already shown; switching away from a real chat aborts and clears its overlay. Top-level failures render an alert with **Retry** (re-sends the last content); tool input/output failures stay inline on their tool row.

## Customization

### Switch model or provider

Edit `ALLOWED_MODELS` in `src/ai/chat-routes.ts`. Shipped IDs: Anthropic `claude-opus-4-8`, `claude-sonnet-5` (default), `claude-haiku-4-5`; OpenAI `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`; Cerebras `gpt-oss-120b`. Unknown IDs return 400; there is no silent fallback.

> **Keep `ChatPanel`'s `DEFAULT_MODELS` and `src/ai/chat-routes.ts`'s `ALLOWED_MODELS` in sync.** The picker shows whatever `DEFAULT_MODELS` lists; the worker only accepts whatever `ALLOWED_MODELS` keys. Adding a new model is a 2-file edit. Drift produces a 400 when the user picks the new option.

The provider routing is handled by `createDeepSpaceAI(env, provider, { authToken })` from `'deepspace/worker'`:

- Pass `{ authToken: jwt }` (the caller's JWT) for **user-billed** calls (default for chat — the user pays for their own conversation).
- Omit `authToken` for **owner-billed** calls (falls back to `env.APP_OWNER_JWT`). Useful for the summarizer if you want compaction absorbed by the developer.

### Customize tools (allow more / fewer)

Edit `ALLOWED_TOOL_NAMES` in `src/ai/tools.ts`. The full catalog from `BUILT_IN_TOOLS` (in `'deepspace/worker'`):

- `schema.list` — list collection names
- `schema.describe` — describe one collection (columns, permissions)
- `records.query` / `records.get` — reads
- `records.create` / `records.update` / `records.delete` — **writes**
- `user.current` — caller's user record
- `user.list` — list / look up other users' records
- `yjs.list` / `yjs.getText` — collaborative-document reads
- `yjs.setText` — **write** collaborative text

The scaffold enables schema, record, and current-user tools by default, including record writes; `user.list` and all Yjs tools are opt-in. The assistant can therefore mutate data. **Per-collection RBAC at the DO is the security boundary**; trim the allowlist to read tools for a read-only assistant and keep explicit confirmation around destructive writes.

`buildSystemPrompt(appName, schemas)` produces a concise prompt that lists every collection (name + columns with required-marker `!`). The scaffold's `buildSystemPrompt` includes mutation guardrails ("Confirm intent before destructive actions", "If a write is denied (RBAC), tell the user plainly — do not retry blindly"). Customize by editing `src/ai/tools.ts` directly.

### Add a custom tool (beyond the built-ins)

Augment `buildTools` in `src/ai/tools.ts` with `tool({ description, inputSchema: z.object({...}), execute })` from the `ai` package. The Zod `inputSchema` doubles as runtime validation; failing input emits a `tool-input-error` chunk the client surfaces.

### Tune context-window compaction

`prepareMessagesWithCompaction(messages, config, { summarizer, cachedSummary })` is the pre-stream pipeline. The default config:

```typescript
DEFAULT_CONTEXT_CONFIG = {
  contextBudget: 240_000,    // chars — ≈ 60–80K tokens. Safe for 200K+ models.
  toolResultCap: 30_000,     // bytes — per-tool-result cap (`capToolResultSize`).
  keepRecentToolResults: 5,  // recent assistant turns whose tool results stay un-truncated.
  minKept: 10,               // sliding-window floor.
}
```

Lower `contextBudget` for shorter-context models:
- 128K-context model (e.g. some OpenAI variants) → ~120,000.
- 32K-context model (some Cerebras open-weights) → ~40,000.

The `capToolResultSize(result, byteCap)` helper replaces oversized tool outputs with a structured error telling the agent to narrow its query (preserves a 2KB preview). The scaffold wraps every tool execution with it — see `chat-routes.ts`'s `buildTools` callback.

### Swap the summarizer

`makeDefaultSummarizer(env, { authToken? })` returns a Claude Haiku summarizer (`claude-haiku-4-5`). **Default (no `authToken`) is owner-billed** — the function falls back to `APP_OWNER_JWT`, treating compaction as infrastructure. The scaffold's `chat-routes.ts` explicitly opts in to user-billing by passing `{ authToken: jwt }` (the caller's JWT) so compaction is part of the user's chat cost. To revert to owner-billed:

```typescript
const summarizer = makeDefaultSummarizer(c.env)  // → APP_OWNER_JWT, owner pays (function default)
```

Roll your own by implementing `Summarizer = (messages: ChatTurn[]) => Promise<string>`. The default summary anchors on the last real message id in the older half (skipping prior-summary system rows so re-summarization doesn't loop) — preserve that anchoring if you replace it, or expect a billing leak.

## Custom UIs (without `ChatPanel`)

If you're building your own chat surface — a sidebar, a modal, a minimal text-only one — call `POST /api/ai/chat` directly and decode the SSE stream with the SDK's wire-format helpers (re-exported from `'deepspace'`):

```tsx
import { parseSseLine, decodeAiStreamChunk, type AiStreamAction, getAuthToken } from 'deepspace'

async function send(chatId: string, userMessageId: string, content: string, modelId?: string) {
  const token = await getAuthToken()
  const res = await fetch('/api/ai/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({ chatId, userMessageId, content, modelId }),
  })
  if (!res.ok || !res.body) throw new Error(`chat ${res.status}`)

  const asstId = res.headers.get('X-Asst-Id')!  // tag in-flight overlay with this; dedup against persisted row by id
  const reader = res.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''

  while (true) {
    const { value, done } = await reader.read()
    if (done) break
    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split('\n')
    buffer = lines.pop() ?? ''
    for (const line of lines) {
      const chunk = parseSseLine(line)
      if (!chunk) continue
      const action = decodeAiStreamChunk(chunk)
      if (action) applyAction(asstId, action)   // reduce against your local state
    }
  }
}
```

`AiStreamAction` is the small action vocabulary the React layer reduces:

- `append-text { delta }` — text-delta token.
- `upsert-tool-call { toolCallId, toolName, input }` — tool started.
- `finalize-tool-call { toolCallId, result }` — tool result arrived.
- `fail-tool-input { toolCallId, toolName, input, errorText }` — Zod input validation rejected the call (no preceding `upsert` was emitted).
- `fail-tool-output { toolCallId, errorText }` — tool's `execute` threw; finalize the existing invocation as failed.
- `stream-error { errorText }` — top-level stream error.
- `abort` — server-side abort with no error chunk to follow.

The `X-Asst-Id` response header is the assistant row's id. Tag your in-flight overlay with it; dedup against the WebSocket-broadcast persisted row by id. **Don't dedup by `spawnTime` (client clock) vs `createdAt` (server clock)** — that breaks for users whose clock is ahead of the server.

For the canonical message list, mount `useQuery<AiMessageData>('ai-messages', { where: { chatId, userId }, orderBy: 'createdAt', orderDir: 'asc' })` against your `RecordScope`. The `parts` field carries the UI-shape tool invocations (`{ type: 'tool-invocation', toolCallId, toolInvocation: { toolName, state: 'result', args, result } }`).

## Testing

The streaming endpoint is large; test it via `api.spec.ts`:

```typescript
test('POST /api/ai/chat streams + persists', async ({ request }) => {
  // Pre-condition: signed-in user, an existing chat row.
  const chatRes = await request.post('/api/ai/chats', {
    headers: { Authorization: `Bearer ${token}` },
    data: { title: 'test' },
  })
  const { chat } = await chatRes.json()

  const res = await request.post('/api/ai/chat', {
    headers: { Authorization: `Bearer ${token}` },
    data: {
      chatId: chat.recordId,
      userMessageId: `umsg-${Date.now()}`,
      content: 'Hi',
    },
  })
  expect(res.status()).toBe(200)
  expect(res.headers()['x-asst-id']).toMatch(/^asst-/)
  // Drain the stream, then assert via useQuery in the UI test that two new
  // ai-messages rows (user + assistant) exist.
})
```

Keep one such test per turn-shape (text-only, tool-using, multi-step, abort). Don't iterate the stream in a loop testing every chunk type — the SDK already unit-tests `decodeAiStreamChunk` against the v5 chunk vocabulary, so app-level tests should assert behavior, not parser fidelity.

For the auth-gating side, follow the standard pattern in `references/testing.md` — assert 401 for unauthenticated callers and 404 for cross-user `chatId`.

## Things that bite

- **Persistence is ordered, not atomic.** Order is user → assistant → metadata. If the user write fails twice, the assistant write is skipped, preventing an assistant-only row. The reverse remains possible if the assistant write fails after the user succeeds; retries reduce but cannot eliminate it. Don't reorder.
- **Concurrent multi-tab writes to the same `chatId`** can interleave. The DO serializes individual writes but not the per-request 3-write group. Realistic impact is rare; documented in `chat-routes.ts` next to the route.
- **Reasoning models are opt-out at the boundary.** `toUIMessageStreamResponse({ sendReasoning: false })` strips `reasoning-*` chunks. The UI doesn't have a "thinking" disclosure block today; flipping `sendReasoning: true` will leave users staring at a stuck spinner during long thinks.
- **Don't relax `AI_*_SCHEMA`'s `create: false`.** That's the forged-assistant-row vector.
- **`stopWhen: stepCountIs(5)`** caps tool-loop rounds per turn. For agentic workflows that legitimately need more steps, raise it carefully — each step is a full LLM round-trip and adds cost.
- **`X-Asst-Id` header is required** for clock-skew-proof dedup. If you proxy the streaming response through another worker, preserve the header.

## See also

- `references/sdk-reference.md` § Server-side AI helpers — full export list (`createDeepSpaceAI`, `prepareMessagesWithCompaction`, `turnsToCoreMessages`, `buildUiParts`, `unwrapToolOutput`, `makeDefaultSummarizer`, `capToolResultSize`, `truncateOldToolResults`, `applySlidingWindow`, `totalChars`, `DEFAULT_CONTEXT_CONFIG`, `ChatContextConfig`, `ChatTurn`, `Summarizer`, plus `getChat` / `createChat` / `updateChat` / `deleteChatCascade` / `loadMessages` / `appendMessage` / `ChatRow` / `ChatMessageRow` from `chat-history`).
- `references/sdk-reference.md` § Frontend wire helpers — `parseSseLine`, `decodeAiStreamChunk`, `AiStreamAction`, `AiStreamChunk`.
- `references/integrations.md` for raw `integration.post('anthropic/...')` calls when you don't need streaming.
