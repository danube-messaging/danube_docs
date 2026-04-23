# Key-Shared Dispatch Architecture

Key-Shared is a subscription model that combines **per-key ordering** with **multi-consumer parallelism**. All messages sharing the same routing key are delivered to exactly one consumer, in order, while different keys are dispatched to different consumers in parallel.

At a high level, the design is built around three ideas:

- keep **key-to-consumer assignment stable and predictable** with consistent hashing
- keep the **subscription cursor safe** by advancing only past contiguously-acked offsets
- keep **poisoned keys isolated** so one misbehaving key cannot stall the entire subscription

This page describes the internal architecture of the Key-Shared dispatcher, how the main components collaborate, and the design decisions behind them.

## Design Goals

- **Per-key ordering**
  - Messages with the same routing key must always be delivered to the same consumer, in order. No two messages for the same key may be in-flight simultaneously.
- **Multi-key parallelism**
  - Different routing keys can be dispatched to different consumers and remain in-flight concurrently. This provides throughput proportional to the number of distinct keys.
- **Key-affinity stability**
  - Adding or removing a consumer should remap only ~1/N of existing key assignments, not all of them.
- **Cursor safety**
  - The persisted subscription cursor must never skip past unacknowledged messages. A broker restart must replay from the correct position.
- **Key-level fault isolation**
  - A poison message (retry-exhausted) on one key should not block other keys from making progress.
- **Per-consumer backpressure**
  - A single slow consumer should not consume the entire in-flight window.

## High-level Model

Each reliable Key-Shared subscription is backed by these components:

1. **`KeySharedConsumerState`** — Maps routing keys to consumers via a consistent hash ring with optional glob-based key filtering.
2. **`InFlightWindow`** — Tracks all in-flight and blocked messages. Enforces single-message-per-key, contiguous cursor advancement, and per-consumer capacity limits.
3. **Dispatch loop** — A `tokio::select!` loop driven by two sources: a command channel (acks, nacks, consumer add/remove, poll requests) and a periodic heartbeat timer.
4. **Heartbeat watchdog** — Runs every 500ms to detect ack timeouts, retry poison messages, evict inactive consumers, and poll for WAL lag.
5. **`SubscriptionEngine`** — Owns the WAL stream cursor, failure policy, and debounced cursor persistence.
6. **Poison handler** — Applies the configured poison policy (Drop, DeadLetter, Block) to retry-exhausted messages.

The non-reliable Key-Shared variant omits components 2, 4, 5, and 6. It routes messages by key hash and drops them on saturation with no ack tracking, no retry, and no cursor persistence.

## Core Components

### `KeySharedConsumerState`

`KeySharedConsumerState` maintains the consumer ring and handles the two-stage key routing algorithm:

1. **Filter**: which consumers accept this key? A consumer with no key filters accepts all keys. A consumer with filters accepts only keys matching at least one glob pattern.
2. **Hash**: among eligible consumers, use consistent hashing to select the owner.

#### Consistent Hashing

Each consumer is allocated `VNODES_PER_CONSUMER` (100) virtual nodes on a 32-bit hash ring. Virtual node hashes are derived from `FNV-1a("{consumer_id}-{vnode_idx}")` with murmur3 finalization to ensure good bit distribution.

When a message arrives, the routing key is hashed to a 32-bit value, and a binary search on the sorted ring finds the first virtual node ≥ that hash (clockwise walk, wrapping at the end).

Important properties:

- **Stability**: adding or removing one consumer remaps approximately 1/N of keys. Verified by unit tests over 1,000 keys, remapping stays within 15-55% when going from 2→3 consumers.
- **Determinism**: the same routing key always selects the same consumer for a given consumer set.
- **Filter interaction**: when some consumers declare key filters, the ring is projected onto only the eligible consumers for that key. The common path (no filters, all consumers eligible) uses the full ring with zero allocation.

#### Key Filtering

Consumers may declare one or more glob patterns at subscription time:

