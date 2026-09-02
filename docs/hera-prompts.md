# Brief: build the `hera_prompts` library

You are building a standalone Python library in an empty repository. It is a prompt compiler: it
holds a structured, serialisable prompt state and translates it into finished messages for an
LLM.

## Context

`hera` is a personal agentic framework, split into independent packages, each with its own repo:

```
heraAPI (FastAPI application, wires everything together)
  hera_profiles       hera_promptevo
  hera_tools          hera_memories       hera_skillsets
  hera_prompts        hera_providers      hera_permissions     hera_chats
                              hera_storage
```

Dependencies point downwards only. `hera_prompts` sits at the foundation level and imports **no**
other `hera_*` library — not even `hera_storage`. It has no persistence, no I/O, no network.

Target models are small local models (gpt-oss-20b via LM Studio, some 3B-class). That shapes the
defaults: plain grammar beats nesting.

## The three hard rules

1. **No domain knowledge.** `hera_prompts` doesn't know what a tool, a memory, a skill, or a chat
   is. Foreign content only ever arrives as pre-rendered strings via named slots.
2. **No evolution vocabulary.** The words generation, fitness, population, selection, parent,
   dream, and mutation must not appear anywhere in this library — not in identifiers, not in
   docstrings. It holds state and applies changes; who generates and scores those changes is
   unknown to it.
3. **Everything is immutable and serialisable.** Every transformation returns a new object. A
   `Prompt` must round-trip losslessly through `model_dump_json()` and back — otherwise the layer
   above it can't persist it.

## Technical requirements

- Python 3.12+, type annotations everywhere, `from __future__ import annotations`.
- One runtime dependency: `pydantic` v2. **No** tiktoken, no Jinja, no lxml.
- Build with `uv` and `hatchling`. Package name `hera-prompts`, import name `hera_prompts`.
- `ruff` and `mypy --strict` run clean; `py.typed` in the wheel.
- Determinism is a contract: the same object plus the same bindings produces byte-identical
  output. Every iteration over dicts runs in a defined order (traits sorted by key).

## Data model

### Roles and messages

```python
class Role(StrEnum):
    SYSTEM = "system"
    DEVELOPER = "developer"
    USER = "user"

class Message(BaseModel):
    role: Role
    content: str
```

`Message.model_dump()` already yields `{"role": ..., "content": ...}` — mapping onto a concrete
provider format is `hera_providers`'s job, not ours.

### Section

```python
class Section(BaseModel):
    key: str                      # stable address, "behavior.character"
    title: str | None = None
    content: str | None = None    # authored text
    slot: str | None = None       # OR the name of a placeholder
    children: list[Section] = []
    role: Role = Role.SYSTEM      # only evaluated at the top level
    priority: int = 100           # lower drops first under budget pressure
    required: bool = False
    locked: bool = False
    enabled: bool = True
```

Validated at construction time, not at render time:

- `key` matches `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`.
- A child's `key` starts with the parent's `key` plus a dot.
- Every `key` in the tree is unique.
- `content` and `slot` are mutually exclusive; a section with `children` has neither.

These rules exist because the predecessor prompt was hand-maintained and drifted (opening tag
`<hera:behavior>`, closing tag `</Hera_behavior>`). Structure is generated, never typed by hand.

### Traits

```python
TraitValue = bool | str | int

class TraitSpec(BaseModel):
    key: str                                  # like Section.key, prefix = target section
    type: Literal["str", "bool", "int"]
    default: TraitValue | None = None
    description: str = ""                     # for the generating layer
    choices: list[TraitValue] | None = None
    render: dict[str, str] | str | None = None
    locked: bool = False

class TraitRegistry(BaseModel):
    specs: list[TraitSpec]
    allow_unknown: bool = True

    def get(self, key: str) -> TraitSpec | None
    def validate_value(self, key: str, value: TraitValue) -> None   # raises TraitError
    def fingerprint(self) -> str
```

`render` is either a mapping from value to sentence
(`{"never": "Don't invent anything. If you don't know something, say so."}`) or a template with
`{value}`. If it's missing, or the trait is unknown to the registry, every renderer falls back to
the raw key/value pair.

`allow_unknown=True` allows traits that appear in no spec. That's the mode in which the layer
above may invent its own traits; `False` enforces the declared set. Both must be switchable
without a code change.

### Patch

```python
class TraitPatch(BaseModel):
    changes: dict[str, TraitValue | None]     # None means delete
    rationale: str | None = None

class RejectedChange(BaseModel):
    key: str
    reason: Literal["locked", "unknown_trait", "invalid_value", "invalid_key"]

class PatchResult(BaseModel):
    prompt: Prompt
    applied: dict[str, TraitValue | None]
    rejected: list[RejectedChange]
```

