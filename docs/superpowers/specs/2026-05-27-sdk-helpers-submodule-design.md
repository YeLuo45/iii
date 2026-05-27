# SDK Helpers Submodule — Design Spec

**Date:** 2026-05-27
**Status:** Approved (pending plan)
**Scope:** All 4 SDKs (Rust, Node, Python, Browser)

## Goal

Move 8 items off the `III` / `ISdk` instance into a dedicated `helpers` submodule in each of the 4 SDKs. Helpers are free functions that take `iii` as their first argument. Expose via subpath imports. Breaking change with no deprecation wrappers.

## Items in Scope

| Category | Items |
|---|---|
| Methods becoming free functions | `register_trigger_type`, `unregister_trigger_type`, `create_channel`, `create_stream` |
| Pure utilities (relocated) | `extract_channel_refs`, `is_channel_ref` |
| Types/enums (relocated) | `ChannelDirection`, `ChannelItem` |

Side effect: Rust SDK gains `create_stream` (currently missing).

## Out of Scope

`register_function`, `register_service`, `register_trigger`, `trigger`, `list_functions`, `list_workers`, `list_triggers`, `on_functions_available`, `on_connection_state_change`, `shutdown` — remain on instance.

## Architecture

### File Layout

| SDK | New file | Public path |
|---|---|---|
| Rust | `sdk/packages/rust/iii/src/helpers.rs` | `iii_sdk::helpers::*` |
| Node | `sdk/packages/node/iii/src/helpers.ts` | `@iii/sdk/helpers` |
| Python | `sdk/packages/python/iii/src/iii/helpers.py` | `iii.helpers` |
| Browser | `sdk/packages/node/iii-browser/src/helpers.ts` | `@iii/sdk-browser/helpers` |

Single file per SDK, no nested folders.

### Package config

- Rust: `pub mod helpers;` in `lib.rs`. Items removed from top-level `pub use` re-exports.
- Node: add `"./helpers"` entry to `exports` field in `sdk/packages/node/iii/package.json`. Build pipeline emits matching `.d.ts` and `.js`.
- Browser: same as Node in `sdk/packages/node/iii-browser/package.json`.
- Python: file resolves automatically via package path. Drop relocated items from `__init__.py` `__all__`.

## Module Surface

### Rust

```rust
// helpers.rs
pub use crate::channels::{
    ChannelDirection, ChannelItem,
    extract_channel_refs, is_channel_ref,
};

pub fn create_channel(iii: &III, buffer_size: Option<usize>)
    -> Result<Channel, IIIError>;

pub fn create_stream<S>(iii: &III, name: impl Into<String>, stream: S)
where S: IStream + 'static;

pub fn register_trigger_type<H, C, R>(
    iii: &III,
    registration: RegisterTriggerType<H, C, R>,
) -> TriggerTypeRef<C, R>
where H: TriggerHandler + 'static;

pub fn unregister_trigger_type(iii: &III, id: impl Into<String>);
```

### Node

```ts
// helpers.ts
export {
    ChannelDirection, ChannelItem,
    extractChannelRefs, isChannelRef,
} from './channels'

export function createChannel(
    iii: ISdk,
    bufferSize?: number,
): Promise<Channel>

export function createStream<TData>(
    iii: ISdk,
    streamName: string,
    stream: IStream<TData>,
): void

export function registerTriggerType<TConfig>(
    iii: ISdk,
    triggerType: Omit<RegisterTriggerTypeMessage, 'message_type'>,
    handler: TriggerHandler<TConfig>,
): TriggerTypeRef<TConfig>

export function unregisterTriggerType(iii: ISdk, id: string): void
```

### Python

```python
# helpers.py
from .channels import ChannelDirection, ChannelItem
from .types import extract_channel_refs, is_channel_ref

def create_channel(
    iii: IIIClient,
    buffer_size: int | None = None,
) -> Channel: ...

async def create_channel_async(
    iii: IIIClient,
    buffer_size: int | None = None,
) -> Channel: ...

def create_stream(
    iii: IIIClient,
    stream_name: str,
    stream: IStream[Any],
) -> None: ...

def register_trigger_type(
    iii: IIIClient,
    trigger_type: RegisterTriggerTypeInput | dict[str, Any],
    handler: TriggerHandler[Any],
) -> TriggerTypeRef[Any, Any]: ...

def unregister_trigger_type(iii: IIIClient, id: str) -> None: ...
```

### Browser

Mirrors Node exactly. Same function names, same arg order. `ISdk` is the browser package's local `ISdk` type.

## Signature Unification Notes

- `unregister_*_trigger_type` everywhere takes `(iii, id: string)`. Previously Node/Python took the full trigger-type object — breaking change.
- `register_trigger_type` keeps language-idiomatic config shape: Rust uses the `RegisterTriggerType<H, C, R>` builder for compile-time generics; Node/Python keep the input-object pattern. Conceptually all three are `(iii, config, handler) -> TriggerTypeRef`.
- `create_channel` in Python keeps both sync `create_channel` and async `create_channel_async` per existing module pattern.
- Item naming follows per-language convention (snake/camel). Type/enum names identical across all 4 SDKs.

## Breaking Changes Summary

### Instance methods removed (all SDKs)

`register_trigger_type` / `registerTriggerType`, `unregister_trigger_type` / `unregisterTriggerType`, `create_channel` / `createChannel`, `create_stream` / `createStream`.

