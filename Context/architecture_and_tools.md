# Architecture and Development Information

## 1. System Architecture
The application runs as a thick client utilizing a Backend-as-a-Service (BaaS) and external LLM APIs. The design focuses on real-time event syncing and agentic function/tool calling.

### Interactions Flow
1. **User Input:** User sends a message via the mobile app.
2. **Backend Orchestrator:** The message route triggers an Edge Function.
3. **LLM Processing:** The LLM processes context, user intent, the summarized history, and the system prompt (which contains the user-defined Persona).
4. **Tool Execution:** Instead of outputting chaotic text, the LLM calls explicit tools (e.g., `create_schedule`, `summarize_long_history`) that result in discrete database writes.
5. **Real-time UI Updates:** The mobile app detects changes in the database (via WebSockets/Real-time) and instantly updates the UI to reflect new chat messages and scheduled tasks.

## 2. Technology Stack & Tools

### Frontend (Mobile Application)
*   **Framework:** **React Native with Expo** 
    *   *Why:* The standard for cross-platform app development. Expo simplifies native module integration and over-the-air updates. While the MVP focuses on Android features, the React Native codebase will easily compile for iOS in the future.
*   **State Management:** **Zustand** or **Jotai**
    *   *Why:* State management ensures that things like "is the user authenticated," "the current chat message list," and "today's tasks" are available to any visual component without passing data manually everywhere. Unlike bulky solutions like Redux, Zustand is extremely modern, lightweight, and requires minimal boilerplate code.
*   **Styling:** **NativeWind** (TailwindCSS for React Native)
    *   *Why:* Allows for rapid, standard UI styling familiar to most web developers, making it natively compatible with AI development tools.

### Backend Infrastructure
*   **BaaS:** **Supabase**
    *   *Database:* PostgreSQL. Stores users, goals, task schedules, and message logs.
    *   *Real-time:* Provides WebSocket subscriptions so the mobile chat UI instantly updates when the AI adds a reply or creates a schedule.
    *   *Edge Functions:* Deno-based serverless functions that act as a secure proxy between your app and the LLM API, ensuring API keys are never exposed on the mobile device.
    *   *Auth:* Built-in support for social logins (Google/Apple) that ties directly into Postgres Row Level Security (RLS) to keep user data private.

### AI Engine & Integration
*   **LLM Provider:** **OpenAI API (GPT-4o / GPT-4o-mini)**
    *   *Why:* They offer the most robust and accurate function-calling capabilities. This is vital so the AI can physically "save" goals to the database rather than simply stating it has done so.
*   **Orchestration SDK:** **Vercel AI SDK**
    *   *Why:* Designed specifically to simplify tool integration and streaming responses directly to a frontend.

## 3. Background Services & Native Modules

### Scheduled Notifications
*   **Mechanism:** Rather than the mobile device polling for when a task starts or ends, Supabase's `pg_cron` (a database job scheduler) will monitor the `tasks` table. 
*   **Trigger:** When a task is supposed to start or end, the CRON strikes and triggers an Edge Function. The Edge Function asks the LLM to generate a situational message (based on the persona), saves that message to the DB, and sends a push notification to wake up the phone.

### App Usage Monitoring (Android MVP)
*   **Mechanism:** Android provides the `UsageStatsManager` API to query foreground app usage. 
*   **Implementation:** Because React Native is written in JavaScript and `UsageStatsManager` requires native Java/Kotlin, a **Custom Expo Native Module** must be developed. 
*   **Flow:** An Android background service periodically checks foreground activity. If "Distracting App X" is active during a scheduled "Work Task Y", it triggers a webhook to Supabase, which in turn commands the LLM to intervene and message the user.

## 4. AI Prompt & Context Management
To pass history to the LLM without exceeding context windows or incurring massive HTTP request costs:
*   **Active Window:** The current session or the last ~20 messages are passed verbatim to the LLM.
*   **Summarized Context:** A background Edge Function runs periodically to condense older messages into a rolling "summary of the user's progress and mindset." This single summary paragraph is passed as system instructions to the LLM on subsequent runs.
