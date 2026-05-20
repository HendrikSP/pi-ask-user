# Plan: answer `ask_user` from another extension

## Decisions (locked)

| Decision | Choice |
|---|---|
| Listener registration strategy | Global inbound handler, registered idempotently per `pi.events` bus using a module-level `WeakSet<object>` + module-level `Map<string, PendingCall>` |
| Pending-call lifetime | A pending entry exists only while `ctx.ui.custom()` is actively unresolved; it is deleted as soon as that await settles, before any dialog fallback begins |
| External cancel semantics | `ask_user:external_response` accepts `response: null` as an explicit cancel; a missing `response` property is invalid and ignored |
| `comment` when `allowComment: false` | Accept, normalize, and pass through — `allowComment` only governs the local TUI |
| Normalization strategy | Canonicalize external payloads through existing helpers (`createSelectionResponse()` / `createFreeformResponse()`) before resolving |
| Non-interactive / `options.length === 0` / post-custom dialog fallback | No-op — external responses only apply while the rich `custom()` UI promise is live |
| Event name | `ask_user:external_response` |

---

## Confirmed API facts

- `pi.events.on(channel, handler)` exists (`EventBus` interface) and returns `() => void`.
- Calling `done(response)` from the `ctx.ui.custom()` path resolves the Promise and closes/tears down the UI.
- `ctx.ui.custom()` may return `undefined` in RPC/headless-style dialog mode, after which the code falls back to `askViaDialogs(...)`.
- The unsubscribe-in-finally pattern already exists in the codebase (`removeOverlayInputListener`).

---

## Scope limit (explicit)

External responses apply **only** while a live `ask_user` call is waiting inside `ctx.ui.custom()`.

They do **not** apply to:
- `!ctx.hasUI` path
- `options.length === 0` path
- any later `askViaDialogs(...)` fallback after `ctx.ui.custom()` has already settled

All of those remain unchanged no-ops for inbound external events.

---

## Phase 1 — module-level state + external payload canonicalization

- [x] Promote `_toolCallId` to `toolCallId` in `execute`'s parameter list.
- [x] Define `PendingCall`:
  ```ts
  interface PendingCall {
    options: QuestionOption[];
    allowMultiple: boolean;
    allowFreeform: boolean;
    resolve: (result: AskUIResult | null) => void;
  }
  ```
- [x] Declare module-level state (outside `export default`):
  ```ts
  const pendingCalls = new Map<string, PendingCall>();
  const registeredExternalResponseBuses = new WeakSet<object>();
  ```
- [x] Add a helper that validates **and canonicalizes** inbound payloads before they reach the live call:
  ```ts
  function canonicalizeExternalResponse(
    pending: PendingCall,
    rawResponse: unknown,
  ): AskUIResult | null | undefined
  ```
  Semantics:
  - returns `AskUIResult` for a valid external answer
  - returns `null` for a valid external cancel (`response: null`)
  - returns `undefined` for invalid payloads that must be ignored
- [x] Validation/canonicalization rules for `canonicalizeExternalResponse(...)`:

### `rawResponse === null`

Treat as an explicit external cancel and return `null`.

### `kind: "selection"`

| Check | Rule |
|---|---|
| `selections` | Must be a non-empty `string[]` |
| Normalization | Trim each selection before matching / returning |
| `allowMultiple: false` | Reject if normalized `selections.length > 1` |
| Option matching | Every normalized selection must exactly match one of `pending.options[].title` |
| `comment` | If present, must be `string`, `null`, or `undefined`; pass through `createSelectionResponse(...)` for normalization |
| Canonical output | Use `createSelectionResponse(normalizedSelections, comment)`; if helper returns `null`, treat as invalid |

### `kind: "freeform"`

| Check | Rule |
|---|---|
| `allowFreeform` | Reject if `false` |
| `text` | Must be a string |
| Canonical output | Use `createFreeformResponse(text)`; if helper returns `null` after trimming, treat as invalid |

### Unknown / malformed payloads

Reject by returning `undefined`.

---

## Phase 2 — rich-UI path integration with correct lifetime boundaries

- [x] Keep the existing `!ctx.hasUI` and `options.length === 0` branches unchanged.
- [x] In the `ctx.ui.custom()` code path, keep overlay-input listener setup/cleanup in the existing outer scope.
- [x] Inside `customFactory`, wrap `done` in a settled-once guard so abort, timeout, local UI, and external response can safely race:
  ```ts
  let settled = false;
  const onceDone = (result: AskUIResult | null) => {
    if (settled) return;
    settled = true;
    done(result);
  };
  ```
- [x] Register the pending call **inside the live `custom()` lifecycle**:
  ```ts
  pendingCalls.set(toolCallId, {
    options,
    allowMultiple,
    allowFreeform,
    resolve: onceDone,
  });
  ```
- [x] Pass `onceDone` — not raw `done` — to:
  - abort listener
  - timeout callback
  - `AskComponent`