Python additionally drops `create_channel_async` from the instance.

### Top-level re-exports removed

- Rust `lib.rs`: drop `ChannelDirection`, `ChannelItem`, `extract_channel_refs`, `is_channel_ref` from `pub use channels::{...}`.
- Node `index.ts`: drop `isChannelRef` if exported.
- Python `__init__.py`: drop `extract_channel_refs`, `is_channel_ref`, `ChannelDirection`, `ChannelItem` from imports and `__all__`.
- Browser `index.ts`: same as Node.

### Signature breaks (beyond relocation)

| Item | Before | After |
|---|---|---|
| Node `unregisterTriggerType` | `(triggerType: Omit<RegisterTriggerTypeMessage,'message_type'>)` | `(iii, id: string)` |
| Python `unregister_trigger_type` | `(trigger_type: RegisterTriggerTypeInput \| dict)` | `(iii, id: str)` |
| Rust `unregister_trigger_type` | `(&self, id)` instance method | `helpers::unregister_trigger_type(&iii, id)` free fn |

### Internal call-site updates

`TriggerTypeRef.unregister()` (all SDKs) calls the instance method today. Switch internal implementation to call `helpers::unregister_trigger_type(iii, id)` directly.

`iii.ts:861` (Node) and `iii.py:558` use `isChannelRef` / `is_channel_ref` internally. Keep the internal imports (relative paths inside the package) — only external re-exports change.

### Examples / sample code

Update `sdk/packages/{rust,node,python}/iii-example/src/` to use new helper imports.

### Version

Major version bump for each SDK (current `iii/v0.16.0-next.3` → next major). No deprecation period; clean removal.

## Testing

### Migrate existing tests

Every test that calls one of the 8 instance methods must switch to the helper free function.

- Rust: `sdk/packages/rust/iii/tests/` — grep for `register_trigger_type`, `create_channel`, `create_stream`, `unregister_trigger_type`. Replace with `helpers::*`.
- Node: `sdk/packages/node/iii/tests/` — grep for `registerTriggerType`, `createChannel`, `createStream`, `unregisterTriggerType`, `isChannelRef`, `extractChannelRefs`. Update imports and call form.
- Python: `sdk/packages/python/iii/tests/` — same scope, Python naming.
- Browser: `sdk/packages/node/iii-browser/tests/integration/` — same as Node.

### New tests

- **Rust `create_stream` unit test** — register a stream via the helper, assert engine message dispatched and stream callable. Mirror existing Node/Python `create_stream` test cases.
- **Helpers import test (each SDK)** — `import { createChannel, isChannelRef, ChannelDirection } from '@iii/sdk/helpers'` (and Python / Rust equivalents) compiles and resolves at runtime.
- **Removal verification** —
  - Node/Browser TS: add a `// @ts-expect-error` line confirming `iii.createChannel` no longer typechecks.
  - Python: assert `getattr(iii, "create_channel", None) is None`.
  - Rust: not needed; removal causes compile errors in any consumer.
- **`unregister_trigger_type` new signature (each SDK)** — register trigger type, call `helpers.unregister_trigger_type(iii, "id")`, assert registry cleared + unregister message sent.

### Integration

- Run full per-SDK suites: `cargo test`, `pnpm test` (Node packages), `pytest` (Python).
- Cross-SDK fixtures in `sdk/fixtures/` and `sdk/test-assets/` — confirm still green.

## Docs Updates

### `architecture/SDK.md`

- Remove `register_trigger_type`, `create_channel`, `create_stream` from core methods table.
- Add new section "Helpers Module" listing all 8 items with subpath import per SDK.
- Update inline examples that call instance methods.

### `architecture/CHANGE-MAP.md`

- Add row: "Adding a new helper function" → file map across 4 SDKs.
- Update "Adding/modifying a module" entry if affected.

### `architecture/CHANNELS.md`

- Update channel API examples to use helper call form.

### `architecture/MODULES.md`

- Update trigger-type registration examples.

### SDK reference docs

- Grep `docs/` for `iii.create_channel`, `iii.registerTriggerType`, `iii.createStream`, `iii.unregisterTriggerType`. Fix every hit.

### Per-SDK READMEs

- `sdk/packages/rust/iii/README.md`, `sdk/packages/node/iii/README.md`, `sdk/packages/python/iii/README.md`, `sdk/packages/node/iii-browser/README.md` if they show examples — update.

### User global file (out of repo)

`~/.claude/iii-sdk-api-consistency.md` lists the affected methods as required instance methods. Plan flags the change for the user to update post-merge; cannot edit from this PR.

### Changelog / release notes

Flag the major-version breaking changes:
- Removed instance methods
- `unregister_*` signature change
- New `/helpers` subpath required for the 8 items

## Risk / Mitigation

- **Cross-SDK drift** — All 4 SDKs change together in one PR to keep the API-consistency contract intact. Single commit per SDK acceptable inside the PR for review clarity.
- **Hidden internal call sites** — `TriggerTypeRef.unregister()` and `isChannelRef` are called inside the SDKs themselves. Plan covers them explicitly; reviewers should confirm via grep before merge.
- **Rust `create_stream` parity gap** — Currently missing. Adding it as part of this work removes the parity hole rather than leaving it for a follow-up.
