# `backend/app/services/__init__.py` — Service Package Exports

## Overview

This file gathers the backend service-layer classes and data structures into one package-level import surface. Like most `__init__.py` files, it contains no business logic of its own; its job is organization and re-export.

---

## Re-exported Symbols

The file re-exports service classes and related dataclasses from:
- [`ontology_generator.py`](ontology_generator.md)
- [`graph_builder.py`](graph_builder.md)
- [`text_processor.py`](text_processor.md)
- [`zep_entity_reader.py`](zep_entity_reader.md)
- [`oasis_profile_generator.py`](oasis_profile_generator.md)
- [`simulation_manager.py`](simulation_manager.md)
- [`simulation_config_generator.py`](simulation_config_generator.md)
- [`simulation_runner.py`](simulation_runner.md)
- [`zep_graph_memory_updater.py`](zep_graph_memory_updater.md)
- [`simulation_ipc.py`](simulation_ipc.md)

### `__all__`

The `__all__` list defines which names are intended to be imported from `app.services` as the package’s public API.

---

## Why This File Exists

It reduces import friction. Instead of importing deep module paths, callers can often write package-level imports such as:

```python
from app.services import SimulationManager, SimulationRunner
```

---

## Related Files

- all service docs listed above
