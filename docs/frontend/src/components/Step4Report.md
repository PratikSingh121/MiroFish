# `frontend/src/components/Step4Report.vue` — Step 4 Workbench

## Overview

`Step4Report.vue` is the most complex frontend component. It visualizes the report-generation process as a live timeline by polling both structured agent logs and console logs, reconstructs the report outline and completed sections incrementally, and renders tool outputs in specialized visual forms.

This component is the UI counterpart of [`services/report_agent.py`](../../../backend/app/services/report_agent.md).

---

## High-Level Responsibilities

- poll `agent_log.jsonl` and `console_log.txt`
- rebuild the report outline and generated section list in real time
- display planning, tool calls, tool outputs, and section completion
- parse tool-specific text payloads into richer UI structures
- let the user jump to step 5 once generation completes

---

## State and Computed Values

Main refs include:
- `agentLogs`, `consoleLogs`
- line cursors for incremental polling
- `reportOutline`, `currentSectionIndex`, `generatedSections`
- expansion/collapse sets for content and logs
- `isComplete`, `startTime`
- panel refs and raw-result visibility state

Computed helpers include:
- status and workflow progress values
- total/completed section counts
- percent complete
- total tool-call count
- elapsed time
- filtered display log list
- active workflow step flags

---

## Interaction Helpers

### `goToInteraction()`
Routes to step 5 once the report is ready.

### `toggleRawResult(timestamp, event)`
Shows or hides full tool results for one log event.

### `toggleSectionContent(idx)`
Expands or collapses section body display.

### `toggleSectionCollapse(idx)`
Collapses a completed section card.

### `toggleLogExpand(log)` and `isLogCollapsed(log)`
Control log-card expansion.

---

## Tool Parsing and Display Helpers

### Tool metadata helpers
- `getToolDisplayName(toolName)`
- `getToolColor(toolName)`
- `getToolIcon(toolName)`

### Parser functions
These convert raw text returned by backend tools into structured data for richer presentation:
- `parseInsightForge(text)`
- `parsePanorama(text)`
- `parseInterview(text)`
- `parseQuickSearch(text)`

### Inline display components
The file defines in-component display objects for each tool result type:
- `InsightDisplay`
- `PanoramaDisplay`
- `InterviewDisplay`
- `QuickSearchDisplay`

---

## Formatting and Rendering Helpers

- `addLog(msg)`
- `isSectionCompleted(sectionIndex)`
- `formatTime(timestamp)`
- `formatParams(params)`
- `formatResultSize(length)`
- `truncateText(text, maxLen)`
- `renderMarkdown(content)`
- `getTimelineItemClass(log, idx, total)`
- `getConnectorClass(log, idx, total)`
- `getActionLabel(action)`
- `getLogLevelClass(log)`
- `extractFinalContent(response)`

These helpers translate backend log structures into readable UI cards and markdown panels.

---

## Polling Methods

### `fetchAgentLog()`
Reads incremental structured events and updates outline, sections, active section state, completion state, and tool metrics.

### `fetchConsoleLog()`
Reads incremental console output for the side log panel.

### `startPolling()`
Starts the periodic polling loop.

### `stopPolling()`
Stops polling when the component unmounts or the report is complete.

---

## Related Files

- [`api/report.js`](../api/report.md)
- [`views/ReportView.vue`](../views/ReportView.md)
- [`components/Step5Interaction.vue`](Step5Interaction.md)
- [`services/report_agent.py`](../../../backend/app/services/report_agent.md)
