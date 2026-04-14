# AI Accountability Partner MVP Implementation (Revised Plan)

Develop an Android-first mobile application using React Native (Expo) that serves as an LLM-powered accountability partner. The app features a chat-based UI, proactive task notifications, push delivery via FCM, and a custom Android native module to monitor distracting app usage and intervene in real-time. Backend: Supabase (Auth, Postgres, Realtime, Edge Functions, pg_cron). AI orchestration: Vercel AI SDK with OpenAI function calling.

## Dev Workflow Note (Read First)
Because we ship custom native modules (Phase 3), **Expo Go cannot run this project past Phase 3**. From Phase 3 onward we build and run via an **EAS Dev Client** (`npx expo run:android` locally, or `eas build --profile development` for a signed APK). Phases 1–2 can use Expo Go.

## Scope Deferrals (Tracked, Not in MVP)
- **Social login (Google/Apple):** mvp_specs calls for it, but we ship email/password first for faster local testing. Social auth is a Phase 8 follow-up.
- **iOS:** Codebase stays cross-platform, but native module + `UsageStatsManager` replacement for iOS is out of scope.

---

## Phase 1: Environment Setup & Project Initialization
1. **Initialize Expo Application**: `npx create-expo-app` with the TypeScript template.
2. **Install Core Dependencies**: `zustand`, `nativewind`, `@supabase/supabase-js`, `@react-navigation/native` + stack/tabs, `expo-notifications`, `expo-device`, `expo-secure-store`.
3. **Initialize Supabase Local Development**: Install the Supabase CLI and run `supabase init`. Start Docker and `supabase start` to confirm the local stack boots.
4. **Environment Variables**: Create `.env.local` with local Supabase URL + anon key. Placeholder entries for `OPENAI_API_KEY` and `NATIVE_WEBHOOK_SECRET` (filled in Phase 6).

## Phase 2: Database Schema & Authentication Modeling
5. **Create Migration**: `supabase migration new init_schema`.
6. **Define Tables**:
   - `users`: profile, synced with `auth.users`, includes `persona_prompt TEXT` (user-defined AI persona) and `timezone TEXT`.
   - `goals`: high-level goals (title, description, status, user_id).
   - `tasks`: scheduled work blocks (goal_id, start_at, end_at, status, title).
   - `messages`: chat log (role: user/assistant/system, content, user_id, created_at).
   - `message_summaries`: rolling summary paragraph per user (user_id, summary_text, covers_up_to).
   - `distracting_apps`: (user_id, package_name, app_label).
   - `push_tokens`: (user_id, expo_push_token, platform, updated_at).
7. **Row Level Security**: Enable RLS on every table; policies restrict reads/writes to `auth.uid() = user_id`.
8. **Auth Trigger**: Postgres function + trigger on `auth.users` insert that seeds a `public.users` row.
9. **Apply Migrations**: `supabase db reset` locally to verify migration idempotency.

