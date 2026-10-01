# AGENT_CONTEXT.md

> **Notice for Future Upstream PRs**: This file and related fork-specific documentation (`DECISIONS.md`, `ARCHITECTURE.md`) exist exclusively to provide operational handoff context for this fork and downstream consumers (ZapFast). When preparing a clean PR for `upstream/oxidezap`, create a separate branch that includes only the functional code changes (`wacore/src/types/events.rs`, `src/features/app_state_settings.rs`, `src/client/app_state.rs`) and excludes fork documentation.

---

## 1. Repository & Fork Overview

| Attribute | Details |
| :--- | :--- |
| **Upstream Repository** | [oxidezap/whatsapp-rust](https://github.com/oxidezap/whatsapp-rust.git) |
| **Fork Repository** | [YeralAndre/whatsapp-rust](https://github.com/YeralAndre/whatsapp-rust.git) |
| **Active Feature Branch** | `feat/unarchive-chats-setting` |
| **Current Upstream Base** | `6f07e3ab7f6a94764c23dfd6298c312b58419e7b` (`6f07e3ab`) |
| **Current Feature Commit** | `80323773` (`feat(appstate): expose unarchive chats setting`) |
| **Superseded Commit (Do Not Use)** | `7f1672d9` (pre-rebase version, obsolete) |
| **Downstream Consumer** | [YeralAndre/zapfast](https://github.com/YeralAndre/zapfast.git) (`fix/archived-chat-sync`) |
| **Downstream Pinned Revision** | `rev = "80323773"` |

---

## 2. Purpose & Implemented Feature

### Why this Fork Exists
WhatsApp does not automatically send `ArchiveUpdate(false)` when a new message arrives in an archived chat. Instead, WhatsApp replicates an account-wide setting (`setting_unarchiveChats` / `UnarchiveChatsSetting`) via AppState sync (Syncd). Downstream clients (such as ZapFast) need to know this user preference to determine whether incoming messages should automatically unarchive a chat locally or keep it archived.

Upstream `whatsapp-rust` defines the Protobuf message and schema entry for `setting_unarchiveChats`, but previously dropped inbound mutations as `AppStateDispatchOutcome::Unclaimed` and offered no public API or event.

### Feature Surface
1. **Inbound AppState Dispatch**: Inbound `setting_unarchiveChats` mutations in `Collection::RegularLow` are parsed and dispatched through the CoreEventBus as typed `Event::UnarchiveChatsSettingUpdate`.
2. **Event Payload (`wacore::types::events::UnarchiveChatsSettingUpdate`)**:
   - `unarchive_chats: bool` (true = auto-unarchive on new messages; false = keep chats archived).
   - `timestamp: DateTime<Utc>`
   - `action: Box<wa::sync_action_value::UnarchiveChatsSetting>`
   - `from_full_sync: bool` (true during initial connection bootstrap/snapshot; false for live push updates).
3. **Outbound Accessor**: `client.app_state_settings().set_unarchive_chats(unarchive: bool).await` creates and pushes a `SyncdMutation::Set` to WhatsApp servers in `Collection::RegularLow`.
4. **Observability**: `src/client/app_state.rs` maps `AppStateShape::UnarchiveChatsSetting` and logs scalar values as `MutationEffectDetail::Bool("unarchive_chats", b)` without exposing sensitive data.

### Consumer Semantics (ZapFast)
- `unarchive_chats == true`: Incoming messages to an archived chat trigger local unarchival.
- `unarchive_chats == false` ("Keep chats archived" setting enabled): Chat remains archived when receiving new messages.

---

## 3. Discriminant Numbering & Append-Only Rules

`wacore::types::events::EventKind` packs event discriminants into a `u128` bitmask (`EventInterest`, capacity 128). Every variant MUST be strictly append-only.

### Current Numbering (Post-Rebase on `6f07e3ab`)
- `EventKind::FavoritesUpdate` = `74` (Upstream)
- `EventKind::StatusPrivacyUpdate` = `75` (Added upstream in commit `7f7dc9f4`)
- `EventKind::UnarchiveChatsSettingUpdate` = `76` (Our feature)

```rust
// wacore/src/types/events.rs
pub enum EventKind {
    // ...
    FavoritesUpdate,             // 74
    StatusPrivacyUpdate,         // 75
    UnarchiveChatsSettingUpdate, // 76
}

// Build-time tripwire:
const _: () = assert!((EventKind::UnarchiveChatsSettingUpdate as u8) < EventKind::CAPACITY);
```

### Known Rebase Pitfall
When rebasing on new upstream commits that add `EventKind` variants:
1. **Do NOT discard upstream variants** (e.g. `StatusPrivacyUpdate`).
2. **Do NOT reuse discriminant `75`** for `UnarchiveChatsSettingUpdate`.
3. Keep `UnarchiveChatsSettingUpdate` as the last variant.
4. Keep the `const _: () = assert!(...)` tripwire pointing to `EventKind::UnarchiveChatsSettingUpdate`.
5. Update the regression test `assert_eq!(EventKind::UnarchiveChatsSettingUpdate as u8, <new_index>);` in `wacore/src/types/events.rs`.

---

## 4. Validation Status

The feature branch is validated after rebase:

```bash
# 1. Feature unit tests (10 passed, 0 failed)
cargo test -p whatsapp-rust --lib app_state_settings

# 2. Discriminant & append-only test (1 passed, 0 failed)
cargo test -p wacore --lib event_kind_discriminants_are_append_only

# 3. Workspace clippy checks
# PASS (Note: On Windows, linker_messages informational warnings may appear, which are excluded from -D warnings)
cargo clippy -p whatsapp-rust -p wacore --all-targets -- -D warnings
```

> **Platform Note (Windows)**: Running `cargo fmt --all` on Windows can encounter stack overflows due to massive generated files (`wacore/src/iq/abprops.rs`). For local formatting verification, run `rustfmt --edition 2024` on modified files directly.

### Runtime Validation & Verification Scope
- **Live Push Updates (Empirically Observed in Runtime)**: Tested with a linked WhatsApp session in ZapFast. Toggling "Keep chats archived" in the WhatsApp mobile application immediately delivered live `UnarchiveChatsSettingUpdate` events with `from_full_sync = false`:
  - Mobile Setting ON ("Keep chats archived"): `unarchive_chats = false`
  - Mobile Setting OFF: `unarchive_chats = true`
  - Downstream consumer in ZapFast properly received, persisted, and honored the setting.
- **Full Sync / Bootstrap Behavior (Implemented & Test-Covered)**:
  - The initial sync code path is designed to pass `event_full_sync = true` to `UnarchiveChatsSettingUpdate` when bootstrapping collection snapshots.
  - This is verified via unit tests (`inbound_set_dispatches_unarchive_chats_flag`), though the runtime verification session was incremental and did not explicitly trigger an initial full-sync snapshot bootstrap.

---

## 5. Upstream Synchronization Strategy

- **Do NOT rebase on every upstream commit blindly**: Upstream moves fast with internal refactors. Keep the fork pinned to a known-good revision in downstream consumers.
- **When to update**:
  - Upstream releases (e.g. `v0.8.0`).
  - Critical protocol/transport fixes (such as keepalive or reconnect fixes).
  - Explicit requirement from ZapFast.
- **Workflow for updating**:
  1. `git fetch upstream`
  2. Audit new commits (`git log HEAD..upstream/main`).
  3. Rebase `feat/unarchive-chats-setting` onto `upstream/main`.
  4. Fix discriminant numbering if upstream added `EventKind` entries.
  5. Run tests & clippy.
  6. Force-push with lease (`git push origin feat/unarchive-chats-setting --force-with-lease`).
  7. Update consumer `rev` in ZapFast's `Cargo.toml`.
