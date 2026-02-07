# ETCD Metadata Structure

This document explains how Danube organizes metadata in ETCD. Understanding this structure is essential for developers working on cluster management, persistence, schema registry, or debugging production issues.

## Overview

Danube uses **ETCD** as its distributed metadata store to maintain cluster state, topic configurations, subscription tracking, schema registry, and cloud storage metadata. All brokers in the cluster share this centralized state.

### MetadataStore and LocalCache Pattern

To balance consistency with performance, Danube implements a dual-layer caching strategy:

**MetadataStorage (ETCD)**

- Source of truth for all cluster metadata
- Handles all **write operations** (PUT/DELETE) to ensure consistency

**LocalCache (per-broker)**

- Fast read access for broker-local operations  
- Continuously synchronized from two sources:
  1. **ETCD Watch** - Real-time updates from ETCD changes
  2. **Synchronizer Topic** - Internal topic broadcasting metadata changes

**Access Pattern:**

```bash
Write: Client → MetadataStore (ETCD) → Broadcast → LocalCache updates
Read:  Client → LocalCache (fast, no ETCD query)
```

## Resource Hierarchy

ETCD paths follow a hierarchical structure that mirrors Danube's logical architecture:

```bash
/
├── cluster/          # Cluster-wide state and broker coordination
├── namespaces/       # Namespace policies and topic lists
├── topics/           # Topic-level metadata (producers, subscriptions, policies)
├── schemas/          # Schema registry (Avro, Protobuf, JSON schemas)
├── danube-data/      # Persistent storage metadata (cloud objects, sealed state)
└── subscriptions/    # (Legacy) Subscription-level consumer tracking
```

---

## 1. Cluster Resources

**Base Path:** `/cluster`  
**Purpose:** Broker discovery, topic assignment, load balancing, and leader election

### 1.1 Cluster Name

**Path:** `/cluster/{CLUSTER_NAME}`  
**Value:** `null`  
**Purpose:** Marker for cluster existence

**Example:**

```bash
/cluster/MY_CLUSTER
null
```

### 1.2 Broker Registration

**Path:** `/cluster/register/{broker_id}`  
**Value:** JSON with broker endpoints  
**Purpose:** Broker metadata for client routing and inter-broker communication

**Example:**

```json
/cluster/register/625722408599041316
{
  "admin_addr": "http://0.0.0.0:50051",
  "advertised_addr": "0.0.0.0:6650",
  "broker_addr": "http://0.0.0.0:6650",
  "prom_exporter": "0.0.0.0:9040"
}
```

**Fields:**

- `broker_addr` - gRPC endpoint for client connections
- `admin_addr` - HTTP endpoint for administrative operations
- `advertised_addr` - Public address advertised to clients
- `prom_exporter` - Prometheus metrics endpoint

### 1.3 Broker State

**Path:** `/cluster/brokers/{broker_id}/state`  
**Value:** JSON with operational mode  
**Purpose:** Track broker lifecycle states (active, draining, maintenance)

**Example:**

```json
/cluster/brokers/625722408599041316/state
{
  "mode": "active",
  "reason": "boot"
}
```

**States:**

- `active` - Normal operation
- `draining` - Graceful shutdown in progress
- (custom states for maintenance/upgrades)

### 1.4 Topic Assignments

**Path:** `/cluster/brokers/{broker_id}/{namespace}/{topic}`  
**Value:** `null`  
**Purpose:** Maps topics to brokers; watched by brokers to load/unload topics

**Example:**

```bash
/cluster/brokers/13308604176970018988/default/reliable_topic
null
```

**Workflow:**

1. Load Manager assigns topic → Creates this key
2. Broker watches `/cluster/brokers/{own_id}/` → Detects new topic
3. Broker loads topic locally (WAL, subscriptions, etc.)

### 1.5 Unassigned Topics

**Path:** `/cluster/unassigned/{namespace}/{topic}`  
**Value:** `null` or JSON with unload reason  
**Purpose:** Queue for newly created or unloaded topics awaiting assignment

**Example (new topic):**

