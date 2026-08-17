# SOC Homelab — Technical Documentation

This directory contains the deeper technical documentation for the SOC homelab. The root [`README.MD`](../README.MD) is intentionally optimized for a quick portfolio review; these pages preserve the implementation detail, screenshots, design rationale, and troubleshooting history.

## Documentation Map

- [Architecture & Trust Boundaries](architecture.md) — logical topology, systems, VLANs, and security objectives.
- [Firewall Policy & Segmentation](firewall-and-segmentation.md) — OPNsense rule strategy, per-VLAN rationale, Suricata, and firewall evidence.
- [Elastic & Endpoint Telemetry](elastic-and-endpoint-telemetry.md) — Filebeat/Logstash/Elasticsearch pipeline, verified TLS, Fleet, Elastic Defend, Sysmon, and GPO-based deployment.
- [Detection Engineering](detection-engineering.md) — Nmap, internal reconnaissance, SSH password-guessing detections, ATT&CK mapping, validation chains, and false-positive tuning.
- [Active Directory](active-directory.md) — AD DS, workstation/domain discovery, Group Policy security deployment, and Kali-to-workstation validation.
- [Infrastructure](infrastructure.md) — physical path, Proxmox roles, switching, management plane, backups, and storage-failure recovery.
- [Vault PKI](pki.md) — root/intermediate hierarchy, service certificates, and browser/service trust.
- [Vulnerability Management](vulnerability-management.md) — OpenVAS placement, scoped scanning, and findings evidence.
- [Lessons Learned](lessons-learned.md) — operational failures, noisy detections, certificate-chain issues, and recovery experience.

## Configuration References

Sanitized active configuration is maintained separately under [`../configs/`](../configs/README.md).

Secrets, passwords, service tokens, private keys, Vault unseal material, and other authentication data are intentionally excluded from the repository.
