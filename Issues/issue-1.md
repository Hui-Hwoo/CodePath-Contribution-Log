# Contribution 1: Implement Dashboard 2FA Approval Endpoints

**Contribution Number:** 1  
**Student:** Hui Hwoo  
**Issue:** [innerwarden #71 — Implement dashboard 2FA approval endpoints](https://github.com/InnerWarden/innerwarden/issues/71)  
**Status:** Phase II [Complete]

---

## Why I Chose This Issue

This issue sits at the intersection of security engineering and API design, two areas I want to deepen. The project already has a working TOTP 2FA flow through Telegram, so the task isn't to invent a new auth mechanism — it's to expose the same state machine through the dashboard's REST API. That makes it a great fit for understanding how existing patterns get extended without duplicating security logic.

I also wanted to work in a real Rust/Axum codebase. Rust's type system enforces correctness at compile time in a way that directly teaches security thinking: you can't accidentally skip an auth check because the compiler won't let the handler signature compile.

---

## Understanding the Issue

### Problem Description

The agent already supports TOTP 2FA for sensitive operations (allowlist changes, detector disabling, mode switching) via Telegram. However, the dashboard — the primary operator UI — has no equivalent API endpoints. Operators who prefer the web dashboard over Telegram cannot approve or deny pending security actions without switching to the bot.

### Expected Behavior

Three REST endpoints in `crates/agent/src/dashboard/`:
- `GET /api/2fa/pending` — returns a list of pending approval requests (with deadlines) so the operator can see what's waiting
- `POST /api/2fa/approve/{approval_id}` — approves the action after verifying the operator's session token AND their TOTP code
- `POST /api/2fa/deny/{approval_id}` — rejects the action (no TOTP required — refusing is never dangerous)

### Current Behavior

These three endpoints do not exist. A `GET /api/2fa/pending` returns a 404.

### Affected Components

- `crates/agent/src/dashboard/mod.rs` — router registration
- `crates/agent/src/dashboard/state.rs` — `DashboardState` struct (needs new fields)
- `crates/agent/src/telegram.rs` — reference for `ApprovalResult` and `PendingConfirmation` data shapes
- `crates/agent/src/dashboard/agent_api.rs` — source of `verify_dashboard_totp()` helper we reuse

---

## Reproduction Process

### Environment Setup

The project is a Rust workspace. Setup steps:
1. Install Rust toolchain via `rustup` (stable)
2. Fork `InnerWarden/innerwarden` on GitHub
3. Clone your fork: `git clone https://github.com/Hui-Hwoo/innerwarden.git`
4. Navigate to the project root
5. Run `cargo check -p innerwarden-agent` to confirm the build baseline is clean

No database or external service is required for a `cargo check`.

### Steps to Reproduce

1. Clone the upstream repo (or your fork before this fix)
2. Start the agent in development mode (or use `cargo test` to run unit tests)
3. Send a `GET` request to `http://localhost:<port>/api/2fa/pending`
   ```
   curl -u admin:password http://localhost:8080/api/2fa/pending
   ```
4. **Observed result:** HTTP 404 — the endpoint does not exist
5. Attempt `POST /api/2fa/approve/any-id` → same 404
6. Attempt `POST /api/2fa/deny/any-id` → same 404

### Reproduction Evidence

- **Branch in my fork:** [feature/dashboard-2fa-approval-endpoints](https://github.com/Hui-Hwoo/innerwarden/tree/feature/dashboard-2fa-approval-endpoints)
- **Draft PR:** [PR #1 on Hui-Hwoo/innerwarden](https://github.com/Hui-Hwoo/innerwarden/pull/1)
- **My findings:** The existing Telegram 2FA flow stores pending confirmations in `AgentState.pending_confirmations` (an in-memory `HashMap`), which is only accessible to the agent loop — the dashboard has no bridge to it. The fix introduces a separate `pending_approvals` map on `DashboardState` that the agent loop can populate, and three handlers that read/write it.

---

## Solution Approach

### Analysis

The root cause is a missing bridge between the existing TOTP infrastructure and the dashboard API layer:

1. `verify_dashboard_totp()` already exists in `agent_api.rs` — it validates a 6-digit TOTP code against the configured secret and is reused by `trust-exec` and orphan-resolution endpoints.
2. `DashboardState` has `two_factor: Arc<TwoFactorSettings>` — the 2FA configuration is already threaded through.
3. What's missing is: (a) a shared store for pending approval requests, (b) an optional back-channel to notify the agent loop of decisions, and (c) the three handler functions wired to routes.

### Proposed Solution

Add a `pending_approvals: Arc<Mutex<HashMap<String, TwoFaPendingRequest>>>` to `DashboardState`. The agent loop can insert entries here when a sensitive operation needs approval; the dashboard handlers read and remove entries when the operator acts.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The dashboard needs GET/POST endpoints backed by a shared in-memory store, with TOTP verification on approve and audit logging on both approve and deny.

**Match:** Existing patterns in the codebase:
- `verify_dashboard_totp()` in `agent_api.rs` — reused directly for the TOTP gate
- `Option<axum::Extension<AuthenticatedUser>>` pattern — extract operator username from request extension
- `append_admin_action()` — write audit rows to the compliance JSONL
- Separate sub-module per feature area (e.g., `fleet.rs`, `push.rs`) — created `two_fa.rs`

**Plan:**

1. Add `TwoFaPendingRequest`, `DashboardApprovalOutcome`, and `TwoFaActionRequest` types to `dashboard/state.rs`
2. Add `pending_approvals` and `approval_outcome_tx` fields to `DashboardState` (and its test helper)
3. Create `dashboard/two_fa.rs` with three handlers: `api_2fa_pending`, `api_2fa_approve`, `api_2fa_deny`
4. Register `mod two_fa` and import `use two_fa::*` in `dashboard/mod.rs`
5. Wire three routes in the `dashboard` router (behind `auth_layer`)
6. Confirm `cargo check` passes; add unit tests inside `two_fa.rs`

**Implement:** [feature/dashboard-2fa-approval-endpoints](https://github.com/Hui-Hwoo/innerwarden/tree/feature/dashboard-2fa-approval-endpoints) — commit `2cc09dab`

**Review:**
- [x] Follows module-per-feature pattern used throughout the dashboard (`fleet.rs`, `push.rs`, etc.)
- [x] Reuses `verify_dashboard_totp()` — no duplicated TOTP logic
- [x] Audit rows written on approve and deny via `append_admin_action()`
- [x] `cargo check -p innerwarden-agent` passes
- [x] CSRF protection applies automatically (routes are inside `auth_layer`/`csrf_protection` middleware chain)
- [x] Denial path does not require TOTP (consistent with the security principle: refusal is never dangerous)

**Evaluate:** Unit tests cover `is_expired`, map insert/remove, `api_2fa_pending` response shape, and expired-entry filtering. Manual testing with `curl` against a running agent.

---

## Testing Strategy

### Unit Tests

- [x] `is_expired_returns_true_past_deadline` — verify deadline logic works
- [x] `is_expired_returns_false_before_deadline` — happy path for active requests
- [x] `pending_approvals_map_insert_and_remove` — confirm the Arc<Mutex<HashMap>> mechanics
- [x] `api_2fa_pending_returns_empty_when_no_requests` — clean-state response
- [x] `api_2fa_pending_filters_expired_requests` — expired entries are hidden from operators

### Integration Tests

- [ ] POST /api/2fa/approve with wrong TOTP returns 401 Unauthorized
- [ ] POST /api/2fa/approve with valid TOTP removes entry and returns 200 with `approved: true`
- [ ] POST /api/2fa/deny removes entry and returns 200 with `approved: false`
- [ ] Both approve and deny on a missing ID return 404
- [ ] Approve on an expired entry returns 410 Gone

### Manual Testing

Run the agent, insert a fake pending approval via test code, then exercise the three endpoints with curl. Verify audit rows appear in the admin-actions JSONL.

---

## Implementation Notes

### Week 1 Progress

Explored the codebase to understand how the Telegram 2FA flow works (`telegram.rs`, `bot_actions.rs`) and how the dashboard handles existing sensitive endpoints (`agent_api.rs` orphan-resolution, `actions.rs` trust-exec). Identified `verify_dashboard_totp()` as the reusable gate. Implemented all three endpoints, confirmed `cargo check` passes, and opened the draft PR.

### Code Changes

- **Files modified:**
  - `crates/agent/src/dashboard/state.rs` — new types + two new fields on `DashboardState`
  - `crates/agent/src/dashboard/mod.rs` — module registration + three route entries
- **Files created:**
  - `crates/agent/src/dashboard/two_fa.rs` — handler implementations + unit tests
- **Key commits:** [`2cc09dab`](https://github.com/Hui-Hwoo/innerwarden/commit/2cc09dab)
- **Approach decisions:** Placed types in `state.rs` (not `two_fa.rs`) to avoid a circular import — `two_fa.rs` uses `super::*` which already includes `state::*`.

---

## Pull Request

**PR Link:** [https://github.com/Hui-Hwoo/innerwarden/pull/1](https://github.com/Hui-Hwoo/innerwarden/pull/1)

**PR Description:** Draft PR implementing the three 2FA endpoints. Security model, audit logging, TOTP gate, and test plan are documented in the PR body.

**Maintainer Feedback:**

- *(awaiting review)*

**Status:** Draft — awaiting maintainer review

---

## Learnings & Reflections

### Technical Skills Gained

- Reading a large Rust codebase and identifying the right extension points
- Axum handler signatures, extractor ordering, and the `Option<Extension<T>>` pattern for optional auth extensions
- How `Arc<Mutex<HashMap>>` provides shared mutable state across concurrent HTTP handlers without data races
- How audit logging is wired through a hash-chained JSONL for compliance

### Challenges Overcome

- Initial attempt used `Extension<AuthenticatedUser>` directly in the handler signature, which caused a compile error because `Extension` is not in scope via `super::*`. Traced the issue to `agent_api.rs` where the existing pattern uses `Option<axum::Extension<...>>` with the full path.
- Avoiding circular imports: defining `TwoFaPendingRequest` in `two_fa.rs` caused a cycle because `state.rs` needed it. Moved the types to `state.rs` to break the cycle.

### What I'd Do Differently Next Time

Start by tracing one complete existing handler end-to-end (state → route → handler → audit) before writing any new code, to avoid discovering patterns like the `Option<axum::Extension>` idiom only after hitting a compile error.

---

## Resources Used

- [Axum extractors documentation](https://docs.rs/axum/latest/axum/extract/index.html)
- Existing `verify_dashboard_totp()` and `api_orphan_clear()` handlers in `agent_api.rs` — used as implementation templates
- `telegram.rs` `PendingConfirmation` struct — reference for the approval data model
