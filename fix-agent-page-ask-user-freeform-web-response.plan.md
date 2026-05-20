Plan: fix `/agent` ask_user freeform responses not being accepted

Goal
- [ ] Web-submitted `ask_user` responses resolve the live tool call in the agent
- [ ] Freeform-only prompts work from `/agent`
- [ ] Frontend only treats the response as complete when the tool call actually closes

Phase 1 — wire external response into `pi-ask-user`
- [ ] Update `/home/pi/.pi/agent/npm/node_modules/pi-ask-user/index.ts`
- [ ] In `execute(toolCallId, params, ...)`, register `pi.events.on("ask_user:external_response", ...)`
- [ ] Ignore events whose `toolCallId` does not match the current pending call
- [ ] Validate incoming response against current params before accepting it
- [ ] Resolve the pending `ask_user` call from the external response
- [ ] Clean up the event listener on success, cancel, abort, timeout, and error

Phase 2 — cover freeform path
- [ ] Make external responses work for prompts with no options (`ctx.ui.input(...)` path)
- [ ] Accept `{ kind: "freeform", text }` only when freeform is allowed
- [ ] Accept `{ kind: "selection", ... }` only when selections match current option titles
- [ ] Prevent double-resolution when local TUI input and external response race

Phase 3 — fix `/agent` completion behavior
- [ ] In `server/frontend/src/pages/agent.rs`, do not treat transport ack as final completion
- [ ] Keep the pending state until the open `ask_user` tool call disappears from the stream
- [ ] Show transport rejection errors immediately
- [ ] Clear/hide the panel only after the tool call actually closes

Phase 4 — tests and checks
- [ ] Add/extend tests for matching `toolCallId` and ignoring stale external responses
- [ ] Add/extend tests for freeform external responses
- [ ] Add/extend tests for invalid selections / disallowed freeform
- [ ] Run `cargo clippy`
- [ ] Run `cargo check -p agent_dashboard_frontend --target wasm32-unknown-unknown`
- [ ] Run `npm run typecheck`

Acceptance
- [ ] A pending freeform `ask_user` prompt can be answered from `/agent`
- [ ] The live tool call resumes without `submit_prompt`
- [ ] Invalid or stale external responses are ignored or rejected correctly
- [ ] The frontend no longer shows false success on ack alone
