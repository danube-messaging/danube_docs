# MCP Mode: AI-Native Cluster Management

Enable AI assistants to manage your Danube cluster using natural language through the Model Context Protocol (MCP).

## Overview

The MCP (Model Context Protocol) mode transforms cluster management from command memorization to natural conversation. Instead of learning CLI syntax, you describe what you want in plain English, and the AI executes the appropriate operations using specialized tools.

### What is Model Context Protocol?

MCP is an open protocol developed by Anthropic that enables AI assistants to interact with external systems through:

- **Tools** - Direct function calls the AI can invoke automatically
- **Prompts** - Pre-built multi-step workflows that guide complex operations
- **Resources** - Contextual information about the system state

Danube's MCP server exposes **40+ tools** and **7 guided prompts** specifically designed for cluster management.

---

## Getting Started

### Prerequisites

- Danube cluster running and accessible
- `danube-admin` binary installed
- AI IDE that supports MCP (Claude Desktop, Windsurf, VSCode with Continue extension)

### Start the MCP Server

```bash
danube-admin serve --mode mcp --broker-endpoint http://localhost:50051
```

### Configure Your AI IDE

See the [Getting Started guide](getting_started.md#mode-2-mcp-mode-ai-native) for detailed configuration instructions for:

- Claude Desktop
- Windsurf
- VSCode

---

## Usage Examples

### Basic Operations

**Check Cluster Health:**

```bash
You: "Check the health of my Danube cluster"
AI: [Calls health_check tool]
    "The cluster has 2 active brokers, leader is broker-001, 
     balance CV is 0.15 (well balanced), 8 topics, 12 consumers..."
```

**List Topics:**

```bash
You: "What topics are in the default namespace?"
AI: [Calls list_topics with namespace=default]
    "The default namespace has 5 topics:
     /default/user-events, /default/orders..."
```

**Create Topic:**

```bash
You: "Create a topic for user events with 4 partitions and reliable delivery"
AI: [Calls create_topic with appropriate parameters]
    "✅ Topic created: /default/user-events
     Partitions: 4, Delivery: reliable"
```

---

### Using Guided Prompts

**From Prompt Picker:**

1. Open prompt picker in your AI IDE
2. Select "Setup New Topic"
3. Fill arguments: namespace="production", use_case="high-throughput"
4. Click "Use Prompt"
5. AI follows multi-step workflow automatically

**Via Natural Language:**

```bash
You: "Run the cluster health check workflow"
AI: [Loads cluster_health_check prompt]
    "🏥 Danube Cluster Health Check Report
     
     Step 1: Cluster Overview
     - 2 active brokers
     - 8 topics, 12 consumers, 7 subscriptions
     
     Step 2: Broker Health ✅
     - All brokers online: 2/2 active
     - Leader: broker-001
     
     Step 3: Load Distribution ✅
     - Well balanced (CV: 0.15)
     
     ..."
```

---

### Advanced Workflows

**Schema Evolution:**

```bash
You: "I need to add a new field to the user-events schema. 
      Can you help me evolve it safely?"
AI: [Uses manage_schema_evolution prompt]
    "I'll guide you through safe schema evolution:
     
     Step 1: Current Schema Analysis
     [Calls get_schema for user-events]
     Current version: 2, Compatibility: BACKWARD
     
     Step 2: Compatibility Check
     Please provide the new schema definition...
     [After receiving schema]
     [Calls check_schema_compatibility]
     ✅ New schema is backward compatible
     
     Step 3: Registration
     [Calls register_schema]
     ✅ New version 3 registered
     
     Step 4: Verification
     [Calls describe_topic for topics using this schema]
     All topics updated successfully"
```

**Load Rebalancing:**

```bash
You: "The cluster seems imbalanced, can you fix it?"
AI: [Uses prepare_cluster_rebalance prompt]
    "Let me check and rebalance the cluster:
     
     Step 1: Current Balance
     [Calls get_cluster_balance]
     CV: 0.45 (severely imbalanced)
     Broker 1: 8 topics, Broker 2: 2 topics
     
     Step 2: Dry-Run Preview
     [Calls trigger_rebalance with dry_run=true]
     Proposed moves: 3 topics from broker1 to broker2
     
     Step 3: Execute Rebalancing
     [Calls trigger_rebalance with dry_run=false]
     ✅ Moved 3 topics
     
     Step 4: Verification
     [Calls get_cluster_balance again]
     New CV: 0.15 (well balanced) ✅"
```

---

## Configuration

### Basic Configuration

Minimum required: just specify the broker endpoint when starting the MCP server.

```bash
danube-admin serve --mode mcp --broker-endpoint http://localhost:50051
```

### Advanced Configuration

For log access and metrics, create `mcp-config.yml`:

```yaml
# Broker admin endpoint
broker_endpoint: http://localhost:50051

# Prometheus for metrics (optional)
prometheus_url: http://localhost:9090

# Deployment configuration for log access (optional)
deployment:
  type: docker  # or: kubernetes, local
  docker:
    container_mappings:
      - id: "broker1"
        container: "danube-broker1"
      - id: "broker2"
        container: "danube-broker2"
```

**Deployment Types:**

- **docker**: Accesses logs via `docker logs` command
- **kubernetes**: Accesses logs via `kubectl logs` command
- **local**: Accesses log files from local filesystem

Use the config file:

```bash
danube-admin serve --mode mcp --config /path/to/mcp-config.yml
```

---

## Best Practices

### 1. Discovery Before Action

Let the AI gather context before making changes:

```bash
❌ "Delete the old-topic"
✅ "What subscriptions are on old-topic? Then help me safely delete it"
```

### 2. Use Prompts for Complex Operations

For multi-step workflows, use prompts instead of individual commands:

```bash
❌ "Check balance, then rebalance if needed"
✅ [Use prepare_cluster_rebalance prompt]
```

### 3. Ask for Explanations

MCP mode understands context - ask why, not just what:

```bash
✅ "Why does broker-001 have more topics than broker-002?"
✅ "What's the best partitioning strategy for my use case?"
✅ "Should I use backward or full compatibility for this schema?"
```

### 4. Dry-Run First

For operations with side effects, ask the AI to preview first:

```bash
✅ "Show me what would happen if I rebalance the cluster"
✅ "Check if this schema change is compatible before registering"
```

### 5. Iterative Troubleshooting

Let the AI guide investigation:

```bash
You: "Consumers are slow on /default/events"
AI: [Checks topic metrics, broker load, subscription lag]
    "I see high lag. Let me check if it's a broker issue..."
    [Continues investigation based on findings]
     Step 1: Current Balance
     [Calls get_cluster_balance]
     CV: 0.45 (severely imbalanced)
     Broker 1: 8 topics, Broker 2: 2 topics
     
     Step 2: Dry-Run Preview
     [Calls trigger_rebalance with dry_run=true]
     Proposed moves: 3 topics from broker1 to broker2
     
     Step 3: Execute Rebalancing
     [Calls trigger_rebalance with dry_run=false]
     ✅ Moved 3 topics
     
     Step 4: Verification
     [Calls get_cluster_balance again]
     New CV: 0.15 (well balanced) ✅"
```

---

## Available Tools

Tools are categorized by function. The AI automatically selects and calls these tools based on your natural language requests.

### Cluster Management (9 tools)

Manage brokers, namespaces, and cluster-wide operations.

**list_brokers**  
List all brokers with their status, roles (leader/follower), and network addresses. Use this first to discover broker IDs.

**get_leader**  
Identify the current cluster leader broker responsible for coordination and metadata management.

**get_cluster_balance**  
Get load distribution metrics across brokers. Returns Coefficient of Variation (CV) where <20% = well balanced, >40% = severely imbalanced.

**trigger_rebalance**  
Redistribute topics across brokers to balance load. Always use `dry_run=true` first to preview moves. Supports `max_moves` parameter to limit scope.

**unload_broker**  
Gracefully unload all topics from a broker for maintenance or decommissioning. Supports dry-run mode, namespace filtering, and parallel unloading.

**activate_broker**  
Bring a broker back into service after maintenance. Changes broker state to active for topic assignment.

**list_namespaces**  
List all namespaces in the cluster. Namespaces provide logical isolation for topics and policies.

**get_namespace_policies**  
Get resource limits and policies for a namespace (max producers/consumers, rate limits, message size limits).

**create_namespace**  
Create a new namespace for organizing topics by environment, team, or application.

**delete_namespace**  
Delete an empty namespace. Fails if any topics exist in the namespace.

---

### Topic Management (8 tools)

Create, configure, and manage topics.

**list_topics**  
List all topics in a namespace with broker assignment and delivery strategy.

**describe_topic**  
Get detailed topic information including broker assignment, delivery strategy, schema configuration, and active subscriptions.

**create_topic**  
Create a new topic with optional partitioning and schema validation. Topic name format: `/namespace/topic-name`. Delivery strategies: `reliable` (durable) or `non_reliable` (fast).

**delete_topic**  
Permanently delete a topic and all its data. Irreversible operation - use with caution.

**list_subscriptions**  
List all active subscriptions (consumer groups) for a topic.

**unsubscribe**  
Remove a specific subscription from a topic, disconnecting consumers using that subscription.

**unload_topic**  
Unload a topic from its current broker for reassignment. Brief disruption during transfer.

**configure_topic_schema**  
Associate a schema subject with a topic and set validation policy (`none`, `warn`, `enforce`). Requires schema to exist in registry.

**set_topic_validation_policy**  
Update validation policy without changing schema subject. Useful for gradually enforcing validation.

**get_topic_schema_config**  
Retrieve current schema configuration for a topic including subject, validation policy, and cached schema ID.

---

### Schema Registry (7 tools)

Manage schemas for data validation and evolution.

**register_schema**  
Register a new schema or create a new version of an existing subject. Automatically checks compatibility. Supported types: `json_schema`, `avro`, `protobuf`, `string`, `number`, `bytes`.

**get_schema**  
Get the latest version of a schema by subject name. Returns schema definition, ID, version, type, and compatibility mode.

**list_schema_versions**  
List all versions of a schema subject with version numbers, schema IDs, timestamps, creators, and descriptions.

**check_schema_compatibility**  
Validate if a new schema is compatible with existing versions before registration. Prevents breaking changes.

**get_schema_compatibility_mode**  
Get the current compatibility mode for a subject: `NONE`, `BACKWARD`, `FORWARD`, or `FULL`.

**set_schema_compatibility_mode**  
Set compatibility mode for future schema versions. Controls how strictly evolution is enforced.

**delete_schema_version**  
Delete a specific schema version. Use with caution - may break topics or consumers using this version.

---

### Diagnostics (3 tools)

Health checks and troubleshooting assistance.

**health_check**  
Comprehensive cluster health assessment. Returns broker status, leader election, cluster balance metrics, and detected issues.

**analyze_consumer_lag**  
Analyze consumer lag for a topic and subscription. Returns diagnostic data about topic, subscription status, and cluster balance.

**get_recommendations**  
Get prioritized recommendations for load imbalance, overloaded brokers, and high availability concerns based on current cluster state.

---

### Log Access (6 tools)

Access broker logs from Docker, Kubernetes, or local deployments.

> **Note**: `get_broker_logs` and `list_configured_brokers` require `mcp-config.yml` with deployment configuration. Other log tools work without config.

**get_broker_logs** *(Config Required)*  
Get logs from a specific broker by ID. Requires deployment configuration in `mcp-config.yml`.

**list_configured_brokers** *(Config Required)*  
List brokers configured in the MCP config file.

**list_docker_containers** *(No Config Required)*  
List all running Docker containers on the host. Use to discover container names.

**fetch_container_logs** *(No Config Required)*  
Fetch logs directly from any Docker container by name.

**list_k8s_pods** *(No Config Required)*  
List Kubernetes pods in a namespace.

**fetch_pod_logs** *(No Config Required)*  
Fetch logs directly from a Kubernetes pod.

---

### Metrics (4 tools)

Query Prometheus metrics for cluster monitoring.

> **Note**: All metrics tools require Prometheus to be running and accessible. Configure `prometheus_url` in `mcp-config.yml` (default: `http://localhost:9090`).

**get_cluster_metrics** *(Prometheus Required)*  
Get cluster-wide metrics summary: broker count, topics, producers, consumers, message rates, throughput, and balance CV.

**get_broker_metrics** *(Prometheus Required)*  
Get metrics for a specific broker: topics owned, RPC count, active producers/consumers, bytes in/out.

**get_topic_metrics** *(Prometheus Required)*  
Get comprehensive topic metrics: message/byte counters, active connections, publish/dispatch rates, lag, and latency percentiles (p50/p95/p99).

**query_prometheus** *(Prometheus Required)*  
Execute raw PromQL queries for advanced metric analysis. Full PromQL syntax supported.

---

## Guided Workflow Prompts

Prompts are pre-built multi-step workflows that guide the AI through complex operations. Access them through your AI IDE's prompt picker.

### Troubleshooting Prompts

**diagnose_consumer_lag**  
*Arguments: topic (required), subscription (optional)*

Step-by-step guide to diagnose why consumers are lagging. Checks topic existence, subscription status, broker assignment, and cluster balance to identify root causes.

**diagnose_broker_issues**  
*Arguments: broker_id (optional)*

Investigate broker health, load distribution, and potential problems. Analyzes specific broker or all brokers if ID omitted.

**analyze_topic_performance**  
*Arguments: topic (required)*

Deep dive into topic metrics including throughput, latency, and errors. Correlates metrics with broker load and cluster state.

**cluster_health_check**  
*Arguments: none*

Comprehensive cluster health assessment with actionable recommendations. Checks broker status, leader election, load balance, and provides prioritized action items.

---

### Operational Prompts

**setup_new_topic**  
*Arguments: namespace (required), use_case (optional)*

Guided workflow to create a new topic with best-practice configuration. Verifies namespace exists, recommends partitioning and delivery strategy based on use case (`high-throughput`, `low-latency`, `persistent`), and validates setup.

**manage_schema_evolution**  
*Arguments: subject (required), schema_type (optional)*

Safe workflow for evolving schemas with compatibility validation. Checks current compatibility mode, validates new schema compatibility, registers new version, and verifies topics picked up the change.

**prepare_cluster_rebalance**  
*Arguments: max_moves (optional), target_broker (optional)*

Safe rebalancing workflow with dry-run validation. Checks current balance, previews moves, executes rebalancing with optional limits, and validates results.

---

## Resources

Resources provide contextual information about the cluster. These are read-only data sources the AI can access.

**danube://cluster/config**  
Cluster configuration including broker list, namespaces, and global settings.

**danube://cluster/metrics**  
Real-time cluster metrics snapshot (if Prometheus is configured).

---

## Limitations

### What MCP Can Do

✅ Read cluster state (brokers, topics, schemas, metrics)  
✅ Create/delete topics and namespaces  
✅ Register and evolve schemas  
✅ Trigger rebalancing and maintenance operations  
✅ Access logs and metrics  
✅ Provide guided workflows and recommendations  

### What MCP Cannot Do

❌ Publish or consume messages (use `danube-cli` for that)  
❌ Modify broker configuration (requires broker restart)  
❌ Access historical data beyond Prometheus retention  
❌ Execute arbitrary shell commands on brokers  

---

## Next Steps

- Read the [Getting Started guide](./getting_started.md) to configure MCP in your AI IDE
- Explore the [AI-Native Messaging article](https://dev-state.com/posts/ai_native_messaging_with_danube/) for background and examples
- Try the troubleshooting prompts to get familiar with guided workflows
