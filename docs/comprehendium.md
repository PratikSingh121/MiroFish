# MiroFish Comprehendium

## What This System Is Really Trying To Do

MiroFish is not just a document-processing app and not just a simulator. Its core idea is to turn messy human source material into a runnable social world.

The system takes unstructured input such as news, reports, or fiction, extracts the important actors and relationships, turns that into a structured world model, populates the world with plausible agents, lets those agents interact under platform rules, and then interprets the result in human-readable form.

In short, the codebase is organized around one question:

> How do we go from text about a world to a simulated future of that world?

---

## The Core Reasoning Pipeline

### 1. Start from text because that is what humans already have

Most real problems arrive as documents, not databases. A policy draft, a breaking-news article, a chapter of a novel, or a market analysis already contains the seeds of a world.

That is why the first part of the system focuses on file parsing and extraction rather than forcing users to manually model entities and relationships.

### 2. Convert text into a graph because simulation needs structure

Raw text is rich but ambiguous. Simulation needs explicit objects:
- who exists
- what they are
- how they relate
- what pressures or affiliations shape behavior

The graph-building services exist to compress long-form text into a machine-usable representation without throwing away the social structure that matters later.

### 3. Convert graph structure into agents because worlds act through individuals

A graph by itself is static. The system needs simulated people or groups that can:
- hold beliefs
- react to events
- post content
- respond to other agents
- accumulate memory over time

The profile-generation and environment-setup code bridges the gap between a knowledge model and an executable social simulation.

### 4. Run the simulation in an external engine because specialized runtimes should stay specialized

The Flask backend does orchestration, persistence, and API serving. OASIS does multi-agent social simulation. That separation is intentional.

The backend does not try to reimplement an agent runtime. Instead, it prepares inputs, launches the simulation as a subprocess, watches it, and exchanges data through a simple IPC layer. This keeps responsibilities narrow and makes failures easier to isolate.

### 5. Turn simulation output into explanation because users need judgment, not logs

Simulation output is noisy. Raw events, posts, and memories are not enough for decision-making.

That is why report generation is treated as a first-class stage. The report agent does not merely summarize files; it queries graph memory and simulation artifacts to answer structured questions about likely outcomes, mechanisms, and implications.

### 6. Keep the world queryable after the report because exploration continues after prediction

The interaction stage exists because a prediction is rarely the end of the user’s reasoning process. After the report, users still want to ask:
- why did this happen
- what did this agent believe
- what relationships mattered most
- what changed over time

The chat and tool layers keep the simulated world explorable instead of freezing it into a single report artifact.

---

## Why The Architecture Looks The Way It Does

## Thin frontend, orchestration-heavy backend

The frontend mainly collects input, launches steps, displays status, and visualizes results. The backend owns the hard parts because the hard parts are stateful, long-running, and integration-heavy.

This is a good fit for the problem: browsers are good at workflow and visualization; the server is better at file handling, subprocess control, retries, and external service coordination.

## Service-oriented backend modules

The backend is split into small services because the pipeline itself is staged:
- parse input
- build graph
- generate agent profiles
- generate simulation config
- run simulation
- update memory
- generate report

Each stage has distinct failure modes and distinct dependencies. Keeping them separate makes the flow easier to debug and easier to rerun partially.

## File-based task persistence

The project and task models are file-backed rather than database-backed. This keeps setup simple and makes workflow artifacts visible on disk.

That choice trades off scalability for transparency and ease of local use. For a research/demo-style system, that is often the right trade.

## Graph memory plus generated report

The graph is the durable knowledge substrate. The report is a human-oriented interpretation layer. They solve different problems.

If the system only stored a report, users would lose inspectability. If it only stored graph data, users would lose readability. The codebase keeps both because prediction systems need both auditability and explanation.

## Subprocess-based simulation execution

Running the simulator out-of-process avoids tangling Flask request handling with a long-running multi-agent runtime. It also gives the system cleaner lifecycle control, easier cancellation, and safer crash boundaries.

---

## Mental Model For Reading The Code

If you want to understand the repository quickly, read it in this order:

1. Frontend workflow views to see the user journey.
2. Flask API blueprints to see the backend entry points.
3. Service modules to see the actual business logic.
4. Utility modules to see cross-cutting helpers.
5. Report and interaction services last, because they make more sense once the simulation pipeline is clear.

A practical mental model is:
- views describe the steps
- APIs expose the steps
- services perform the steps
- utils support the steps
- docs explain the steps

---

## Intuition Behind The Five User-Facing Steps

### Step 1: Graph Build

This step exists to answer: what world are we talking about?

### Step 2: Environment Setup

This step exists to answer: who lives in that world and what constraints shape them?

### Step 3: Simulation

This step exists to answer: what happens when those agents begin interacting over time?

### Step 4: Report

This step exists to answer: what does the simulation mean?

### Step 5: Interaction

This step exists to answer: what else does the user want to inspect after the main report is done?

---

## Where The Important Complexity Lives

The difficult parts of the codebase are not the routes or the Vue pages. The real complexity lives in four translation boundaries:

1. Text to graph
2. Graph to agent profiles
3. Backend orchestration to simulator runtime
4. Simulation traces to report-quality explanation

Most modules exist because one of those translations needs to be made more reliable.

---

## The Main Tradeoffs

### Speed vs fidelity

Higher-fidelity extraction and richer agent setups improve simulation quality but cost more in latency and LLM usage.

### Modularity vs simplicity

Many service files make the architecture easier to reason about, but they also increase the number of moving pieces.

### Transparency vs convenience

Persisting artifacts on disk and keeping explicit intermediate files improves inspectability, but it can feel less polished than hiding everything behind a database.

### Determinism vs generative flexibility

The system relies on LLM-driven extraction and interpretation, which creates richer behavior but also makes outputs less strictly deterministic.

---

## What To Keep In Mind While Modifying It

When changing this codebase, the key question is not only whether a function works in isolation. The key question is whether it preserves the handoff into the next stage.

A change is usually good if it does one of these:
- makes a stage output more explicit
- reduces ambiguity passed to the next stage
- improves recovery when an external dependency fails
- keeps artifacts inspectable
- makes report conclusions easier to trace back to evidence

A change is risky if it does one of these:
- hides intermediate state
- couples the Flask layer too tightly to the simulation runtime
- weakens graph/report traceability
- makes retry behavior or failure handling less visible

---

## Short Map Of The Repository

- `frontend/src/views` defines the user journey.
- `frontend/src/components` implements step-specific UI and visualization.
- `frontend/src/api` maps browser actions to backend endpoints.
- `backend/app/api` exposes the pipeline as HTTP routes.
- `backend/app/services` contains the real application logic.
- `backend/app/utils` holds helper code for files, retries, LLM access, logging, and paging.
- `backend/scripts` contains operational and simulation-side scripts.

---

## Related Docs

- [`README.md`](README.md)
- [`backend/app/services/graph_builder.md`](backend/app/services/graph_builder.md)
- [`backend/app/services/simulation_manager.md`](backend/app/services/simulation_manager.md)
- [`backend/app/services/report_agent.md`](backend/app/services/report_agent.md)
- [`frontend/src/views/MainView.md`](frontend/src/views/MainView.md)
- [`frontend/src/views/SimulationView.md`](frontend/src/views/SimulationView.md)
- [`frontend/src/views/ReportView.md`](frontend/src/views/ReportView.md)
