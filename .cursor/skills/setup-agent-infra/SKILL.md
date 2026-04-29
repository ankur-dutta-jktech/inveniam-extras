# Setup Agent Infrastructure

Wire infrastructure pieces into an already-scaffolded agent container at
`apps/ai-service/containers/<name>/`. Each piece uses the corresponding
shared package directly (thin wiring, no thick local wrappers).

## Available pieces

| # | Piece | Shared package |
|---|-------|----------------|
| a | Logging | `inveniam-logger` |
| b | Vault | `inveniam-vault` |
| c | Config manager | Pydantic (in-container) |
| d | Global state | singleton (in-container) |
| e | MongoDB | `agent-adapters-mongodb` |
| f | Weaviate | `agent-adapters-weaviate` |

---

## MANDATORY GATE — only build what was asked

Before touching any file:

1. Read the user's request and enumerate which of the six pieces (`logging`, `vault`, `config_manager`, `global_state`, `mongodb`, `weaviate`) were explicitly requested.
2. If the request is ambiguous (e.g. "set up infra for `appraisal_validity_agent`" without naming pieces), call `AskQuestion` with a single multi-select question listing the six pieces. Do NOT proceed until the user picks.
3. Implement ONLY the pieces in the confirmed set. Do not add anything else "to be helpful". For example, if only `mongodb` is requested, do NOT add vault/global state/logging even though they are common companions — the user will ask later.
4. Dependency note: if the user asks for `vault`, `mongodb`, or `weaviate`, those sections' startup hooks reference `GlobalState`. If `global_state` was not also asked, still write the startup hook but comment in the response that "these hooks require `global_state` — add it with this skill if you want the cached-client pattern". Do NOT implicitly create `global_state`.

---

## Trigger

Use this skill when the user asks any of:

- "Set up logging / vault / mongodb / weaviate / config / global state for `<name>` agent"
- "Wire shared packages into `<name>` container"
- "Add `inveniam-vault` to `<name>`"

If the user names a container that does not exist under `apps/ai-service/containers/`, tell them to run the `scaffold-agent-container` skill first.

---

## Resolve inputs

1. `<container-name>` — required. Derive from the user's message or ask.
2. `CONTAINER_PATH` = `apps/ai-service/containers/<container-name>`
3. Confirm `CONTAINER_PATH` exists and contains `pyproject.toml`.
4. Read `CONTAINER_PATH/pyproject.toml` once so you know which deps are already present; do not add duplicate entries.

For each section below, after applying file + `pyproject.toml` changes, add the corresponding "startup hook" snippet to `CONTAINER_PATH/src/main.py`. The hook lines are additive — preserve anything already in `startup()`.

After all requested sections are done, run:

```sh
cd apps/ai-service/containers/<container-name>
uv sync
```

---

## Section (a) — Logging

Shared package: `inveniam-logger`.

### Files

No new files. Add this import at the top of any module that needs a logger:

```python
from inveniam_logger import get_logger

logger = get_logger(__name__)
```

Replace `src/main.py`'s stdlib `logging.getLogger(__name__)` with `get_logger(__name__)` from `inveniam_logger`, and remove the `import logging` line if nothing else uses it.

### `pyproject.toml` delta

Add to `[project] dependencies` (skip if already present):

```toml
"inveniam-logger",
```

Add to `[tool.uv.sources]`:

```toml
inveniam-logger = { path = "../../../../shared-packages/inveniam-logger/v1.0.0" }
```

### Dockerfile delta

Add before the runtime stage (next to the existing `uv build && uv pip install` blocks):

```dockerfile
WORKDIR /app/shared-packages/inveniam-logger/v1.0.0
RUN uv build && \
    uv pip install --no-cache-dir dist/*.whl --python /app/apps/ai-service/containers/<container-name>/.venv/bin/python
```

### Env vars to document in README

- `LOG_LEVEL` (default `INFO`) — `DEBUG` / `INFO` / `WARNING` / `ERROR`
- `LOG_JSON_FORMAT` (default `true`) — set to `false` for human-readable console output

### Startup hook

None. Importing `inveniam_logger` auto-configures logging.

---

## Section (b) — Vault

