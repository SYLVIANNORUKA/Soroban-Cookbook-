# Instance Storage Pattern

A focused demonstration of Soroban instance storage, the storage map attached
to a deployed contract instance.

## What You'll Learn

- How to read and write values with `env.storage().instance()`
- Why instance storage is useful for small contract-wide state
- How shared instance TTL differs from persistent storage's per-key TTL
- When to choose instance storage and when to use persistent storage instead
- How to test counters, runtime config, and TTL keep-alive behavior

## Overview

Instance storage is physically attached to the contract instance ledger entry.
It shares the contract instance TTL, so a live contract instance keeps its
instance storage live too. This makes instance storage a good fit for small,
bounded state that is useful across most calls (admin address, runtime
config, counters, small metadata).

Do not use instance storage for unbounded per-user or per-entity data; use
persistent storage for that.

## Contract API (example)

| Function | Purpose |
| --- | --- |
| `set_instance(key, value)` | Store a `u64` under a named config key |
| `get_instance(key)` | Read a named config key as `Option<u64>` |
| `increment_counter()` | Increment a contract-wide transaction counter |
| `get_counter()` | Read the counter, defaulting to `0` |
| `extend_ttl()` | Explicitly refresh the instance TTL |

## Key Pattern

The example uses a typed key enum instead of raw symbols:

```rust
#[contracttype]
#[derive(Clone)]
pub enum InstanceKey {
    TxCounter,
    Config(Symbol),
}
```

# Instance Storage Pattern

A focused demonstration of Soroban instance storage, the storage map attached
to a deployed contract instance.

## What You'll Learn

- How to read and write values with `env.storage().instance()`
- Why instance storage is useful for small contract-wide state
- How shared instance TTL differs from persistent storage's per-key TTL
- When to choose instance storage and when to use persistent storage instead
- How to test counters, runtime config, and TTL keep-alive behavior

## Overview

Instance storage is physically attached to the contract instance ledger entry.
It shares the contract instance TTL, so a live contract instance keeps its
instance storage live too.

This makes instance storage a good fit for small, bounded state that is useful
across most calls:

- Admin or owner address
- Protocol configuration, such as fee basis points or limits
- Contract metadata and version flags
- Counters or aggregate totals
- Token pair metadata for a pool contract

Do not use instance storage for unbounded per-user or per-entity data. Instance
data is loaded with the contract instance, so large instance storage can make
every invocation more expensive.

## Instance vs Persistent Storage

| Feature | Instance Storage | Persistent Storage |
| --- | --- | --- |
| SDK API | `env.storage().instance()` | `env.storage().persistent()` |
| TTL scope | Shared with the contract instance | Independent per key |
| Size model | Limited by instance ledger-entry size | Supports unbounded keys |
| Best for | Small shared config/state | User, asset, proposal, or record data |
| Invocation footprint | Loaded with the contract instance | Loaded only when accessed |
| Expiration behavior | Archived with the instance | Archived per entry |

Use instance storage when the data is small, shared, and has a known upper
bound. Use persistent storage when the data grows with users, tokens, proposals,
orders, or other entities.

## Contract API

This example exposes six functions:

| Function | Purpose |
| --- | --- |
| `set_instance(key, value)` | Store a `u64` under a named config key |
| `get_instance(key)` | Read a named config key as `Option<u64>` |
| `increment_counter()` | Increment a contract-wide transaction counter |
| `get_counter()` | Read the counter, defaulting to `0` |
| `set_config(key, value)` | Semantic wrapper for runtime config |
| `get_config(key)` | Read runtime config as `Option<u64>` |
| `extend_ttl()` | Explicitly refresh the instance TTL |

## Key Pattern

The contract uses a typed key enum instead of raw symbols:
>>>>>>> 0de102b (docs: document instance storage pattern)

```rust
#[contracttype]
#[derive(Clone)]
pub enum InstanceKey {
    TxCounter,
    Config(Symbol),
}
```

<<<<<<< HEAD
## Common Operations

```rust
// Write a value and extend the instance TTL
env.storage().instance().set(&InstanceKey::TxCounter, &count);
env.storage().instance().extend_ttl(1_000, 10_000);

// Read with default
let count: u64 = env.storage().instance().get(&InstanceKey::TxCounter).unwrap_or(0);

// Check existence
let exists: bool = env.storage().instance().has(&InstanceKey::TxCounter);

// Remove
env.storage().instance().remove(&InstanceKey::TxCounter);
```