`apply()` does **not** raise when a patch runs into a lock — it drops the change and records it
in `rejected`. A caller that repeatedly touches locked traits is a signal the layer above wants to
see; aborting would kill an entire run instead.

### Prompt

```python
class Prompt(BaseModel):
    sections: list[Section]
    traits: dict[str, TraitValue] = {}
    locked_traits: set[str] = set()
    renderer: RendererConfig = RendererConfig()

    # Navigation
    def paths(self) -> list[str]
    def get(self, key: str) -> Section | None

    # Transformation, each returns a new object
    def replace(self, key: str, *, content: str | None = None, title: str | None = None) -> Prompt
    def insert(self, parent: str, section: Section, *, after: str | None = None) -> Prompt
    def remove(self, key: str) -> Prompt
    def reorder(self, parent: str, order: list[str]) -> Prompt
    def set_enabled(self, key: str, enabled: bool) -> Prompt
    def apply(self, patch: TraitPatch, *, registry: TraitRegistry | None = None) -> PatchResult

    # Identity
    def fingerprint(self) -> str

    # Output
    def render(self, *, bindings: Mapping[str, str] | None = None,
               registry: TraitRegistry | None = None,
               budget: TokenBudget | None = None) -> RenderResult
```

`replace`, `remove` and `reorder` respect `Section.locked` the same way `apply` respects
`locked_traits`: the change is dropped, not raised. `replace`/`remove` have no return channel for
that, so document that these methods return the unchanged object on a locked section, and provide
a helper method `is_locked(key)`.

`fingerprint()` is a SHA-256 over canonical JSON of sections, traits, and renderer config with
sorted keys. Two prompts with identical rendering must have the same fingerprint, or identical
work gets done twice.

Module function `diff(a: Prompt, b: Prompt) -> PromptDiff` with separate fields for changed
sections, changed traits, and changed renderer options.

## Rendering

```python
class RendererConfig(BaseModel):
    format: Literal["keyvalue", "xml", "markdown"] = "keyvalue"
    qualified_tags: bool = True          # <behavior:character> instead of <character>
    constraints_first: bool = True       # traits before authored text
    developer_role: Literal["fold_into_system", "native"] = "fold_into_system"
    trait_group_separator: str = " "     # "BEHAVIOR tone = terse"
```

The config lives **inside** the prompt object, not as a separate argument. That's the only way a
saved variant is fully described by the object — and the only way the layer above can vary the
format itself.

`developer_role="fold_into_system"` is the default because LM Studio and Ollama, through their
OpenAI-compatible endpoints, at best silently fold developer messages into system messages. When
folding, developer sections are appended after system sections into **one** message.

### Trait routing

The key determines the target: `behavior.tone` renders into the `behavior` section,
`formatting.max_words` into `formatting`. A trait with no dot lands in a general block at the
start of the system message. If a prefix points at a section that doesn't exist or is disabled,
the trait also moves to the general block — never dropped, never raised.

### Trait rendering per format

- `keyvalue`: always the raw pair, `GROUP name = value`. The `render` template is deliberately
  ignored — this grammar is itself the signal.
- `xml` and `markdown`: the `render` template, falling back to the raw pair.

That's a behavioural difference between renderers and must be tested as one.

### Slots

`bindings` is a mapping from slot name to a finished string. A slot with no binding drops the
section; if it's `required=True`, it raises `MissingBinding`. A binding with no matching slot
doesn't raise — it appears in `RenderResult.unused_bindings`.

### Budget

```python
class TokenBudget(BaseModel):
    limit: int
    counter: Callable[[str], int]     # default: len(text) // 4
    reserve: int = 0                  # for the expected answer
```

If rendering exceeds the budget, sections are removed in ascending `priority` order and
re-rendered until it fits. `required=True` protects against removal; if it's still too big
afterwards, it raises `BudgetExceeded`. Removed keys are in `RenderResult.dropped_keys` — that
field exists so a bad result can later be traced back to content missing under budget pressure.

### Result

```python
class PromptSnapshot(BaseModel):
    content_hash: str                 # SHA-256 over the rendered messages
    prompt_fingerprint: str
    registry_fingerprint: str | None
    renderer: RendererConfig
    traits: dict[str, TraitValue]
    dropped_keys: list[str]
    token_estimate: int
    component_versions: dict[str, UUID] = {}   # filled in by the layer above

class RenderResult(BaseModel):
    messages: list[Message]
    snapshot: PromptSnapshot
    unused_bindings: list[str]
```