Shared package: `inveniam-vault`.

### Files

Create `src/services/vault/vault_config.py`:

```python
"""Vault credential resolution (thin wrapper around `inveniam_vault`)."""

from __future__ import annotations

import os
from typing import Any, Dict, Optional, Tuple

from inveniam_logger import get_logger
from pydantic import BaseModel, Field

try:
    from inveniam_vault import K8S_TOKEN_PATH, get_vault_client
except ModuleNotFoundError:  # pragma: no cover
    K8S_TOKEN_PATH = "/var/run/secrets/kubernetes.io/serviceaccount/token"

    def get_vault_client():
        raise ModuleNotFoundError("inveniam_vault is required for Vault integration.")

logger = get_logger(__name__)


class ConfigValidationError(Exception):
    """Raised when Vault-derived configuration cannot be initialized."""


def vault_client_precheck() -> Tuple[bool, str]:
    """Validate env preconditions before calling `get_vault_client`."""
    auth_method = (os.getenv("VAULT_AUTH_METHOD") or "AWS").upper()
    missing: list[str] = []

    if not os.getenv("VAULT_ENDPOINT"):
        missing.append("VAULT_ENDPOINT")
    if not os.getenv("VAULT_ROLE"):
        missing.append("VAULT_ROLE")
    if auth_method == "AWS" and not os.getenv("AWS_COMMON_REGION"):
        missing.append("AWS_COMMON_REGION")
    if auth_method == "KUBERNETES" and not os.path.exists(K8S_TOKEN_PATH):
        missing.append(f"{K8S_TOKEN_PATH} (k8s service-account token)")

    if missing:
        return False, f"Vault precheck failed; missing: {', '.join(missing)}"
    return True, "ok"


def resolve_vault_client(force_refresh: bool = False) -> Any:
    """Return cached Vault client or initialize one when env preconditions pass.

    Falls back to a fresh `get_vault_client()` call if `GlobalState` is not
    available in this container.
    """
    try:
        from utils.global_state import GlobalState, GlobalStateKey  # type: ignore

        state: Optional[Any] = GlobalState()
    except Exception:  # pylint: disable=broad-exception-caught
        state = None

    if state is not None and not force_refresh:
        try:
            cached = state.get_state(GlobalStateKey.VAULT_CLIENT)
            if cached:
                return cached
        except Exception:  # pylint: disable=broad-exception-caught
            pass

    ready, reason = vault_client_precheck()
    if not ready:
        logger.warning(reason)
        return None

    try:
        client = get_vault_client()
        if client and state is not None:
            state.set_state(GlobalStateKey.VAULT_CLIENT, client)
        return client
    except Exception as exc:  # pylint: disable=broad-exception-caught
        logger.warning("Vault client initialization failed: %s", exc)
        return None


class VaultConfig(BaseModel):
    """Resolved vault-backed credentials (extend with agent-specific fields)."""

    llm_config: Optional[Dict[str, Optional[str]]] = Field(default=None)

    @classmethod
    def from_env(cls, credential_path: Optional[str] = None) -> "VaultConfig":
        raw_credential = credential_path if credential_path is not None else os.getenv("LLM_API_KEY")
        raw_base = os.getenv("LLM_API_BASE")

        if not raw_credential:
            return cls(llm_config={"api_key": None, "base_url": raw_base})

        if not raw_credential.startswith("vault://"):
            return cls(llm_config={"api_key": raw_credential, "base_url": raw_base})

        vault_client = resolve_vault_client()
        if not vault_client:
            logger.warning("Vault credential path configured but no vault client available")
            return cls(llm_config={"api_key": None, "base_url": raw_base})

        try:
            api_key = cls._read_vault_key(raw_credential, vault_client)
            base_url = cls._read_vault_key(raw_base, vault_client) if raw_base and raw_base.startswith("vault://") else raw_base
            return cls(llm_config={"api_key": api_key, "base_url": base_url})
        except Exception as exc:  # pylint: disable=broad-exception-caught
            logger.warning("Failed to read vault-backed credentials: %s", exc)
            return cls(llm_config={"api_key": None, "base_url": raw_base})

    @staticmethod
    def _read_vault_key(var: Optional[str], vault_client: Any) -> Optional[str]:
        if var is None:
            return None
        if not var.startswith("vault://"):
            return var
        parts = var.split("/")
        if len(parts) < 4:
            raise ValueError(f"Invalid vault path: {var}")
        secret_engine = parts[2]
        secret_path = "/".join(parts[3:-1])
        key = parts[-1]
        response = vault_client.read(f"/{secret_engine}/data/{secret_path}")
        if not response:
            raise ConfigValidationError(f"Failed to read secret from Vault: {var}")
        return response.get("data", {}).get("data", {}).get(key)
```

