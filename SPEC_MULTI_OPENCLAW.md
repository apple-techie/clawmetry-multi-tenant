# ClawMetry Multi-OpenClaw Support Specification

## Overview

Enable ClawMetry to track metrics from multiple OpenClaw instances, with per-node visibility and filtering in the dashboard.

## Problem Statement

Currently, when multiple OpenClaw containers send metrics to a single ClawMetry via OTLP, all metrics are pooled together with no way to distinguish which OpenClaw instance sent what data.

## Solution Architecture

### 1. Node Identification via OTLP Resource Attributes

Each OpenClaw instance sends a unique identifier via OTLP resource attributes:

```yaml
# OpenClaw OTLP config (per instance)
OTEL_RESOURCE_ATTRIBUTES: "service.name=openclaw-prod,openclaw.node=prod-instance"
```

ClawMetry extracts and stores this identifier with each metric.

### 2. Data Model Changes

**Current metric entry:**
```json
{
  "timestamp": 1234567890,
  "input": 1000,
  "output": 500,
  "model": "claude-sonnet-4",
  "channel": "telegram",
  "provider": "anthropic"
}
```

**New metric entry with node_id:**
```json
{
  "timestamp": 1234567890,
  "input": 1000,
  "output": 500,
  "model": "claude-sonnet-4",
  "channel": "telegram",
  "provider": "anthropic",
  "node_id": "openclaw-prod"
}
```

### 3. Node Registry

Extend the existing fleet infrastructure:

- Use `openclaw_nodes` table for OpenClaw instances (separate from ClawMetry fleet nodes)
- Auto-register nodes when first metric is received
- Track: node_id, name, first_seen, last_seen, status

### 4. API Changes

**New/Modified Endpoints:**

```
GET /api/metrics/tokens?node_id=openclaw-prod
GET /api/metrics/cost?node_id=openclaw-prod
GET /api/nodes/openclaw  # List OpenClaw instances
GET /api/nodes/openclaw/<node_id>  # Node details
```

**OTLP Resource Attribute Extraction:**

Priority order for node_id:
1. `openclaw.node` (explicit identifier)
2. `service.name` (if starts with "openclaw")
3. `service.instance.id`
4. Fallback: "default"

### 5. UI Changes

**Node Selector Component:**
- Dropdown in dashboard header
- "All Nodes" option (default)
- Individual node selection
- Node status indicator (online/offline)

**Per-Node Views:**
- Overview cards show aggregated or per-node stats
- Charts can be filtered by node
- Fleet tab shows all OpenClaw instances

### 6. Docker Compose Configuration

```yaml
services:
  openclaw-prod:
    image: openclaw/openclaw:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://clawmetry:8900
      - OTEL_RESOURCE_ATTRIBUTES=service.name=openclaw-prod,openclaw.node=prod
  
  openclaw-staging:
    image: openclaw/openclaw:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://clawmetry:8900
      - OTEL_RESOURCE_ATTRIBUTES=service.name=openclaw-staging,openclaw.node=staging
```

## Implementation Plan

1. **Phase 1: Core Changes**
   - Modify `_process_otlp_metrics()` to extract node_id
   - Update `_add_metric()` to accept node_id
   - Add node_id to all metric store entries

2. **Phase 2: Node Registry**
   - Create `openclaw_nodes` table
   - Auto-register on first metric
   - API endpoints for node listing

3. **Phase 3: API Filtering**
   - Add node_id parameter to existing endpoints
   - Create aggregate stats per node

4. **Phase 4: UI Updates**
   - Add node selector component
   - Filter dashboard by node
   - Update fleet view for OpenClaw nodes

5. **Phase 5: Docker & Deployment**
   - Update docker-compose.yml
   - Documentation updates

## Testing

1. Single OpenClaw → ClawMetry (existing behavior preserved)
2. Multiple OpenClaws → One ClawMetry (new feature)
3. Node filtering in UI
4. Aggregate vs per-node stats
5. Node offline detection

## Backward Compatibility

- Existing single-OpenClaw setups work unchanged (node_id defaults to "default")
- Existing metrics without node_id are treated as "default" node
- Fleet API remains unchanged for ClawMetry-to-ClawMetry communication