`PromptSnapshot` is a plain model with no table — it's persisted in `heraAPI`.

Document explicitly: `messages` is the **frame**, not the full conversation. A conversation
history belongs between the system message(s) and the final user message, and is inserted by the
calling layer. `hera_prompts` knows nothing about history.

### Escaping

The XML renderer escapes `<`, `>` and `&` in content. The keyvalue renderer rejects trait values
containing a newline or `=` with `TraitError`, because they would break the grammar.

## Errors

```python
class PromptError(Exception): ...
class SectionError(PromptError): ...      # invalid key, duplicate, content+slot
class TraitError(PromptError): ...        # unknown with allow_unknown=False, type, choices
class MissingBinding(PromptError): ...
class BudgetExceeded(PromptError): ...
```

## Reference example

This example belongs in the repo as a doctest or as a test with an exact expected output. It
pins down the semantics.

Object: sections `identity` (SYSTEM, locked), `behavior` with child `behavior.character`
(DEVELOPER), `tools` (DEVELOPER, slot `tools`, locked), `memories` (USER, slot), `request` (USER,
slot, required). Traits `behavior.tone="terse"` and `behavior.hallucinate="never"`. Renderer
`keyvalue`, `constraints_first=True`.

Expected system message:

```
#IDENTITY
You are Hera, an attentive assistant with a mind of her own.

#BEHAVIOR
BEHAVIOR tone = terse
BEHAVIOR hallucinate = never
You have an opinion and you voice it. When unsure, you say so.

#TOOLS
CALL search(query=~~QUERY~~)
```

Expected user message:

```
#MEMORIES
MEMORY city = Chemnitz

#REQUEST
What was that again about the ablation?
```

The same object with `format="xml"` yields, for `behavior`:

```xml
<behavior>
  <behavior:constraints>
    Answer tersely. No preamble, no wind-down.
    Don't invent anything. If you don't know something, say so.
  </behavior:constraints>
  <behavior:character>
    You have an opinion and you voice it. When unsure, you say so.
  </behavior:character>
</behavior>
```

Note the difference: the same trait appears once as `BEHAVIOR tone = terse`, once as a
fully-formed sentence from the `render` template.

## Tests

At minimum, these cases each as their own test:

- JSON round-trip of a complete prompt, followed by an identical `fingerprint()`.
- Calling `render()` twice on the same object produces byte-identical output.
- Both reference examples above with exact string equality.
- A trait with a `render` template: keyvalue yields the raw pair, xml yields the sentence.
- A trait targeting an unknown prefix lands in the general block.
- `apply()` on a locked trait: prompt unchanged, entry in `rejected`, no exception.
- `apply()` with `None` deletes the trait.
- `apply()` with `allow_unknown=False` and an unknown key: `rejected` with reason
  `unknown_trait`.
- `replace()` on a locked section returns the unchanged object.
- An invalid child `key` with no parent prefix raises `SectionError`.
- A slot with no binding drops; with `required=True` it raises `MissingBinding`.
- Budget removes the section with the lowest `priority` first and lists it in `dropped_keys`.
- Budget with only `required` sections over the limit raises `BudgetExceeded`.
- `developer_role="fold_into_system"` produces exactly one system message; `"native"` produces
  two.
- A trait value containing `=` or a newline raises `TraitError` in the keyvalue renderer.
- The XML renderer escapes angle brackets in content.
- `diff()` between a parent and a child object lists exactly the three changed traits.

Coverage at least 95% on `render/` and `prompt.py`.

## Order of work

1. `pyproject.toml`, repo structure (`src/hera_prompts/`), tooling.
2. `models.py` — Role, Message, Section, RendererConfig with validation.
3. `traits.py` — TraitSpec, TraitRegistry, TraitPatch, PatchResult.
4. `prompt.py` — Prompt with navigation, transformations, `apply`, `fingerprint`, `diff`.
5. `render/` — protocol, KeyValueRenderer, XMLRenderer, MarkdownRenderer, budget.
6. `snapshot.py`, `errors.py`.
7. Tests, README.

Pause after step 3 and show me Section, the Prompt signatures, and TraitSpec before building the
rendering. That's the contract four other libraries will write against.

## Do not build

No prompt inheritance or overlays (base plus patch prompt), no template engine, no variable
interpolation in content, no caching, no tokenizer, no persistence, no history management, no
tool/memory/skill knowledge, no provider specifics beyond the three roles. If something useful
occurs to you along the way that isn't listed here: don't add it — name it as a suggestion at the
end instead.
