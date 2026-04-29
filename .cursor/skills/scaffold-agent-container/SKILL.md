# Scaffold AI-Service Agent Container

Create a new agent container under `apps/ai-service/containers/<name>/` that follows the Reusable Agent Framework architecture (per confluence: domain / application / composition / services / config / database / utils / models / exceptions). The layout mirrors `apps/ai-service/containers/react_demo_agent/`, which is the reference implementation of the new framework.

This skill only produces the **file/folder skeleton**. It does NOT wire logging, vault, config_manager, global state, mongodb, or weaviate — those are the job of the `setup-agent-infra` skill and must stay opt-in.

---

## Trigger

Use this skill when the user asks any of:

- "Scaffold a new agent container"
- "Create a new agent under ai-service"
- "Set up the file structure for `<name>` agent per the confluence"
- "Create agent container skeleton"

---

## Phase 1 — Gather inputs

Before writing anything, resolve:

1. `<container-name>` — snake_case, e.g. `appraisal_validity_agent`. If the user didn't give one, ask via `AskQuestion` with a single free-form question (or propose one based on context).
2. Verify the parent exists: `ls apps/ai-service/containers/`
3. Verify the target does NOT exist: if `apps/ai-service/containers/<container-name>/` already has files, STOP and tell the user; do not overwrite.

Derive from the container name:
- `<AgentClassName>` = PascalCase of the container name with the trailing "Agent" normalized (e.g. `appraisal_validity_agent` -> `AppraisalValidityAgent`).

---

## Phase 2 — Create the directory tree

Create these empty directories (use `mkdir -p`):

```
apps/ai-service/containers/<container-name>/
├── src/
│   ├── domain/
│   ├── application/
│   │   ├── nodes/
│   │   ├── tools/
│   │   └── callbacks/
│   ├── composition/
│   ├── services/
│   │   ├── litellm/
│   │   └── vault/
│   ├── config/
│   ├── database/
│   ├── vectorstore/
│   ├── utils/
│   ├── models/
│   └── exceptions/
└── tests/
    ├── unit/
    └── integration/
```

---

## Phase 3 — Create files

### 3.1 Container-root files

Copy each template below to the new container, substituting `__CONTAINER_NAME__` with the snake_case name and `__AGENT_CLASS__` with the PascalCase class name.

The templates live alongside this skill in `templates/`:

- `templates/pyproject.toml.tmpl` → `apps/ai-service/containers/<container-name>/pyproject.toml`
- `templates/Dockerfile.tmpl` → `apps/ai-service/containers/<container-name>/Dockerfile`
- `templates/Makefile.tmpl` → `apps/ai-service/containers/<container-name>/Makefile`
- `templates/README.md.tmpl` → `apps/ai-service/containers/<container-name>/README.md`
- `templates/requirements.txt.tmpl` → `apps/ai-service/containers/<container-name>/requirements.txt`

Read each template with the `Read` tool, perform the string substitutions, then `Write` the result.

### 3.2 `src/` Python stubs

Write each file with exactly the content shown. All `__init__.py` files start empty unless specified below.

#### `src/main.py`

```python
"""Main entrypoint for the __AGENT_CLASS__."""

from __future__ import annotations

import asyncio
import signal
import sys

# Logging is wired by the setup-agent-infra skill. Until then, a stdlib logger is fine.
import logging

logger = logging.getLogger(__name__)


def startup():
    """Initialize configuration and dependencies.

    Infrastructure wiring (vault, config, mongodb, weaviate, global state) is
    added by the `setup-agent-infra` skill on demand.
    """
    logger.info("__AGENT_CLASS__ startup")
    return None


def cleanup_resources() -> None:
    """Release resources held during execution."""
    logger.info("__AGENT_CLASS__ cleanup")


def signal_handler(_signum, _frame):
    sys.exit(0)


async def main() -> int:
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    try:
        startup()
        logger.info("__AGENT_CLASS__ initialized")
        return 0
    except Exception:  # pylint: disable=broad-exception-caught
        logger.exception("__AGENT_CLASS__ failed")
        return 1
    finally:
        cleanup_resources()


if __name__ == "__main__":
    sys.exit(asyncio.run(main()))
```

