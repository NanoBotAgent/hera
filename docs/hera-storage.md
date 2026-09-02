# Brief: build the `hera_storage` library

You are building a standalone Python library in an empty repository. It is the foundation of a
larger system (`hera`), but it must know **nothing** about it.

## Context

`hera` is a personal agentic framework, split into several independent Python packages, each with
its own repo:

```
heraAPI (FastAPI application, wires everything together)
  hera_profiles       hera_promptevo
  hera_tools          hera_memories       hera_skillsets
  hera_prompts        hera_providers      hera_permissions     hera_chats
                              hera_storage          <-- this repo
```

Dependencies point downwards only. `hera_storage` sits at the very bottom and imports **no**
other `hera_*` library — not now, not ever.

The domain libraries above it (`hera_chats`, `hera_memories`, `hera_promptevo`, `hera_profiles`)
each define their **own** SQLModel tables and inherit from the base classes in `hera_storage` to
do so. They don't know about each other; cross-references between libraries exist only as bare
`UUID` fields with no foreign-key constraint. Linking tables live in `heraAPI`.

## The one hard rule

**`hera_storage` contains not a single table and not a single domain concept.** No chat, no
message, no prompt, no provider, no memory. If you find yourself typing the word "chat" while
writing this, something is wrong. The library must be usable unchanged in a completely unrelated
project (a recipe manager, say).

Second rule: no `table=True` class in this library. All base classes are mixins with no table of
their own.

## Technical requirements

- Python 3.12+, type annotations everywhere, `from __future__ import annotations`.
- Dependencies: `sqlmodel`, `pydantic-settings`. Nothing else. **No Alembic** as a runtime
  dependency — migrations run in `heraAPI`; we only provide the `MetaData` and the naming
  convention.
- Build with `uv` and `hatchling`, package name `hera-storage`, import name `hera_storage`.
- **Synchronous** SQLAlchemy, no async sessions. Reasoning: the target environment is SQLite on a
  Mac Mini for a single user; DB access sits in the microsecond range, while the real latency is
  in the LLM calls. Async would only add complexity with no payoff. The session API must still be
  thread-safe to use (one session per request, never shared globally).
- Primary database is SQLite; PostgreSQL should work without code changes (no SQLite-specific
  types in the public API).

## Public API

Everything below is importable directly from `hera_storage`. Stick to these signatures — they are
the contract five other libraries will build on.

### 1. Settings

```python
class StorageSettings(BaseSettings):
    # env prefix: HERA_STORAGE_
    url: str = "sqlite:///hera.db"
    echo: bool = False
    sqlite_wal: bool = True
    busy_timeout_ms: int = 5000
    pool_size: int = 5
```

### 2. Database

```python
class Database:
    def __init__(self, settings: StorageSettings | None = None, *, url: str | None = None) -> None
    @classmethod
    def from_env(cls) -> Database
    @classmethod
    def in_memory(cls) -> Database          # StaticPool, for tests

    @property
    def engine(self) -> Engine
    @property
    def metadata(self) -> MetaData          # SQLModel.metadata, for Alembic in heraAPI

    @contextmanager
    def session(self) -> Iterator[Session]  # commit on success, rollback on exception, always close
    def dependency(self) -> Callable[[], Iterator[Session]]   # for FastAPI Depends()
    def create_all(self) -> None            # tests/bootstrap only, not for production migrations
    def dispose(self) -> None
```

For SQLite URLs, a `connect` event listener must always set:
`PRAGMA journal_mode=WAL` (when `sqlite_wal`), `PRAGMA foreign_keys=ON`,
`PRAGMA busy_timeout=<busy_timeout_ms>`.
For `in_memory()`, also `StaticPool` and `check_same_thread=False` — otherwise every connection
attempt sees an empty DB.

### 3. Base classes (mixins, no `table=True`)

```python
class EntityStatus(StrEnum):
    ACTIVE = "active"
    REVOKED = "revoked"

class Entity(SQLModel):
    id: UUID              # default_factory=uuid4, primary_key
    created_at: datetime  # UTC, default_factory
    updated_at: datetime  # UTC, onupdate

class SoftDeletable(SQLModel):
    status: EntityStatus = EntityStatus.ACTIVE
    revoked_at: datetime | None = None

class Versioned(SQLModel):
    version: int = 1
    supersedes_id: UUID | None = None   # points at the previous version
    origin: str | None = None           # "manual" | "dream:<uuid>" | "selection:gen7"
    is_current: bool = True
```

`updated_at` must update even on a plain attribute assignment with no explicit call — via
`sa_column_kwargs={"onupdate": ...}`, not via manual setting in the repository.

All three mixins must be freely combinable: `class Foo(Entity, SoftDeletable, Versioned,
table=True)` must work with no MRO or field conflicts. Write an explicit test for that.