```bash
/cluster/unassigned/default/reliable_topic
null
```

**Example (topic unload):**

```json
/cluster/unassigned/default/reliable_topic
{
  "reason": "unload",
  "from_broker": 12549595323552083708
}
```

**Workflow:**

1. Producer creates topic → Broker writes to `/cluster/unassigned/`
2. Load Manager watches this path
3. Load Manager assigns to least-loaded broker
4. Unassigned marker deleted after successful assignment

### 1.6 Load Reports

**Path:** `/cluster/load/{broker_id}`  
**Value:** JSON with resource usage and topic list  
**Purpose:** Load Manager uses this to make assignment decisions

**Example:**

```json
/cluster/load/13308604176970018988
{
  "resources_usage": [
    {"resource": "CPU", "usage": 30},
    {"resource": "Memory", "usage": 30}
  ],
  "topic_list": ["/default/reliable_topic"],
  "topics_len": 1
}
```

### 1.7 Leader Election

**Path:** `/cluster/leader`  
**Value:** broker_id (u64)  
**Purpose:** Identifies the current cluster leader

**Example:**

```bash
/cluster/leader
625722408599041316
```

**Usage:**

- Load Manager runs only on the leader broker
- Schema registry operations coordinate through leader

---

## 2. Namespace Resources

**Base Path:** `/namespaces`  
**Purpose:** Namespace-level policies and topic organization

### 2.1 Namespace Policy

**Path:** `/namespaces/{namespace}/policy`  
**Value:** JSON with namespace-wide limits  
**Purpose:** Default policies for all topics in the namespace

**Example:**

```json
/namespaces/default/policy
{
  "max_consumers_per_subscription": 0,
  "max_consumers_per_topic": 0,
  "max_message_size": 10485760,
  "max_producers_per_topic": 0,
  "max_publish_rate": 0,
  "max_subscription_dispatch_rate": 0,
  "max_subscriptions_per_topic": 0
}
```

**Note:** `0` means unlimited

### 2.2 Topic Registry

**Path:** `/namespaces/{namespace}/topics/{namespace}/{topic}`  
**Value:** `null`  
**Purpose:** List all topics in a namespace

**Example:**

```bash
/namespaces/default/topics/default/reliable_topic
null
```

---

## 3. Topic Resources

**Base Path:** `/topics`  
**Purpose:** Topic-specific configuration, producers, and subscriptions

### 3.1 Topic Root

**Path:** `/topics/{namespace}/{topic}`  
**Value:** Number of partitions (usually `0` for non-partitioned)  
**Purpose:** Topic existence marker

**Example:**

```bash
/topics/default/reliable_topic
0
```

### 3.2 Delivery Mode

**Path:** `/topics/{namespace}/{topic}/delivery`  
**Value:** Dispatch strategy type  
**Purpose:** Determines message delivery guarantees

**Example:**

```bash
/topics/default/reliable_topic/delivery
"Reliable"
```

**Values:**

- `"Reliable"` - At-least-once delivery with persistence
- `"NonReliable"` - Best-effort delivery

### 3.3 Producer Registration

**Path:** `/topics/{namespace}/{topic}/producers/{producer_id}`  
**Value:** JSON with producer metadata  
**Purpose:** Track active producers on the topic

**Example:**

```json
/topics/default/reliable_topic/producers/13940288943180594845
{
  "access_mode": 0,
  "producer_id": 13940288943180594845,
  "producer_name": "prod_json_reliable",
  "status": true,
  "topic_name": "/default/reliable_topic"
}
```

### 3.4 Subscription Metadata

**Path:** `/topics/{namespace}/{topic}/subscriptions/{subscription_name}`  
**Value:** JSON with subscription configuration  
**Purpose:** Subscription settings and consumer tracking

**Example:**

```json
/topics/default/reliable_topic/subscriptions/subs_reliable
{
  "consumer_id": null,
  "consumer_name": "cons_reliable",
  "subscription_name": "subs_reliable",
  "subscription_type": 0
}
```

**Subscription Types:**

