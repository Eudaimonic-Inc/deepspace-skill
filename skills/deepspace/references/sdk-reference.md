# DeepSpace SDK reference

Use this as an export index, not a workflow guide. Load the task-shaped reference (`auth.md`, `schemas.md`, `ai-chat.md`, `bindings.md`, `payments.md`, and so on) for implementation guidance. Confirm exact signatures in `node_modules/deepspace/dist/{index,worker,server,testing}.d.ts`; do not guess.

| Import | Owns |
|---|---|
| `deepspace` | React hooks, providers, client helpers, shared types. |
| `deepspace/schema` | Runtime-neutral collection types and built-in schema constants for browser/shared code. |
| `deepspace/worker` | Durable Objects, JWT, actions, AI, bindings, metering, and worker-only helpers. |
| `deepspace/server` | Platform-backed subscriptions, refunds, and screenshots. |
| `deepspace/testing` | Playwright multi-user fixtures; test files only. |

## Frontend (`deepspace`)

### Auth

| Export | Contract |
|---|---|
| `DeepSpaceAuthProvider` | Required above auth hooks. |
| `AuthOverlay`, `SignedIn`, `SignedOut`, `AuthGate`, `GuestBanner` | Sign-in and conditional UI. |
| `useAuth()` | `{ isLoaded, isSignedIn, userId, sessionId }`; use for auth readiness. |
| `useAuthUser()` | Auth-layer Better Auth user. |
| `useUser()` | Storage-layer `{ user, isLoading, refetch }`; fields are under `user`. |
| `useAuthStatus()` | Auth-only readiness; safe outside `RecordProvider`. |
| `useAuthProfileReady()` | Auth plus profile readiness; requires `RecordProvider`. |
| `useDisplayName()` | Resolved display name or `null`. |
| `authClient`, `useSession`, `signIn`, `signOut`, `getAuthToken`, `clearAuthToken` | Low-level session access. |

Read `auth.md` before changing provider order or public/gated layout.

### Records

| Export | Contract |
|---|---|
| `RecordProvider` | WebSocket/store root. Preserve scaffolded `onWriteError`: optimistic write denials surface there. |
| `RecordScope` | Selects room/schemas/app/shared scope. |
| `ScopeRegistryProvider` | Required once when using shared scopes. |
| `useQuery<T>(collection, options?)` | `{ records, status, error }`; records are envelopes, so use `record.recordId` and `record.data.field`. |
| `useMutations<T>(collection)` | `{ ready, create, put, remove, createConfirmed, putConfirmed, removeConfirmed }`. Disable write controls until `ready`; all methods fail with `RecordRoomNotReadyError` / code `not_ready` before then. `put` merges a patch. Plain methods return after optimistic apply; use confirmed methods when the next operation depends on server acceptance. |
| `useUsers()` | User list, admin `setRole`, and `refresh`. |
| `useUserLookup()` | O(1) `getUser/getEmail/getName`; other fields come from `getUser(id)`. |
| `useRecordContext()`, `RecordStore` | Low-level store access. |

### Messaging and directory

| Export | Contract |
|---|---|
| `useChannels()` | Channel CRUD/archive; `create` requires `{ name, type }`. |
| `useMessages(channelId)` | Send/edit/remove/soft-delete channel messages. |
| `useReactions(channelId)` | Toggle and group reactions. |
| `useChannelMembers(channelId)` | Join/leave/membership. |
| `useReadReceipts()` | Mark/read unread counts. |
| `useConversation()` | DM/conversation collections inside a `conv:<id>` scope; not the channel collections. |
| `useConversations()` | Cross-app inbox state and DM/channel creation in the directory DO. |
| `useCommunities()` | Communities and memberships. |
| `usePosts()` | Directory posts and conversation linkage. |

Types: `Channel`, `Message`, `Reaction`, `ChannelMember`, `ChannelInvitation`, `ReadReceipt`. Helpers: `groupReactionsForMessage`, `shouldGroupMessages`, `getThreadCounts`, `formatMessageTime`, `formatFullTimestamp`, `getConversationDisplayName`, `getConversationParticipantIds`, `isDMConversation`, `parseMessageMetadata`.

### Collaboration and background work

| Export | Contract |
|---|---|
| `useYjsField`, `useYjsText`, `useYjsRoom` | Collaborative data/text. `connected` is transport state; `synced` is document convergence. Gate editing on `connected && synced && canWrite`; Yjs awareness carries cursors and selections. |
| `useCanvas(roomId)` | Shapes, viewports, undo/redo. Writes no-op until `canWrite`; viewport broadcast remains available to viewers. |
| `usePresence()` | Users-collection heartbeat/last-seen state, not cursors. |
| `usePresenceRoom(scopeId)` | Ephemeral cursor/typing/viewport peers. |
| `useGameRoom(roomId)` | Game DO client; game binding/manifest/route are not scaffolded. |
| `useCronMonitor(roomId)` | Task state plus trigger/pause/resume. See `cron.md` for authorization. |
| `useJobs(roomId)` | Durable queue state plus enqueue/cancel/retry. Worker-side code uses `enqueueJob`. |

