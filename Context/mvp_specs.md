# App MVP Specifications: AI Accountability Partner

## 1. Overview
The MVP of the application will be an Android-only mobile application (built cross-platform for future iOS portability) that acts as an LLM-powered accountability partner. The app primarily utilizes a chat-style interface for all interactions, helping users define goals, plan tasks, and stay on track through proactive notifications and app usage monitoring.

## 2. Core Features

### A. Authentication
*   **Social Login:** The app will support quick onboarding using social login (e.g., Google or Apple Sign-In). 

### B. Chat Interface & Goal Setting
*   **Primary UI:** The interface behaves like a standardized messaging app.
*   **Goal Extraction:** Users converse with the AI to discuss their productivity goals. The AI parses the conversation to create structured "Goals" and a "Plan" with scheduled tasks.
*   **AI Persona/Role:** During the goal-setting phase, the user defines the role of the AI in the Plan (e.g., "strict drill sergeant", "supportive friend"). The AI adopts this persona moving forward.
*   **Chat History Optimization:** Past weeks' chat history will be automatically summarized by the system before being passed back to the LLM to conserve context window tokens and reduce API costs.

### C. Proactive Task Notifications
*   **Task Starts:** The AI will proactively send a notification to the user when a scheduled task is supposed to begin based on the Plan.
*   **Task Ends:** The AI will prompt the user at the end of the scheduled block to ask if they successfully completed the task.
*   **In-Chat Display:** Every proactive notification or check-in sent by the AI will also be appended to the chat interface conversation history.

### D. App Usage Monitoring (The "Gotcha" Feature)
*   **Android-First Focus:** For the MVP, we are exclusively focusing on Android to utilize the `UsageStatsManager` API, bypassing Apple's restrictive Screen Time sandbox limitations.
*   **Distraction Detection:** The app will monitor foreground app usage data. If a user opens a known distracting app (e.g., YouTube, Instagram) during a block designated for work, it triggers an event.
*   **AI Intervention:** The system alerts the LLM of the distraction event. The LLM generates a personalized message asking the user why they are off-task and sends it as an immediate notification and a chat message.

## 3. Data Entities (High-Level)
*   **User:** Auth details.
*   **Goals/Plan:** Data defined and extracted by the AI.
*   **Tasks:** Specific time blocks mapped out.
*   **Messages:** Real-time chat logs (and summaries for older content).
*   **AI Persona Rules:** Set instructions guiding the AI's tone and enforcement strategy.
