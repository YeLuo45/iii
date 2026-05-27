# SDK Helpers Submodule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move 8 SDK items (`register_trigger_type`, `unregister_trigger_type`, `create_channel`, `create_stream`, `extract_channel_refs`, `is_channel_ref`, `ChannelDirection`, `ChannelItem`) off the `III`/`ISdk` instance into a dedicated `helpers` submodule across all 4 SDKs (Rust, Node, Python, Browser) with subpath imports.

**Architecture:** Each SDK gets a single `helpers` source file exposing free functions that take `iii` as their first argument. Public access via subpath (`@iii/sdk/helpers`, `iii.helpers`, `iii_sdk::helpers`, `@iii/sdk-browser/helpers`). Instance methods are removed cleanly — no deprecation wrappers. Major version bump. Rust gains a new `IStream` trait and `create_stream` helper to reach parity.

**Tech Stack:** Rust (cargo + tokio + tungstenite), Node 18+ (TypeScript + tsdown + vitest), Python 3.10+ (hatchling + pytest + pydantic), Browser (TypeScript + tsdown + vitest + happy-dom).

**Spec:** [docs/superpowers/specs/2026-05-27-sdk-helpers-submodule-design.md](../specs/2026-05-27-sdk-helpers-submodule-design.md)

---

## Phase 1 — Rust SDK

### Task 1: Add `IStream` trait to Rust SDK

**Files:**
- Create: `sdk/packages/rust/iii/src/stream_provider.rs`
- Modify: `sdk/packages/rust/iii/src/lib.rs`
- Test: `sdk/packages/rust/iii/tests/stream_provider_trait.rs`

- [ ] **Step 1: Write the failing test**

Create `sdk/packages/rust/iii/tests/stream_provider_trait.rs`:

```rust
use async_trait::async_trait;
use iii_sdk::{
    IStream, StreamDeleteInput, StreamGetInput, StreamListGroupsInput, StreamListInput,
    StreamSetInput, StreamUpdateInput, DeleteResult, SetResult, UpdateResult,
};
use serde_json::Value;

struct DummyStream;

#[async_trait]
impl IStream for DummyStream {
    async fn get(&self, _: StreamGetInput) -> Result<Option<Value>, iii_sdk::IIIError> {
        Ok(None)
    }
    async fn set(&self, _: StreamSetInput) -> Result<Option<SetResult>, iii_sdk::IIIError> {
        Ok(None)
    }
    async fn delete(&self, _: StreamDeleteInput) -> Result<DeleteResult, iii_sdk::IIIError> {
        Ok(DeleteResult::default())
    }
    async fn list(&self, _: StreamListInput) -> Result<Vec<Value>, iii_sdk::IIIError> {
        Ok(vec![])
    }
    async fn list_groups(
        &self,
        _: StreamListGroupsInput,
    ) -> Result<Vec<String>, iii_sdk::IIIError> {
        Ok(vec![])
    }
    async fn update(
        &self,
        _: StreamUpdateInput,
    ) -> Result<Option<UpdateResult>, iii_sdk::IIIError> {
        Ok(None)
    }
}

#[test]
fn dummy_stream_implements_istream() {
    let _: Box<dyn IStream> = Box::new(DummyStream);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package iii-sdk --test stream_provider_trait`
Expected: FAIL — `cannot find trait IStream in iii_sdk`.

- [ ] **Step 3: Implement `IStream` trait**

Create `sdk/packages/rust/iii/src/stream_provider.rs`:

```rust
use async_trait::async_trait;
use serde_json::Value;

use crate::error::IIIError;
use crate::types::{
    DeleteResult, SetResult, StreamDeleteInput, StreamGetInput, StreamListGroupsInput,
    StreamListInput, StreamSetInput, StreamUpdateInput, UpdateResult,
};

/// Custom stream-provider trait. Implementors override the engine's built-in
/// stream storage for a specific stream name when registered through
/// [`crate::helpers::create_stream`].
#[async_trait]
pub trait IStream: Send + Sync + 'static {
    async fn get(&self, input: StreamGetInput) -> Result<Option<Value>, IIIError>;
    async fn set(&self, input: StreamSetInput) -> Result<Option<SetResult>, IIIError>;
    async fn delete(&self, input: StreamDeleteInput) -> Result<DeleteResult, IIIError>;
    async fn list(&self, input: StreamListInput) -> Result<Vec<Value>, IIIError>;
    async fn list_groups(
        &self,
        input: StreamListGroupsInput,
    ) -> Result<Vec<String>, IIIError>;
    async fn update(
        &self,
        input: StreamUpdateInput,
    ) -> Result<Option<UpdateResult>, IIIError>;
}
```

Edit `sdk/packages/rust/iii/src/lib.rs` — add module declaration and re-export. Insert after the existing `pub mod stream;` line:

```rust
pub mod stream_provider;
```

And in the re-export block, add:

```rust
pub use stream_provider::IStream;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --package iii-sdk --test stream_provider_trait`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/rust/iii/src/stream_provider.rs \
        sdk/packages/rust/iii/src/lib.rs \
        sdk/packages/rust/iii/tests/stream_provider_trait.rs
git commit -m "feat(sdk-rust): add IStream trait for custom stream providers"
```

---

### Task 2: Create Rust `helpers` module with relocated items

**Files:**
- Create: `sdk/packages/rust/iii/src/helpers.rs`
- Modify: `sdk/packages/rust/iii/src/lib.rs`
- Test: `sdk/packages/rust/iii/tests/helpers_module.rs`

- [ ] **Step 1: Write the failing test**

Create `sdk/packages/rust/iii/tests/helpers_module.rs`:

```rust
use iii_sdk::helpers::{
    ChannelDirection, ChannelItem, create_channel, extract_channel_refs, is_channel_ref,
    register_trigger_type, unregister_trigger_type,
};
use serde_json::json;

#[test]
fn helpers_module_reexports_channel_utilities() {
    let value = json!({});
    assert!(!is_channel_ref(&value));
    let refs = extract_channel_refs(&value);
    assert!(refs.is_empty());
    let _: ChannelDirection = ChannelDirection::Read;
    let _: ChannelItem = ChannelItem::Text("x".into());
}

#[test]
fn helpers_module_exposes_free_functions() {
    let _ = create_channel as fn(&iii_sdk::III, Option<usize>) -> Result<iii_sdk::Channel, iii_sdk::IIIError>;
    let _ = unregister_trigger_type as fn(&iii_sdk::III, String);
    // `register_trigger_type` is generic — typed function-pointer cast is impractical;
    // existence in the import above is enough to verify the symbol exists.
    let _ = register_trigger_type::<DummyHandler, (), ()>;
}

struct DummyHandler;

