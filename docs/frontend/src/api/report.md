# `frontend/src/api/report.js` — Report API Wrapper

## Overview

`api/report.js` wraps all frontend calls related to report generation, log polling, report retrieval, and report-agent chat.

---

## Functions

### `generateReport(data)`
Starts report generation through `/api/report/generate`.

### `getReportStatus(reportId)`
Polls the report-generation status endpoint.

### `getAgentLog(reportId, fromLine = 0)`
Loads incremental structured agent logs from `agent_log.jsonl`.

### `getConsoleLog(reportId, fromLine = 0)`
Loads incremental raw console logs from `console_log.txt`.

### `getReport(reportId)`
Loads report metadata and full markdown content.

### `chatWithReport(data)`
Sends a chat message to the backend report-agent chat endpoint.

---

## Related Files

- [`components/Step4Report.vue`](../components/Step4Report.md) polls both log streams
- [`components/Step5Interaction.vue`](../components/Step5Interaction.md) uses `chatWithReport()` for the report-agent chat mode
- [`views/ReportView.vue`](../views/ReportView.md) and [`views/InteractionView.vue`](../views/InteractionView.md) load report context
