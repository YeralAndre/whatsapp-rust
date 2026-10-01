# DECISIONS.md

## Architectural Decision Records (ADRs)

This document records the architectural and design decisions established for this fork of `whatsapp-rust`.

---

### ADR 0001: Expose `setting_unarchiveChats` via Typed Event and Symmetric Accessor

#### Status
Accepted

#### Context
WhatsApp protocol replicates account settings through Syncd (AppState) mutations. The auto-unarchive setting (`setting_unarchiveChats` in `RegularLow`) controls whether receiving a new message in an archived chat automatically unarchives it. WhatsApp servers do not send `ArchiveUpdate(false)` on incoming messages.

#### Decision
Implement both inbound and outbound support matching upstream's `AppStateSettings` convention:
1. Parse and dispatch inbound mutations into `Event::UnarchiveChatsSettingUpdate`.
2. Provide `client.app_state_settings().set_unarchive_chats(bool)` to push setting mutations.
3. Do not persist chat state or implement unarchive business logic inside `whatsapp-rust`; leave chat list state management to downstream consuming applications (ZapFast).

#### Consequences
- Fully compliant with upstream `src/features/app_state_settings.rs` design patterns.
- Downstream applications have complete visibility and control over chat archival behavior.

---

### ADR 0002: Append-Only `EventKind` Discipline and Rebase Protocol

#### Status
Accepted

#### Context
`EventKind` variants in `wacore::types::events` are mapped directly to bits in a `u128` bitmask (`EventInterest`, max capacity 128) and used for binary serialization stability. Inserting variants out-of-order or shifting existing discriminants breaks binary compatibility and causes rebase regressions.

#### Decision
1. Always append new custom variants strictly at the end of `EventKind`.
2. Keep the build-time capacity tripwire pointing to the last variant.
3. When rebasing on upstream commits that introduce new `EventKind` variants (such as `StatusPrivacyUpdate` = 75), resolve conflicts by shifting custom variants (e.g. `UnarchiveChatsSettingUpdate` = 76) and updating regression tests accordingly.

#### Consequences
- Prevents silent bitmask corruption or serialization mismatches.
- Clear, predictable conflict resolution procedure during upstream rebases.

---

### ADR 0003: Consumer Dependency Management via Pinned Revisions

#### Status
Accepted

#### Context
Downstream consumers like ZapFast depend on custom fork features (`UnarchiveChatsSettingUpdate`). Using loose branch tracking (`branch = "feat/unarchive-chats-setting"`) can cause build inconsistencies across different developer machines or CI runs if the branch is rebased or updated.

#### Decision
Consumers MUST pin the exact git commit revision (e.g. `rev = "80323773"`) in their `Cargo.toml`. When the fork is rebased or updated, the consumer's `Cargo.toml` is explicitly updated in lockstep.

#### Consequences
- Deterministic selection of the fork revision across developer machines and CI.
- Clean traceability between downstream releases and fork state.

---

### ADR 0004: Deliberate and Batched Upstream Synchronization

#### Status
Accepted

#### Context
Upstream `oxidezap/whatsapp-rust` undergoes frequent updates and internal refactors. Continuously merging every upstream commit creates maintenance churn and risk of regressions.

#### Decision
Synchronize with upstream deliberately rather than continuously:
1. Audit and sync on major upstream releases (e.g. `v0.8.0`).
2. Sync when critical connection, keepalive, or crypto fixes land upstream.
3. Sync when specifically required by downstream applications.
4. Keep PR preparation separate: maintain fork handoff docs in the fork branch, and construct clean doc-free branches when contributing upstream.

#### Consequences
- Maximizes stability for downstream applications.
- Minimizes merge overhead while maintaining upstream compatibility.