- `0` - Exclusive (single consumer)
- `1` - Shared (round-robin across consumers)
- `2` - Failover (primary/backup consumers)

### 3.5 Subscription Cursor

**Path:** `/topics/{namespace}/{topic}/subscriptions/{subscription_name}/cursor`  
**Value:** Last acknowledged offset (u64)  
**Purpose:** Track consumer progress through the topic

**Example:**

```bash
/topics/default/reliable_topic/subscriptions/subs_reliable/cursor
13
```

**Note:** This is written periodically (every 1000 acks or 5 seconds) for performance

---

## 4. Schema Registry

**Base Path:** `/schemas`  
**Purpose:** Schema versioning, validation, and compatibility management

### 4.1 Global Schema ID Counter

**Path:** `/schemas/_global/next_schema_id`  
**Value:** Next available schema ID (u64)  
**Purpose:** Monotonically increasing schema ID generation

**Example:**

```bash
/schemas/_global/next_schema_id
2
```

### 4.2 Schema Metadata

**Path:** `/schemas/{subject}/metadata`  
**Value:** JSON with schema evolution history  
**Purpose:** Complete schema subject information

**Example:**

```json
/schemas/product-catalog/metadata
{
  "compatibility_mode": "Backward",
  "created_at": 1768728611,
  "created_by": "danube-client",
  "id": 1,
  "latest_version": 2,
  "subject": "product-catalog",
  "topics_using": [],
  "updated_at": 1768728613,
  "versions": [
    {
      "created_at": 1768728611,
      "created_by": "danube-client",
      "version": 1,
      "fingerprint": "sha256:270e3390...",
      "schema_def": {"Avro": {...}},
      "is_deprecated": false,
      "tags": []
    },
    {
      "created_at": 1768728613,
      "version": 2,
      "fingerprint": "sha256:d6c3da89...",
      "schema_def": {"Avro": {...}}
    }
  ]
}
```

### 4.3 Schema Version

**Path:** `/schemas/{subject}/versions/{version}`  
**Value:** JSON with schema definition  
**Purpose:** Immutable schema version storage

**Example:**

```json
/schemas/product-catalog/versions/1
{
  "created_at": 1768728611,
  "created_by": "danube-client",
  "version": 1,
  "fingerprint": "sha256:270e3390980bc143...",
  "is_deprecated": false,
  "schema_def": {
    "Avro": {
      "fingerprint": "sha256:270e3390...",
      "raw_schema": "{\n  \"type\": \"record\",\n  \"name\": \"Product\",\n  ..."
    }
  },
  "tags": []
}
```

### 4.4 Schema ID Index

**Path:** `/schemas/_index/by_id/{schema_id}`  
**Value:** JSON mapping ID to subject  
**Purpose:** Reverse lookup from schema ID to subject name

**Example:**

```json
/schemas/_index/by_id/1
{
  "schema_id": 1,
  "subject": "product-catalog"
}
```

---

## 5. Cloud Storage Metadata

**Base Path:** `/danube-data/storage`  
**Purpose:** Track uploaded message segments in cloud storage (S3, GCS, filesystem)

### 5.1 Object Descriptors

**Path:** `/danube-data/storage/topics/{namespace}/{topic}/objects/{padded_start_offset}`  
**Value:** JSON with cloud object metadata  
**Purpose:** Map message offsets to cloud storage objects

**Example:**

```json
/danube-data/storage/topics/default/reliable_topic/objects/00000000000000000000
{
  "completed": true,
  "created_at": 1768728599,
  "end_offset": 29,
  "etag": null,
  "object_id": "data-0-29.dnb1",
  "offset_index": [[0, 0]],
  "size": 7164,
  "start_offset": 0
}
```

**Fields:**

- `object_id` - Filename in cloud storage
- `start_offset` / `end_offset` - Offset range contained in this object
- `offset_index` - Sparse index for fast offset lookups
- `size` - Object size in bytes
- `completed` - Whether upload finished successfully

**Offset Padding:** Start offsets are zero-padded to 20 digits for lexicographic sorting:

- `0` → `00000000000000000000`
- `30` → `00000000000000000030`

### 5.2 Current Object Cursor

**Path:** `/danube-data/storage/topics/{namespace}/{topic}/objects/cur`  
**Value:** JSON pointer to latest object  
**Purpose:** Quick lookup for most recent uploaded segment

**Example:**

```json
/danube-data/storage/topics/default/reliable_topic/objects/cur
{
  "start": "00000000000000000030"
}
```

### 5.3 Sealed State (Topic Move)

**Path:** `/danube-data/storage/topics/{namespace}/{topic}/state`  
**Value:** JSON with last committed offset  
**Purpose:** Preserve offset continuity when topic moves between brokers

**Example:**

```json
/danube-data/storage/topics/default/reliable_topic/state
{
  "sealed": true,
  "last_committed_offset": 21,
  "broker_id": 10285063371164059634,
  "timestamp": 1768625254
}
```

**When Created:**

- Written during topic unload (`danube-admin topics unload`)
- Read by new broker to initialize WAL at `last_committed_offset + 1`
- Deleted after successful topic load on new broker

**Critical for:** Preventing offset collisions during topic migrations

---

## 6. Subscriptions Resources (Legacy)

**Base Path:** `/subscriptions`  
**Purpose:** Consumer metadata (largely deprecated in favor of `/topics/.../subscriptions/`)

**Path:** `/subscriptions/{subscription_name}/{consumer_id}`  
**Value:** Consumer metadata JSON

**Note:** This structure is mostly legacy. Current implementation stores subscription state under `/topics/{namespace}/{topic}/subscriptions/`.

---

## Common Patterns

### Watching for Changes

Brokers watch specific ETCD prefixes to react to cluster changes:

```rust
// Broker watches its own assignment path
broker.watch("/cluster/brokers/{own_broker_id}/")
  → Loads/unloads topics dynamically

// Load Manager watches unassigned topics
load_manager.watch("/cluster/unassigned/")
  → Assigns topics to brokers

// Schema registry watches schema changes
schema_service.watch("/schemas/")
  → Invalidates local schema cache
```

### Key Naming Conventions

- **Broker IDs:** 64-bit unsigned integers (e.g., `625722408599041316`)
- **Topic Paths:** `/{namespace}/{topic}` (e.g., `/default/reliable_topic`)
- **Padded Offsets:** Zero-padded to 20 digits for sorting
- **Null Values:** Used as existence markers when no data needed

### Performance Considerations

**LocalCache First:**  
All read-heavy operations (e.g., routing clients, looking up producers) use LocalCache to avoid ETCD query overhead.

**Batch Updates:**  
Subscription cursors are updated in batches (every 1000 ACKs or 5s) to reduce ETCD write load.

**Watch-Based Sync:**  
ETCD watches provide millisecond-latency updates to LocalCache, keeping reads fast and consistent.

---

## Debugging Tips

### List All Keys

```bash
etcdctl get / --prefix --keys-only
```

### Watch Cluster Changes Live

```bash
etcdctl watch / --prefix
```

### Query Specific Topic

```bash
# Topic existence
etcdctl get /topics/default/my_topic

# Find which broker owns it
etcdctl get /cluster/brokers/ --prefix | grep my_topic

# Check cloud storage objects
etcdctl get /danube-data/storage/topics/default/my_topic/objects/ --prefix
```

### Inspect Schema Evolution

```bash
# Get all versions
etcdctl get /schemas/product-catalog/versions/ --prefix

# Check compatibility mode
etcdctl get /schemas/product-catalog/metadata
```

---

## Summary

Danube's ETCD structure provides:

✅ **Cluster coordination** - Broker discovery, leader election, topic assignment  
✅ **Metadata persistence** - Topics, producers, subscriptions, schemas  
✅ **Cloud storage tracking** - Object descriptors for tiered storage  
✅ **State continuity** - Sealed state for seamless topic migration  

Understanding this structure is essential for developing features that interact with cluster state, debugging production issues, or contributing to Danube's distributed systems architecture.