Create `src/services/vault/__init__.py` (if not present):

```python
from .vault_config import VaultConfig, resolve_vault_client, vault_client_precheck

__all__ = ["VaultConfig", "resolve_vault_client", "vault_client_precheck"]
```

### `pyproject.toml` delta

```toml
# dependencies
"inveniam-vault",

# [tool.uv.sources]
inveniam-vault = { path = "../../../../shared-packages/inveniam-vault/v1.0.0" }
```

### Dockerfile delta

```dockerfile
WORKDIR /app/shared-packages/inveniam-vault/v1.0.0
RUN uv build && \
    uv pip install --no-cache-dir dist/*.whl --python /app/apps/ai-service/containers/<container-name>/.venv/bin/python
```

### Env vars to document

- `VAULT_ENDPOINT` (required)
- `VAULT_ROLE` (required)
- `VAULT_AUTH_METHOD` (default `AWS`; also `AZURE`, `KUBERNETES`)
- `AWS_COMMON_REGION` (required for AWS auth)

### Startup hook (add to `src/main.py` `startup()`)

```python
from services.vault.vault_config import resolve_vault_client

vault_client = resolve_vault_client(force_refresh=True)
if vault_client:
    logger.info("Vault client initialized")
else:
    logger.warning("Vault client unavailable; continuing without vault-backed secrets")
```

If `global_state` is present in this container, also:

```python
from utils.global_state import GlobalState, GlobalStateKey
GlobalState().set_state(GlobalStateKey.VAULT_CLIENT, vault_client)
```

---

## Section (c) — Config manager

In-container Pydantic configuration with `@lru_cache get_config()`. No shared package required beyond `pydantic` (already in base deps) and optionally `inveniam-vault` (if `getenv` should resolve `vault://` URIs).

### Files

Create `src/config/config_helper.py`:

```python
"""Environment-variable helper with optional vault:// resolution."""

from __future__ import annotations

import os
from typing import Callable, Optional, TypeVar

from inveniam_logger import get_logger

T = TypeVar("T")

logger = get_logger(__name__)


def _resolve_vault(raw: str) -> str:
    """Resolve a `vault://` URI to its secret value. Requires `inveniam-vault`
    and (optionally) a cached client in `GlobalState`."""
    try:
        from inveniam_vault import convert_vault_path, get_vault_client  # type: ignore
    except ModuleNotFoundError:
        raise ValueError("vault:// URI encountered but inveniam-vault is not installed")

    client = None
    try:
        from utils.global_state import GlobalState, GlobalStateKey  # type: ignore

        client = GlobalState().get_state(GlobalStateKey.VAULT_CLIENT)
    except Exception:  # pylint: disable=broad-exception-caught
        client = None
    if client is None:
        client = get_vault_client()
    if client is None:
        raise ValueError("Vault client is not initialized")

    secret_path, key = convert_vault_path(raw)
    response = client.read(secret_path)
    if not response:
        raise ValueError(f"Failed to read secret from vault path: {secret_path}")
    data = response.get("data", {}).get("data", {}).get(key)
    if data is None:
        raise ValueError(f"Key '{key}' not found at {secret_path}")
    return data


def getenv(
    key: str,
    default: Optional[T] = None,
    parse: Callable[[str], T] = lambda x: x,
    required: bool = False,
) -> T:
    """Read env var with optional parsing, default, and vault:// resolution."""
    raw = os.getenv(key)
    if raw and raw.startswith("vault://"):
        raw = _resolve_vault(raw)
    if raw is None:
        if required:
            raise ValueError(f"Required environment variable '{key}' is not set.")
        if default is not None and parse is not None and isinstance(default, str):
            try:
                return parse(default)
            except Exception as exc:  # pylint: disable=broad-exception-caught
                raise ValueError(f"Error parsing default for env var '{key}': {exc}") from exc
        return default  # type: ignore[return-value]
    try:
        return parse(raw)
    except Exception as exc:
        raise ValueError(f"Error parsing env var '{key}': {exc}") from exc
