---
layout: post
title:  "Extending Alert Automation to MongoDB Ops Manager"
date:   2025-12-21 12:00:00 +0800
categories: mongodb ops-manager automation python
---

In a [previous post](/llm/mongodb/atlas/automation/python/2025/11/27/automating-mongodb-atlas-alerts.html), I covered automating MongoDB Atlas alert configurations using Excel and the Atlas CLI. This post extends that approach to self-managed MongoDB deployments using Ops Manager.

## Why Ops Manager Needs Different Tooling

While Atlas and Ops Manager share similar alert concepts, the implementation differs significantly:

| Aspect | Atlas | Ops Manager |
|--------|-------|-------------|
| Authentication | Atlas CLI with API keys | HTTP Digest Auth |
| API Endpoint | `cloud.mongodb.com` | Your Ops Manager server |
| Backup Alerts | `CPS_SNAPSHOT_*` events | `OPLOG_BEHIND`, `RESYNC_REQUIRED` |
| Agent Alerts | N/A | `MONITORING_AGENT_DOWN`, `AUTOMATION_AGENT_DOWN` |

The same Excel-driven workflow applies, but the underlying API calls and available alert types change.

## How It Works

```
┌─────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│  Excel Config File  │ ──▶ │  Python Script       │ ──▶ │  Ops Manager API    │
│  (your thresholds)  │     │  (generates JSON)    │     │  (creates alerts)   │
└─────────────────────┘     └──────────────────────┘     └─────────────────────┘
```

1. **Read Excel** - Alert names and thresholds defined in a spreadsheet
2. **Generate JSON** - Script converts thresholds to Ops Manager API format
3. **Create via API** - HTTP Digest authentication to your Ops Manager instance

## Ops Manager-Specific Alert Types

Beyond the standard replica set and host metrics, Ops Manager provides alerts for its agent infrastructure:

### Agent Health Alerts

```python
# Monitoring Agent
{
    "eventTypeName": "MONITORING_AGENT_DOWN",
    "enabled": true,
    "notifications": [...]
}

# Automation Agent
{
    "eventTypeName": "AUTOMATION_AGENT_DOWN",
    "enabled": true,
    "notifications": [...]
}

# Backup Agent
{
    "eventTypeName": "BACKUP_AGENT_DOWN",
    "enabled": true,
    "notifications": [...]
}
```

These alerts trigger when agents stop reporting to Ops Manager - critical for catching infrastructure issues before they affect your databases.

### Backup-Specific Alerts

```python
# Oplog falling behind
{
    "eventTypeName": "OPLOG_BEHIND",
    "typeName": "BACKUP",
    "enabled": true,
    "notifications": [...]
}

# Full resync required
{
    "eventTypeName": "RESYNC_REQUIRED",
    "typeName": "BACKUP",
    "enabled": true,
    "notifications": [...]
}
```

## Authentication: HTTP Digest vs Atlas CLI

Atlas uses the CLI for authentication, which handles token refresh and credential management. Ops Manager requires HTTP Digest authentication directly:

```python
from requests.auth import HTTPDigestAuth

auth = HTTPDigestAuth(public_key, private_key)
response = requests.post(
    f"{base_url}/api/public/v1.0/groups/{project_id}/alertConfigs",
    auth=auth,
    json=alert_config,
    verify=ca_cert_path  # For self-signed certificates
)
```

The authentication never sends credentials in plain text - Digest auth uses a challenge-response mechanism.

## Handling Self-Signed Certificates

Many Ops Manager deployments use self-signed TLS certificates. The script supports two approaches:

```bash
# Option 1: Provide CA certificate (recommended)
./run_alerts.sh --project-id YOUR_PROJECT_ID --ca-cert /path/to/ca.crt

# Option 2: Disable SSL verification (not recommended for production)
./run_alerts.sh --project-id YOUR_PROJECT_ID --no-verify-ssl
```