| Pattern | Matches |
|---------|---------|
| `"payment"` | Exact match only |
| `"ship*"` | `"shipping"`, `"shipment"`, etc. |
| `"eu-west-?"` | `"eu-west-1"`, `"eu-west-2"`, etc. |
| `"*"` | Everything (same as no filter) |

Glob matching is implemented with an iterative backtracking algorithm (`simple_glob`) that handles `*` and `?` without regex overhead.

If a message's routing key matches no consumer's filters, the message is skipped, the offset is treated as implicitly acked and the cursor advances past it.

### `InFlightWindow`

`InFlightWindow` replaces the single-slot `Option<PendingDelivery>` used by Shared and Exclusive dispatchers with a multi-message concurrent tracking window.

#### Key Invariant

**At most one message per routing key is in-flight at any time.** Additional messages for the same key are held in `blocked_queue` until the in-flight one is resolved.

#### Data Structures

| Structure | Type | Purpose |
|-----------|------|---------|
| `in_flight` | `BTreeMap<u64, InFlightEntry>` | Messages dispatched and awaiting ack, keyed by WAL offset |
| `active_keys` | `HashMap<String, u64>` | Which routing keys currently have an in-flight message |
| `blocked_queue` | `VecDeque<StreamMessage>` | Messages polled from WAL but blocked because their key is active |
| `acked_offsets` | `BTreeSet<u64>` | Offsets that have been acked but are ahead of the contiguous frontier |
| `safe_cursor` | `Option<u64>` | The highest offset where all offsets ≤ this value are resolved |
| `per_consumer_in_flight` | `HashMap<u64, usize>` | Per-consumer in-flight count for backpressure |

#### Contiguous Cursor Advancement

The `safe_cursor` advances only when offsets form a contiguous chain from the current position. This is critical because the `SubscriptionEngine` persists this cursor and uses it as the replay start point after restart.

Example:

```text
Dispatched: offsets 10, 11, 12, 13
Acked:      offsets 10, 12, 13  (11 still in-flight)

safe_cursor = 10  (can't advance past 11)

When offset 11 is acked:
safe_cursor = 13  (10,11,12,13 all contiguous)
```

Out-of-order acks are common in Key-Shared dispatch because different keys are acked independently. The `acked_offsets` set accumulates future acks, and `advance_safe_cursor()` walks forward from the current position, draining contiguous entries.

#### Per-Consumer Backpressure

Each consumer has a maximum in-flight capacity (default: 1,000 messages). When a consumer's count reaches this limit, new messages destined for that consumer are pushed to the blocked queue instead. This prevents a single slow consumer from occupying the entire window.

#### Window Capacity

The global window has a `max_window_size` (default: 10,000) that limits the total number of in-flight + blocked messages. When the window is full, the heartbeat stops polling new messages from the WAL, creating natural backpressure into the storage layer.

### Dispatch Loop

The main loop is a `tokio::select!` between two sources:

```
loop {
    select! {
        cmd = control_rx.recv() => handle_command(cmd, ...)
        _   = heartbeat.tick()  => handle_heartbeat(...)
    }
}
```

**Commands** are sent by the broker's consumer handler and subscription engine. They include consumer add/remove, message ack/nack, poll-and-dispatch triggers, and flush requests.

**Heartbeat** fires every 500ms and performs five phases of maintenance work.

### Heartbeat Watchdog

The heartbeat runs every 500ms and performs five phases:

#### Phase 1: Ack Timeouts

Scans all in-flight entries for messages past their `ack_timeout`. Timed-out messages are marked for retry by recording the timeout event and making them eligible for re-dispatch.

#### Phase 2: Poison Handling

Scans for in-flight entries that have exhausted their retry budget. Delegates to `resolve_poisoned_delivery` from the shared poison handler:

- **Drop**: The message is skipped by this subscription — it is no longer tracked or retried for this subscription's consumers. However, the message still remains in the topic WAL and is available for delivery to other subscriptions on the same topic. The offset is marked as acked in `acked_offsets` so the cursor can advance. Any blocked messages for the same key are also drained.
- **DeadLetter**: Message is published to the configured DLQ topic via the replicator, then treated like Drop.
- **Block**: Message stays in place. The key remains active, preventing new messages for that key from being dispatched. Other keys continue normally.