## Use Cases in this Example

1. Transaction counter — per-instance state that changes frequently and can expire with the instance.
2. Runtime configuration — operator-tunable parameters shared across calls but not required to survive upgrades.

## Build & Test

From this directory:

```bash
cargo test
cargo build --target wasm32-unknown-unknown --release
```

From repository root:

```bash
cargo test -p instance-storage
cargo build -p instance-storage --target wasm32-unknown-unknown --release
```

## Related Examples

- [02-storage-patterns](../02-storage-patterns/) — Compare all three storage types side-by-side
- [persistent-storage](../persistent-storage/) — Per-key TTL for user-specific data
- [temporary_storage](../temporary_storage/) — Ephemeral, single-ledger data

### TTL Management

Instance storage shares a single TTL for all entries. Calling `extend_ttl` on the instance refreshes the lifetime of _all_ instance keys at once.

```rust
const TTL_THRESHOLD: u32 = 1_000;
const TTL_EXTEND_TO: u32 = 10_000;

env.storage().instance().extend_ttl(TTL_THRESHOLD, TTL_EXTEND_TO);
```

## Use Cases in this Example

1.  **Transaction Counter**: A classic candidate for instance storage. It's per-instance state, changes often (benefiting from lower costs), and doesn't strictly need to survive upgrades.
2.  **Runtime Configuration**: Operator-tunable parameters (like fee rates or limits) that are shared across all invocations but can be reset if the contract is upgraded.

## Running the Example

### 1. Build the Contract

```bash
cargo build --target wasm32-unknown-unknown --release
```

### 2. Run Tests

```bash
cargo test
```

## Lessons Learned

- Instance storage is ideal for shared "instance-global" state.
- Single TTL management significantly simplifies housekeeping compared to persistent storage.
- Always call `extend_ttl` during both reads and writes to ensure the instance doesn't expire while in use.
>>>>>>> 0f17a2b (instance storage)
=======
Typed keys make the storage layout explicit and reduce accidental collisions.

## Writing Instance Data

```rust
pub fn set_instance(env: Env, key: Symbol, value: u64) {
    let storage_key = InstanceKey::Config(key);
    env.storage().instance().set(&storage_key, &value);
    env.storage().instance().extend_ttl(1_000, 10_000);
}
```

The important difference from persistent storage is that the TTL extension does
not name a specific key. A single instance TTL extension applies to the contract
instance and all instance storage entries.

## Use Case: Runtime Configuration

Runtime config is one of the safest instance-storage use cases because it is
small and contract-wide:

```rust
client.set_config(&symbol_short!("fee_bps"), &30);
assert_eq!(client.get_config(&symbol_short!("fee_bps")), Some(30));
```

Examples include protocol fees, caps, cooldown durations, feature flags, and
small version markers.

## Use Case: Contract Counter

The example also includes a counter:

```rust
assert_eq!(client.increment_counter(), 1);
assert_eq!(client.increment_counter(), 2);
assert_eq!(client.get_counter(), 2);
```

A counter is appropriate here because it is one small piece of shared state. A
counter per user would belong in persistent storage instead.

## Best Practices

1. Keep instance storage small and bounded.
2. Store user-specific and entity-specific records in persistent storage.
3. Refresh instance TTL from read/write paths that indicate active use.
4. Use typed key enums with `#[contracttype]`.
5. Return `Option<T>` for keys that may be unset.

## Running Tests

From the repository root:

```bash
cargo test -p instance-storage
```

Build the contract as Wasm:

```bash
cargo build -p instance-storage --target wasm32-unknown-unknown --release
```

Output:

```text
target/wasm32-unknown-unknown/release/instance_storage.wasm
```

## Related Examples

- [Storage Patterns](../02-storage-patterns/) - compares persistent, instance, and temporary storage
- [Persistent Storage](../persistent-storage/) - focused per-key durable storage example
- [Temporary Storage](../temporary_storage/) - short-lived storage example

## Further Reading

- [Use instance storage in a contract](https://developers.stellar.org/docs/build/guides/storage/use-instance)
- [Choosing the right storage type](https://developers.stellar.org/docs/build/guides/storage/choosing-the-right-storage)
- [State archival](https://developers.stellar.org/docs/learn/fundamentals/contract-development/storage/state-archival)
>>>>>>> 0de102b (docs: document instance storage pattern)