#### `src/domain/__init__.py`

```python
"""Agent-specific domain entities (Layer 1).

Extend `agent_core.domain.base_state.AgentState` for per-agent execution state.
Keep this layer free of infrastructure imports.
"""
```

#### `src/domain/state.py`

```python
"""Agent-specific execution state.

Extend this class with fields unique to __AGENT_CLASS__.
"""

from __future__ import annotations

from agent_core.domain.base_state import AgentState


class __AGENT_CLASS__State(AgentState):
    """Execution state for __AGENT_CLASS__."""
```

#### `src/application/__init__.py`

```python
"""Application services (Layer 2).

Agent-specific workflow nodes, tools, and callbacks live here.
Nodes consume ports defined in `agent_core.ports` and tools from the tool registry.
"""
```

#### `src/application/nodes/__init__.py`

```python
"""Workflow nodes for __AGENT_CLASS__.

Each node subclasses `agent_core.application.base_node.BaseNode`.
"""
```

#### `src/application/tools/__init__.py`

```python
"""Agent-specific tools.

Tools subclass `agent_core.application.base_tool.BaseTool`.
Prefer generic tools from `agent_core.application.tools.*` when suitable.
"""
```

#### `src/application/callbacks/__init__.py`

```python
"""Workflow callbacks (observability, tracing, metrics hooks)."""
```

#### `src/composition/__init__.py` (empty)

#### `src/composition/agent.py`

```python
"""Composition root (Layer 5) for __AGENT_CLASS__.

Wires ports to adapters and exposes the public entrypoint used by `main.py`.
"""

from __future__ import annotations


class __AGENT_CLASS__:
    """Composition root for __AGENT_CLASS__.

    Responsibilities:
      * Build the workflow (static or ReAct) via a workflow factory.
      * Resolve ports -> adapters (memory, model, tools).
      * Expose `run()` for the entrypoint.
    """

    def __init__(self) -> None:
        pass

    async def run(self, *args, **kwargs):
        raise NotImplementedError("Implement the agent workflow here.")
```

#### `src/services/__init__.py` (empty)
#### `src/services/litellm/__init__.py` (empty)
#### `src/services/vault/__init__.py` (empty)

#### `src/config/__init__.py` (empty)

Leave this empty — it is populated by the `setup-agent-infra` skill when `config_manager` is requested.

#### `src/database/__init__.py` (empty)

Populated by `setup-agent-infra` when `mongodb` is requested.

#### `src/vectorstore/__init__.py` (empty)

Populated by `setup-agent-infra` when `weaviate` is requested.

#### `src/utils/__init__.py` (empty)

Populated by `setup-agent-infra` when `global state` is requested.

#### `src/models/__init__.py`

```python
"""Data-transfer objects (execution request/result, workflow definition)."""
```

#### `src/models/execution_request.py`

```python
"""ExecutionRequest DTO."""

from __future__ import annotations

from typing import Any, Dict, Optional

from pydantic import BaseModel, Field


class ExecutionRequest(BaseModel):
    """Request envelope for an agent execution."""

    execution_id: str = Field(..., min_length=1)
    configuration: Dict[str, Any] = Field(default_factory=dict)
    metadata: Optional[Dict[str, Any]] = None
```

#### `src/models/execution_result.py`

```python
"""ExecutionResult DTO."""

from __future__ import annotations

from typing import Any, Dict, Optional

from pydantic import BaseModel, Field


class ExecutionResult(BaseModel):
    """Response envelope for an agent execution."""

    execution_id: str
    status: str = "success"
    output: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)
```

#### `src/exceptions/__init__.py`