Also export `NAMING_CONVENTION: dict[str, str]` (the standard SQLAlchemy convention for
`ix`/`uq`/`ck`/`fk`/`pk`) and apply it to the metadata. Without named constraints, Alembic's batch
mode fails under SQLite on every column change — that's the reason this exists.

### 4. Repository

```python
T = TypeVar("T", bound=SQLModel)

class Repository(Generic[T]):
    def __init__(self, model: type[T], session: Session) -> None

    def get(self, id: UUID, *, include_revoked: bool = False) -> T | None
    def get_or_raise(self, id: UUID, *, include_revoked: bool = False) -> T
    def list(self, *where: Any, order_by: Any = None, limit: int | None = None,
             offset: int = 0, include_revoked: bool = False) -> list[T]
    def add(self, obj: T) -> T
    def add_all(self, objs: Iterable[T]) -> list[T]
    def save(self, obj: T) -> T
    def revoke(self, id: UUID) -> T        # sets status + revoked_at
    def restore(self, id: UUID) -> T
    def hard_delete(self, id: UUID) -> None
    def count(self, *where: Any, include_revoked: bool = False) -> int
    def exists(self, id: UUID) -> bool
```

Important: `include_revoked=False` only filters when the model actually inherits from
`SoftDeletable` — otherwise the parameter is ignored rather than raising. `revoke`/`restore` raise
`TypeError` when the model isn't soft-deletable.

`Repository` is meant to be subclassed: domain libraries write `class ChatRepository(Repository
[Chat])` and add domain-specific methods. Document that in the README with an example.

### 5. Versioning

```python
def new_version(session: Session, obj: V, *, origin: str, **changes: Any) -> V
def version_history(session: Session, model: type[V], id: UUID) -> list[V]
def current_version(session: Session, model: type[V], id: UUID) -> V | None
```

`new_version` copies the object, applies `**changes`, sets `version += 1`,
`supersedes_id = obj.id`, `origin`, assigns a new `id`, sets `is_current=False` on the old object
and `True` on the new one. One snapshot is stored per version, not a diff.

`version_history` follows the `supersedes_id` chain backwards and returns versions in ascending
chronological order. It must still terminate if the chain were ever cyclic due to a bug — build in
a guard limit.

### 6. Errors

```python
class StorageError(Exception): ...
class NotFound(StorageError): ...       # carries model_name and id
class Conflict(StorageError): ...       # wraps IntegrityError
```

`Database.session()` catches `IntegrityError` and raises `Conflict` with the original exception
as `__cause__`. Other DB errors are not swallowed.

### 7. Test support

A `hera_storage.testing` module with pytest fixtures (`db`, `session`), registered as a pytest
plugin via the `pytest11` entry point in `pyproject.toml`. That gives every domain library
`def test_x(session): ...` for free, with no duplicated setup.

## Conventions to document in the README

- **Table prefixes.** Every domain library sets `__tablename__` explicitly with its own prefix
  (`chat_messages`, `mem_entries`, `evo_generations`). All models land in the same `MetaData`, so
  identically named tables from two libraries would otherwise collide silently.
- **No cross-library foreign keys.** References to entities in other libraries are bare `UUID`
  fields. Integrity is enforced by the application layer, not the DB.
- **Migrations run in heraAPI.** That's where every library gets imported, which registers its
  models; `alembic autogenerate` then sees the full schema. Write this as a short section in the
  README, including a note on `render_as_batch=True` for SQLite.

## Quality requirements

- `ruff` (lint + format) and `mypy --strict` run clean.
- pytest with in-memory SQLite. Tests define their own dummy models
  (`class Widget(Entity, SoftDeletable, Versioned, table=True)`) — there is nothing
  domain-specific to test in this library.
- Test coverage at least 90% on `repository.py` and `versioning.py`.
- At minimum, these cases are covered explicitly: rollback on an exception inside the `session()`
  block; `include_revoked` on a model without `SoftDeletable`; `revoke` on a model without
  `SoftDeletable` raises; combined mixin inheritance; `new_version` across three generations with
  correct history; `Conflict` on a unique-constraint violation; the WAL pragma is actually set on
  file-backed SQLite.
- README covering: purpose, installation, a 20-line quickstart, an example of a derived
  repository, a section on migrations, and an explicit "what does **not** belong here" section.

## Order of work

1. `pyproject.toml`, repo structure (`src/hera_storage/`), tooling configuration.
2. `settings.py`, `errors.py`, `base.py`.
3. `database.py`, including the SQLite pragma listener.
4. `repository.py`.
5. `versioning.py`.
6. `testing.py` and the entry point.
7. Tests.
8. README.

Pause briefly after step 3 and show me the base classes plus `Database` before continuing —
everything else depends on them, and a mistake there is expensive later.

## Do not build

No caching, no connection retry, no event/outbox system, no full-text search, no embeddings, no
multi-tenancy, no async variant, no CLI. If a feature seems useful while you're working on this
and it isn't listed here: don't add it — name it as a suggestion at the end instead.