Low-level Yjs exports include `Awareness`, encoder/decoder helpers, sync/awareness handlers, and `MSG_SYNC*` / `MSG_AWARENESS`. Use the `.d.ts` only when built-in hooks are insufficient.

### Files, integrations, voice, and payments

| Export | Contract |
|---|---|
| `useR2Files()` | `upload`, `uploadBase64`, `deleteFile`, `downloadFile`, `readFile`, `list`, `getUrl`, `isUploading`. `list()` is async; there is no reactive `files` field. |
| `R2FileInfo` | `{ key, size, uploaded, url, originalName?, uploadedBy? }`; no MIME field. |
| `isImageFile`, `formatFileSize` | Display helpers. |
| `integration` | `get/post/put/delete`; returns `{ success, data }` or `{ success, error, issues? }`. Read validation `issues` before changing payload fields. |
| `useAsyncResource` | Abortable single-resource loading with explicit status/reload. |
| `usePagedResource` | Abortable paged feeds/search with load-more, retry, and refresh. |
| `useVoiceAgent` | Managed OpenAI Realtime WebRTC session; no LiveKit room hook exists. |
| `useSubscription`, `useCheckout`, `PricingTable` | Client entitlements and purchases; load `payments.md`. |

R2, subscription, checkout, and screenshot paths require `APP_IDENTITY_TOKEN`. A fresh app receives it after registration/deploy; rerun `dev start` afterward to refresh `.dev.vars`.

### Platform context, UI, and environment

`PlatformProvider` is opt-in and not scaffolded. It owns `usePlatform()`, `useInbox()`, `usePlatformWS()`, and `PlatformContext`; mount it only for cross-app inbox/platform streams.

- Theme: `DeepSpaceThemeProvider`, `useIsDarkTheme`, `isDarkColor`, `applyDeepSpaceTheme`, `clearDeepSpaceTheme`, `readThemeFromDOM`, `applyUIThemeTokens`, `DEEPSPACE_THEME_PROPERTIES`.
- User colors: `DEFAULT_USER_COLORS`, `getUserColor`.
- Environment: `detectEnvironment`, `getEnvironmentConfig`, `getApiUrl`, `getPlatformWorkerUrl`, `getAuthUrl`, `isLocalDev`, `isProduction`, `resetEnvironmentCache`, `ENV`.
- RBAC: `ROLES`, `ROLE_CONFIG`, `Role`. Collection schemas—not `ROLE_CONFIG`—own permissions.

### Frontend wire helpers

For custom AI chat UIs, `parseSseLine` decodes SSE framing and `decodeAiStreamChunk` reduces supported AI SDK v5 chunks to `AiStreamAction`. Types: `AiStreamChunk`, `AiStreamAction`. Prefer the `ai-chat` feature and load `ai-chat.md` before building this layer yourself.

For a custom DeepSpace WebSocket client, use shared `MSG`, `ClientMessage`, `ServerMessage`, `clientBuild`, `dispatch`, and `encode`. Most apps should use the hooks above.

## Worker (`deepspace/worker`)

### Durable Objects and manifest

| Export | Contract |
|---|---|
| `BaseRoom` | Rare custom-room base; built-in room classes are preferred. |
| `RecordRoom<Env>` | Primary schema/RBAC record DO. |
| `YjsRoom<Env>` | Collaborative Yjs documents. |
| `CanvasRoom<Env>` | Shapes and viewports. |
| `PresenceRoom<Env>` | Ephemeral peer state. |
| `CronRoom<Env>` | Scheduled tasks; override `onTask`. |
| `JobRoom<Env>` | Durable queue; override `onJob`. |
| `GameRoom<Env>` | Tick/state game loop; override `onHydrateState` for stored-state migrations. |

The scaffold wires Record, Yjs, Canvas, Presence, Cron, and Job rooms. Adding or removing a class requires coordinated `__DO_MANIFEST__`, Wrangler binding/migration, export, and route changes.

- `DOManifest`, `DOManifestEntry`, `DOBindings<typeof manifest>` type the manifest and env bindings.
- `DEFAULT_DO_MANIFEST` contains only Record and Yjs rooms; it is not the full scaffold manifest.
- `enqueueJob(namespace, roomId, type, payload, opts?)` enqueues outside the Job DO isolate.
- Room-specific types include `UserAttachment`, `CanvasShape`, `Viewport`, `PresencePeer`, `CronTask`, `CronExecution`, `Job`, `JobContext`, `JobStatus`, `JobRoomConfig`, `Player`, and `GameInput`.

### Auth and routes

- `verifyJwt(config, token)` returns `{ result, error? }`; invalid tokens do not throw.
- `decodeJwtPayload(token)` does not verify and must not establish trust.
- `createDeepSpaceAuth(config)` is for custom auth-worker variants, not ordinary scaffold apps.
- Route/manifest helpers are authoritative in the `.d.ts`; preserve the scaffolded route ordering and identity stripping described in `architecture.md`.

### Server-side AI helpers