- [x] Wrap **only** the `await ctx.ui.custom(...)` call in a nested `try/finally` so cleanup happens before any later dialog fallback:
  ```ts
  let customResult: AskUIResult | null | undefined;
  try {
    customResult = await ctx.ui.custom(...);
  } finally {
    pendingCalls.delete(toolCallId);
  }
  ```
- [x] If `customResult === undefined`, proceed to the existing `askViaDialogs(...)` fallback exactly as today — with no pending-map entry remaining.

---

## Phase 3 — global inbound event handler, registered idempotently per bus

- [x] Add a small helper before `pi.registerTool(...)`:
  ```ts
  function ensureExternalResponseHandlerRegistered(pi: ExtensionAPI) {
    // guard with registeredExternalResponseBuses
  }
  ```
- [x] Registration behavior:
  - if `pi.events` has already been seen in `registeredExternalResponseBuses`, do nothing
  - otherwise add it to the `WeakSet` and register one permanent handler for that event bus
- [x] Call `ensureExternalResponseHandlerRegistered(pi)` before `pi.registerTool(...)`.
- [x] Handler logic:
  1. Verify `data` is an object.
  2. Extract `toolCallId`.
  3. Require that the payload **has its own** `response` property so `response: null` can be distinguished from a missing field.
  4. Look up `pendingCalls.get(toolCallId)`; if absent, return early.
  5. Run `canonicalizeExternalResponse(pending, data.response)`.
  6. If the helper returns `undefined`, ignore the event.
  7. Otherwise call `pending.resolve(canonicalResponse)`.
- [x] The handler must not emit `ask:answered` / `ask:cancelled` directly; it only resolves the live pending call so the existing result path emits those events once.

---

## Phase 4 — result behavior

- [x] No public result-shape changes.
- [x] Externally answered calls return the same `details.response` union shape as locally answered calls because they are canonicalized through the same helper functions first.
- [x] `ask:answered` fires for externally answered calls.
- [x] `ask:cancelled` fires for external `response: null` cancellation.
- [x] Late or stale external events become no-ops because the pending entry is gone once `ctx.ui.custom()` settles.
- [x] Headless mode, zero-option mode, overlay toggle behavior, abort, timeout, and dialog fallback behavior remain unchanged.

---

## Phase 5 — tests

### Harness updates

- [x] Extend the test event bus so `pi.events.on(...)` stores handlers and `pi.events.emit(...)` both:
  - invokes registered handlers
  - records outbound events for assertions
- [x] Keep `setupTool()` creating a fresh `pi.events` object per test so idempotent-per-bus registration is exercised naturally.

### Behavior tests

- [x] Registers the inbound handler once per event bus:
  - same `pi` / same `pi.events` initialized twice → still one handler
  - fresh `pi.events` object → new handler registered
- [x] External response resolves a live pending call:
  - start an unresolved `ctx.ui.custom()` call
  - emit `ask_user:external_response` with a valid selection response
  - assert `details.response` and `cancelled: false`
- [x] External freeform response resolves a live pending call when `allowFreeform: true`.
- [x] External `response: null` cancels a live pending call and emits `ask:cancelled`.
- [x] `ask:answered` is emitted for externally answered calls.
- [x] Stale `toolCallId` is ignored.
- [x] Invalid payloads are ignored:
  - missing `response` property
  - `allowMultiple: false` with multiple selections
  - selection title not in options list
  - `kind: "freeform"` with `allowFreeform: false`
  - unknown `kind`
  - whitespace-only freeform text
  - invalid `comment` type
- [x] `comment` is normalized and passed through even when `allowComment: false`.
- [x] Race: local UI resolution wins over a late external response.
- [x] Race: external response wins over a slower local UI interaction.
- [x] Abort still works unchanged.
- [x] Timeout still works unchanged.
- [x] Dialog fallback is not externally resolvable:
  - `ctx.ui.custom()` returns `undefined`
  - `askViaDialogs(...)` is still pending
  - external event is ignored
- [x] `!ctx.hasUI` and `options.length === 0` paths remain unaffected.
- [x] No pending-call bleed between tests (validated behaviorally via stale-event no-ops after prior tests complete).

---

## Phase 6 — docs / changelog

- [x] Update `README.md` to document the inbound `ask_user:external_response` event payload and its scope limit.
- [x] Update `CHANGELOG.md` with the new external-response capability.

---

## Acceptance

- [x] An external `ask_user:external_response` event can resolve a live pending `ask_user` tool call.
- [x] Valid external payloads are canonicalized to the same response shape used by local answers.
- [x] `response: null` performs an explicit external cancel; a missing `response` field is ignored.
- [x] Invalid or stale external payloads are silently ignored — wrong calls never resolve.
- [x] `comment` from external payloads flows through to the agent regardless of `allowComment`.
- [x] External resolution emits `ask:answered` / `ask:cancelled` via the existing result path, not from the event handler itself.
- [x] Local TUI, overlay toggle, abort, timeout, and dialog fallback all still work as before.
- [x] Non-interactive and `options.length === 0` paths are unaffected.
- [x] No pending-call entry survives past the end of `ctx.ui.custom()`.
