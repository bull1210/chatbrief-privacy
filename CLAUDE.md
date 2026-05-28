# ChatBrief — Codebase Guide

**App name:** ChatBrief (never "Briefly")  
**Active source folder:** `app/` (root of this repo)  
**Package:** `com.chatbrief.app`  
**GitHub remote:** `https://github.com/bull1210/briefly-ai` — main branch

---

## Stack

- Kotlin + Jetpack Compose + Material3
- Room + SQLCipher (encrypted local DB)
- DataStore (settings/prefs)
- Ktor (cloud AI calls — Gemini Flash or Claude Haiku)
- Hilt (DI)
- WorkManager (daily digest at 8 AM)
- NotificationListenerService (live WhatsApp group detection)

---

## Key Feature: Notification Listener (WhatsApp group auto-detection)

This is a core differentiator. The app uses Android's `NotificationListenerService` to read WhatsApp notifications in the background — no export needed for live groups.

**How it works:**
1. User grants Notification Access (Settings → Special App Access → Notification Access)
2. `WhatsAppNotificationListenerService` intercepts every WhatsApp notification
3. Extracts group name from `EXTRA_CONVERSATION_TITLE` (groups) or `EXTRA_TITLE` (1:1)
4. **Known group** → parses message via `NotificationParser`, inserts into DB, increments unread count
5. **Unknown group** → stages messages in `GroupDiscoveryRepository`, fires a "New group detected" push notification, adds to discovery list
6. User opens `GroupDiscoverySheet` (bottom sheet on HomeScreen), selects groups, taps Add
7. Group is created and all staged messages are flushed in immediately

**Key files:**
- `service/WhatsAppNotificationListenerService.kt` — main listener, routes known/unknown groups
- `service/WhatsAppNotificationListener.kt` — badge notifications + live-update notifications for known groups
- `service/NotificationParser.kt` — parses WhatsApp notification extras into message objects
- `data/repository/GroupDiscoveryRepository.kt` — stores discovered group names + staged messages
- `ui/home/GroupDiscoverySheet.kt` — UI to review and add detected groups (+ GroupDiscoveryViewModel)
- `service/BootReceiver.kt` — rebinds the listener after device reboot

**Permission:** `android.permission.BIND_NOTIFICATION_LISTENER_SERVICE` (declared in manifest, granted by user in system settings — cannot be requested at runtime like normal permissions)

**HomeScreen integration:**
- Banner shown if notification access not granted
- Snackbar fires when a new unknown group is detected live
- "Detected Groups" card visible when there are pending groups to add

---

## Other Import Method (manual)

- User exports WhatsApp chat as `.txt` or `.zip`
- Shares to ChatBrief → `ImportSheet` opens → file parsed → group created
- `WhatsAppParser.kt` handles DD/MM/YY format, lowercase am/pm, zip container, system messages

---

## UI Structure

- 4-tab bottom nav: Home / Brief / Archive / Settings
- `MainScaffold.kt` owns all bottom sheets: FilterSheet, ImportSheet, GroupDiscoverySheet
- `HomeScreen` → group list, add button, notification permission banner, snackbar
- `GroupDiscoverySheet` → detected groups list with select/exclude/restore + PostAdd phase

---

## Multilingual

All UI strings must be added to **all 6 strings.xml files**:  
`values/` (EN), `values-hi/` (HI), `values-ta/` (TA), `values-te/` (TE), `values-kn/` (KN), `values-mr/` (MR)  
Always use `stringResource()` — never hardcode strings.

---

## AI / Summarization

- Providers: Gemini Flash, Claude Haiku (user's own API key via DataStore)
- temperature: 0 on all providers
- max_tokens: 2048
- Claude requires `system` param (not just user message)
- Output: structured JSON (action items, events, announcements, FYI)