```python
"""Custom exception hierarchy for __AGENT_CLASS__.

See confluence "Error Handling Strategy" — each infrastructure layer owns its own
exception module so callers can distinguish retryable from permanent failures.
"""

from .base_exception import AgentException
from .memory_exceptions import MemoryAccessError, MemoryConnectionError
from .tool_exceptions import ToolExecutionError, ToolNotFoundError
from .workflow_exceptions import NodeFailureError, WorkflowExecutionError

__all__ = [
    "AgentException",
    "WorkflowExecutionError",
    "NodeFailureError",
    "MemoryAccessError",
    "MemoryConnectionError",
    "ToolExecutionError",
    "ToolNotFoundError",
]
```

#### `src/exceptions/base_exception.py`

```python
"""Base exception hierarchy."""

from __future__ import annotations

from typing import Any, Dict, Optional


class AgentException(Exception):
    """Root of the agent exception hierarchy."""

    is_retryable: bool = False
    is_critical: bool = False
    error_code: str = "AGENT_ERROR"

    def __init__(
        self,
        message: str,
        *,
        error_code: Optional[str] = None,
        is_retryable: Optional[bool] = None,
        is_critical: Optional[bool] = None,
        context: Optional[Dict[str, Any]] = None,
    ) -> None:
        super().__init__(message)
        if error_code is not None:
            self.error_code = error_code
        if is_retryable is not None:
            self.is_retryable = is_retryable
        if is_critical is not None:
            self.is_critical = is_critical
        self.context: Dict[str, Any] = context or {}
```

#### `src/exceptions/workflow_exceptions.py`

```python
"""Workflow-layer exceptions."""

from __future__ import annotations

from .base_exception import AgentException


class WorkflowExecutionError(AgentException):
    """Raised when the workflow fails outside a single node."""

    error_code = "WORKFLOW_EXECUTION_ERROR"


class NodeFailureError(AgentException):
    """Raised when a single workflow node fails."""

    error_code = "NODE_FAILURE"
```

#### `src/exceptions/memory_exceptions.py`

```python
"""Memory adapter exceptions."""

from __future__ import annotations

from .base_exception import AgentException


class MemoryAccessError(AgentException):
    """Raised when reading/writing memory fails."""

    error_code = "MEMORY_ACCESS_ERROR"


class MemoryConnectionError(AgentException):
    """Raised when the underlying store cannot be reached."""

    error_code = "MEMORY_CONNECTION_ERROR"
    is_retryable = True
```

#### `src/exceptions/tool_exceptions.py`

```python
"""Tool-layer exceptions."""

from __future__ import annotations

from .base_exception import AgentException


class ToolExecutionError(AgentException):
    """Raised when a tool invocation fails."""

    error_code = "TOOL_EXECUTION_ERROR"


class ToolNotFoundError(AgentException):
    """Raised when a requested tool is not registered."""

    error_code = "TOOL_NOT_FOUND"
```

### 3.3 Tests scaffolding

#### `tests/__init__.py` (empty)
#### `tests/unit/__init__.py` (empty)
#### `tests/integration/__init__.py` (empty)

#### `tests/conftest.py`

```python
"""Shared pytest fixtures for __AGENT_CLASS__."""
```

#### `tests/unit/test_placeholder.py`

```python
"""Placeholder unit test — replace with real tests."""


def test_placeholder() -> None:
    assert True
```

---

## Phase 4 — Verify

Run (inside the new container):

```sh
cd apps/ai-service/containers/<container-name>
uv sync
```

If `uv sync` succeeds, the scaffold is valid.

---

## Phase 5 — Next steps

Tell the user:

> Scaffold complete at `apps/ai-service/containers/<container-name>/`.
>
> To wire infrastructure (logging, vault, config_manager, global state, mongodb, weaviate), invoke the `setup-agent-infra` skill and list the pieces you want. Only the ones you name will be added.

---

## Rules

- Do NOT wire logging, vault, config_manager, global state, mongodb, or weaviate here. That is the job of `setup-agent-infra`.
- Do NOT invent business-logic nodes/tools. Stubs only.
- Always substitute both `__CONTAINER_NAME__` and `__AGENT_CLASS__` placeholders.
- If the container already exists, STOP and report to the user.
