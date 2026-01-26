# Danube Load Manager & Rebalancing Architecture

[Danube](https://github.com/danube-messaging/danube)'s Load Manager is a distributed system that ensures optimal topic placement across broker clusters. It combines intelligent topic assignment for new topics with automated rebalancing to maintain cluster health as workloads evolve. This creates a self-optimizing messaging infrastructure that adapts to changing conditions without manual intervention.

This page explains what the Load Manager is, why it's essential for production Danube clusters, how it assigns topics to brokers, and how automated rebalancing keeps your cluster balanced over time.

---

## What is the Load Manager?

The **Load Manager** is a critical subsystem in Danube that handles the distribution of topics across brokers in a cluster. Think of it as the "traffic controller" that decides which broker should handle which topic, ensuring that no single broker becomes overwhelmed while others sit idle.

### Core Responsibilities

- **Topic Assignment** - Places new topics on the least loaded broker using intelligent ranking algorithms
- **Load Monitoring** - Continuously tracks broker resource utilization (CPU, memory, throughput, topic count)
- **Cluster Rebalancing** - Automatically moves topics between brokers to maintain balanced load distribution
- **Failover Management** - Reassigns topics from failed brokers to healthy ones
- **Resource Optimization** - Prevents hotspots and ensures efficient cluster utilization

### Why It Matters

Without intelligent load management, clusters suffer from:

- **Hotspots** - One broker handles most traffic while others are idle
- **Resource waste** - Underutilized brokers mean wasted infrastructure costs
- **Performance degradation** - Overloaded brokers cause latency spikes and message backlogs
- **Manual toil** - Operators spend time manually moving topics to fix imbalances
- **Scaling challenges** - Hard to add/remove brokers without disrupting service

The Load Manager solves these problems automatically, keeping your cluster healthy 24/7.

---

## High-Level Architecture

```bash
┌─────────────────────────────────────────────────────────────────┐
│                        ETCD Metadata Store                       │
│  • Broker registrations    • Topic assignments                  │
│  • Load reports            • Rebalancing history                │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ Watch events
                 │
┌────────────────▼───────────────┐         ┌────────────────────┐
│      Load Manager (Leader)      │         │   Other Brokers    │
│                                 │         │  (Followers)       │
│  1. Monitor Load Reports        │◄────────┤  • Publish loads   │
│  2. Calculate Rankings          │         │  • Watch events    │
│  3. Assign New Topics           │         │  • Execute unloads │
│  4. Detect Imbalance (CV)       │         └────────────────────┘
│  5. Select Topics to Move       │
│  6. Execute Rebalancing         │
│                                 │
│  Components:                    │
│  ├─ Rankings Calculator         │
│  ├─ Imbalance Detector          │
│  ├─ Candidate Selector          │
│  └─ Rebalancing Executor        │
└─────────────────────────────────┘

Flow for New Topic Assignment:
1. Topic created → Added to /unassigned/ path
2. Load Manager (leader) watches unassigned topics
3. Calculates broker rankings (least loaded first)
4. Assigns topic to top-ranked broker
5. Broker receives assignment and loads topic

Flow for Automated Rebalancing:
1. Load Manager checks cluster balance every N minutes
2. Calculates imbalance (coefficient of variation)
3. If CV > threshold → Select topics to move
4. Creates unassigned marker with target broker hint
5. Deletes assignment from overloaded broker
6. Topic moves to underloaded broker
7. Records move in history for rate limiting
```

### Component Roles

**Load Manager (Leader-Only)**  
Centralized coordinator that assigns topics and executes rebalancing. Only the elected cluster leader runs these operations to ensure consistency.

**Broker Load Reports**  
Every broker publishes resource utilization metrics (CPU, memory, disk I/O, network I/O) and per-topic statistics (throughput, connections, backlog) to ETCD every 30 seconds.

**Rankings Calculator**  
Scores all brokers based on their current load using configurable strategies (Fair, Balanced, WeightedLoad). Lower scores mean less loaded.

**Imbalance Detector**  
Calculates statistical measures (coefficient of variation, standard deviation) to determine if cluster needs rebalancing.

**Rebalancing Executor**  
Orchestrates topic moves using graceful unload workflow. Enforces rate limits and cooldowns to prevent disruption.

**ETCD Metadata Store**  
Single source of truth for cluster state. Stores broker registrations, topic assignments, load reports, and rebalancing history.

---

## Core Concepts

### Brokers and Broker States

A **broker** is a server instance running the Danube broker process. Each broker has a unique ID and maintains:

- **Topic assignments** - Which topics it currently serves
- **Resource metrics** - CPU, memory, disk I/O, network I/O usage
- **Topic metrics** - Per-topic throughput, connections, backlog

**Broker states:**

- **Active** - Normal operation, can receive new topics
- **Draining** - Preparing to shut down, topics being unloaded
- **Drained** - All topics moved, ready for removal

Only active brokers are considered for topic assignment and rebalancing calculations.

### Load Reports

Every broker generates a **LoadReport** every 30 seconds containing:

**System-Level Metrics:**

- CPU usage percentage (0-100%)
- Memory usage percentage (0-100%)
- Disk I/O throughput (bytes/second)
- Network I/O throughput (bytes/second)

**Topic-Level Metrics:**

- Message rate (messages/second)
- Byte rate (bytes/second, converted to Mbps)
- Producer count
- Consumer count
- Subscription count
- Backlog messages

**Aggregate Metrics:**

- Total throughput (Mbps across all topics)
- Total message rate
- Total lag/backlog

These reports are published to ETCD where the Load Manager consumes them for ranking and rebalancing decisions.

### Broker Rankings

**Rankings** are a sorted list of `(broker_id, load_score)` pairs ordered by load (ascending = less loaded first). The Load Manager maintains these rankings in-memory and updates them whenever new load reports arrive.

Rankings determine:

- Which broker receives the next topic assignment
- Which brokers are overloaded (candidates for offloading)
- Which brokers are underloaded (candidates for receiving topics)

The ranking algorithm is configurable (see Assignment Strategies).

### Coefficient of Variation (CV)

The **coefficient of variation** measures cluster balance:

```
CV = (standard_deviation / mean_load) × 100%
```

**Interpretation:**

- **CV < 20%** - Excellent balance, loads are very similar
- **CV 20-30%** - Good balance, acceptable variance
- **CV 30-40%** - Moderate imbalance, consider rebalancing
- **CV > 40%** - Significant imbalance, rebalancing recommended

The Load Manager uses CV thresholds to trigger automated rebalancing based on the configured aggressiveness level.

### Assignment Strategies

Three strategies control how new topics are assigned:

| Strategy | Algorithm | Best For | Overhead |
|----------|-----------|----------|----------|
| **Fair** | Topic count only | Development, testing, predictable placement | Lowest |
| **Balanced** | Weighted: topic_load (30%) + CPU (35%) + Memory (35%) | General production, mixed workloads | Medium |
| **WeightedLoad** | Adaptive bottleneck detection | Variable workloads, auto-optimization | Highest |

**Fair Strategy:**  
Simple topic counting. Broker with fewest topics gets the next assignment. Ignores actual resource usage and topic throughput.

**Balanced Strategy (RECOMMENDED):**  
Multi-factor scoring combining weighted topic load (message rate, connections, backlog) with system resources. Most reliable for production clusters.

**WeightedLoad Strategy:**  
Smart algorithm that detects which resource is under most pressure (CPU, memory, throughput, etc.) and prioritizes it in scoring. Adapts automatically to workload patterns.

### Rebalancing Aggressiveness

Three aggressiveness levels control automated rebalancing behavior:

| Level | CV Threshold | Check Interval | Max Moves/Hour | Best For |
|-------|--------------|----------------|----------------|----------|
| **Conservative** | 40% | 10 minutes | 5 | Stable clusters, minimal disruption |
| **Balanced** | 30% | 5 minutes | 10 | General production use |
| **Aggressive** | 20% | 2 minutes | 20 | Fast-changing workloads, test clusters |

Higher aggressiveness means:

- More frequent balance checks
- Lower tolerance for imbalance
- More topic moves per hour
- Faster response to load changes
- Higher metadata store overhead

### Topic Blacklists

Administrators can configure topic patterns that should **never be rebalanced**. This is useful for:

- Critical topics that must stay on specific brokers
- Low-latency topics where moves would cause disruption
- Topics with large backlogs (expensive to move)
- System topics used for coordination

Blacklist patterns support wildcards:

```yaml
blacklist_topics:
  - "/system/*"           # All system topics
  - "/default/critical-*" # Critical workload topics
  - "/analytics/logs"     # Specific high-volume topic
```

---

## How It Works: Topic Assignment

When a new topic is created, it must be assigned to a broker. This process ensures the topic lands on the broker best suited to handle it.

### Step-by-Step Flow

**1. Topic Creation**

A producer creates a topic by sending a message to a topic that doesn't exist yet. The broker creates the topic metadata in ETCD under `/topics/{namespace}/{topic}`.

**2. Unassigned Entry Creation**

The broker creates an entry in ETCD at:

```bash
/cluster/unassigned/{namespace}/{topic}
```

This signals to the Load Manager that a topic needs assignment. The value is empty (no marker data).

**3. Load Manager Detection**

The Load Manager (running on the leader broker) watches the `/cluster/unassigned/` path. When a new entry appears, it triggers the assignment workflow.

**4. Rankings Calculation**

The Load Manager retrieves the latest load reports for all active brokers and calculates rankings using the configured strategy:

**Fair Strategy:**

```bash
score = topic_count
```

**Balanced Strategy:**

```bash
weighted_topic_load = (count × 0.2 + throughput × 0.3 + 
                       connections × 0.3 + backlog × 0.2)
score = (weighted_topic_load × 0.3) + (CPU × 0.35) + (Memory × 0.35)
```

**WeightedLoad Strategy:**

```bash
1. Normalize all metrics (0.0 to 1.0)
2. Find bottleneck (max utilization metric)
3. If bottleneck > 70% → Weight it heavily (50%)
4. Otherwise → Balance evenly (25% each metric)
```

Brokers are sorted by score (ascending), so the least loaded broker appears first.

**5. Broker Selection**

The Load Manager selects brokers from the top of the rankings (lowest scores). To prevent all assignments going to one broker when multiple brokers have similar loads, it uses **round-robin within threshold**:

```bash
1. Get minimum score from rankings
2. Find all brokers within 10% of minimum score
3. Round-robin through these candidates
4. Track last selected broker to ensure rotation
```

This ensures even distribution when brokers have comparable loads.

**6. Assignment Creation**

The Load Manager writes the assignment to ETCD:

```bash
/cluster/brokers/{broker_id}/{namespace}/{topic} → null
```

The target broker watches this path and receives the assignment event.

**7. Topic Loading**

The target broker:

1. Creates the topic instance locally
2. Initializes subscriptions and storage
3. Begins accepting producer connections
4. Starts reporting topic metrics in load reports

**8. Cleanup**

The Load Manager deletes the unassigned entry:

```bash
DELETE /cluster/unassigned/{namespace}/{topic}
```

This prevents duplicate assignments.

**9. Internal State Update**

The Load Manager updates its in-memory broker usage map:

```bash
brokers_usage[broker_id].topics.push(new_topic_placeholder)
```

This ensures the next assignment sees the updated load, even before the broker's next load report arrives.

### Assignment Visualization

```bash
New Topic Created
       │
       ▼
  ┌─────────────────┐
  │ /topics/ns/top  │  Topic metadata created
  └─────────────────┘
       │
       ▼
  ┌─────────────────┐
  │ /unassigned/... │  Unassigned marker created
  └─────────────────┘
       │
       │ Watch event
       ▼
  ┌──────────────────────────────────────┐
  │        Load Manager (Leader)         │
  │                                      │
  │  1. Read load reports from ETCD     │
  │  2. Calculate broker rankings        │
  │     Broker 1: score=20  ← least     │
  │     Broker 2: score=25              │
  │     Broker 3: score=45  ← most      │
  │  3. Select Broker 1 (round-robin)   │
  └──────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────┐
  │ /brokers/1/ns/top   │  Assignment created
  └─────────────────────┘
       │
       │ Watch event
       ▼
  ┌─────────────────┐
  │   Broker 1      │  Loads topic, starts serving
  └─────────────────┘
       │
       ▼
  ┌─────────────────┐
  │ DELETE /unassigned/... │  Cleanup
  └─────────────────┘
```

### Special Cases

**Broker Unload (Manual):**  
When an administrator unloads a broker, topics create unassigned markers with:

```json
{
  "reason": "unload",
  "from_broker": 12345
}
```

The Load Manager excludes the source broker when selecting the target.

**Broker Failure:**  
When a broker crashes, the leader detects the registration deletion in ETCD and:

1. Deletes all assignments from the failed broker
2. Creates unassigned markers for all topics (no hint)
3. Normal assignment workflow reassigns topics to healthy brokers

**All Brokers Equally Loaded:**  
When all brokers have identical scores, the Load Manager uses stable round-robin rotation to distribute topics evenly.

---

## How It Works: Automated Rebalancing

As workloads change over time, clusters can become imbalanced. Some brokers handle more traffic than others, leading to hotspots and inefficiency. Automated rebalancing solves this by continuously monitoring cluster health and moving topics to restore balance.

### Rebalancing Lifecycle

```bash
┌──────────────────────────────────────────────────────────────┐
│                  Rebalancing Loop (Leader)                    │
│                                                               │
│  Every N minutes (configured by check_interval):             │
│                                                               │
│  1. Check Leadership                                          │
│     ├─ If follower → Skip cycle                              │
│     └─ If leader → Continue                                   │
│                                                               │
│  2. Check Configuration                                       │
│     ├─ If rebalancing disabled → Skip cycle                  │
│     └─ If enabled → Continue                                  │
│                                                               │
│  3. Check Cluster Health                                      │
│     ├─ If < 2 brokers → Skip cycle                          │
│     └─ If >= 2 brokers → Continue                            │
│                                                               │
│  4. Calculate Imbalance Metrics                               │
│     ├─ Get broker rankings                                    │
│     ├─ Calculate mean, std_deviation, CV                     │
│     └─ Identify overloaded/underloaded brokers               │
│                                                               │
│  5. Decide If Rebalancing Needed                             │
│     ├─ If CV < threshold → Cluster balanced, skip            │
│     └─ If CV > threshold → Continue                          │
│                                                               │
│  6. Select Topics to Move                                     │
│     ├─ For each overloaded broker                            │
│     ├─ Filter blacklisted topics                             │
│     ├─ Filter recently moved topics (cooldown)               │
│     ├─ Filter topics too young (min_topic_age)               │
│     ├─ Sort by load (lightest first)                         │
│     └─ Select lightest topic                                 │
│                                                               │
│  7. Select Target Brokers                                     │
│     ├─ Find underloaded brokers from rankings                │
│     ├─ Exclude source broker                                 │
│     └─ Select least loaded active broker                     │
│                                                               │
│  8. Execute Rebalancing Moves                                 │
│     ├─ Check rate limit (max_moves_per_hour)                │
│     ├─ Create unassigned marker with target hint             │
│     ├─ Delete assignment from source broker                  │
│     ├─ Wait for topic reassignment                           │
│     ├─ Record move in history                                │
│     ├─ Apply cooldown delay                                  │
│     └─ Repeat for next move                                  │
│                                                               │
│  9. Update Metrics & Logs                                     │
│     └─ Log cycle completion and move count                   │
└──────────────────────────────────────────────────────────────┘
```

### Imbalance Detection

The Load Manager calculates statistical measures to determine if rebalancing is needed:

**Metrics Calculated:**

- **Mean load** - Average load across all active brokers
- **Standard deviation** - Spread of loads around the mean
- **Coefficient of variation (CV)** - `(std_dev / mean) × 100%`
- **Max load** - Highest loaded broker
- **Min load** - Lowest loaded broker

**Broker Classification:**

- **Overloaded** - Load > (mean + 1 std_dev)
- **Underloaded** - Load < (mean - 1 std_dev) AND load < (mean × 0.5)
- **Normal** - Between overloaded and underloaded

**Example:**

```bash
Broker 1: load = 45
Broker 2: load = 30
Broker 3: load = 25

mean = 33.33
std_dev = 8.50
CV = (8.50 / 33.33) = 0.255 = 25.5%

Overloaded: load > 41.83 → Broker 1
Underloaded: load < 25.00 AND load < 16.67 → None
Normal: Broker 2, Broker 3
```

If `CV > threshold` (based on aggressiveness level), rebalancing is triggered.

### Candidate Selection

Once imbalance is detected, the Load Manager selects topics to move:

**Selection Criteria:**

1. **Source brokers** - Only overloaded brokers (load > mean + std_dev)
2. **Topic age** - Skip topics younger than `min_topic_age_seconds` (default 5 minutes)
3. **Blacklist** - Skip topics matching blacklist patterns
4. **Cooldown** - Skip topics moved in the last `cooldown_seconds` (default 60 seconds)
5. **Load score** - Prefer lightest topics (easier to move, less disruption)

**Topic Load Scoring:**

```bash
load_score = (topic_count × 0.2) + 
             (throughput_mbps × 0.3) + 
             (connections × 0.3) + 
             (backlog/10000 × 0.2)
```

Topics are sorted by score (ascending), and the lightest topic is selected first.

**Move Limits:**

- Maximum 1 topic moved per rebalancing cycle (single-move design)
- Maximum N moves per hour (rate limiting, configurable)
- Cooldown delay between individual moves (prevents oscillations)

### Target Broker Selection

For each topic to move, the Load Manager selects a target broker:

1. **Get rankings** - Sorted list of brokers by load (ascending)
2. **Filter active** - Exclude brokers in draining/drained state
3. **Exclude source** - Can't move topic to same broker
4. **Select first** - Pick the least loaded broker meeting criteria

If no suitable target exists (e.g., only one broker active), the move is skipped with a warning.

### Move Execution

The Load Manager executes moves using the existing graceful unload workflow:

**Step 1: Create Unassigned Marker**

```json
Path: /cluster/unassigned/{namespace}/{topic}
Value: {
  "reason": "rebalance",
  "from_broker": 12345,
  "to_broker": 67890,
  "timestamp": 1234567890
}
```

The `to_broker` hint tells the assignment logic to prefer broker 67890 (if active).

**Step 2: Delete Source Assignment**

```bash
DELETE /cluster/brokers/12345/{namespace}/{topic}
```

This triggers the source broker's topic watcher, which:

1. Seals the topic (no new producers)
2. Drains in-flight messages
3. Unloads the topic from memory
4. Releases resources

**Step 3: Target Receives Assignment**

The Load Manager's assignment logic sees the unassigned marker and:

1. Checks if target broker (67890) is active
2. If active → Assign to target broker (respecting hint)
3. If not active → Select alternative broker from rankings

**Step 4: Topic Loads on Target**

The target broker:

1. Receives assignment from ETCD watch
2. Loads topic metadata
3. Initializes subscriptions
4. Accepts producer reconnections
5. Begins serving traffic

**Step 5: Record Move in History**

The Load Manager records the move:

```json
Path: /cluster/rebalancing_history/{timestamp}
Value: {
  "topic": "/default/my-topic",
  "from_broker": 12345,
  "to_broker": 67890,
  "reason": "LoadImbalance",
  "estimated_load": 15.3,
  "timestamp": 1234567890
}
```

This history is used for:

- Rate limiting (counting moves in last hour)
- Cooldown enforcement (preventing rapid re-moves)
- Audit trail (troubleshooting and analysis)
- Metrics (tracking rebalancing activity)

**Step 6: Cooldown Delay**

After a successful move, the Load Manager waits for `cooldown_seconds` before the next move. This prevents:

- Rapid oscillations (topic bouncing between brokers)
- Metadata store overload (too many writes)
- Cluster instability (constant topic movements)

### Rebalancing Visualization

```bash
Imbalance Detected (CV > threshold)
       │
       ▼
  ┌──────────────────────────────────┐
  │  Select Topics to Move           │
  │                                  │
  │  Broker 1 (overloaded):         │
  │    ├─ topic-A: load = 5.2       │
  │    ├─ topic-B: load = 8.1       │
  │    └─ topic-C: load = 12.3      │
  │                                  │
  │  Select lightest: topic-A       │
  └──────────────────────────────────┘
       │
       ▼
  ┌──────────────────────────────────┐
  │  Select Target Broker            │
  │                                  │
  │  Rankings:                       │
  │    Broker 3: load = 20 ← target │
  │    Broker 2: load = 30          │
  │    Broker 1: load = 45          │
  └──────────────────────────────────┘
       │
       ▼
  ┌───────────────────────────────────┐
  │  Create Move Plan                 │
  │  topic-A: Broker 1 → Broker 3    │
  └───────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────┐
  │  Execute Move                │
  │  1. Create unassigned marker │
  │     with target hint         │
  │  2. Delete assignment        │
  │     from Broker 1            │
  └─────────────────────────────┘
       │
       ├──────────────────┬──────────────────┐
       ▼                  ▼                  ▼
  ┌──────────┐     ┌────────────┐     ┌──────────┐
  │ Broker 1 │     │Load Manager│     │ Broker 3 │
  │          │     │            │     │          │
  │ Unload   │     │ Assign to  │     │ Load     │
  │ topic-A  │     │ Broker 3   │     │ topic-A  │
  └──────────┘     └────────────┘     └──────────┘
       │                  │                  │
       └──────────────────┴──────────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Record in       │
                 │ History         │
                 │ Apply Cooldown  │
                 └─────────────────┘
```

### Safety Mechanisms

Automated rebalancing includes multiple safety mechanisms to prevent disruption:

**1. Single Move Per Cycle**  

Only one topic is moved per rebalancing check cycle. This prevents:

- Overshooting (moving too many topics at once)
- Cascading imbalances (fixing one problem creates another)
- Metadata store overload

**2. Rate Limiting**  
Maximum moves per hour enforced via history tracking. Prevents rebalancing storms during cluster instability.

**3. Cooldown Period**  
Topics can't be moved again within `cooldown_seconds` (default 60s). Prevents oscillations where topics bounce back and forth.

**4. Minimum Topic Age**  
Topics younger than `min_topic_age_seconds` (default 300s) are never moved. Allows topics to stabilize after creation before considering them for rebalancing.

**5. Blacklist**  
Critical topics can be permanently excluded from rebalancing using pattern matching.

**6. Leader-Only Execution**  
Only the elected cluster leader performs rebalancing. Prevents multiple brokers from executing conflicting moves.

**7. Active Broker Check**  
Target brokers must be in "active" state. Draining or drained brokers are excluded.

**8. Graceful Unload**  
Topic moves use the same graceful unload workflow as manual operations:

- Seal topic (no new producers)
- Drain in-flight messages
- Unload cleanly

**9. Metrics Logging**  
All rebalancing activity is logged with full context for troubleshooting and auditing.

---

## Configuration and Tuning

### Basic Configuration

Load Manager configuration in `config/danube_broker.yml`:

```yaml
load_manager:
  # How often brokers report load to ETCD
  load_report_interval_seconds: 30
  
  # Assignment strategy for new topics
  # Options: fair, balanced, weighted_load
  assignment_strategy: balanced
  
  # Automated rebalancing settings
  rebalancing:
    enabled: false                    # Start disabled, enable after testing
    aggressiveness: balanced          # conservative | balanced | aggressive
    check_interval_seconds: 300       # How often to check balance (5 min)
    max_moves_per_hour: 10            # Rate limit
    cooldown_seconds: 60              # Wait between moves
    min_brokers_for_rebalance: 2      # Need at least 2 brokers
    min_topic_age_seconds: 300        # Don't move young topics (5 min)
    blacklist_topics: []              # Topics to never rebalance
```

### Assignment Strategy Selection

| Scenario | Recommended Strategy | Reason |
|----------|---------------------|---------|
| Development/testing | `fair` | Simple, predictable, low overhead |
| Production (general) | `balanced` | Best balance of accuracy and overhead |
| Heterogeneous hardware | `balanced` | Accounts for CPU/memory differences |
| Uniform hardware | `fair` or `balanced` | Fair works well when brokers are identical |
| Variable workloads | `weighted_load` | Adapts to changing resource bottlenecks |
| High-throughput topics | `balanced` or `weighted_load` | Considers throughput in scoring |

### Aggressiveness Tuning

**Conservative** - Use when:

- Cluster is mostly stable
- Topic moves are expensive (large backlogs)
- Minimizing disruption is critical
- Resources are plentiful

**Balanced** - Use when:

- General production workloads
- Normal mix of topic sizes
- Moderate load variability
- Default choice for most clusters

**Aggressive** - Use when:

- Workloads change frequently
- Fast response to imbalance is needed
- Testing/development environments
- Cluster has capacity for frequent moves

### Load Report Interval Tuning

**Lower intervals (5-10s):**

- Faster response to load changes
- More accurate rankings
- Higher ETCD traffic
- Better for testing

**Higher intervals (30-60s):**

- Lower overhead
- Suitable for stable production clusters
- Slower response to changes
- Industry standard

**Recommendation:** Start with 30s, reduce only if needed for fast-changing workloads.

### Rate Limiting Tuning

The `max_moves_per_hour` setting prevents rebalancing storms:

**Conservative:**

- `max_moves_per_hour: 5`
- Maximum 5 topics moved per hour
- Best for large topics with backlogs

**Balanced:**

- `max_moves_per_hour: 10`
- Moderate rebalancing speed
- Good for most clusters

**Aggressive:**

- `max_moves_per_hour: 20`
- Faster rebalancing
- Suitable for test clusters or small topics

### Blacklist Patterns

Exclude critical topics from rebalancing:

```yaml
blacklist_topics:
  # System topics
  - "/system/*"
  
  # Critical low-latency topics
  - "/default/critical-*"
  
  # High-volume topics (expensive to move)
  - "/analytics/logs"
  - "/metrics/*"
  
  # Topics with specific placement requirements
  - "/pinned/special-topic"
```

---

## Metrics and Monitoring

### Load Manager Metrics

The Load Manager exposes Prometheus metrics for observability:

**Cluster Health Metrics:**

```bash
danube_cluster_balance_cv            # Current coefficient of variation
danube_cluster_balance_mean_load     # Mean broker load
danube_cluster_balance_max_load      # Max broker load
danube_cluster_balance_min_load      # Min broker load
danube_cluster_balance_std_dev       # Standard deviation
danube_cluster_active_brokers        # Count of active brokers
```

**Rebalancing Metrics:**

```bash
danube_rebalancing_checks_total      # Total balance check cycles
danube_rebalancing_moves_total       # Total topic moves executed
danube_rebalancing_failures_total    # Failed move attempts
danube_rebalancing_cycle_duration_seconds  # Duration of rebalancing cycles
```

**Topic Assignment Metrics:**

```bash
danube_topic_assignments_total       # Total topic assignments
  labels: broker_id, action (assign/unassign)
```

**Broker Load Metrics:**

```bash
danube_broker_topics_owned           # Topics per broker
danube_broker_cpu_usage              # CPU percentage
danube_broker_memory_usage           # Memory percentage
danube_broker_throughput_mbps        # Aggregate throughput
```

---

## Use Cases and Best Practices

### Scenario 1: Adding Brokers to a Cluster

**Situation:**  
You start with 3 brokers, each handling 100 topics. You add 2 new brokers to scale capacity.

**Without Rebalancing:**

- New brokers sit idle (0 topics each)
- Old brokers remain at 100 topics
- Cluster is imbalanced (CV = 100%)
- New capacity is wasted

**With Rebalancing:**

1. New brokers join and register
2. Load Manager detects imbalance (CV > threshold)
3. Selects 40 lightest topics from old brokers
4. Moves them to new brokers over time (rate limited)
5. Final state: 60 topics per broker (balanced)
6. CV drops to < 10%

**Best Practice:**

- Enable rebalancing before adding brokers
- Use aggressive mode temporarily for faster rebalancing
- Monitor rebalancing progress via metrics
- Return to balanced mode after completion

### Scenario 2: Handling Broker Failure

**Situation:**  
A broker crashes, and its 150 topics need reassignment.

**Without Load Manager:**

- Manual intervention required
- Topics go offline until reassigned
- Risk of creating new hotspots

**With Load Manager:**

1. Leader detects broker deregistration
2. Deletes all assignments from failed broker
3. Creates 150 unassigned markers
4. Assignment workflow distributes topics across remaining brokers
5. Topics reassigned in < 1 minute

**Best Practice:**

- Ensure at least 3 brokers for redundancy
- Use balanced or weighted_load strategy for failover
- Monitor failover metrics
- Set appropriate `min_brokers_for_rebalance`

### Scenario 3: Workload Changes Over Time

**Situation:**  
Topics created in the morning (low traffic) grow to high traffic in the afternoon, creating imbalance.

**Without Rebalancing:**

- Initial assignment is fair
- As traffic grows, some brokers become hotspots
- Performance degrades on overloaded brokers

**With Rebalancing:**

1. Load reports show increasing CPU/throughput on some brokers
2. CV rises above threshold
3. Rebalancing moves high-traffic topics to underloaded brokers
4. Cluster remains balanced throughout the day

**Best Practice:**

- Use balanced or weighted_load strategy (accounts for throughput)
- Enable rebalancing for dynamic workloads
- Set check_interval to match workload variability (5-10 minutes)
- Use balanced aggressiveness

### Scenario 4: Heterogeneous Hardware

**Situation:**  
Cluster has mix of hardware: 2 high-memory brokers, 3 standard brokers.

**Without Intelligence:**

- Fair strategy distributes topics evenly
- High-memory brokers are underutilized
- Standard brokers may run out of memory

**With Balanced Strategy:**

- Memory usage is factored into scoring (35% weight)
- High-memory brokers receive more topics
- Standard brokers stay within capacity
- Resources are optimized

**Best Practice:**

- Always use balanced or weighted_load for heterogeneous clusters
- Monitor per-broker resource utilization
- Adjust load_report_interval if needed (lower = more responsive)

---

## Troubleshooting

### Rebalancing Not Triggering

**Symptoms:**

- CV is high but no moves occur
- Logs show "cluster is balanced"

**Possible Causes:**

1. Rebalancing disabled: Check `rebalancing.enabled: true`
2. Not leader: Only leader executes rebalancing
3. CV below threshold: Increase aggressiveness or lower threshold
4. Too few brokers: Check `min_brokers_for_rebalance`
5. All topics blacklisted: Review blacklist patterns

### Topics Moving Too Frequently

**Symptoms:**

- Topics move multiple times per hour
- Cluster never stabilizes
- High metadata store traffic

**Possible Causes:**

1. Cooldown too short: Increase `cooldown_seconds`
2. Threshold too low: Use less aggressive mode
3. Topic age check disabled: Set `min_topic_age_seconds > 0`
4. Workload is truly unstable

### Imbalance After Rebalancing

**Symptoms:**

- Rebalancing executes but CV remains high
- Moves don't reduce imbalance

**Possible Causes:**

1. Moving wrong topics (heavy topics not moving)
2. New traffic creating imbalance faster than moves fix it
3. Assignment strategy doesn't match rebalancing goals

**Solutions:**

- Review topic load scores (ensure heaviest topics are candidates)
- Increase `max_moves_per_hour`
- Switch to weighted_load strategy
- Check for blacklisted heavy topics

---

## Summary

The [Danube](https://github.com/danube-messaging/danube) Load Manager is a sophisticated system that keeps broker clusters healthy and balanced automatically. By combining intelligent topic assignment with proactive rebalancing, it ensures:

- **Optimal resource utilization** - No broker is overloaded or idle
- **High availability** - Automatic failover when brokers crash
- **Performance consistency** - Even load distribution prevents hotspots
- **Operational simplicity** - Self-optimizing cluster reduces manual intervention
- **Scalability** - Easy to add/remove brokers without manual rebalancing

The system is production-ready with multiple safety mechanisms, configurable strategies, and comprehensive monitoring. Start with defaults (balanced strategy, disabled rebalancing) and gradually tune based on your workload characteristics.