Always prefer providing the CA certificate. Disabling verification opens you to man-in-the-middle attacks.

## Excel Configuration Format

The spreadsheet format remains similar to the Atlas version:

| Alert Name | Alert Type | Low Threshold | High Threshold |
|------------|------------|---------------|----------------|
| Replication Lag | Replica Set | > 60s for 5 minutes | > 120s for 2 minutes |
| Disk space % used on Data Partition | Host | > 80% for 5 minutes | > 90% for 5 minutes |
| Monitoring Agent Down | Agent | Any occurrence | - |

The script maps alert names to the correct Ops Manager metric names via an `ALERT_MAPPINGS` dictionary.

## Finding Metric Names

Metric names can vary between Ops Manager versions. The most reliable approach is to create an alert manually via the UI, then inspect it via API:

```bash
# Query all alerts and filter by keyword
curl -sk -u "${PUBLIC_KEY}:${PRIVATE_KEY}" --digest \
  "${BASE_URL}/api/public/v1.0/groups/${PROJECT_ID}/alertConfigs" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for alert in data.get('results', []):
    if 'DISK' in str(alert):
        print(json.dumps(alert, indent=2))
"
```

Look for the `metricThreshold.metricName` field in the output. That's the exact string to use in your configuration.

## Disk Partition Metrics

Disk metrics often cause confusion because Ops Manager tracks partitions separately. The DATA partition metrics use a `_DATA` suffix:

| Alert | Metric Name |
|-------|-------------|
| Disk space % used on Data Partition | `DISK_PARTITION_SPACE_USED_DATA` |
| Disk read IOPS on Data Partition | `DISK_PARTITION_READ_IOPS_DATA` |
| Disk write latency on Data Partition | `DISK_PARTITION_WRITE_LATENCY_DATA` |

For JOURNAL or INDEX partitions, substitute the appropriate suffix.

## Usage

```bash
# Set credentials via environment variables
export OPS_MANAGER_BASE_URL=https://opsmanager.example.com:8080
export OPS_MANAGER_PUBLIC_KEY=your_public_key
export OPS_MANAGER_PRIVATE_KEY=your_private_key

# Preview what will be created (dry run)
./run_alerts.sh --project-id YOUR_PROJECT_ID --dry-run

# Create the alerts
./run_alerts.sh --project-id YOUR_PROJECT_ID

# Delete only automation-created alerts (preserves defaults)
./run_alerts.sh --project-id YOUR_PROJECT_ID --delete-existing
```

The `--delete-existing` flag only removes alerts that were created by this automation. It tracks alert IDs in `.automation_alert_ids.json`, so manually created alerts or Ops Manager defaults are preserved.

## Tracking Created Alerts

The script maintains a JSON file mapping project IDs to alert IDs:

```json
{
  "60f1a2b3c4d5e6f7": [
    "alert_id_1",
    "alert_id_2",
    "alert_id_3"
  ]
}
```

This enables safe cleanup without affecting alerts created through other means.

## Combining with Atlas Automation

If you run both Atlas and Ops Manager deployments, you can use both automation tools with a shared Excel format. The alert names and thresholds stay consistent; only the underlying API calls differ.

| Environment | Tool | Authentication |
|-------------|------|----------------|
| MongoDB Atlas | atlas-alerts-creation | Atlas CLI |
| MongoDB Ops Manager | ops-manager-alerts-creation | HTTP Digest |

This provides a unified alerting strategy across your entire MongoDB estate.

## Key Differences from Atlas Version

1. **No Atlas CLI dependency** - Direct HTTP calls with Digest auth
2. **Additional alert types** - Agent and Ops Manager-specific backup alerts
3. **SSL certificate handling** - Support for self-signed certificates
4. **Different metric names** - Some metrics have different names in Ops Manager

The code is available at [research/ops-manager-alerts-creation](https://github.com/tzehon/research/tree/main/ops-manager-alerts-creation).