The Key-Shared dispatcher manages its own cursor advancement rather than delegating to the engine, because multi-message contiguous tracking requires coordination with `InFlightWindow`.

#### Phase 3: Retry-Ready Re-dispatch

Messages that have completed their backoff delay are re-dispatched to a consumer. The consumer selection uses the same hash ring, so a retried message may land on a different consumer if the ring has changed (e.g., due to eviction).

#### Phase 4: Inactive Consumer Eviction

Tracks per-consumer "inactive tick" counters. A consumer is considered inactive when its `Arc<AtomicBool>` status flag is `false` (set by the gRPC handler when the client disconnects). After 6 consecutive inactive ticks (3 seconds), the consumer is evicted:

1. All its in-flight entries are removed and their offsets marked as acked in `acked_offsets`
2. The consumer is removed from the hash ring (triggers ring rebuild)
3. Freed keys are dispatched to remaining consumers via `dispatch_unblocked`
4. The cursor is persisted if it advanced

#### Phase 5: Lag-Driven Polling

If the window has capacity and the WAL has unread messages (lag), the heartbeat polls new messages. This ensures continuous throughput when the WAL is being written to faster than consumers can ack.

### Consumer Eviction and Cursor Safety

When a consumer is evicted, its in-flight entries become orphaned, they can no longer be acked because the consumer is gone. Without special handling, these offsets would create permanent gaps in the contiguous cursor, stalling the subscription forever.

The fix: `remove_consumer_entries()` treats evicted offsets as implicitly acked by inserting them into `acked_offsets`. This allows the safe cursor to advance past them. The tradeoff is that messages assigned to the evicted consumer may be lost (at-most-once for those specific messages during eviction), but the subscription as a whole continues making progress.

## Message Flow

A message's lifecycle through the Key-Shared dispatcher:

1. **Poll**: `poll_next()` reads the next message from the WAL stream.
2. **Filter**: `select_consumer()` checks key filters and hashes to a consumer. If no consumer matches, the offset is skipped.
3. **Block check**: if the message's routing key is already active (another message for the same key is in-flight), the message is pushed to `blocked_queue`.
4. **Capacity check**: if the consumer has reached its per-consumer limit, the message is also blocked.
5. **Send**: `send_message()` pushes the message to the consumer's gRPC output channel.
6. **Ack**: the consumer acks, freeing the key lock and triggering `dispatch_unblocked()` to check if blocked messages can now be sent.
7. **Cursor**: `advance_safe_cursor()` walks forward from the current cursor, advancing past contiguously-acked offsets.

## Non-Reliable Key-Shared

The non-reliable variant uses the same `KeySharedConsumerState` (consistent hashing + key filters) but skips all reliability machinery:

- **No `InFlightWindow`**: no per-key blocking, no cursor tracking
- **No ack/nack**: messages are fire-and-forget
- **No retry or DLQ**: dropped messages are lost
- **No background task**: dispatch is inline, called directly by the message write path
- **Drop-on-saturation**: if the target consumer's channel is full, `try_send_message()` drops the message

This mode is appropriate when per-key routing is desired but message loss is acceptable (e.g., real-time metrics, live streaming).

## Comparison with Other Subscription Models

| Aspect | Exclusive | Shared | Key-Shared |
|--------|-----------|--------|------------|
| Consumers | 1 | N | N |
| Ordering | Total | None | Per-key |
| In-flight tracking | Single slot | Single slot | Multi-message window |
| Cursor advancement | Sequential | Sequential | Contiguous-ack |
| Key affinity | N/A | N/A | Consistent hash ring |
| Key filtering | N/A | N/A | Glob patterns |
| Backpressure | Per-subscription | Per-subscription | Per-consumer + global |
| Consumer rebalancing | Failover | Round-robin | Hash ring rebuild (~1/N remapped) |
