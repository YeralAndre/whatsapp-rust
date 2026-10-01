# ARCHITECTURE.md

## AppState Settings Architecture

This document describes the end-to-end integration of AppState synchronization settings, focusing on `setting_unarchiveChats`, from WhatsApp Syncd protocol mutations to typed domain events and downstream consumption.

---

## 1. Architectural Layers & Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                       WhatsApp Server                       │
└──────────────────────────────┬──────────────────────────────┘
                               │ Syncd IQ / Push Notifications
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 whatsapp-rust (Connection Layer)            │
│  - Parses WAPatchList (RegularLow Collection)               │
│  - Decodes SyncdMutations with LTHash                       │
└──────────────────────────────┬──────────────────────────────┘
                               │ Mutation: ["setting_unarchiveChats"]
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 AppState Settings Dispatcher                │
│  (src/features/app_state_settings.rs)                       │
│  - Validates schema against UNARCHIVE_CHATS_SETTING         │
│  - Extracts SyncActionValue.unarchive_chats_setting         │
│  - Emits Event::UnarchiveChatsSettingUpdate                 │
└──────────────────────────────┬──────────────────────────────┘
                               │ CoreEventBus
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Downstream Application (ZapFast)            │
│  - Listens to Event::UnarchiveChatsSettingUpdate            │
│  - Persists setting: auto_unarchive_chats = val             │
│  - When new message arrives in archived chat:               │
│      if auto_unarchive_chats { unarchive_chat() }           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Inbound Pipeline (Sync & Live Push)

### Full Sync (Initial Bootstrap — Implemented & Test-Covered)
1. On initial pairing or key reconnect, the client requests a snapshot of collection `RegularLow`.
2. `src/client/app_state.rs` processes the patch list with `full_sync = true`.
3. Every mutation is handed to `dispatch_app_state_mutation_inner(..., event_full_sync = true, ...)`.
4. `dispatch_app_state_setting_mutation_outcome` matches `schemas::UNARCHIVE_CHATS_SETTING.name` (`"setting_unarchiveChats"`).
5. Dispatches `Event::UnarchiveChatsSettingUpdate` with `from_full_sync: true`.
6. Consuming applications are designed to receive the active account setting immediately on initial bootstrap without requiring user interaction (covered by unit test suites).

### Live Push Updates (Empirically Verified in Runtime)
1. When the user changes "Keep chats archived" on their mobile device or another linked device, WhatsApp sends an incremental Syncd patch.
2. The mutation is processed with `event_full_sync = false`.
3. Dispatches `Event::UnarchiveChatsSettingUpdate` with `from_full_sync: false`.
4. The consumer dynamically updates its cached preference in real time (observed and validated live with ZapFast).

---

## 3. Outbound Pipeline (Setting Mutation)

To update the setting from the client:

1. Downstream calls `client.app_state_settings().set_unarchive_chats(unarchive).await`.
2. `AppStateSettings` constructs a `wa::SyncActionValue` wrapping `UnarchiveChatsSetting { unarchive_chats: Some(unarchive) }` with current timestamp.
3. Calls `client.send_app_state_action(&schemas::UNARCHIVE_CHATS_SETTING, &[], &value)`.
4. Encodes and pushes a `SyncdMutation::Set` to the `RegularLow` collection on the WhatsApp server.

---

## 4. Privacy & Observability Policy

Following the repository's redaction policy in `src/client/app_state.rs`:
- Syncd setting names are classified under `AppStateShape::UnarchiveChatsSetting`.
- Telemetry logs project only the scalar boolean via `MutationEffectDetail::Bool("unarchive_chats", b)`.
- No sensitive user data or unencrypted action buffers are exposed in logs.
