---
name: SLO Warden
category: reliability
purpose: SLO monitoring and alerting
tags:
  - SLO
  - monitoring
  - alerting
  - reliability
  - SLI
  - error budgets
version: 1.0.0
requires:
  - prometheus
  - alertmanager
  - grafana
---

# SLO Warden

SLO Warden is OpenClaw's reliability specialist for Service Level Objectives monitoring, alerting, and error budget management. It provides commands to create, monitor, and alert on SLOs across your infrastructure.

## Quick Start

```bash
openclaw slo list
openclaw slo create --service api --target 99.9 --window 30d
openclaw slo status api
```

## Commands

### Create SLO

Creates a new Service Level Objective with specified targets.

```bash
openclaw slo create --service <name> --target <percentage> --window <duration>
```

**Options:**
- `--service, -s`: Service name (required)
- `--target, -t`: Target availability percentage (e.g., 99.9)
- `--window, -w`: Time window (1h, 24h, 7d, 30d)
- `--indicator-type`: SLI type (latency, error-rate, availability)
- `--threshold`: Error threshold for latency SLOs (default: 500ms)

**Example:**
```bash
openclaw slo create -s api-gateway -t 99.95 -w 30d --indicator-type latency --threshold 200
```

### List SLOs

Lists all configured SLOs with their current status.

```bash
openclaw slo list [flags]
```

**Flags:**
- `--env`: Filter by environment (production, staging, development)
- `--status`: Filter by status (healthy, warning, critical)

**Example:**
```bash
openclaw slo list --env production
```

### SLO Status

Shows detailed status of a specific SLO including error budget consumption.

```bash
openclaw slo status <service> [flags]
```

**Example:**
```bash
openclaw slo status api-gateway --window 7d
```

**Output includes:**
- Current SLI value
- Error budget remaining
- Burn rate
- Time to exhaustion
- Recent violations

### Alert Rules

Manages alerting rules for SLO violations.

```bash
openclaw slo alert create <service> [flags]
openclaw slo alert list [service]
openclaw slo alert delete <rule-id>
```

**Flags:**
- `--warning-threshold`: Warning alert threshold (default: 75% budget consumed)
- `--critical-threshold`: Critical alert threshold (default: 90% budget consumed)
- `--channel`: Notification channel (slack, pagerduty, email)

**Example:**
```bash
openclaw slo alert create api-gateway --warning-threshold 70 --critical-threshold 90 --channel slack
```

### Error Budget

Manages and reports on error budgets.

```bash
openclaw slo budget <service>
openclaw slo budget burn-rate <service>
```

**Example:**
```bash
openclaw slo budget api-gateway
```

### Dashboard

Opens Grafana dashboard for SLO visualization.

```bash
openclaw slo dashboard [service]
```

## Real Use Cases

### Use Case 1: API Gateway Availability SLO

Monitor API gateway availability with 99.9% target over 30 days.

```bash
openclaw slo create -s api-gateway -t 99.9 -w 30d --indicator-type availability
openclaw slo alert create api-gateway --channel slack
```

**Expected behavior:**
- Creates SLO tracking 99.9% availability
- Sets up alerts at 75% (warning) and 90% (critical) budget consumption
- Slack notifications sent to #reliability-alerts

### Use Case 2: Latency SLO with Custom Threshold

Monitor p99 latency for payment service, alerting if >500ms.

```bash
openclaw slo create -s payment-service -t 99.0 -w 7d --indicator-type latency --threshold 500
openclaw slo alert create payment-service --warning-threshold 60 --critical-threshold 85 --channel pagerduty
```

**Use case:** Payments must complete within 500ms for 99% of requests over 7 days.

### Use Case 3: Multi-Environment SLO Comparison

Compare SLO status across environments.

```bash
openclaw slo list --env production --status warning
openclaw slo status auth-service --window 24h
```

**Use case:** Identify which production services are at risk and need immediate attention.

### Use Case 4: Error Budget Burn Rate Analysis

Investigate why an SLO is burning budget faster than expected.

```bash
openclaw slo budget burn-rate user-service
```

**Use case:** Detect if an incident is causing accelerated budget consumption (burn rate > 2x normal).