```

Create `src/config/config_manager.py`:

```python
"""Pydantic configuration for the agent.

Extend the `AgentConfig` sub-model with agent-specific fields, and add
new sub-models to `Config` as needed.
"""

from __future__ import annotations

import uuid
from functools import lru_cache
from typing import Any, Dict, Optional

from inveniam_logger import get_logger
from pydantic import BaseModel, ConfigDict, Field

from .config_helper import getenv

logger = get_logger(__name__)


class AuthConfig(BaseModel):
    m2m_aia_app_id: Optional[str] = Field(default=None)
    m2m_client_id: Optional[str] = Field(default=None)
    m2m_client_secret: Optional[str] = Field(default=None)
    auth_url: str = Field(default="https://inveniam-dev.fusionauth.io/oauth2/token")


class OrganizationConfig(BaseModel):
    organisation_id: Optional[uuid.UUID] = Field(default=None)
    user_id: Optional[str] = Field(default=None)
    job_id: Optional[str] = Field(default=None)


class VaultSettings(BaseModel):
    vault_endpoint: Optional[str] = Field(default=None)
    vault_role: Optional[str] = Field(default=None)
    vault_auth_method: str = Field(default="AWS")


class RetryConfig(BaseModel):
    retry_max_attempts: int = Field(default=3)
    retry_min_wait_seconds: int = Field(default=1)
    retry_max_wait_seconds: int = Field(default=30)
    retry_multiplier: float = Field(default=1.0)


class AgentConfig(BaseModel):
    """Extend this class with agent-specific fields."""

    agent_id: str = Field(default="agent")
    agent_request_id: Optional[str] = Field(default=None)


class Config(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)

    auth: AuthConfig
    organization: OrganizationConfig
    vault: VaultSettings
    retry: RetryConfig
    agent: AgentConfig
    additional: Dict[str, Any] = Field(default_factory=dict)


@lru_cache()
def get_config() -> Config:
    """Load configuration from environment. Cached for the process lifetime."""
    return Config(
        auth=AuthConfig(
            m2m_aia_app_id=getenv("M2M_AIA_APP_ID"),
            m2m_client_id=getenv("M2M_CLIENT_ID"),
            m2m_client_secret=getenv("M2M_CLIENT_SECRET"),
            auth_url=getenv(
                "AUTH_URL",
                default="https://inveniam-dev.fusionauth.io/oauth2/token",
            ),
        ),
        organization=OrganizationConfig(
            organisation_id=getenv("ORGANISATION_ID", parse=uuid.UUID),
            user_id=getenv("USER_ID"),
            job_id=getenv("JOB_ID"),
        ),
        vault=VaultSettings(
            vault_endpoint=getenv("VAULT_ENDPOINT"),
            vault_role=getenv("VAULT_ROLE"),
            vault_auth_method=getenv("VAULT_AUTH_METHOD", default="AWS"),
        ),
        retry=RetryConfig(
            retry_max_attempts=getenv("RETRY_MAX_ATTEMPTS", default=3, parse=int),
            retry_min_wait_seconds=getenv("RETRY_MIN_WAIT_SECONDS", default=1, parse=int),
            retry_max_wait_seconds=getenv("RETRY_MAX_WAIT_SECONDS", default=30, parse=int),
            retry_multiplier=getenv("RETRY_MULTIPLIER", default=1.0, parse=float),
        ),
        agent=AgentConfig(
            agent_id=getenv("AGENT_ID", default="agent"),
            agent_request_id=getenv("AGENT_REQUEST_ID"),
        ),
    )
```

Overwrite `src/config/__init__.py`:

```python
from .config_manager import (
    AgentConfig,
    AuthConfig,
    Config,
    OrganizationConfig,
    RetryConfig,
    VaultSettings,
    get_config,
)

