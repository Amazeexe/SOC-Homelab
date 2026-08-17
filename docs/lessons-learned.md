# Lessons Learned & Recovery

[← Back to main README](../README.MD) · [Documentation index](README.md)

The most valuable parts of the lab have often come from failures, noisy detections, or unexpected traffic rather than from the initial installation of a product.

## Firewall Telemetry Can Expose Configuration Problems

After hardening VLAN 99, Elastic showed repeated TCP/25 blocks from a Proxmox host. Investigation traced them to the local Postfix/sendmail notification path attempting direct SMTP delivery while a separate authenticated SMTP notification target was already configured.

The firewall was doing exactly what it should: denying direct outbound SMTP while preserving evidence of unexpected application behavior.

The notification path was later moved to the intended authenticated SMTP configuration, removing the unnecessary direct SMTP behavior without weakening the firewall.

## Not Every Unusual Port Is Malicious

TCP/4460 blocks from the Vault server initially looked suspicious. Investigation identified them as Network Time Security key-establishment traffic associated with secure time synchronization.

The result was a narrow, source-specific exception rather than a broad allow rule. The lesson was to identify the application behavior before changing policy simply because a port number looked unfamiliar.

## IDS Rules Require Context

The Suricata signature `ET SCAN NMAP -sA (1)` produced a large number of events that initially appeared to indicate scanning.

Traffic review showed a different pattern:

- public HTTPS servers as sources,
- source port `443`,
- internal clients as destinations,
- high ephemeral destination ports.

That was consistent with return HTTPS traffic being misclassified by the legacy signature, not actual internal ACK scans.

Instead of disabling the signature completely, the Elastic Security detection was scoped to internal source and destination CIDRs. This preserved useful scan coverage while reducing false positives.

## Field Mappings Affect Detection Quality

The older custom Suricata indices dynamically mapped `source.ip` and `destination.ip` as text/keyword fields. That made CIDR logic unreliable in Elastic Security.

A dedicated `suricata-alerts-ecs-*` path and template were created with native ECS `ip` mappings. Historical indices were retained rather than force-migrated, while new detection data received the correct types.

This reinforced that detection engineering depends on data engineering: a logically correct rule is still limited by the schema beneath it.

## Certificate Chains Are Part of Telemetry Reliability

Elastic Agent components stopped publishing when the configured trust information could not be validated against the certificate chain presented by Elasticsearch.

Troubleshooting showed that the Elasticsearch HTTP endpoint was presenting only the leaf certificate. The HTTP certificate configuration was rebuilt to present the leaf together with its issuing intermediate, and Fleet trust was corrected to the issuing CA.

After the fix, ingestion recovered.

The key lesson was that PKI failures can become monitoring failures. A service may be reachable and still effectively invisible if the telemetry client no longer trusts it.

## Verify Security Before Restarting Production Pipelines

When Logstash-to-Elasticsearch TLS verification was hardened, the original pipeline configuration was backed up first. The replacement was applied surgically, validated with Logstash's configuration test mode, and only then restarted.

The final output uses the internal root CA and full server identity verification instead of `ssl_certificate_verification => false`.

This provided a practical change-management pattern for the lab:

```text
backup -> make narrow change -> validate configuration -> restart -> verify live ingestion
```

## Major Firewall Upgrades Should Be Treated as Change Management

The OPNsense firewall was upgraded in stages rather than jumping directly from the older release to the newest available build. The path progressed from the 26.1 branch through the final 26.1 maintenance release, then to 26.7, and finally to `26.7.2_2`.

Before the upgrade, the firewall configuration and security telemetry components were backed up separately. Existing ZFS boot environments were preserved so the system retained rollback options if the operating-system or package transition caused a failure.

The validation plan focused on the services that would be most damaging to silently lose:

```text
OPNsense upgrade
    -> verify firewall boots and interfaces recover
    -> verify Filebeat configuration is unchanged
    -> run Filebeat configuration test
    -> verify TLS connection to Logstash
    -> validate Suricata configuration
    -> confirm fresh EVE events are produced
    -> confirm Elastic ingestion continues
```

Filebeat's active configuration was hashed before and after the major upgrade and remained unchanged. `filebeat test config` and `filebeat test output` also confirmed that the Beats-to-Logstash path still used certificate verification and successfully negotiated the configured TLS connection.

Suricata was separately validated with its configuration test mode before relying on live IDS telemetry again. Fresh EVE output confirmed that the network-detection side of the pipeline had recovered after the operating-system upgrade.

New ZFS feature flags were deliberately **not** enabled immediately after the upgrade. Keeping the pool compatible with older boot environments was more valuable than enabling new filesystem features that were not required by the lab.

The lesson was that a firewall upgrade is not complete when the dashboard shows a new version number. It is complete when routing, policy enforcement, IDS, telemetry export, certificate trust, and rollback options have all been checked.

## Immutable Flags Can Break Configuration Management

A later Suricata configuration change exposed an older manual hardening decision: generated IDS files had been marked with the FreeBSD system-immutable `schg` flag.

When OPNsense attempted to regenerate its IDS configuration, the rule installer failed because it could not rewrite files including:

```text
/usr/local/etc/suricata/opnsense.rules/local.rules
/usr/local/etc/suricata/installed_rules.yaml
```

The failure initially appeared during a normal GUI Apply operation. Troubleshooting the rule installer directly made the underlying filesystem error visible.

A second issue was discovered at the same time: the editable source rule file and the generated OPNsense copy had drifted. The source file under:

```text
/usr/local/etc/suricata/rules/local.rules
```

still contained an older internal scan rule, while the generated copy contained the newer tuned rules.

Both copies were backed up before repair. The current rule set was restored to the editable source, the inappropriate immutable flags were removed from OPNsense-managed generated files, and the IDS configuration was regenerated normally.

Validation included:

```text
installRules.py -> exit 0
suricata -T     -> exit 0
fresh EVE data  -> confirmed
```

The final check also confirmed that no Suricata files remained protected by `schg`.

The lesson is that filesystem immutability is not automatically a security improvement. Files that belong to a configuration-management or package-generation workflow must remain writable by that workflow. Protecting generated files manually can create configuration drift, break upgrades, and turn a normal policy change into an outage.

## Segmentation and Detection Reinforce Each Other

The firewall does more than stop traffic. Logged denies create telemetry that can be correlated with Suricata and endpoint events.

A blocked Kali connection therefore becomes both:

- a **preventive control**, because the connection is denied, and
- a **detection artifact**, because the attempt is visible in Elastic.

This is why the red-team VLAN is deliberately constrained rather than being placed on a trusted network simply to make testing easier.

## Infrastructure Failures Are Part of Security Engineering

A failed pve2 NVMe drive required a Proxmox reinstall and VM rebuild.

Recovery included more than bringing guests back online. Vault, certificates, Elastic components, enrollments, and service trust relationships also had to be restored and validated.

That experience reinforced the value of:

- scheduled VM backups,
- documented dependencies,
- reproducible configuration,
- post-restore trust validation,
- verifying telemetry after infrastructure recovery.

## Operational Takeaway

The lab has become most useful when something behaves unexpectedly. The recurring workflow is:

```text
Observe anomaly -> identify the actual source -> change the narrowest control necessary
-> validate the result -> document what was learned
```

That troubleshooting history is intentionally preserved because it demonstrates how the environment is operated, not merely how it was installed.