## Phase 3: Android Native Modules (Usage & App Lists)
10. **EAS Prebuild**: `npx expo prebuild -p android` to generate `/android`. Commit the config plugin approach so prebuild is reproducible.
11. **App List Module (Kotlin)**: Native module querying `PackageManager` for installed user apps (label, package, icon base64).
12. **Usage Module (Kotlin)**: `UsageStatsManager`-based poller that returns the current foreground package.
13. **Permissions**:
    - Manifest: `PACKAGE_USAGE_STATS`, `QUERY_ALL_PACKAGES`, `POST_NOTIFICATIONS`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_SPECIAL_USE` (Android 14+), `INTERNET`.
    - Runtime: guide the user through Settings → Usage Access grant (cannot be auto-granted).
14. **Foreground Service**: Android foreground service with a persistent notification ("Accountability Partner is watching"). Poll `UsageStatsManager` every 10–15s while an active task is live. Request battery optimization exemption via `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` on first enable.
15. **Distraction Webhook**: On detection, native code POSTs to the `distraction_alert` Edge Function with an HMAC signature derived from `NATIVE_WEBHOOK_SECRET` (stored in Android Keystore, provisioned on login).

## Phase 4: State Management & Navigation Setup
16. **Zustand Stores**: `authStore`, `chatStore` (messages + realtime sub handle), `taskStore` (today's tasks), `settingsStore` (persona, distracting apps cache).
17. **Navigation**: React Navigation with an Auth stack (Login/Signup) and an authenticated Tab navigator (Dashboard, Chat, Settings).

## Phase 5: UI/UX Implementation
18. **Auth Screens**: Email/password flows wired to Supabase Auth. On successful auth, register the Expo push token to `push_tokens`.
19. **App Selection UI**: Calls the App List native module; lets the user toggle distracting apps; writes to `distracting_apps`.
20. **Persona Setup UI**: First-run screen that captures the AI persona prompt and writes it to `users.persona_prompt`. Editable later in Settings.
21. **Chat Interface**: Message list + composer. Supports streamed assistant responses.
22. **Realtime Subscriptions**: Supabase Realtime on `messages` filtered by `user_id` — assistant replies and proactive nudges appear instantly.
23. **Dashboard UI**: Today's tasks with progress, next upcoming nudge time, and active-task banner.

## Phase 6: AI Orchestration, Edge Functions & Notifications
24. **OpenAI Setup**: User creates an OpenAI API key and adds it to Supabase secrets (`supabase secrets set OPENAI_API_KEY=...`). Also set `NATIVE_WEBHOOK_SECRET` and `EXPO_ACCESS_TOKEN` (for push).
25. **`chat_handler` Edge Function**:
    - Accepts user messages.
    - Loads last 20 messages + latest `message_summaries.summary_text` + `users.persona_prompt` → system prompt.
    - Streams via Vercel AI SDK with OpenAI.
    - Tools exposed to the LLM: `save_goal`, `schedule_task`, `update_persona`, `mark_task_complete`.
    - Persists assistant message to `messages`.
26. **`summarize_context` Edge Function**: Condenses messages older than the active window into `message_summaries`. Invoked daily.
27. **`task_notifier` Edge Function + pg_cron**:
    - `pg_cron` job runs every minute, selects tasks whose `start_at` or `end_at` crossed in the last minute.
    - For each, invokes `task_notifier`: LLM generates a persona-appropriate nudge, writes it to `messages`, and sends a push via Expo Push API.
28. **`distraction_alert` Edge Function**:
    - Verifies the HMAC signature from the native webhook (Phase 3.15).
    - Confirms a task is currently active for that user.
    - Asks the LLM to generate an intervention message → writes to `messages` → sends push.
29. **`summarize_context` scheduling**: Separate `pg_cron` job, daily at 03:00 local (or UTC; compute per-user based on `users.timezone`).

## Phase 7: Deployment & End-to-End Testing
30. **Dev Client Launch**: `npx expo run:android` against a physical device (emulators don't expose `UsageStatsManager` reliably).
31. **Verify App Selection**: Native module lists apps; toggles persist to Supabase.
32. **Verify Persona Flow**: Persona set in onboarding appears in system prompt (inspect Edge Function logs).
33. **Verify AI Agent Flow**: Chat → goal → LLM calls `schedule_task` → row appears in `tasks` → Dashboard renders it.
34. **Verify Proactive Notifications**: Create a task starting 2 minutes out → push lands + chat message appears at start and end.
35. **Verify Distraction Flow**: With an active task, open a flagged app → foreground service fires webhook → intervention push + chat message arrive.
36. **Verify Summarization**: Seed 50+ messages, run the summarization function manually, confirm `message_summaries` row updates and chat handler uses it.

## Phase 8 (Deferred, Post-MVP)
- Google/Apple social login.
- iOS build (with Screen Time API limitations documented).
- OTA updates via EAS Update.
- Analytics + crash reporting.

---

## External Requirements
Before Phase 6:
1. OpenAI account + API key + billing at [platform.openai.com](https://platform.openai.com).
2. Expo account (free) for push notifications via the Expo Push API.
3. Physical Android device (API 29+) for testing usage monitoring.

If you approve, we'll initialize `task.md` from this plan and start Phase 1.