__all__ = [
    "AgentConfig",
    "AuthConfig",
    "Config",
    "OrganizationConfig",
    "RetryConfig",
    "VaultSettings",
    "get_config",
]
```

### `pyproject.toml` delta

None (Pydantic already in base deps). Config-manager relies on `inveniam-logger`
and (for vault:// env vars) on `inveniam-vault`; add those via their own
sections if the user requested them.

### Startup hook

```python
from config import get_config

config = get_config()
logger.info("Config loaded", extra={"agent_id": config.agent.agent_id})
```

---

## Section (d) — Global state

In-container singleton. No shared package.

### Files

Create `src/utils/singleton.py`:

```python
"""Simple class-level singleton decorator."""

from __future__ import annotations

from typing import Type, TypeVar

T = TypeVar("T")


def singleton(cls: Type[T]) -> Type[T]:
    """Decorate a class so instantiating it always returns the same instance."""
    instances: dict[Type[T], T] = {}

    def get_instance(*args, **kwargs) -> T:
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance  # type: ignore[return-value]
```

Create `src/utils/global_state.py`:

```python
"""Process-wide state manager (keyed access + shutdown flag)."""

from __future__ import annotations

from enum import Enum
from typing import Any

from utils.singleton import singleton


class GlobalStateKey(Enum):
    """Add keys here as needed. Keep values in sync with consumers."""

    IS_SHUTTING_DOWN = "is_shutting_down"
    DATABASE_CONFIG = "database_config"
    ORG_LLM_PROVIDER = "org_llm_provider"
    VAULT_CLIENT = "vault_client"
    VAULT_CONFIG = "vault_config"
    MONGO_CLIENT = "mongo_client"
    WEAVIATE_CLIENT = "weaviate_client"
    LLM_CLIENT = "llm_client"
    COMPLETION_SERVICE = "completion_service"


@singleton
class GlobalState:
    """Global application state manager.

    Not thread-safe. Set state during single-threaded startup, then read
    during execution.
    """

    def __init__(self) -> None:
        self._is_shutting_down: bool = False
        self._state: dict[GlobalStateKey, Any] = {}

    @property
    def is_shutting_down(self) -> bool:
        return self._is_shutting_down

    def mark_shutdown(self) -> None:
        self._is_shutting_down = True

    def reset(self) -> None:
        self._is_shutting_down = False

    def set_state(self, key: GlobalStateKey, value: Any) -> None:
        self._state[key] = value

    def get_state(self, key: GlobalStateKey) -> Any:
        return self._state.get(key)
```

### `pyproject.toml` delta

None.

### Startup hook

No new hook; other sections call `GlobalState().set_state(...)` themselves.
Add this line to `cleanup_resources()`:

```python
from utils.global_state import GlobalState
GlobalState().mark_shutdown()
```

### Extending keys

To add a new key, append a member to `GlobalStateKey`. Keep key names aligned
with the shared-package clients you cache.

---

## Section (e) — MongoDB

Shared package: `agent-adapters-mongodb`. Uses `MongoDBClient` + `MongoDBConfig`
directly.

### Files

Create `src/database/__init__.py`:

```python
from .mongodb.client_factory import build_mongo_client

__all__ = ["build_mongo_client"]
```

Create `src/database/mongodb/__init__.py` (empty).

Create `src/database/mongodb/client_factory.py`:

```python
"""Thin factory that builds a `MongoDBClient` from config + vault credentials."""

from __future__ import annotations

import os
from typing import Any, Dict, Optional

from agent_adapters_mongodb import (
    AUTH_MECHANISM_X509,
    DEFAULT_AUTH_SOURCE,
    MongoDBClient,
    MongoDBConfig,
)
from inveniam_logger import get_logger

logger = get_logger(__name__)


