# Sanitized Configuration Artifacts

These files are sanitized reference copies of the active configuration used in the SOC homelab. They are included to show how the telemetry pipeline, TLS trust, ECS normalization, and local IDS rules are implemented without publishing credentials, private keys, service tokens, or other sensitive material.

## Files

| File | Purpose |
|---|---|
| [`sanitized/filebeat-opnsense.yml`](sanitized/filebeat-opnsense.yml) | OPNsense Filebeat inputs and TLS-encrypted forwarding to Logstash on TCP/5044. |
| [`sanitized/logstash-opnsense.conf`](sanitized/logstash-opnsense.conf) | Beats ingestion, Suricata ECS enrichment, severity tagging, firewall/DNS/DHCP/auth parsing, and Elasticsearch index routing. |
| [`sanitized/elasticsearch.yml`](sanitized/elasticsearch.yml) | Elasticsearch security and TLS configuration, including the HTTP certificate chain used by Kibana, Logstash, and Elastic Agent clients. |
| [`sanitized/kibana.yml`](sanitized/kibana.yml) | Kibana HTTPS, Elasticsearch trust, Fleet output configuration, and sanitized service-account/encryption settings. |
| [`sanitized/suricata-local.rules`](sanitized/suricata-local.rules) | Active custom internal port-scan rule used for lab detection testing. |
| [`sanitized/suricata-alerts-ecs-template.json`](sanitized/suricata-alerts-ecs-template.json) | Elasticsearch index template that maps Suricata alert fields to ECS-compatible types, including `source.ip` and `destination.ip` as native `ip` fields. |

## Telemetry Flow

```text
OPNsense
  └─ Filebeat
       └─ TLS / TCP 5044
            └─ Logstash
                 ├─ Suricata alerts -> suricata-alerts-ecs-*
                 ├─ Suricata events -> suricata-logs-*
                 ├─ Firewall logs   -> firewall-logs-*
                 ├─ DNS logs        -> dns-logs-*
                 ├─ DHCP logs       -> dhcp-logs-*
                 └─ OPNsense auth   -> opnsense-auth-*

Elasticsearch -> Kibana / Elastic Security
```

## ECS Normalization

Suricata alert events are normalized in Logstash with:

```text
event.kind     = alert
event.category = intrusion_detection
event.type     = allowed
```

The dedicated `suricata-alerts-ecs-*` template maps the core network fields using ECS-compatible types:

```text
source.ip       -> ip
destination.ip  -> ip
source.port     -> long
destination.port-> long
```

This allows Elastic Security rules to use native IP/CIDR queries instead of string-based workarounds.

## Security Notes

- Passwords, API keys, service-account tokens, encryption keys, private keys, Vault tokens, and unseal material are not included.
- Certificate and key paths are retained because they document service relationships; certificate/key contents are not published.
- Values such as `<REDACTED>` and `<CA_FINGERPRINT>` are placeholders.
- These files document the homelab architecture and are not intended to be copied unchanged into another environment.
- The current sanitized Logstash artifact reflects the lab state before final Logstash-to-Elasticsearch certificate-verification hardening; that setting is tracked as the next TLS improvement.
