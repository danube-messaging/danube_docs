# Getting Started with danube-admin

The unified administration tool for Danube cluster management.

## Overview

`danube-admin` is a single binary that provides three powerful interfaces for managing your Danube cluster:

1. **CLI Mode** - Command-line interface for scripts, automation, and quick operations
2. **HTTP Server Mode** - REST API for application integrations and web UI
3. **MCP Mode** - AI-native interface for natural language cluster management

All three modes share the same core functionality, giving you flexibility in how you interact with your cluster.

---

## Installation

### Download Pre-built Binaries

Download the latest release for your platform from [GitHub Releases](https://github.com/danube-messaging/danube/releases):

**Linux:**

```bash
curl -L -o danube-admin https://github.com/danube-messaging/danube/releases/latest/download/danube-admin-linux
chmod +x danube-admin
sudo mv danube-admin /usr/local/bin/
```

**macOS (Apple Silicon):**

```bash
curl -L -o danube-admin https://github.com/danube-messaging/danube/releases/latest/download/danube-admin-macos
chmod +x danube-admin
sudo mv danube-admin /usr/local/bin/
```

**Windows:**  
Download `danube-admin-windows.exe` from the releases page and add it to your PATH.

### Docker

```bash
docker pull ghcr.io/danube-messaging/danube-admin:latest
```

### Verify Installation

```bash
danube-admin --version
```

---

## Mode 1: CLI Mode

The CLI mode provides traditional command-line access to all cluster management functions.

### When to Use

- **Automation & Scripts** - CI/CD pipelines, deployment scripts, cron jobs
- **Quick Operations** - One-off administrative tasks, troubleshooting
- **Terminal Workflows** - SSH sessions, infrastructure as code
- **Human-Readable Output** - Formatted tables and lists for easy scanning

### Basic Usage

```bash
# Set broker endpoint (required)
export DANUBE_BROKER_ENDPOINT=http://localhost:50051

# List all brokers
danube-admin brokers list

# Create a topic
danube-admin topics create /default/events --partitions 4

# Register a schema
danube-admin schemas register user-events \
  --schema-type json_schema \
  --file schema.json
```

### Configuration

**Environment Variables:**

```bash
# Broker admin endpoint (required)
export DANUBE_BROKER_ENDPOINT=http://localhost:50051

# Optional: Default namespace
export DANUBE_NAMESPACE=default

# Optional: Output format
export DANUBE_OUTPUT_FORMAT=json
```

**Command-line Flags:**

```bash
# Specify broker endpoint
danube-admin --broker-endpoint http://broker1:50051 brokers list

# JSON output for automation
danube-admin brokers list --output json

# Help for any command
danube-admin topics create --help
```

### Available Commands

- **brokers** - Manage cluster brokers (list, unload, activate)
- **namespaces** - Manage namespaces (create, delete, list topics)
- **topics** - Manage topics (create, describe, delete, subscriptions)
- **schemas** - Manage schema registry (register, get, check compatibility)
- **cluster** - Cluster operations (balance, rebalance, health)

See individual command documentation for detailed usage.

---

## Mode 2: MCP Mode (AI-Native)

The MCP (Model Context Protocol) mode enables AI assistants to manage your cluster using natural language.

### When to Use

- **AI-Assisted Operations** - Manage clusters through conversation with Claude, Windsurf, or VSCode
- **Exploratory Analysis** - Ask questions about cluster state and health
- **Guided Workflows** - Multi-step operations with AI guidance
- **Learning & Discovery** - Understand cluster behavior through AI explanations

### Start MCP Server

```bash
# For AI IDE integration (stdio mode)
danube-admin serve --mode mcp \
  --broker-endpoint http://localhost:50051
```

### Configure in AI IDEs

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "danube-admin": {
      "command": "/usr/local/bin/danube-admin",
      "args": ["serve", "--mode", "mcp", "--broker-endpoint", "http://localhost:50051"],
      "env": {
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

**Windsurf** (`~/.codeium/windsurf/mcp_config.json`):

```json
{
  "mcpServers": {
    "danube-admin": {
      "command": "sh",
      "args": [
        "-c",
        "/usr/local/bin/danube-admin serve --mode mcp --broker-endpoint http://localhost:50051"
      ],
      "env": {
        "PATH": "/usr/local/bin:/usr/bin:/bin",
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

**VSCode** (with Continue or similar MCP extension):

```json
{
  "servers": {
    "danube-admin": {
      "type": "stdio",
      "command": "/usr/local/bin/danube-admin",
      "args": ["serve", "--mode", "mcp", "--broker-endpoint", "http://localhost:50051"],
      "env": {
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

### Using MCP Mode

Once configured, interact with your cluster using natural language:

```bash
You: "List all brokers in my Danube cluster"
AI: [Calls list_brokers tool and presents results]

You: "Check if the cluster is balanced"
AI: [Calls get_cluster_balance and analyzes metrics]

You: "Create a new topic for user events with 4 partitions"
AI: [Calls create_topic with appropriate parameters]
```

See the [MCP Mode documentation](ai_admin_assistant.md) for detailed information about available tools and guided workflows.

---

## Mode 3: HTTP Server Mode

The HTTP server mode exposes a REST API that powers the Danube Admin UI.

### When to Use

- **Web UI** - Visual cluster management through [Danube Admin UI](https://github.com/danube-messaging/danube-admin-ui)
- **Monitoring Dashboards** - Query cluster state for custom dashboards

### Start the Server

```bash
# Basic server
danube-admin serve --mode http --port 8080 \
  --broker-endpoint http://localhost:50051

# With all options
danube-admin serve \
  --mode http \
  --port 8080 \
  --broker-endpoint http://localhost:50051 \
  --host 0.0.0.0
```

### Example API Usage

```bash
# List brokers
curl http://localhost:8080/api/brokers

# Create topic
curl -X POST http://localhost:8080/api/topics \
  -H "Content-Type: application/json" \
  -d '{
    "name": "/default/events",
    "partitions": 4,
    "dispatch_strategy": "reliable"
  }'

# Get cluster balance
curl http://localhost:8080/api/cluster/balance
```

### Danube Admin UI

The [Danube Admin UI](https://github.com/danube-messaging/danube-admin-ui) is a modern React-based web interface that connects to the HTTP server:

1. Start the HTTP server
2. Configure the UI to point to `http://localhost:8080`
3. Access visual cluster management, monitoring, and operations

---

## Common Usage

### 1. CLI Mode

```bash
# Set your broker endpoint
export DANUBE_BROKER_ENDPOINT=http://localhost:50051

# Check cluster status
danube-admin brokers list
danube-admin brokers leader

# Create a namespace
danube-admin namespaces create production

# Create your first topic
danube-admin topics create /production/events --partitions 3

# Verify
danube-admin topics describe /production/events
```

### 2. UI Mode

```bash
# Start HTTP server
danube-admin serve --mode http --port 8080 \
  --broker-endpoint http://localhost:50051

# Access UI at http://localhost:8080 (if UI is deployed)
# Or use API directly for custom dashboards
```

### 3. AI-Assisted Management (MCP Mode)

```bash
# Configure MCP in your AI IDE (one-time setup)
# Then interact naturally:

"Show me the cluster health"
"What topics are in the default namespace?"
"Help me set up a new topic with schema validation"
"Why is broker-001 handling more topics than broker-002?"
```

---

## Configuration File for MCP (Optional)

For advanced features like log access and metrics, create `mcp-config.yml`:

```yaml
broker_endpoint: http://localhost:50051
prometheus_url: http://localhost:9090

deployment:
  type: docker
  docker:
    container_mappings:
      - id: "broker1"
        container: "danube-broker1"
      - id: "broker2"
        container: "danube-broker2"
```

Use the config file:

```bash
# MCP mode
danube-admin serve --mode mcp --config /path/to/mcp-config.yml
```

---

## Next Steps

- **CLI Commands**: See [Brokers](./admin_cli/brokers.md), [Topics](./admin_cli/topics.md), [Namespaces](./admin_cli/namespaces.md), [Schema Registry](./admin_cli/schema_registry.md)
- **Try MCP Mode**: Read the [MCP documentation](./ai_admin_assistant.md) for AI-assisted operations
- **Deploy Web UI**: Set up [Danube Admin UI](https://github.com/danube-messaging/danube-admin-ui) for visual management
