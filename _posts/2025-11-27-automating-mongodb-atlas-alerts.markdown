---
layout: post
title:  "Automating MongoDB Atlas Alert Configuration with Excel and CLI"
date:   2025-11-27 10:00:00 +0800
categories: mongodb atlas automation python
---

Managing alerts across multiple MongoDB Atlas projects manually is tedious and error-prone. I built an automation script that reads alert configurations from an Excel spreadsheet and deploys them via the [MongoDB Atlas CLI](https://www.mongodb.com/docs/atlas/cli/current/), making it easy to maintain consistent alerting across environments.

## The Problem

Atlas provides excellent default alerts, but production environments often need customized thresholds. When managing multiple projects or clusters, manually configuring 20+ alert types per project becomes a maintenance nightmare. Changes need to be tracked, documented, and applied consistently.

## The Solution

The script takes an Excel file defining alert configurations and translates them into Atlas CLI commands. This provides:

- **Version-controlled configuration** - Excel files can be tracked in git
- **Bulk operations** - Create dozens of alerts in one command
- **Dry-run mode** - Generate JSON configs without creating alerts
- **Safe deletion** - Track automation-created alerts separately from defaults

## How It Works

The automation handles three main workflows:

### 1. Threshold Parsing

Excel cells contain human-readable thresholds that need translation to Atlas API format:

```
"> 4000 for 2 minutes" → operator: GREATER_THAN, threshold: 4000, units: MINUTES, value: 2
"< 24h for 5 minutes"  → operator: LESS_THAN, threshold: 24, units: HOURS, value: 5
"> 90%"                → operator: GREATER_THAN, threshold: 90, units: PERCENTAGE
```

The parser handles various formats including milliseconds, gigabytes, percentages, and plain durations.

### 2. Alert Generation

Each row in the Excel file can generate multiple alerts:
- If low and high priority thresholds differ, two separate alerts are created
- Event-based alerts (like backup failures) trigger on any occurrence
- Metric-based alerts use the parsed threshold values

### 3. Alert Tracking

The script maintains `.automation_alert_ids.json` to track which alerts it created. This enables:
- Deleting only automation-created alerts while preserving Atlas defaults
- Re-running safely without creating duplicates
- Auditing what was deployed

## Alert Types Covered

The configuration supports 20+ alert types across categories:

| Category | Examples |
|----------|----------|
| Replication | Oplog window, replication lag, primary elections |
| Performance | Disk IOPS, latency, page faults, queue depths |
| Resources | CPU usage, disk space, swap usage |
| Availability | Host down, no primary |
| Backup | Failed/successful snapshots, schedule behind |

## Usage

Basic deployment:

```bash
./run_alerts.sh --project-id YOUR_PROJECT_ID
```

Preview without creating:

```bash
./run_alerts.sh --project-id YOUR_PROJECT_ID --dry-run
```

Remove automation alerts only:

```bash
./run_alerts.sh --project-id YOUR_PROJECT_ID --delete-existing
```

## Implementation Notes

The script uses the Atlas CLI rather than the REST API directly. This provides:
- Simpler authentication (reuses `atlas auth login`)
- Consistent JSON format for alert configs
- Built-in validation before submission

Generated JSON files are saved to `./alerts/` for review. Each alert config follows the [Atlas Alert Configuration File format](https://www.mongodb.com/docs/atlas/cli/current/reference/json/alert-config-file/).

## Practical Considerations

1. **Low vs High Priority** - Separate thresholds create separate alerts, useful for different notification channels
2. **Notification Roles** - Defaults to `GROUP_OWNER` but can target specific emails
3. **Idempotency** - The tracking file prevents duplicate alerts on re-runs
4. **Atlas Defaults** - The `--delete-existing` flag preserves Atlas default alerts

The code is available at [research/atlas-alerts-creation](https://github.com/tzehon/research/tree/main/atlas-alerts-creation).
