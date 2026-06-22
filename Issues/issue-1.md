# Contribution 1: Implement Dashboard 2FA Approval Endpoints

**Contribution Number:** 1  
**Student:** Hui Hwoo  
**Issue:** [innerwarden #71 — Implement dashboard 2FA approval endpoints](https://github.com/InnerWarden/innerwarden/issues/71)  
**Status:** Phase III [Complete]

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
- `GET /api/2fa/pending` — returns a list of pending approval requests (sorted by deadline) so the operator can see what's waiting
- `POST /api/2fa/approve/:approval_id` — approves the action after verifying the operator's session token AND their TOTP code
- `POST /api/2fa/deny/:approval_id` — rejects the action (no TOTP required — refusing is never dangerous)

### Current Behavior

These three endpoints do not exist. `GET /api/2fa/pending` returns a 404.

### Affected Components

- `crates/agent/src/dashboard/mod.rs` — router registration and background cleanup task
- `crates/agent/src/dashboard/state.rs` — `DashboardState` struct (new fields and types)
- `crates/agent/src/dashboard/two_fa.rs` — **new file** with handler implementations
- `crates/agent/src/telegram.rs` — reference for `ApprovalResult` / `PendingConfirmation` data shapes
- `crates/agent/src/dashboard/agent_api.rs` — source of `verify_dashboard_totp()` helper

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

1. Clone the upstream repo (or your fork on `main` before this fix)
2. Send a `GET` request to `http://localhost:<port>/api/2fa/pending`
   ```bash
   curl -u admin:password http://localhost:8080/api/2fa/pending
   ```
3. **Observed result:** HTTP 404 — the endpoint does not exist
4. Attempt `POST /api/2fa/approve/any-id` → same 404
5. Attempt `POST /api/2fa/deny/any-id` → same 404

### Reproduction Evidence

- **Branch in my fork:** [feature/dashboard-2fa-approval-endpoints](https://github.com/Hui-Hwoo/innerwarden/tree/feature/dashboard-2fa-approval-endpoints)
- **Draft PR:** [PR #1 on Hui-Hwoo/innerwarden](https://github.com/Hui-Hwoo/innerwarden/pull/1)
- **My findings:** The existing Telegram 2FA flow stores pending confirmations in `AgentState.pending_confirmations`, which is only accessible to the agent loop — the dashboard has no bridge to it. The fix introduces a separate `pending_approvals` map on `DashboardState` that the agent loop can populate, and three handlers that read/write it.

---

## Solution Approach

### Analysis

The root cause is a missing bridge between the existing TOTP infrastructure and the dashboard API layer:

1. `verify_dashboard_totp()` already exists in `agent_api.rs` — validated by `trust-exec` and orphan-resolution endpoints.
2. `DashboardState` has `two_factor: Arc<TwoFactorSettings>` — the 2FA config is already threaded through.
3. What's missing: (a) a shared store for pending approval requests, (b) an optional back-channel to notify the agent loop of decisions, and (c) the three handler functions wired to routes.

### Proposed Solution

Add `pending_approvals: Arc<Mutex<HashMap<String, TwoFaPendingRequest>>>` to `DashboardState`. The agent loop inserts entries here when a sensitive operation needs approval; dashboard handlers read and remove entries when the operator acts.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The dashboard needs GET/POST endpoints backed by a shared in-memory store, with TOTP verification on approve and audit logging on both approve and deny.

**Match:** Existing patterns in the codebase:
- `verify_dashboard_totp()` in `agent_api.rs` — reused directly for the TOTP gate
- `Option<axum::Extension<AuthenticatedUser>>` — extract operator username from request extension
- `append_admin_action()` — write audit rows to the compliance JSONL
- Separate sub-module per feature area (`fleet.rs`, `push.rs`) — created `two_fa.rs`

**Plan:**
1. Add `TwoFaPendingRequest`, `DashboardApprovalOutcome`, `TwoFaActionRequest` types to `dashboard/state.rs`
2. Add `pending_approvals` and `approval_outcome_tx` fields to `DashboardState`
3. Create `dashboard/two_fa.rs` with three handlers and unit tests
4. Register `mod two_fa` + `use two_fa::*` in `dashboard/mod.rs`
5. Wire routes in the `dashboard` router (behind `auth_layer`)
6. Add background cleanup for `pending_approvals` in `serve()`

**Implement:** [feature/dashboard-2fa-approval-endpoints](https://github.com/Hui-Hwoo/innerwarden/tree/feature/dashboard-2fa-approval-endpoints)

**Review:**
- [x] Module-per-feature pattern (`fleet.rs`, `push.rs`, etc.)
- [x] Reuses `verify_dashboard_totp()` — no duplicated TOTP logic
- [x] Audit rows on approve and deny via `append_admin_action()`
- [x] `cargo check -p innerwarden-agent` passes
- [x] CSRF protection automatic (routes inside `csrf_protection` middleware chain)
- [x] Denial does not require TOTP
- [x] `approval_id` validated (empty / > 128 bytes → 400)
- [x] Raw TOTP code not stored/passed downstream (CWE-532)
- [x] Background cleanup task prevents unbounded map growth

**Evaluate:** 14 unit tests in `two_fa.rs`. Manual testing with `curl` against a running agent.

---

## Testing Strategy

### Unit Tests (14 passing)

| Test | What it covers |
|------|---------------|
| `is_expired_returns_true_past_deadline` | Deadline logic — expired |
| `is_expired_returns_false_before_deadline` | Deadline logic — active |
| `pending_approvals_map_insert_and_remove` | `Arc<Mutex<HashMap>>` mechanics |
| `api_2fa_pending_returns_empty_when_no_requests` | Clean-state response shape |
| `api_2fa_pending_filters_expired_requests` | Expired entries hidden from operator |
| `api_2fa_pending_sorts_by_deadline_ascending` | Most urgent request listed first |
| `api_2fa_approve_rejects_empty_approval_id` | 400 on empty id |
| `api_2fa_approve_rejects_oversized_approval_id` | 400 on id > 128 bytes |
| `api_2fa_approve_returns_not_found_for_unknown_id` | 404 for missing entry |
| `api_2fa_approve_returns_gone_for_expired_entry` | 410 when deadline passed |
| `api_2fa_deny_rejects_empty_approval_id` | 400 on empty id |
| `api_2fa_deny_accepts_no_body` | 200 when client sends no JSON body |
| `api_2fa_deny_returns_not_found_for_unknown_id` | 404 for missing entry |
| `api_2fa_deny_removes_entry_from_map` | Entry gone after deny |

### Build Validation
```
cargo check -p innerwarden-agent   # ✅ zero errors
```

### Manual Testing

```bash
# 1. List pending (empty state)
curl -u admin:pass -H "X-Requested-With: XMLHttpRequest" \
  http://localhost:8080/api/2fa/pending

# 2. Approve with wrong TOTP → 401
curl -X POST -u admin:pass -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/json" -d '{"totp":"000000"}' \
  http://localhost:8080/api/2fa/approve/some-id

# 3. Deny with no body → 200 (if entry exists)
curl -X POST -u admin:pass -H "X-Requested-With: XMLHttpRequest" \
  http://localhost:8080/api/2fa/deny/some-id
```

---

## Implementation Notes

### Week 1 Progress — Initial implementation

Explored the codebase to trace how the Telegram 2FA flow works (`telegram.rs`, `bot_actions.rs`) and how the dashboard handles existing sensitive endpoints (`agent_api.rs` orphan-resolution, `actions.rs` trust-exec). Identified `verify_dashboard_totp()` as the reusable gate.

Implemented all three endpoints, confirmed `cargo check` passes, and opened Draft PR #1.

### Week 3 Progress — Code review and refinements

After a self-review of the initial commit I found and fixed five issues:

1. **Race / silent-discard bug in `api_2fa_approve`**: The original code validated TOTP, then removed from map, then checked expiry — meaning an expired entry was silently dropped with no error returned to the caller. Fixed by peeking at the entry first (check existence + expiry), then validating TOTP, then doing the atomic remove.

2. **CWE-532 — raw TOTP in struct**: `DashboardApprovalOutcome.totp_supplied: String` stored the actual 6-digit code. Replaced with `totp_verified: bool` — the agent loop only needs to know whether verification passed, not the digit string itself.

3. **Missing `approval_id` validation**: The endpoint accepted any string without length/emptiness checks. Added a `MAX_APPROVAL_ID_LEN = 128` guard consistent with the orphan-id validation in `agent_api.rs`.

4. **Deny body was required**: `Json<TwoFaActionRequest>` in the deny handler caused axum to return 422 if no body was sent. Changed to `Option<Json<TwoFaActionRequest>>` — a `reason` is optional.

5. **Background cleanup missing**: The module header comment promised a cleanup task in `serve()` for expired entries but none existed. Added it to the existing 60-second session/advisory cleanup loop.

Also added: sorting by deadline ascending, richer success responses (`action_description` in body), and 9 additional unit tests (14 total).

### Code Changes

- **Files modified:**
  - `crates/agent/src/dashboard/state.rs` — `Mutex` import; new types; two new `DashboardState` fields; `totp_supplied → totp_verified`
  - `crates/agent/src/dashboard/mod.rs` — module + routes; `cleanup_pending_approvals` clone; cleanup task extended
- **Files created:**
  - `crates/agent/src/dashboard/two_fa.rs` — handler implementations + 14 unit tests

- **Key commits:**
  - [`2cc09dab`](https://github.com/Hui-Hwoo/innerwarden/commit/2cc09dab) — Initial implementation
  - [`c89cf83c`](https://github.com/Hui-Hwoo/innerwarden/commit/c89cf83c) — Code-review fixes (race, CWE-532, validation, optional body, cleanup task)

- **Approach decisions:**
  - Types defined in `state.rs` (not `two_fa.rs`) to avoid circular import — `two_fa.rs` uses `super::*` which already pulls in `state::*`.
  - `approval_outcome_tx` is `Option<mpsc::Sender<...>>` so the dashboard can run standalone (e.g. in tests) without requiring the agent loop channel to be wired up.

---

## Pull Request

**PR Link:** [https://github.com/Hui-Hwoo/innerwarden/pull/1](https://github.com/Hui-Hwoo/innerwarden/pull/1)

**PR Description:** Full implementation notes, operation order rationale, security considerations table, per-commit summary, and manual test plan documented in the PR body.

**Maintainer Feedback:**

- *(awaiting review)*

**Status:** Draft — awaiting maintainer review

---

## Learnings & Reflections

### Technical Skills Gained

- Reading a large Rust codebase and identifying the right extension points without disrupting existing patterns
- Axum handler signatures, extractor ordering, and the `Option<Extension<T>>` pattern for optional auth extensions
- How `Arc<Mutex<HashMap>>` provides shared mutable state across concurrent async HTTP handlers without data races
- TOCTOU (time-of-check/time-of-use) races in API handlers and how to reason about them
- CWE-532 — why credentials should never appear in structs that could be serialised or logged
- How audit logging is wired through a hash-chained JSONL for compliance surfaces

### Challenges Overcome

- **Circular import**: defining `TwoFaPendingRequest` in `two_fa.rs` caused a compile error because `state.rs` needed it. Moved the types to `state.rs`.
- **`Extension` not in scope**: Initial attempt used `Extension<AuthenticatedUser>` directly in the handler signature. Traced the pattern in `agent_api.rs` which uses `Option<axum::Extension<...>>` with the full path.
- **Move after use**: `cleanup_pending_approvals` clone had to come before `state` was moved into the router builder. Rust's ownership model made this error unambiguous.
- **Silent discard bug**: Caught during self-review — the initial operation order meant expired entries were removed from the map but no error was returned to the caller.

### What I'd Do Differently Next Time

Trace one complete handler end-to-end (state field → route → handler → audit row) before writing anything new. The patterns (`Option<axum::Extension>`, type placement to avoid circular imports) become obvious from reading rather than from hitting compile errors.

---

## Resources Used

- [Axum extractors documentation](https://docs.rs/axum/latest/axum/extract/index.html)
- Existing `verify_dashboard_totp()` and `api_orphan_clear()` handlers in `agent_api.rs` — implementation templates
- `telegram.rs` `PendingConfirmation` struct — reference for the approval data model
- [CWE-532: Insertion of Sensitive Information into Log File](https://cwe.mitre.org/data/definitions/532.html)