#[async_trait::async_trait]
impl iii_sdk::TriggerHandler for DummyHandler {
    async fn register_trigger(
        &self,
        _: iii_sdk::TriggerConfig,
    ) -> Result<(), iii_sdk::IIIError> {
        Ok(())
    }
    async fn unregister_trigger(
        &self,
        _: iii_sdk::TriggerConfig,
    ) -> Result<(), iii_sdk::IIIError> {
        Ok(())
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package iii-sdk --test helpers_module`
Expected: FAIL — `unresolved import iii_sdk::helpers`.

- [ ] **Step 3: Implement the helpers module**

Create `sdk/packages/rust/iii/src/helpers.rs`:

```rust
//! Helper free functions that operate on an [`III`] instance.
//!
//! These were previously instance methods on `III`. They take `&III` as the
//! first argument so the public API surface of `III` stays focused on the
//! core lifecycle and registration methods.

pub use crate::channels::{
    ChannelDirection, ChannelItem, extract_channel_refs, is_channel_ref,
};

use crate::error::IIIError;
use crate::iii::{III, RegisterTriggerType, TriggerTypeRef};
use crate::stream_provider::IStream;
use crate::triggers::TriggerHandler;
use crate::types::Channel;

/// Create a streaming channel pair for worker-to-worker data transfer.
///
/// Free-function form of the previous `III::create_channel` instance method.
pub async fn create_channel(
    iii: &III,
    buffer_size: Option<usize>,
) -> Result<Channel, IIIError> {
    iii.create_channel(buffer_size).await
}

/// Register a custom stream provider for a stream name.
///
/// Wires the 5 callable `stream::*` functions (`get`, `set`, `delete`, `list`,
/// `list_groups`) on the engine through the supplied [`IStream`] implementor.
/// `update` is **not** registered — atomic updates remain engine-side.
pub fn create_stream<S>(iii: &III, name: impl Into<String>, stream: S)
where
    S: IStream,
{
    use std::sync::Arc;
    let stream = Arc::new(stream);
    let name = name.into();

    let s = stream.clone();
    iii.register_function_async(format!("stream::get({name})"), move |input| {
        let s = s.clone();
        async move { s.get(input).await.map(|v| serde_json::to_value(v).unwrap_or_default()) }
    });
    let s = stream.clone();
    iii.register_function_async(format!("stream::set({name})"), move |input| {
        let s = s.clone();
        async move {
            let res = s.set(input).await?;
            Ok(serde_json::to_value(res).unwrap_or_default())
        }
    });
    let s = stream.clone();
    iii.register_function_async(format!("stream::delete({name})"), move |input| {
        let s = s.clone();
        async move {
            let res = s.delete(input).await?;
            Ok(serde_json::to_value(res).unwrap_or_default())
        }
    });
    let s = stream.clone();
    iii.register_function_async(format!("stream::list({name})"), move |input| {
        let s = s.clone();
        async move {
            let res = s.list(input).await?;
            Ok(serde_json::to_value(res).unwrap_or_default())
        }
    });
    let s = stream.clone();
    iii.register_function_async(format!("stream::list_groups({name})"), move |input| {
        let s = s.clone();
        async move {
            let res = s.list_groups(input).await?;
            Ok(serde_json::to_value(res).unwrap_or_default())
        }
    });
}

/// Register a custom trigger type with the engine.
///
/// Free-function form of the previous `III::register_trigger_type` method.
pub fn register_trigger_type<H, C, R>(
    iii: &III,
    registration: RegisterTriggerType<H, C, R>,
) -> TriggerTypeRef<C, R>
where
    H: TriggerHandler + 'static,
{
    iii.register_trigger_type(registration)
}

/// Unregister a previously registered trigger type by id.
pub fn unregister_trigger_type(iii: &III, id: impl Into<String>) {
    iii.unregister_trigger_type(id);
}
```

Edit `sdk/packages/rust/iii/src/lib.rs` — add `pub mod helpers;` after `pub mod channels;` line.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --package iii-sdk --test helpers_module`
Expected: PASS. All four import items resolve, all three function-pointer assertions compile.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/rust/iii/src/helpers.rs \
        sdk/packages/rust/iii/src/lib.rs \
        sdk/packages/rust/iii/tests/helpers_module.rs
git commit -m "feat(sdk-rust): add helpers submodule with relocated free fns"
```

---

### Task 3: Drop top-level re-exports of relocated channel items

**Files:**
- Modify: `sdk/packages/rust/iii/src/lib.rs:15-18`

- [ ] **Step 1: Write the failing test**

Edit `sdk/packages/rust/iii/tests/helpers_module.rs` — append:

```rust
#[test]
fn channel_items_no_longer_at_top_level() {
    // Importing from the crate root must fail at compile time. The
    // `compile_fail` doctest below proves it.
}

/// ```compile_fail
/// use iii_sdk::ChannelDirection;
/// ```
#[allow(dead_code)]
fn _ensure_channel_direction_not_top_level() {}

/// ```compile_fail
/// use iii_sdk::is_channel_ref;
/// ```
#[allow(dead_code)]
fn _ensure_is_channel_ref_not_top_level() {}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package iii-sdk --test helpers_module --doc`
Expected: FAIL — both compile_fail tests fail because the imports currently succeed.

- [ ] **Step 3: Remove relocated items from top-level re-exports**

Edit `sdk/packages/rust/iii/src/lib.rs` — current block:

```rust
pub use channels::{
    ChannelDirection, ChannelItem, ChannelReader, ChannelWriter, StreamChannelRef,
    extract_channel_refs, is_channel_ref,
};
```

Replace with:

```rust
pub use channels::{ChannelReader, ChannelWriter, StreamChannelRef};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --package iii-sdk --test helpers_module --doc`
Expected: PASS — both compile_fail tests now pass (imports correctly fail).

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/rust/iii/src/lib.rs sdk/packages/rust/iii/tests/helpers_module.rs
git commit -m "feat(sdk-rust)!: drop top-level re-exports for relocated channel items"
```

---

### Task 4: Remove relocated methods from `III` instance

**Files:**
- Modify: `sdk/packages/rust/iii/src/iii.rs` — remove `register_trigger_type` (~lines 918-949), `unregister_trigger_type` (~lines 951-957), `create_channel` (~lines 1110-1140)
- Modify: `sdk/packages/rust/iii/src/helpers.rs` — inline the implementations now that the methods no longer exist

- [ ] **Step 1: Write the failing test**

Edit `sdk/packages/rust/iii/tests/helpers_module.rs` — append:

```rust
/// ```compile_fail
/// let iii = iii_sdk::III::new("ws://x");
/// iii.create_channel(None);
/// ```
#[allow(dead_code)]
fn _ensure_create_channel_not_on_instance() {}

/// ```compile_fail
/// let iii = iii_sdk::III::new("ws://x");
/// iii.unregister_trigger_type("id");
/// ```
#[allow(dead_code)]
fn _ensure_unregister_not_on_instance() {}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package iii-sdk --test helpers_module --doc`
Expected: FAIL — both compile_fail tests fail (instance methods still callable).

- [ ] **Step 3: Move bodies into helpers, delete instance methods**

In `sdk/packages/rust/iii/src/helpers.rs`, replace the three thin-wrapper bodies with inlined implementations using crate-internal accessors. Replace `create_channel` body:

```rust
pub async fn create_channel(
    iii: &III,
    buffer_size: Option<usize>,
) -> Result<Channel, IIIError> {
    use crate::iii::internal_create_channel;
    internal_create_channel(iii, buffer_size).await
}
```

Replace `register_trigger_type` body:

```rust
pub fn register_trigger_type<H, C, R>(
    iii: &III,
    registration: RegisterTriggerType<H, C, R>,
) -> TriggerTypeRef<C, R>
where
    H: TriggerHandler + 'static,
{
    use crate::iii::internal_register_trigger_type;
    internal_register_trigger_type(iii, registration)
}
```

Replace `unregister_trigger_type` body:

```rust
pub fn unregister_trigger_type(iii: &III, id: impl Into<String>) {
    use crate::iii::internal_unregister_trigger_type;
    internal_unregister_trigger_type(iii, id.into());
}
```

In `sdk/packages/rust/iii/src/iii.rs`:

1. **Delete** the public `register_trigger_type`, `unregister_trigger_type`, and `create_channel` methods on `impl III { ... }` (the three blocks around lines 918, 951, 1110).
2. **Add** crate-visible helper functions at module scope (after the `impl III` block):

```rust
pub(crate) fn internal_register_trigger_type<H, C, R>(
    iii: &III,
    registration: RegisterTriggerType<H, C, R>,
) -> TriggerTypeRef<C, R>
where
    H: TriggerHandler + 'static,
{
    let message = crate::protocol::RegisterTriggerTypeMessage {
        id: registration.id,
        description: registration.description,
        trigger_request_format: registration.trigger_request_format,
        call_request_format: registration.call_request_format,
    };
    let trigger_type_id = message.id.clone();
    iii.inner.trigger_types.lock_or_recover().insert(
        message.id.clone(),
        crate::structs::RemoteTriggerTypeData {
            message: message.clone(),
            handler: std::sync::Arc::new(registration.handler),
        },
    );
    let _ = iii.send_message(message.to_message());
    TriggerTypeRef {
        iii: iii.clone(),
        trigger_type_id,
        _phantom: std::marker::PhantomData,
    }
}

pub(crate) fn internal_unregister_trigger_type(iii: &III, id: String) {
    iii.inner.trigger_types.lock_or_recover().remove(&id);
    let msg = crate::protocol::UnregisterTriggerTypeMessage { id };
    let _ = iii.send_message(msg.to_message());
}

pub(crate) async fn internal_create_channel(
    iii: &III,
    buffer_size: Option<usize>,
) -> Result<Channel, IIIError> {
    // Move the original body verbatim from the deleted instance method,
    // replacing `self` with `iii`.
    iii.create_channel_internal(buffer_size).await
}
```

Where the existing private channel-creation code in `iii.rs` already does the work, expose it via a `pub(crate) async fn create_channel_internal(&self, ...)`. If that path doesn't yet exist as a private fn, rename the previous public method to `create_channel_internal` and mark `pub(crate)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --package iii-sdk --test helpers_module --doc && cargo test --package iii-sdk`
Expected: PASS — both compile_fail tests pass; existing crate tests still green.

- [ ] **Step 5: Update `TriggerTypeRef::unregister` to call helper**

Locate the `TriggerTypeRef` impl in `sdk/packages/rust/iii/src/iii.rs`. Find any `self.iii.unregister_trigger_type(...)` call site and replace with:

```rust
crate::iii::internal_unregister_trigger_type(&self.iii, self.trigger_type_id.clone());
```

Run: `cargo build --package iii-sdk` — Expected: clean build.

- [ ] **Step 6: Commit**

```bash
git add sdk/packages/rust/iii/src/iii.rs \
        sdk/packages/rust/iii/src/helpers.rs \
        sdk/packages/rust/iii/tests/helpers_module.rs
git commit -m "refactor(sdk-rust)!: remove relocated instance methods from III"
```

---

### Task 5: Migrate Rust example + integration tests to helpers

**Files:**
- Modify: `sdk/packages/rust/iii-example/src/**/*.rs` (every file that calls the relocated methods)
- Modify: `sdk/packages/rust/iii/tests/common/**/*.rs` and any top-level integration test that calls the relocated methods

- [ ] **Step 1: Locate call sites**

Run: `rg -n 'register_trigger_type|unregister_trigger_type|create_channel|extract_channel_refs|is_channel_ref|ChannelDirection|ChannelItem' sdk/packages/rust/`
Capture the list of files (excluding `helpers.rs`, the new `helpers_module` test, and `iii.rs` internals which are already updated).

- [ ] **Step 2: Update each call site**

For each match, change one of:
- `iii.register_trigger_type(reg)` → `iii_sdk::helpers::register_trigger_type(&iii, reg)`
- `iii.unregister_trigger_type(id)` → `iii_sdk::helpers::unregister_trigger_type(&iii, id)`
- `iii.create_channel(buf)` → `iii_sdk::helpers::create_channel(&iii, buf)`
- `iii_sdk::ChannelDirection::*` → `iii_sdk::helpers::ChannelDirection::*`
- `iii_sdk::ChannelItem::*` → `iii_sdk::helpers::ChannelItem::*`
- `iii_sdk::extract_channel_refs(v)` → `iii_sdk::helpers::extract_channel_refs(v)`
- `iii_sdk::is_channel_ref(v)` → `iii_sdk::helpers::is_channel_ref(v)`

- [ ] **Step 3: Run full Rust suite**

Run: `cargo test --package iii-sdk --all-targets`
Expected: PASS, no warnings about deprecated/missing items.

- [ ] **Step 4: Commit**

```bash
git add sdk/packages/rust/
git commit -m "refactor(sdk-rust): migrate examples and tests to helpers submodule"
```

---

## Phase 2 — Node SDK

### Task 6: Create Node `helpers.ts` with relocated symbols

**Files:**
- Create: `sdk/packages/node/iii/src/helpers.ts`
- Modify: `sdk/packages/node/iii/package.json` — add `./helpers` exports entry
- Test: `sdk/packages/node/iii/tests/helpers.test.ts`

- [ ] **Step 1: Write the failing test**

Create `sdk/packages/node/iii/tests/helpers.test.ts`:

```ts
import { describe, expect, it } from 'vitest'

import {
    ChannelDirection,
    ChannelItem,
    createChannel,
    createStream,
    extractChannelRefs,
    isChannelRef,
    registerTriggerType,
    unregisterTriggerType,
} from '../src/helpers'

describe('helpers module', () => {
    it('exposes channel utilities and types', () => {
        expect(typeof isChannelRef).toBe('function')
        expect(typeof extractChannelRefs).toBe('function')
        expect(isChannelRef({})).toBe(false)
        expect(extractChannelRefs({})).toEqual([])
        expect(ChannelDirection).toBeDefined()
        expect(ChannelItem).toBeDefined()
    })

    it('exposes free functions taking iii as first arg', () => {
        expect(createChannel.length).toBe(2)
        expect(createStream.length).toBe(3)
        expect(registerTriggerType.length).toBe(3)
        expect(unregisterTriggerType.length).toBe(2)
    })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter iii-sdk test helpers.test.ts`
Expected: FAIL — cannot find module `../src/helpers`.

- [ ] **Step 3: Implement the helpers module**

Create `sdk/packages/node/iii/src/helpers.ts`:

```ts
import type {
    Channel,
    ISdk,
    RegisterTriggerTypeInput,
    TriggerTypeRef,
} from './types'
import type { IStream } from './stream'
import type { TriggerHandler } from './triggers'

export {
    ChannelDirection,
    type ChannelItem,
} from './channels'
export { isChannelRef, extractChannelRefs } from './utils'

/**
 * Create a streaming channel pair for worker-to-worker data transfer.
 *
 * Free-function form of the previous `ISdk.createChannel` instance method.
 */
export function createChannel(
    iii: ISdk,
    bufferSize?: number,
): Promise<Channel> {
    return (iii as unknown as { __helpers_create_channel(b?: number): Promise<Channel> })
        .__helpers_create_channel(bufferSize)
}

/**
 * Register a custom stream implementation by wiring its 5 callable methods
 * to `stream::get/set/delete/list/list_groups`.
 *
 * Free-function form of the previous `ISdk.createStream` instance method.
 */
export function createStream<TData>(
    iii: ISdk,
    streamName: string,
    stream: IStream<TData>,
): void {
    ;(iii as unknown as {
        __helpers_create_stream<T>(name: string, s: IStream<T>): void
    }).__helpers_create_stream(streamName, stream)
}

/**
 * Register a custom trigger type with the engine.
 *
 * Free-function form of the previous `ISdk.registerTriggerType` method.
 */
export function registerTriggerType<TConfig>(
    iii: ISdk,
    triggerType: RegisterTriggerTypeInput,
    handler: TriggerHandler<TConfig>,
): TriggerTypeRef<TConfig> {
    return (iii as unknown as {
        __helpers_register_trigger_type<T>(
            t: RegisterTriggerTypeInput,
            h: TriggerHandler<T>,
        ): TriggerTypeRef<T>
    }).__helpers_register_trigger_type(triggerType, handler)
}

/**
 * Unregister a previously registered trigger type by id.
 */
export function unregisterTriggerType(iii: ISdk, id: string): void {
    ;(iii as unknown as { __helpers_unregister_trigger_type(id: string): void })
        .__helpers_unregister_trigger_type(id)
}
```

> **Note:** The `__helpers_*` symbol shim lets the helper module call into the SDK without exposing the implementation as a public method. Task 7 renames the existing instance methods to these `__helpers_*` names and removes them from the public `ISdk` type.

Edit `sdk/packages/node/iii/package.json` — extend the `exports` block to include the helpers subpath:

```json
"./helpers": {
    "types": "./dist/helpers.d.ts",
    "import": "./dist/helpers.mjs",
    "require": "./dist/helpers.cjs"
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm --filter iii-sdk test helpers.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/node/iii/src/helpers.ts \
        sdk/packages/node/iii/package.json \
        sdk/packages/node/iii/tests/helpers.test.ts
git commit -m "feat(sdk-node): add helpers submodule with relocated free fns"
```

---

### Task 7: Rename Node instance methods to `__helpers_*` and remove from `ISdk` type

**Files:**
- Modify: `sdk/packages/node/iii/src/iii.ts:174-216` (rename `registerTriggerType` → `__helpers_register_trigger_type` and `unregisterTriggerType` → `__helpers_unregister_trigger_type`)
- Modify: `sdk/packages/node/iii/src/iii.ts:408-430` (rename `createChannel` → `__helpers_create_channel`)
- Modify: `sdk/packages/node/iii/src/iii.ts:570-576` (rename `createStream` → `__helpers_create_stream`)
- Modify: `sdk/packages/node/iii/src/types.ts` — remove `registerTriggerType`, `unregisterTriggerType`, `createChannel`, `createStream` from the `ISdk` interface
- Modify: `sdk/packages/node/iii/src/iii.ts:205` — internal `TriggerTypeRef.unregister` callsite uses the new name

- [ ] **Step 1: Write the failing test**

Append to `sdk/packages/node/iii/tests/helpers.test.ts`:

```ts
import { registerWorker } from '../src/iii'

describe('ISdk public surface', () => {
    it('no longer exposes relocated methods', () => {
        const iii = registerWorker({ address: 'ws://localhost:9' }) as unknown as Record<
            string,
            unknown
        >
        expect(iii.createChannel).toBeUndefined()
        expect(iii.createStream).toBeUndefined()
        expect(iii.registerTriggerType).toBeUndefined()
        expect(iii.unregisterTriggerType).toBeUndefined()
    })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter iii-sdk test helpers.test.ts`
Expected: FAIL — instance methods are still defined.

- [ ] **Step 3: Rename instance methods**

In `sdk/packages/node/iii/src/iii.ts`, replace:

```ts
registerTriggerType = <TConfig>(
    triggerType: Omit<RegisterTriggerTypeMessage, 'message_type'>,
    handler: TriggerHandler<TConfig>,
): TriggerTypeRef<TConfig> => {
```

with:

```ts
__helpers_register_trigger_type = <TConfig>(
    triggerType: Omit<RegisterTriggerTypeMessage, 'message_type'>,
    handler: TriggerHandler<TConfig>,
): TriggerTypeRef<TConfig> => {
```

Inside the body, change `this.unregisterTriggerType(triggerType)` to `this.__helpers_unregister_trigger_type(triggerType.id)`.

Replace:

```ts
unregisterTriggerType = (triggerType: Omit<RegisterTriggerTypeMessage, 'message_type'>): void => {
    this.sendMessage(MessageType.UnregisterTriggerType, triggerType, true)
}
```

with:

```ts
__helpers_unregister_trigger_type = (id: string): void => {
    this.sendMessage(MessageType.UnregisterTriggerType, { id }, true)
}
```

Replace:

```ts
createChannel = async (bufferSize?: number): Promise<import('./types').Channel> => {
```

with:

```ts
__helpers_create_channel = async (bufferSize?: number): Promise<import('./types').Channel> => {
```

Replace:

```ts
createStream = <TData>(streamName: string, stream: IStream<TData>): void => {
```

with:

```ts
__helpers_create_stream = <TData>(streamName: string, stream: IStream<TData>): void => {
```

In `sdk/packages/node/iii/src/types.ts`, remove the four method declarations (`registerTriggerType`, `unregisterTriggerType`, `createChannel`, `createStream`) from the `ISdk` interface.

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm --filter iii-sdk test helpers.test.ts && pnpm --filter iii-sdk test`
Expected: PASS — relocation test passes; full suite still green.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/node/iii/src/iii.ts sdk/packages/node/iii/src/types.ts \
        sdk/packages/node/iii/tests/helpers.test.ts
git commit -m "refactor(sdk-node)!: remove relocated methods from ISdk"
```

---

### Task 8: Drop top-level re-exports + migrate Node example/tests/internal call sites

**Files:**
- Modify: `sdk/packages/node/iii/src/index.ts` — remove relocated symbols if exported
- Modify: `sdk/packages/node/iii/src/iii.ts:61, 861` — internal `isChannelRef` imports stay relative (keep)
- Modify: `sdk/packages/node/iii-example/src/**/*.ts` — every call site of the four relocated methods

- [ ] **Step 1: Locate every call site**

Run: `rg -n 'registerTriggerType|unregisterTriggerType|createChannel|createStream|isChannelRef|extractChannelRefs|ChannelDirection|ChannelItem' sdk/packages/node/iii sdk/packages/node/iii-example | rg -v '__helpers_|src/helpers\.ts|tests/helpers\.test\.ts'`

- [ ] **Step 2: Rewrite call sites in `iii-example`**

For each match in `iii-example`, change:
- `iii.registerTriggerType(t, h)` → `import { registerTriggerType } from 'iii-sdk/helpers'; registerTriggerType(iii, t, h)`
- `iii.unregisterTriggerType(t)` → `import { unregisterTriggerType } from 'iii-sdk/helpers'; unregisterTriggerType(iii, t.id)`
- `iii.createChannel(b)` → `import { createChannel } from 'iii-sdk/helpers'; createChannel(iii, b)`
- `iii.createStream(n, s)` → `import { createStream } from 'iii-sdk/helpers'; createStream(iii, n, s)`
- `import { isChannelRef } from 'iii-sdk'` → `import { isChannelRef } from 'iii-sdk/helpers'`
- `import { extractChannelRefs } from 'iii-sdk'` → `import { extractChannelRefs } from 'iii-sdk/helpers'`
- `import { ChannelDirection } from 'iii-sdk'` → `import { ChannelDirection } from 'iii-sdk/helpers'`
- `import { ChannelItem } from 'iii-sdk'` → `import { ChannelItem } from 'iii-sdk/helpers'`

- [ ] **Step 3: Remove any top-level exports**

In `sdk/packages/node/iii/src/index.ts`, ensure none of the 8 symbols are re-exported. Inspect the current file (already known to not export them from `./channels` or `./utils` for these specific names — verify by grep). If any are present, delete them.

- [ ] **Step 4: Run full Node suite + build**

Run: `pnpm --filter iii-sdk build && pnpm --filter iii-sdk test && pnpm --filter iii-example build`
Expected: clean build, all tests pass.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/node/iii sdk/packages/node/iii-example
git commit -m "refactor(sdk-node)!: migrate examples and call sites to helpers subpath"
```

---

## Phase 3 — Browser SDK

### Task 9: Create Browser `helpers.ts` mirroring Node SDK

**Files:**
- Create: `sdk/packages/node/iii-browser/src/helpers.ts`
- Modify: `sdk/packages/node/iii-browser/package.json` — add `./helpers` exports entry
- Test: `sdk/packages/node/iii-browser/tests/helpers.test.ts`

- [ ] **Step 1: Write the failing test**

Create `sdk/packages/node/iii-browser/tests/helpers.test.ts`:

```ts
import { describe, expect, it } from 'vitest'

import {
    ChannelDirection,
    ChannelItem,
    createChannel,
    createStream,
    extractChannelRefs,
    isChannelRef,
    registerTriggerType,
    unregisterTriggerType,
} from '../src/helpers'

describe('browser helpers module', () => {
    it('exposes the same surface as node helpers', () => {
        expect(typeof isChannelRef).toBe('function')
        expect(typeof extractChannelRefs).toBe('function')
        expect(typeof createChannel).toBe('function')
        expect(typeof createStream).toBe('function')
        expect(typeof registerTriggerType).toBe('function')
        expect(typeof unregisterTriggerType).toBe('function')
        expect(ChannelDirection).toBeDefined()
        expect(ChannelItem).toBeDefined()
    })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter iii-browser-sdk test helpers.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement Browser helpers**

Create `sdk/packages/node/iii-browser/src/helpers.ts` — content identical to the Node `helpers.ts` from Task 6, except imports point at the browser package's own `./types`, `./stream`, `./triggers`, `./channels`, `./utils`. Copy the file verbatim from `sdk/packages/node/iii/src/helpers.ts`, no other changes needed because the relative paths are the same.

Edit `sdk/packages/node/iii-browser/package.json` — extend `exports`:

```json
"./helpers": {
    "types": "./dist/helpers.d.ts",
    "import": "./dist/helpers.mjs",
    "require": "./dist/helpers.cjs"
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm --filter iii-browser-sdk test helpers.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/node/iii-browser/src/helpers.ts \
        sdk/packages/node/iii-browser/package.json \
        sdk/packages/node/iii-browser/tests/helpers.test.ts
git commit -m "feat(sdk-browser): add helpers submodule mirroring node SDK"
```

---

### Task 10: Remove Browser instance methods and migrate call sites

**Files:**
- Modify: `sdk/packages/node/iii-browser/src/iii.ts:116-160` (rename `registerTriggerType` / `unregisterTriggerType` to `__helpers_*`)
- Modify: `sdk/packages/node/iii-browser/src/iii.ts:279-300` (rename `createChannel` → `__helpers_create_channel`)
- Modify: `sdk/packages/node/iii-browser/src/iii.ts:395-410` (rename `createStream` → `__helpers_create_stream`)
- Modify: `sdk/packages/node/iii-browser/src/types.ts` — remove the four methods from `ISdk`
- Modify: any callers in `tests/integration/**/*.ts`

- [ ] **Step 1: Write the failing test**

Append to `sdk/packages/node/iii-browser/tests/helpers.test.ts`:

```ts
import { registerWorker } from '../src/iii'

describe('Browser ISdk public surface', () => {
    it('no longer exposes relocated methods', () => {
        const iii = registerWorker({ address: 'ws://localhost:9' }) as unknown as Record<
            string,
            unknown
        >
        expect(iii.createChannel).toBeUndefined()
        expect(iii.createStream).toBeUndefined()
        expect(iii.registerTriggerType).toBeUndefined()
        expect(iii.unregisterTriggerType).toBeUndefined()
    })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter iii-browser-sdk test helpers.test.ts`
Expected: FAIL.

- [ ] **Step 3: Apply the same renaming as Task 7**

In `sdk/packages/node/iii-browser/src/iii.ts`:

- Rename `registerTriggerType = …` to `__helpers_register_trigger_type = …`.
- Rename `unregisterTriggerType = …` to `__helpers_unregister_trigger_type = …`. Update the signature to take a `string id` instead of the message object, body matches Task 7.
- Rename `createChannel = …` to `__helpers_create_channel = …`.
- Rename `createStream = …` to `__helpers_create_stream = …`.
- Inside the renamed register handler body, change `this.unregisterTriggerType(triggerType)` to `this.__helpers_unregister_trigger_type(triggerType.id)`.

In `sdk/packages/node/iii-browser/src/types.ts`, remove `registerTriggerType`, `unregisterTriggerType`, `createChannel`, `createStream` from the `ISdk` interface.

- [ ] **Step 4: Migrate integration tests and internal call sites**

Run: `rg -n 'registerTriggerType|unregisterTriggerType|createChannel|createStream|isChannelRef|extractChannelRefs|ChannelDirection|ChannelItem' sdk/packages/node/iii-browser | rg -v '__helpers_|src/helpers\.ts|tests/helpers\.test\.ts'`

For each non-helper match in `tests/integration/`, rewrite using the same patterns as Task 8 but importing from `iii-browser-sdk/helpers` (for source-built tests) or relative `../src/helpers` (for in-package tests).

- [ ] **Step 5: Run full Browser suite + build**

Run: `pnpm --filter iii-browser-sdk build && pnpm --filter iii-browser-sdk test && pnpm --filter iii-browser-sdk test:integration`
Expected: clean build, all tests pass.

- [ ] **Step 6: Commit**

```bash
git add sdk/packages/node/iii-browser
git commit -m "refactor(sdk-browser)!: remove relocated methods + migrate call sites"
```

---

## Phase 4 — Python SDK

### Task 11: Create Python `helpers.py` with relocated symbols

**Files:**
- Create: `sdk/packages/python/iii/src/iii/helpers.py`
- Test: `sdk/packages/python/iii/tests/test_helpers.py`

- [ ] **Step 1: Write the failing test**

Create `sdk/packages/python/iii/tests/test_helpers.py`:

```python
import inspect

import pytest


def test_helpers_module_exports_expected_names() -> None:
    from iii import helpers

    expected = {
        "ChannelDirection",
        "ChannelItem",
        "create_channel",
        "create_channel_async",
        "create_stream",
        "extract_channel_refs",
        "is_channel_ref",
        "register_trigger_type",
        "unregister_trigger_type",
    }
    actual = {name for name in dir(helpers) if not name.startswith("_")}
    missing = expected - actual
    assert not missing, f"missing: {missing}"


def test_helpers_free_functions_take_iii_first() -> None:
    from iii import helpers

    for name in (
        "create_channel",
        "create_channel_async",
        "create_stream",
        "register_trigger_type",
        "unregister_trigger_type",
    ):
        sig = inspect.signature(getattr(helpers, name))
        params = list(sig.parameters)
        assert params and params[0] == "iii", f"{name} signature: {sig}"


def test_is_channel_ref_works_via_helpers() -> None:
    from iii import helpers

    assert helpers.is_channel_ref({}) is False
    assert helpers.extract_channel_refs({}) == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd sdk/packages/python/iii && uv run pytest tests/test_helpers.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'iii.helpers'`.

- [ ] **Step 3: Implement the helpers module**

Create `sdk/packages/python/iii/src/iii/helpers.py`:

```python
"""Helper free functions that operate on an :class:`IIIClient` instance.

These were previously instance methods. They take the ``iii`` client as the
first argument so the public surface of :class:`IIIClient` stays focused on
the core lifecycle and registration methods.
"""

from __future__ import annotations

from typing import Any

from .channels import ChannelDirection, ChannelItem
from .stream import IStream
from .triggers import TriggerHandler, TriggerTypeRef
from .types import Channel, IIIClient, extract_channel_refs, is_channel_ref
from .iii_types import RegisterTriggerTypeInput

__all__ = [
    "ChannelDirection",
    "ChannelItem",
    "create_channel",
    "create_channel_async",
    "create_stream",
    "extract_channel_refs",
    "is_channel_ref",
    "register_trigger_type",
    "unregister_trigger_type",
]


def create_channel(iii: IIIClient, buffer_size: int | None = None) -> Channel:
    """Create a streaming channel pair (sync wrapper)."""
    return iii._helpers_create_channel(buffer_size)


async def create_channel_async(
    iii: IIIClient, buffer_size: int | None = None
) -> Channel:
    """Create a streaming channel pair (async)."""
    return await iii._helpers_create_channel_async(buffer_size)


def create_stream(iii: IIIClient, stream_name: str, stream: IStream[Any]) -> None:
    """Register a custom stream implementation."""
    iii._helpers_create_stream(stream_name, stream)


def register_trigger_type(
    iii: IIIClient,
    trigger_type: RegisterTriggerTypeInput | dict[str, Any],
    handler: TriggerHandler[Any],
) -> TriggerTypeRef[Any, Any]:
    """Register a custom trigger type with the engine."""
    return iii._helpers_register_trigger_type(trigger_type, handler)


def unregister_trigger_type(iii: IIIClient, id: str) -> None:
    """Unregister a previously registered trigger type by id."""
    iii._helpers_unregister_trigger_type(id)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd sdk/packages/python/iii && uv run pytest tests/test_helpers.py::test_helpers_module_exports_expected_names tests/test_helpers.py::test_helpers_free_functions_take_iii_first tests/test_helpers.py::test_is_channel_ref_works_via_helpers -v`
Expected: PASS — these three tests only verify the helper module's public surface and the re-exported `is_channel_ref` / `extract_channel_refs`. They do not exercise the `_helpers_*` shim methods, which are added in Task 12.

- [ ] **Step 5: Commit (module only)**

```bash
git add sdk/packages/python/iii/src/iii/helpers.py \
        sdk/packages/python/iii/tests/test_helpers.py
git commit -m "feat(sdk-python): add helpers submodule (impl wired in next task)"
```

---

### Task 12: Rename Python instance methods to `_helpers_*` and remove from public API

**Files:**
- Modify: `sdk/packages/python/iii/src/iii/iii.py:746-810` (`register_trigger_type` → `_helpers_register_trigger_type`, change unregister body to accept id string)
- Modify: `sdk/packages/python/iii/src/iii/iii.py:811-835` (`unregister_trigger_type` → `_helpers_unregister_trigger_type`, accept `id: str`)
- Modify: `sdk/packages/python/iii/src/iii/iii.py:1145-1190` (`create_channel`/`create_channel_async` → `_helpers_create_channel`/`_helpers_create_channel_async`)
- Modify: `sdk/packages/python/iii/src/iii/iii.py:1251-1306` (`create_stream` → `_helpers_create_stream`)
- Modify: `sdk/packages/python/iii/src/iii/__init__.py` — drop `extract_channel_refs`, `is_channel_ref`, `ChannelDirection`, `ChannelItem` from imports and `__all__`
- Modify: `sdk/packages/python/iii/src/iii/triggers.py` — `TriggerTypeRef.unregister()` callsite uses new name

- [ ] **Step 1: Write the failing test**

Append to `sdk/packages/python/iii/tests/test_helpers.py`:

```python
def test_iii_no_longer_exposes_relocated_methods() -> None:
    from iii import register_worker

    client = register_worker(address="ws://localhost:9", connect=False)

    for name in (
        "register_trigger_type",
        "unregister_trigger_type",
        "create_channel",
        "create_channel_async",
        "create_stream",
    ):
        assert not hasattr(client, name), f"client still has {name}"


def test_init_no_longer_exports_relocated_channel_items() -> None:
    import iii

    for name in (
        "extract_channel_refs",
        "is_channel_ref",
        "ChannelDirection",
        "ChannelItem",
    ):
        assert name not in iii.__all__, f"{name} still in iii.__all__"
```

If `register_worker` does not currently expose a `connect=False` option, locate the equivalent flag in `register_worker` and use that, or instantiate `IIIClient` directly with the same arguments the existing connection-disabled test fixtures use (grep `tests/` for examples).

- [ ] **Step 2: Run test to verify it fails**

Run: `cd sdk/packages/python/iii && uv run pytest tests/test_helpers.py -v`
Expected: FAIL — `client` still has the instance methods, `__init__` still re-exports the channel items.

- [ ] **Step 3: Rename instance methods**

In `sdk/packages/python/iii/src/iii/iii.py`, rename each `def name(...)` to `def _helpers_name(...)`:
- `register_trigger_type` → `_helpers_register_trigger_type` (body unchanged)
- `unregister_trigger_type` → `_helpers_unregister_trigger_type` — change the signature to `def _helpers_unregister_trigger_type(self, id: str) -> None:` and inside the body replace any logic that derived the id from a message object with direct use of the `id` parameter. The message sent to the engine becomes `UnregisterTriggerTypeMessage(id=id)`.
- `create_channel` → `_helpers_create_channel`
- `create_channel_async` → `_helpers_create_channel_async`
- `create_stream` → `_helpers_create_stream`

In `sdk/packages/python/iii/src/iii/triggers.py`, find `TriggerTypeRef.unregister` and change any `self.iii.unregister_trigger_type(...)` call to `self.iii._helpers_unregister_trigger_type(self.trigger_type_id)`.

In `sdk/packages/python/iii/src/iii/__init__.py`:

- Remove `extract_channel_refs`, `is_channel_ref` from the imports near the top (currently in the `.types` import or similar — grep verifies).
- Remove `ChannelDirection`, `ChannelItem` from any imports.
- Remove all four names from `__all__`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd sdk/packages/python/iii && uv run pytest tests/test_helpers.py -v && uv run pytest`
Expected: PASS for helpers tests, full suite still green.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/python/iii
git commit -m "refactor(sdk-python)!: remove relocated methods from IIIClient"
```

---

### Task 13: Migrate Python example and remaining call sites

**Files:**
- Modify: `sdk/packages/python/iii-example/src/**/*.py`
- Modify: any test fixtures or other call sites discovered by grep

- [ ] **Step 1: Locate call sites**

Run: `rg -n 'register_trigger_type|unregister_trigger_type|create_channel|create_stream|extract_channel_refs|is_channel_ref|ChannelDirection|ChannelItem' sdk/packages/python | rg -v '_helpers_|src/iii/helpers\.py|tests/test_helpers\.py|src/iii/iii_types\.py'`

- [ ] **Step 2: Rewrite call sites**

For each match:
- `iii.register_trigger_type(t, h)` → `from iii.helpers import register_trigger_type; register_trigger_type(iii, t, h)`
- `iii.unregister_trigger_type(t)` → `from iii.helpers import unregister_trigger_type; unregister_trigger_type(iii, t.id if hasattr(t, 'id') else t["id"])`
- `iii.create_channel(b)` → `from iii.helpers import create_channel; create_channel(iii, b)`
- `iii.create_channel_async(b)` → `from iii.helpers import create_channel_async; await create_channel_async(iii, b)`
- `iii.create_stream(n, s)` → `from iii.helpers import create_stream; create_stream(iii, n, s)`
- `from iii import is_channel_ref` → `from iii.helpers import is_channel_ref`
- `from iii import extract_channel_refs` → `from iii.helpers import extract_channel_refs`
- `from iii import ChannelDirection` → `from iii.helpers import ChannelDirection`
- `from iii import ChannelItem` → `from iii.helpers import ChannelItem`

- [ ] **Step 3: Run full Python suite**

Run: `cd sdk/packages/python/iii && uv run pytest && cd ../iii-example && uv run pytest 2>/dev/null || true`
Expected: PASS for the SDK suite.

- [ ] **Step 4: Commit**

```bash
git add sdk/packages/python
git commit -m "refactor(sdk-python): migrate examples and tests to helpers submodule"
```

---

## Phase 5 — Docs & Version Bump

### Task 14: Update architecture docs

**Files:**
- Modify: `architecture/SDK.md`
- Modify: `architecture/CHANNELS.md`
- Modify: `architecture/MODULES.md`
- Modify: `architecture/CHANGE-MAP.md`

- [ ] **Step 1: Update `architecture/SDK.md`**

Find the core methods table (search for `register_trigger_type` or `create_channel`). Remove the rows for `register_trigger_type`, `unregister_trigger_type`, `create_channel`, `create_stream`. Add a new section after the table:

```markdown
## Helpers Submodule

The following items live in the `helpers` submodule of each SDK and are imported via subpath:

| Symbol | Rust | Node / Browser | Python |
| --- | --- | --- | --- |
| `register_trigger_type` | `iii_sdk::helpers::register_trigger_type` | `import { registerTriggerType } from 'iii-sdk/helpers'` | `from iii.helpers import register_trigger_type` |
| `unregister_trigger_type` | `iii_sdk::helpers::unregister_trigger_type` | `import { unregisterTriggerType } from 'iii-sdk/helpers'` | `from iii.helpers import unregister_trigger_type` |
| `create_channel` | `iii_sdk::helpers::create_channel` | `import { createChannel } from 'iii-sdk/helpers'` | `from iii.helpers import create_channel` |
| `create_stream` | `iii_sdk::helpers::create_stream` | `import { createStream } from 'iii-sdk/helpers'` | `from iii.helpers import create_stream` |
| `extract_channel_refs` | `iii_sdk::helpers::extract_channel_refs` | `import { extractChannelRefs } from 'iii-sdk/helpers'` | `from iii.helpers import extract_channel_refs` |
| `is_channel_ref` | `iii_sdk::helpers::is_channel_ref` | `import { isChannelRef } from 'iii-sdk/helpers'` | `from iii.helpers import is_channel_ref` |
| `ChannelDirection` | `iii_sdk::helpers::ChannelDirection` | `import { ChannelDirection } from 'iii-sdk/helpers'` | `from iii.helpers import ChannelDirection` |
| `ChannelItem` | `iii_sdk::helpers::ChannelItem` | `import { ChannelItem } from 'iii-sdk/helpers'` | `from iii.helpers import ChannelItem` |

Helpers take the `iii` client as the first argument:

```rust
iii_sdk::helpers::create_channel(&iii, Some(16)).await?;
```

```ts
await createChannel(iii, 16)
```

```python
create_channel(iii, 16)
```
```

Update any inline example in the same file that uses `iii.create_channel`, `iii.registerTriggerType`, etc. — replace with the helper call.

- [ ] **Step 2: Update `architecture/CHANNELS.md`**

Find every example using `iii.create_channel` or `iii.createChannel` and rewrite to use the helper. Replace `ChannelDirection`/`ChannelItem` mentions to clarify they live in `helpers`.

- [ ] **Step 3: Update `architecture/MODULES.md`**

Same — rewrite trigger-type registration examples to use `helpers::register_trigger_type` form.

- [ ] **Step 4: Update `architecture/CHANGE-MAP.md`**

Add a new row:

```
| Adding a new helper free fn | sdk/packages/{rust,node,python,node/iii-browser}/iii/src/helpers.{rs,ts,py,ts} + matching test files |
```

- [ ] **Step 5: Commit**

```bash
git add architecture/
git commit -m "docs(architecture): document helpers submodule across SDKs"
```

---

### Task 15: Update mintlify docs

**Files:**
- Modify: any file under `docs/` referencing the relocated symbols

- [ ] **Step 1: Locate doc references**

Run: `rg -nl 'iii\.create_channel|iii\.createChannel|iii\.registerTriggerType|iii\.register_trigger_type|iii\.createStream|iii\.create_stream|iii\.unregisterTriggerType|iii\.unregister_trigger_type|extractChannelRefs|isChannelRef|ChannelDirection|ChannelItem|extract_channel_refs|is_channel_ref' docs/`

- [ ] **Step 2: Rewrite each doc**

For each file: change call form to use helpers and update the import line in code blocks. Example:

Before:
```ts
import { ChannelDirection } from 'iii-sdk'
const channel = await iii.createChannel()
```

After:
```ts
import { ChannelDirection, createChannel } from 'iii-sdk/helpers'
const channel = await createChannel(iii)
```

Apply the same rewrite pattern across Rust / Python code blocks.

- [ ] **Step 3: Verify mintlify build (if available)**

Run: `pnpm --filter docs build 2>/dev/null || echo "no docs build script"`
Expected: clean build (or skip if no script).

- [ ] **Step 4: Commit**

```bash
git add docs/
git commit -m "docs: update SDK reference + how-tos for helpers submodule"
```

---

### Task 16: Bump SDK versions and changelog

**Files:**
- Modify: `sdk/packages/rust/iii/Cargo.toml` (version)
- Modify: `sdk/packages/node/iii/package.json` (version)
- Modify: `sdk/packages/node/iii-browser/package.json` (version)
- Modify: `sdk/packages/python/iii/pyproject.toml` (version)
- Modify: each package's `CHANGELOG.md` (or repo-level changelog if that's the convention)

- [ ] **Step 1: Determine next major**

Current published is `0.16.0-next.3`. The next major candidate is `0.17.0-next.0`.

- [ ] **Step 2: Bump versions**

In each package config, update the version string:

- `sdk/packages/rust/iii/Cargo.toml`: `version = "0.17.0-next.0"`
- `sdk/packages/node/iii/package.json`: `"version": "0.17.0-next.0"`
- `sdk/packages/node/iii-browser/package.json`: `"version": "0.17.0-next.0"`
- `sdk/packages/python/iii/pyproject.toml`: `version = "0.17.0.dev0"` (Python uses PEP 440)

- [ ] **Step 3: Add changelog entry per package (or repo-level)**

Determine the convention by running `ls sdk/packages/rust/iii/CHANGELOG.md sdk/packages/node/iii/CHANGELOG.md sdk/packages/python/iii/CHANGELOG.md 2>/dev/null` and `ls CHANGELOG.md 2>/dev/null`. Edit whichever exists.

Add the following block at the top:

```markdown
## 0.17.0-next.0 — Helpers submodule

### Breaking changes

- Removed instance methods from `III` / `ISdk`: `register_trigger_type`, `unregister_trigger_type`, `create_channel`, `create_stream` (Python also drops `create_channel_async` on the instance).
- These items, plus `ChannelDirection`, `ChannelItem`, `extract_channel_refs`, `is_channel_ref`, are now imported from a new subpath:
  - Rust: `iii_sdk::helpers::*`
  - Node: `iii-sdk/helpers`
  - Browser: `iii-browser-sdk/helpers`
  - Python: `iii.helpers`
- `unregister_trigger_type` now takes the trigger-type id (string) instead of the full message object across Node and Python.

### New

- Rust SDK gains `create_stream` and the `IStream` trait for parity.
```

- [ ] **Step 4: Run all suites one more time**

Run: `cargo test --package iii-sdk && pnpm --filter iii-sdk --filter iii-browser-sdk test && cd sdk/packages/python/iii && uv run pytest`
Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add sdk/packages/rust/iii/Cargo.toml \
        sdk/packages/node/iii/package.json \
        sdk/packages/node/iii-browser/package.json \
        sdk/packages/python/iii/pyproject.toml \
        sdk/packages/*/iii/CHANGELOG.md CHANGELOG.md 2>/dev/null
git commit -m "chore: bump SDK versions to 0.17.0-next.0 for helpers submodule"
```

---

## Final Verification

- [ ] **Run full per-SDK test suites**

```bash
cargo test --package iii-sdk --all-targets
pnpm --filter iii-sdk --filter iii-browser-sdk build
pnpm --filter iii-sdk --filter iii-browser-sdk test
cd sdk/packages/python/iii && uv run pytest
```

All commands should exit 0.

- [ ] **Confirm zero remaining call sites of old API**

```bash
rg -n 'iii\.create_channel|iii\.createChannel|iii\.registerTriggerType|iii\.register_trigger_type\(|iii\.createStream|iii\.create_stream\(|iii\.unregisterTriggerType|iii\.unregister_trigger_type\(' sdk/ docs/ architecture/
```

Expected: no output.

- [ ] **Confirm no top-level re-exports of relocated items**

```bash
rg -n 'pub use channels::\{[^}]*(ChannelDirection|ChannelItem|extract_channel_refs|is_channel_ref)' sdk/packages/rust/iii/src/lib.rs
rg -n 'extract_channel_refs|is_channel_ref|ChannelDirection|ChannelItem' sdk/packages/python/iii/src/iii/__init__.py
```

Expected: no output.
