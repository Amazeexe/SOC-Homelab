# Elastic Operations & Lifecycle Management

This page documents operational maintenance performed on the Elastic Stack after the initial SIEM and endpoint-telemetry deployment. The goal was to treat the stack as a maintained security platform rather than a static lab service: back up state before upgrades, validate migrations and ingestion afterward, troubleshoot recovery behavior, and enforce finite retention for high-volume telemetry.

## Staged Elastic Stack Upgrade

The stack was upgraded in stages from **8.19.11 → 8.19.20 → 9.5.1** rather than jumping directly between major versions.

Before the first upgrade, an Elasticsearch snapshot named `pre_upgrade_8_19_11_2026_08_16` was created in the configured snapshot repository. Package candidates, service configuration, and available system resources were reviewed before changing the stack.

After the intermediate 8.19.20 upgrade, Elasticsearch and Kibana service health were validated before proceeding. Kibana retained its configuration and completed saved-object migrations successfully.

The stack was then upgraded to **9.5.1**. During the Elasticsearch/Kibana restart and shard-recovery window, several existing Elastic Security detection rules temporarily reported `no_shard_available_action_exception`, query gaps, and missed executions. Affected rules included internal Nmap/TCP reconnaissance, Linux SSH password guessing, and Elastic Defend detections.

Rather than treating the transient errors as rule failures, cluster recovery was validated first. Once primary shards were available again, the affected indices returned to their expected state and detection execution resumed. A second snapshot, `post_upgrade_9_5_1_2026_08_18`, was then created to establish a known-good recovery point after the migration.

### Upgrade validation workflow

1. Review package candidates, service state, and system resources.
2. Create a pre-upgrade Elasticsearch snapshot.
3. Upgrade to the latest compatible 8.19 maintenance release.
4. Validate Elasticsearch health and Kibana saved-object migrations.
5. Upgrade to Elastic 9.5.1.
6. Monitor shard recovery, ingestion, and Security rule execution.
7. Confirm recovered indices and resumed detections.
8. Create a post-upgrade snapshot.

This process reinforced that a SIEM upgrade is not complete when packages finish installing; data availability, migrations, ingestion, detection execution, and recovery state all need to be validated afterward.

## Retention & Storage Management

After the upgrade, historical Elastic Agent metrics and log backing indices were consuming substantially more storage than needed for the lab. The existing lifecycle behavior primarily provided rollover, so custom finite-retention policies were introduced.

### Retention model

| Data class | Policy | Retention goal |
|---|---|---:|
| General logs | `homelab-logs-30-day` | 30 days |
| Metrics | `homelab-metrics-30-day` | 30 days |
| Endpoint alerts | `homelab-endpoint-alerts-90-day` | 90 days |
| Firewall / DNS / DHCP / Suricata legacy indices | `homelab-30-day` | 30 days |

Templates for Fleet-managed log and metric data streams were updated so **new backing-index generations** inherit the custom policies. Existing streams were then rolled over and checked with `_ilm/explain`.

The expected transition was visible immediately: older generations remained attached to the built-in `logs` or `metrics` policy, while the newly created generations showed `homelab-logs-30-day` or `homelab-metrics-30-day`.

Endpoint alerts were kept longer than general telemetry. The `logs-endpoint.alerts-default` stream was configured with `homelab-endpoint-alerts-90-day` and armed with a lazy rollover so the next alert write creates the new generation without forcing an empty backing index.

## Safe Historical Cleanup

A key lesson from the cleanup was that **backing-index creation dates are not sufficient evidence that all documents in an index are expired**.

Several older-looking backing indices had continued receiving events long after their creation date. For example, June 19 Fleet Server, system authentication, and syslog generations still contained events from August 14–15. Deleting those indices based only on their names or creation timestamps would have removed current telemetry.

To avoid that, historical deletion used the newest actual event timestamp in each backing index:

```json
GET /.ds-logs-*/_search?expand_wildcards=all&filter_path=aggregations.by_index.buckets
{
  "size": 0,
  "aggs": {
    "by_index": {
      "terms": {
        "field": "_index",
        "size": 200
      },
      "aggs": {
        "oldest_event": {
          "min": { "field": "@timestamp" }
        },
        "newest_event": {
          "max": { "field": "@timestamp" }
        }
      }
    }
  }
}
```

Only non-write backing indices whose **newest `@timestamp` was older than the 30-day cutoff** were deleted. This removed 24 genuinely expired log backing indices while preserving old generations that still contained recent data.

## Post-Cleanup Validation

After the metric and log cleanup, node-level allocation showed:

```text
Elasticsearch index data:  23 GB
Total disk used:            85 GB
Disk available:            109 GB
Disk total:                 194 GB
Disk utilization:           43%
Assigned shards:            487
Unassigned shards:           36
```

The remaining yellow state was investigated with the cluster allocation explain API rather than assumed to be a storage or corruption issue.

For an unassigned Endpoint replica shard, Elasticsearch returned the `same_shard` allocation decider:

```text
decider: same_shard
decision: NO
explanation: a copy of this shard is already allocated to this node
```

The cluster is intentionally single-node, so replica shards cannot be allocated to a second node. Primary shards were started and available; the unassigned replicas therefore reflected the architecture rather than a failed primary or allocation defect.

## Operational Takeaways

- Take recoverable snapshots before and after major SIEM upgrades.
- Validate saved-object migrations, shard health, ingestion, and detections after maintenance.
- Treat temporary detection failures during shard recovery as an operational signal to investigate, not automatically as broken rule logic.
- Use explicit retention policies for high-volume telemetry instead of relying on indefinite rollover behavior.
- Roll existing data streams when a new lifecycle policy must apply to future generations.
- Do not infer document age from a backing-index name or creation date; verify `@timestamp` before destructive cleanup.
- Investigate yellow cluster health with allocation explanations before changing replica settings or assuming data loss.

This maintenance cycle turned a simple version upgrade into a broader exercise in **SIEM administration, lifecycle management, backup/recovery planning, capacity management, and operational troubleshooting**.