def build_mongo_config(vault_credentials: Optional[Dict[str, Any]] = None) -> MongoDBConfig:
    """Build `MongoDBConfig` from env + optional vault-resolved credentials.

    Supports both X.509 (when `client_cert_content` / `ca_cert_content` are
    provided in vault) and username/password fallbacks.
    """
    host = os.getenv("MONGO_HOST", "localhost")
    port = int(os.getenv("MONGO_PORT", "27017"))
    database_name = os.getenv("MONGO_DATABASE", "agent_db")

    creds = vault_credentials or {}
    if creds.get("clientCertificateKeyFileContent"):
        return MongoDBConfig(
            host=host,
            port=port,
            database_name=database_name,
            client_cert_content=creds["clientCertificateKeyFileContent"],
            ca_cert_content=creds.get("caCertificateFileContent"),
            client_key_password=creds.get("clientKeyPassword"),
            auth_mechanism=AUTH_MECHANISM_X509,
            auth_source=DEFAULT_AUTH_SOURCE,
        )

    return MongoDBConfig(
        host=host,
        port=port,
        database_name=database_name,
        username=os.getenv("MONGO_USERNAME"),
        password=os.getenv("MONGO_PASSWORD"),
        auth_source=os.getenv("MONGO_AUTH_SOURCE", "admin"),
        tls_enabled=os.getenv("MONGO_TLS_ENABLED", "true").lower() in {"1", "true", "yes"},
    )


def build_mongo_client(vault_credentials: Optional[Dict[str, Any]] = None) -> MongoDBClient:
    """Construct and connect an `MongoDBClient`."""
    config = build_mongo_config(vault_credentials)
    client = MongoDBClient(config)
    client.connect()
    logger.info("MongoDB client connected", extra={"database": config.database_name})
    return client
```

### `pyproject.toml` delta

```toml
# dependencies
"agent-adapters-mongodb",