- Provider: `createDeepSpaceAI(env, provider, { authToken? })`. Pass the caller token for user billing; omission falls back to owner billing.
- Context: `prepareMessagesWithCompaction`, `truncateOldToolResults`, `applySlidingWindow`, `capToolResultSize`, `totalChars`, `turnsToCoreMessages`, `buildUiParts`, `unwrapToolOutput`, `makeDefaultSummarizer`, `DEFAULT_CONTEXT_CONFIG`; types `ChatContextConfig`, `ChatTurn`, `Summarizer`.
- History: `getChat`, `createChat`, `updateChat`, `deleteChatCascade`, `loadMessages`, `appendMessage`; types `ChatRow`, `ChatMessageRow`. These helpers bypass caller RBAC, so routes must verify ownership.
- Schemas/tools: `AI_CHATS_SCHEMA`, `AI_MESSAGES_SCHEMA`, `BUILT_IN_TOOLS`, `ToolSchema`, `applyAiToolDefaults`.

Load `ai-chat.md` for the workflow, billing choices, persistence boundary, and testing.

### Server action types

- `ActionHandler<TEnv>` receives `ActionContext<TEnv>`: `{ userId, params, tools, env, callerJwt }`.
- `ActionTools` exposes typed `create`, `update`, `remove`, `get`, `query`, `integration`, and `registerUser`; all bypass caller RBAC.
- `ActionResult<T>` is `{ success: true, data } | { success: false, error }`; narrow on `success`.

Load `server-actions.md` for authorization and caller-token forwarding.

### Bindings, storage, and metering

- `createScopedR2Handler` serves scoped R2 operations.
- `runMigrations(db, migrations)` appends idempotent D1 migrations; never reorder applied entries.
- `meterAi`, `meterVectorize`, `meterUsage`, and `COST_RATES` emit/price usage without breaking the caller when analytics is unavailable.
- Manifest exports: `AUTO_PROVISION_SENTINEL`, `AUTO_PROVISIONABLE_TYPES`, `ALLOWED_BINDING_TYPES`, `RESERVED_BINDING_NAMES`, `validateBindingManifest`, `isAutoProvision`, `bindingManifestFromOutputConfig`, `CustomBinding`.

Load `bindings.md` for declaration, provisioning, quota, and cleanup rules.

### Upstream worker proxy helpers

- `apiWorkerFetch(env, path, init?)` prefers `API_WORKER`, then the dev URL fallback.
- `platformWorkerFetch(env, pathOrRequest, init?)` prefers `PLATFORM_WORKER`, then the dev URL fallback.
- `authWorkerFetch(env, path, init?)` uses `AUTH_WORKER_URL`; auth has no service binding.
- Env types: `ApiWorkerEnv`, `PlatformWorkerEnv`, `AuthWorkerEnv`.

Keep these helpers in scaffolded workers. Deployed cross-worker requests require service bindings; plain `*.workers.dev` cross-worker URLs fail with Cloudflare 1042.

### Schema constants

- Users/messages: `USERS_COLUMNS`, `BASE_USERS_SCHEMA`, `CHANNELS_SCHEMA`, `MESSAGES_SCHEMA`, `REACTIONS_SCHEMA`, `CHANNEL_MEMBERS_SCHEMA`, `CHANNEL_INVITATIONS_SCHEMA`, `READ_RECEIPTS_SCHEMA`, `SYSTEM_COLLECTIONS`.
- Conversations/directory: `CONVERSATION_SCHEMAS`, `DIRECTORY_SCHEMAS`, `VOTING_SCHEMAS`.
- Workspace: `WORKSPACE_SCHEMAS`, `workspaceTeamsSchema`.
- Global registry: `GLOBAL_DO_TYPES`, `GLOBAL_DO_TYPE_NAMES`, `getGlobalDOType`, `getGlobalDOSchemas`, `RESERVED_COLLECTION_NAMES`.

Extend `BASE_USERS_SCHEMA`; do not replace it. Load `schemas.md` before defining permissions or shared scopes.

## Platform-backed helpers (`deepspace/server`)

- `captureScreenshot(env, url)` returns a fixed-profile capture or `null`; apps do not declare Browser Rendering for it. Screenshots are an optional targeted helper.
- `requireSubscription`, `getSubscription`, `cancelSubscription`, `refundInvoice` and `SubscriptionRequiredError`, `SubscriptionAuthError`, `CancelSubscriptionError`, `RefundError` implement server-side payment gates and operations. Load `payments.md`.

Preserve the starter's `/_deepspace/*` proxy: it keeps `APP_IDENTITY_TOKEN` out of the browser.

## Testing (`deepspace/testing`)

- `test`, `expect` are Playwright re-exports with the users fixture.
- `users(N | names[])` yields signed-in `MultiplayerUser`s and closes their contexts.
- Account selection: `loadAllTestAccounts`, `pickTestAccounts`, `findTestAccountByName`.
- Auth context: `ensureStorageState`, `newSignedInContext`, `getStatePathForEmail`, `readCachedState`.
- Types: `MultiplayerUser`, `UsersFixture`, `TestAccount`, `EnsureStorageStateOptions`.

Load `testing.md` for account-pool and multi-user workflow.
