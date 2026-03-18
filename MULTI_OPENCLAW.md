# ClawMetry Multi-OpenClaw Support

This fork of ClawMetry adds support for monitoring multiple OpenClaw instances from a single dashboard.

## Features

- **Node Identification**: Each OpenClaw instance is identified by its OTLP resource attributes
- **Per-Node Metrics**: Track tokens, costs, runs, and messages per OpenClaw instance
- **Node Selector UI**: Filter dashboard views by specific OpenClaw instance
- **Auto-Discovery**: OpenClaw nodes are automatically registered when they send metrics
- **Status Tracking**: See which nodes are online/offline based on recent activity

## Quick Start

### 1. Configure OpenClaw Instances

Each OpenClaw instance needs to send OTLP metrics with a unique identifier. Set these environment variables:

```bash
# Required: OTLP endpoint pointing to ClawMetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://clawmetry:8900
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Required: Node identification (use one of these)
OTEL_RESOURCE_ATTRIBUTES=openclaw.node=my-instance-name
# OR
OTEL_RESOURCE_ATTRIBUTES=service.name=openclaw-my-instance
```

### 2. Docker Compose Example

```yaml
services:
  clawmetry:
    image: python:3.12-slim
    command: bash -c "pip install clawmetry && clawmetry --host 0.0.0.0"
    ports:
      - "8900:8900"

  openclaw-prod:
    image: openclaw/openclaw:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://clawmetry:8900
      - OTEL_RESOURCE_ATTRIBUTES=openclaw.node=prod

  openclaw-staging:
    image: openclaw/openclaw:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://clawmetry:8900
      - OTEL_RESOURCE_ATTRIBUTES=openclaw.node=staging
```

### 3. View in Dashboard

1. Open ClawMetry dashboard (http://localhost:8900)
2. The node selector appears in the navigation bar when multiple nodes are detected
3. Select a specific node to filter metrics, or view all nodes combined

## API Endpoints

### OpenClaw Nodes

```bash
# List all OpenClaw instances
GET /api/nodes/openclaw

# Get specific node details
GET /api/nodes/openclaw/<node_id>

# Get list of node IDs
GET /api/nodes/openclaw/ids
```

### Metrics with Node Filtering

```bash
# Get overview metrics (optionally filtered by node)
GET /api/overview?node_id=openclaw-prod

# Get usage/metrics data
GET /api/history/metrics?node_id=openclaw-prod
```

### OTLP Status (includes node info)

```bash
GET /api/otel-status
```

Returns:
```json
{
  "available": true,
  "hasData": true,
  "lastReceived": 1234567890,
  "counts": { "tokens": 100, "cost": 100, ... },
  "nodes": {
    "total": 2,
    "online": 2,
    "ids": ["openclaw-prod", "openclaw-staging"]
  }
}
```

## Node ID Priority

When extracting the node identifier from OTLP data, ClawMetry uses this priority:

1. `openclaw.node` - Explicit identifier (recommended)
2. `service.name` - If it starts with "openclaw"
3. `service.instance.id` - Fallback
4. `"default"` - If none of the above are present

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | ClawMetry OTLP endpoint | - |
| `OTEL_RESOURCE_ATTRIBUTES` | Resource attributes including node ID | - |

### OpenClaw Node Attributes

Add these to `OTEL_RESOURCE_ATTRIBUTES` (comma-separated):

- `openclaw.node=your-node-name` - Explicit node identifier
- `service.name=openclaw-prod` - Service name (if starts with "openclaw")
- `service.instance.id=instance-1` - Instance identifier

## Monitoring Multiple Environments

### Production Setup

```yaml
# docker-compose.prod.yml
services:
  openclaw:
    environment:
      - OTEL_RESOURCE_ATTRIBUTES=openclaw.node=prod,deployment.environment=production
```

### Staging Setup

```yaml
# docker-compose.staging.yml
services:
  openclaw:
    environment:
      - OTEL_RESOURCE_ATTRIBUTES=openclaw.node=staging,deployment.environment=staging
```

## Node Status

- **Online**: Node has sent metrics within the last 5 minutes
- **Offline**: No metrics received for 5+ minutes

The dashboard updates node status every 30 seconds automatically.

## Backward Compatibility

- Single OpenClaw setups work unchanged (metrics appear under "default" node)
- Existing metrics without node_id are treated as "default" node
- Fleet API for ClawMetry-to-ClawMetry communication remains unchanged

## Troubleshooting

### Node Not Appearing

1. Check OTLP endpoint is correct: `echo $OTEL_EXPORTER_OTLP_ENDPOINT`
2. Verify resource attributes are set: `echo $OTEL_RESOURCE_ATTRIBUTES`
3. Check ClawMetry logs for incoming metrics

### Metrics Not Filtered

1. Ensure `openclaw.node` or `service.name` is set in resource attributes
2. Check `/api/nodes/openclaw` to see registered nodes
3. Verify the node selector shows your nodes

## Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  OpenClaw Prod  │     │ OpenClaw Stage  │     │  OpenClaw Dev   │
│  node_id=prod   │     │ node_id=staging │     │   node_id=dev   │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │    OTLP (metrics)     │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │      ClawMetry         │
                    │   ┌────────────────┐   │
                    │   │  Node Selector │   │
                    │   └────────────────┘   │
                    │   ┌────────────────┐   │
                    │   │ Filtered Views │   │
                    │   └────────────────┘   │
                    └────────────────────────┘
```
