# `frontend/src/components/Step5Interaction.vue` — Step 5 Workbench

## Overview

`Step5Interaction.vue` is the final interactive workspace. It combines three things in one screen:
- a report reader showing the generated sections
- a chat interface for talking either to the report agent or to a selected simulated agent
- a survey mode that sends the same question to multiple agents

This component is the frontend layer over both report-agent chat and live simulation interviews.

---

## State

### Mode and selection state
- `activeTab`
- `chatTarget`
- `showAgentDropdown`
- `selectedAgent`
- `selectedAgentIndex`
- `showFullProfile`
- `showToolsDetail`

### Chat state
- `chatInput`
- `chatHistory`
- `chatHistoryCache`
- `isSending`
- message/input refs

### Survey state
- `selectedAgents`
- `surveyQuestion`
- `surveyResults`
- `isSurveying`

### Report and profile state
- `reportOutline`
- `generatedSections`
- `collapsedSections`
- `currentSectionIndex`
- `profiles`

---

## Functions

### Report display helpers
- `isSectionCompleted(sectionIndex)` checks whether a report section has finished
- `toggleSectionCollapse(idx)` expands or collapses completed sections
- `renderMarkdown(content)` renders markdown for both report sections and replies
- `formatTime(timestamp)` formats message timestamps

### Shared helper
- `addLog(msg)` emits logs to the parent view

### Chat target switching
- `selectChatTarget(target)` updates the current chat mode
- `saveChatHistory()` caches the current conversation by target
- `selectReportAgentChat()` switches to report-agent chat and restores its cached history
- `toggleAgentDropdown()` opens or closes the target-agent dropdown
- `selectAgent(agent, idx)` switches to one simulated agent and restores cached history
- `selectSurveyTab()` switches from chat mode to survey mode

### Message sending
- `sendMessage()` dispatches to the current chat target
- `sendToReportAgent(message)` calls the backend report-agent chat API
- `sendToAgent(message)` sends a one-agent interview request through the backend simulation API
- `scrollToBottom()` keeps the chat scrolled to the latest message

### Survey helpers
- `toggleAgentSelection(idx)` toggles one selected survey target
- `selectAllAgents()` selects every loaded profile
- `clearAgentSelection()` clears the selection
- `submitSurvey()` sends the same question to all chosen agents and collects the replies

### Data loading
- `loadReportData()` loads report outline and previously generated section content
- `loadAgentLogs()` reads the agent log to infer completed sections and active section state
- `loadProfiles()` loads available simulation-agent profiles for chat/survey targeting
- `handleClickOutside(e)` closes the agent dropdown when clicking elsewhere

---

## Related Files

- [`api/report.js`](../api/report.md)
- [`api/simulation.js`](../api/simulation.md)
- [`views/InteractionView.vue`](../views/InteractionView.md)
- [`services/report_agent.py`](../../../backend/app/services/report_agent.md)