### Use Case 5: Post-Incident SLO Review

After resolving an incident, review SLO impact.

```bash
openclaw slo status user-service --window 7d
openclaw slo budget user-service
```

**Use case:** Determine if the incident consumed significant error budget and if remediation is needed.

## Troubleshooting

### SLO shows "No Data"

**Cause:** Prometheus not recording metrics for this service.

**Solution:**
```bash
# Verify service metrics exist
openclaw metrics query <service>_requests_total

# Check Prometheus scrape status
openclaw prometheus targets | grep <service>
```

### Alert not firing at expected threshold

**Cause:** Alert rule configuration mismatch or Alertmanager routing issue.

**Solution:**
```bash
# Verify alert rules loaded
openclaw alertmanager rules | grep <service>

# Check routing config
openclaw alertmanager config
```

### Error budget shows negative

**Cause:** SLO has been violated; more errors than allowed budget.

**Solution:**
```bash
# Review recent errors
openclaw logs <service> --since 24h | grep error

# Check if known incident
openclaw incidents list --service <service> --recent
```

### Burn rate calculation incorrect

**Cause:** Insufficient historical data or metric recording gap.

**Solution:**
```bash
# Verify metric continuity
openclaw prometheus query "rate(<service>_errors_total[1h])"

# Check for recording rules
openclaw prometheus recording-rules | grep sli
```

### Dashboard not loading

**Cause:** Grafana datasource misconfigured or service not tagged.

**Solution:**
```bash
# Verify Grafana connectivity
openclaw grafana health

# Check datasource
openclaw grafana datasources

# Verify service annotations
openclaw annotate list <service>
```

## Configuration

### Required Environment Variables

```bash
PROMETHEUS_URL=<prometheus-server>
ALERTMANAGER_URL=<alertmanager-server>
GRAFANA_URL=<grafana-server>
SLACK_WEBHOOK_URL=<slack-webhook> (optional)
PAGERDUTY_KEY=<pagerduty-key> (optional)
```

### SLO Configuration File

Create `~/.openclaw/config/slo-config.yaml`:

```yaml
defaults:
  warning_threshold: 75
  critical_threshold: 90
  window: 30d
  channels:
    - slack

services:
  api-gateway:
    target: 99.9
    indicator_type: availability
    
  payment-service:
    target: 99.95
    indicator_type: latency
    threshold: 500
    window: 7d

environments:
  production:
    slack_channel: "#prod-alerts"
  staging:
    slack_channel: "#staging-alerts"
```

## Examples

### Complete SLO Setup Workflow

```bash
# 1. Create availability SLO for new service
openclaw slo create -s checkout-service -t 99.9 -w 30d --indicator-type availability

# 2. Configure alerts
openclaw slo alert create checkout-service --warning-threshold 70 --critical-threshold 90 --channel slack

# 3. Open dashboard to verify
openclaw slo dashboard checkout-service

# 4. Check initial status
openclaw slo status checkout-service

# 5. List all SLOs to confirm
openclaw slo list
```

### Daily SLO Health Check

```bash
#!/bin/bash
# Daily SLO health check script

echo "=== SLO Health Report ==="
echo ""

echo "Production Services:"
openclaw slo list --env production --status warning
openclaw slo list --env production --status critical
echo ""

echo "Error Budget Status:"
for service in api-gateway auth-service payment-service; do
    echo "--- $service ---"
    openclaw slo budget "$service"
done
```

### Incident Response SLO Commands

```bash
# During incident
openclaw slo status <affected-service> --window 1h

# After incident
openclaw slo status <affected-service> --window 24h
openclaw slo budget <affected-service>
```

## Best Practices

1. **Start with 99% target** - Move to 99.9% after establishing baseline
2. **Use 30-day windows** - Avoid weekly patterns distorting data
3. **Alert at 75%/90%** - Catch issues before SLO breach
4. **Document SLO rationale** - Include in service README
5. **Review quarterly** - Assess if SLOs match business needs

## Related Commands

```bash
openclaw incident create      # Create new incident
openclaw metrics query        # Query service metrics  
openclaw grafana dashboard    # Open Grafana
openclaw alertmanager status  # Check alert status
```
```