# [tool.uv.sources]
agent-adapters-mongodb = { path = "../../../../shared-packages/agent-adapters-mongodb/v1.0.0" }
```

### Dockerfile delta

```dockerfile
WORKDIR /app/shared-packages/agent-adapters-mongodb/v1.0.0
RUN uv build && \
    uv pip install --no-cache-dir dist/*.whl --python /app/apps/ai-service/containers/<container-name>/.venv/bin/python
```

### Env vars to document

- `MONGO_HOST`, `MONGO_PORT`, `MONGO_DATABASE`
- `MONGO_USERNAME`, `MONGO_PASSWORD`, `MONGO_AUTH_SOURCE` (for password auth)
- `MONGO_TLS_ENABLED` (default `true`)

### Startup hook (add to `startup()`)

```python
from database.mongodb.client_factory import build_mongo_client

try:
    mongo_client = build_mongo_client()  # pass vault_credentials dict if loaded
except Exception:  # pylint: disable=broad-exception-caught
    logger.exception("MongoDB initialization failed")
    mongo_client = None
```

If `global_state` is present:

```python
from utils.global_state import GlobalState, GlobalStateKey
GlobalState().set_state(GlobalStateKey.MONGO_CLIENT, mongo_client)
```

### Cleanup hook (add to `cleanup_resources()`)

```python
try:
    from utils.global_state import GlobalState, GlobalStateKey
    mongo_client = GlobalState().get_state(GlobalStateKey.MONGO_CLIENT)
    if mongo_client:
        mongo_client.close()
except Exception:  # pylint: disable=broad-exception-caught
    logger.exception("MongoDB cleanup failed")
```

---

## Section (f) — Weaviate

Shared package: `agent-adapters-weaviate`. Uses `WeaviateClient` + `WeaviateConfig`.

### Files

Create `src/vectorstore/__init__.py`:

```python
from .weaviate.client_factory import build_weaviate_client

__all__ = ["build_weaviate_client"]
```

Create `src/vectorstore/weaviate/__init__.py` (empty).

Create `src/vectorstore/weaviate/client_factory.py`:

```python
"""Thin factory that builds a `WeaviateClient` from config + vault credentials."""

from __future__ import annotations

import os
from typing import Any, Dict, Optional

from agent_adapters_weaviate import WeaviateClient, WeaviateConfig
from inveniam_logger import get_logger

logger = get_logger(__name__)


def _as_bool(raw: Optional[str], default: bool) -> bool:
    if raw is None:
        return default
    return raw.lower() in {"1", "true", "yes", "t", "y", "on"}


def build_weaviate_config(vault_credentials: Optional[Dict[str, Any]] = None) -> WeaviateConfig:
    """Construct `WeaviateConfig` from env + optional vault-resolved credentials."""
    creds = vault_credentials or {}
    return WeaviateConfig(
        http_host=creds.get("httpHost") or os.getenv("WEAVIATE_DB_HTTP_HOST", ""),
        http_port=int(creds.get("httpPort") or os.getenv("WEAVIATE_DB_HTTP_PORT", "443")),
        http_secure=_as_bool(os.getenv("WEAVIATE_DB_HTTP_SECURE"), creds.get("httpSecure", True)),
        grpc_host=creds.get("grpcHost") or os.getenv("WEAVIATE_DB_GRPC_HOST", ""),
        grpc_port=int(creds.get("grpcPort") or os.getenv("WEAVIATE_DB_GRPC_PORT", "443")),
        grpc_secure=_as_bool(os.getenv("WEAVIATE_DB_GRPC_SECURE"), creds.get("grpcSecure", True)),
        api_key=creds.get("apiKey") or os.getenv("WEAVIATE_DB_API_KEY", ""),
    )


def build_weaviate_client(vault_credentials: Optional[Dict[str, Any]] = None) -> WeaviateClient:
    """Construct a `WeaviateClient` (auto-connects per the adapter)."""
    config = build_weaviate_config(vault_credentials)
    client = WeaviateClient(config)
    logger.info("Weaviate client initialized", extra={"http_host": config.http_host})
    return client
```

### `pyproject.toml` delta

```toml
# dependencies
"agent-adapters-weaviate",
"weaviate-client>=4.0.0",

# [tool.uv.sources]
agent-adapters-weaviate = { path = "../../../../shared-packages/agent-adapters-weaviate/v1.0.0" }
```

### Dockerfile delta

```dockerfile
WORKDIR /app/shared-packages/agent-adapters-weaviate/v1.0.0
RUN uv build && \
    uv pip install --no-cache-dir dist/*.whl --python /app/apps/ai-service/containers/<container-name>/.venv/bin/python
```

### Env vars to document

- `WEAVIATE_DB_HTTP_HOST`, `WEAVIATE_DB_HTTP_PORT`, `WEAVIATE_DB_HTTP_SECURE`
- `WEAVIATE_DB_GRPC_HOST`, `WEAVIATE_DB_GRPC_PORT`, `WEAVIATE_DB_GRPC_SECURE`
- `WEAVIATE_DB_API_KEY`

### Startup hook (add to `startup()`)

```python
from vectorstore.weaviate.client_factory import build_weaviate_client

try:
    weaviate_client = build_weaviate_client()
except Exception:  # pylint: disable=broad-exception-caught
    logger.exception("Weaviate initialization failed")
    weaviate_client = None
```

If `global_state` is present:

```python
from utils.global_state import GlobalState, GlobalStateKey
GlobalState().set_state(GlobalStateKey.WEAVIATE_CLIENT, weaviate_client)
```

### Cleanup hook

```python
try:
    from utils.global_state import GlobalState, GlobalStateKey
    weaviate_client = GlobalState().get_state(GlobalStateKey.WEAVIATE_CLIENT)
    if weaviate_client and hasattr(weaviate_client, "close"):
        weaviate_client.close()
except Exception:  # pylint: disable=broad-exception-caught
    logger.exception("Weaviate cleanup failed")
```

---

## Phase — Verify

After applying only the requested sections:

```sh
cd apps/ai-service/containers/<container-name>
uv sync
```

Report back to the user:

- The exact list of files created or modified.
- The exact list of `pyproject.toml` entries added.
- The exact sections that were **skipped** because the user did not request them, so the gate is visible.

---

## Anti-patterns (do not do)

- Do NOT create `src/vault/`, `src/database/`, `src/vectorstore/`, or `src/utils/` files unless the matching section was requested.
- Do NOT add `agent-adapters-postgres`, `agent-adapters-workflow`, `agent-llm-services`, or any other shared package not covered by a requested section.
- Do NOT modify `main.py`'s `startup()` to initialize systems that were not requested.
- Do NOT rewrite existing wiring if it's already present — only add missing pieces.
