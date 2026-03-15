# MiroFish — Codebase Documentation

MiroFish is a **swarm-intelligence prediction engine** powered by multi-agent simulation. Upload any document (news report, novel, policy draft, financial analysis) and the system will automatically extract real-world "seed" information, build a knowledge graph, generate hundreds of AI agents with individual personalities and memories, run a dual-platform social-media simulation, and finally produce a detailed prediction report.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Frontend (Vue 3)                  │
│  Home → MainView → SimulationView → SimulationRunView│
│           → ReportView → InteractionView            │
└──────────────────────┬──────────────────────────────┘
                       │ REST API (axios)
┌──────────────────────▼──────────────────────────────┐
│              Backend (Flask, Python)                 │
│  /api/graph  /api/simulation  /api/report           │
│                                                     │
│  Services: graph_builder, ontology_generator,       │
│   oasis_profile_generator, simulation_manager,      │
│   simulation_runner, report_agent, zep_tools, ...   │
│                                                     │
│  External: Zep Cloud (graph memory)                │
│            OASIS (multi-agent simulation)          │
│            LLM API (OpenAI-compatible)             │
└─────────────────────────────────────────────────────┘
```

---

## 5-Step Workflow

| Step | Name | Key Backend Service | Frontend View/Component |
|------|------|---------------------|------------------------|
| 1 | Graph Build | `ontology_generator`, `graph_builder` | `MainView` → `Step1GraphBuild` |
| 2 | Env Setup | `simulation_manager`, `oasis_profile_generator`, `simulation_config_generator` | `SimulationView` → `Step2EnvSetup` |
| 3 | Simulation | `simulation_runner`, `zep_graph_memory_updater` | `SimulationRunView` → `Step3Simulation` |
| 4 | Report | `report_agent`, `zep_tools` | `ReportView` → `Step4Report` |
| 5 | Interaction | `report_agent` (chat), `simulation_ipc` | `InteractionView` → `Step5Interaction` |

---

## Documentation Structure

```
docs/
├── README.md                    ← This file
├── comprehendium.md
├── Dockerfile.md
├── docker-compose.md
├── package.md
├── repo-readme.md
├── repo-readme-en.md
├── backend/
│   ├── pyproject.md
│   ├── requirements.md
│   ├── run.md
│   └── app/
│       ├── __init__.md
│       ├── config.md
│       ├── api/
│       │   ├── __init__.md
│       │   ├── graph.md
│       │   ├── simulation.md
│       │   └── report.md
│       ├── models/
│       │   ├── project.md
│       │   └── task.md
│       ├── services/
│       │   ├── graph_builder.md
│       │   ├── ontology_generator.md
│       │   ├── oasis_profile_generator.md
│       │   ├── simulation_manager.md
│       │   ├── simulation_runner.md
│       │   ├── simulation_config_generator.md
│       │   ├── simulation_ipc.md
│       │   ├── report_agent.md
│       │   ├── text_processor.md
│       │   ├── zep_entity_reader.md
│       │   ├── zep_graph_memory_updater.md
│       │   └── zep_tools.md
│       └── utils/
│           ├── file_parser.md
│           ├── llm_client.md
│           ├── logger.md
│           ├── retry.md
│           └── zep_paging.md
└── frontend/
    ├── package.md
    ├── vite.config.md
    └── src/
        ├── main.md
        ├── App.md
        ├── api/
        │   ├── index.md
        │   ├── graph.md
        │   ├── simulation.md
        │   └── report.md
        ├── router/
        │   └── index.md
        ├── store/
        │   └── pendingUpload.md
        ├── views/
        │   ├── Home.md
        │   ├── MainView.md
        │   ├── SimulationView.md
        │   ├── SimulationRunView.md
        │   ├── ReportView.md
        │   └── InteractionView.md
        └── components/
            ├── GraphPanel.md
            ├── Step1GraphBuild.md
            ├── Step2EnvSetup.md
            ├── Step3Simulation.md
            ├── Step4Report.md
            ├── Step5Interaction.md
            └── HistoryDatabase.md
```

---

## Key Concepts

### Zep Cloud
Zep is a graph-based memory system used as the persistent knowledge graph store. MiroFish creates a standalone Zep graph per project. The graph holds entities (people, organizations) and relationships extracted by the LLM from uploaded documents. The graph is also updated in real-time *during* simulation as agents act.

### OASIS
OASIS is the multi-agent social-media simulation platform. It supports two platforms: **Twitter** and **Reddit**. MiroFish feeds OASIS with generated agent profiles, a simulation config, and lets it run for multiple rounds.

### IPC (Inter-Process Communication)
The Flask backend runs the OASIS simulation as a **subprocess**. They communicate via a simple file-based IPC system (commands/responses written to JSON files in the simulation directory). See [`simulation_ipc.md`](backend/app/services/simulation_ipc.md).

### Report Agent (ReACT)
After simulation, `ReportAgent` uses a ReACT (Reason + Act) loop with a rich toolset to query the Zep graph and produce a detailed prediction report section by section.

---

## Repository-Level Docs

- [`comprehendium.md`](comprehendium.md) gives a shorter intuition-first explanation of how the whole codebase fits together and why it is structured this way.
- [`Dockerfile.md`](Dockerfile.md) explains the combined Python + Node development image.
- [`docker-compose.md`](docker-compose.md) explains container runtime deployment using the published image.
- [`package.md`](package.md) explains the workspace-level npm scripts used to install and run the project.
- [`repo-readme.md`](repo-readme.md) documents the Chinese root repository readme.
- [`repo-readme-en.md`](repo-readme-en.md) documents the English root repository readme.
- [`backend/pyproject.md`](backend/pyproject.md) explains backend Python packaging and dependency declaration.
- [`backend/requirements.md`](backend/requirements.md) explains the backend pip compatibility dependency list.
- [`frontend/package.md`](frontend/package.md) explains the frontend app package definition.
- [`frontend/vite.config.md`](frontend/vite.config.md) explains the Vite dev-server and API proxy configuration.
