# SOC Homelab — Detection Engineering, Segmentation & Purple-Team Validation

A security-focused homelab built to practice **network segmentation, SIEM engineering, endpoint telemetry, PKI, vulnerability management, Active Directory, and purple-team validation** in an environment designed to behave more like a small enterprise than a collection of isolated VMs.

The lab combines a bare-metal OPNsense firewall, Cisco switching, a two-node Proxmox cluster, Elastic Stack, Suricata, HashiCorp Vault, Greenbone/OpenVAS, Windows Active Directory, Elastic Defend, Sysmon, and a Kali Linux validation host.

> **Portfolio focus:** defensive engineering, telemetry pipelines, firewall policy design, detection validation, alert tuning, endpoint visibility, PKI, and vulnerability management.

## What This Project Demonstrates

- **Eight segmented VLANs** with explicit service exceptions, `Protected_Networks` controls, and default-deny behavior.
- **End-to-end security telemetry** from OPNsense Filebeat through Logstash to Elasticsearch over verified TLS.
- **Elastic Defend and Sysmon** deployed to scoped Windows systems through Active Directory Group Policy.
- **Closed-loop detection validation** using controlled Kali and internal-host activity with confirmation in Elastic Security.
- **MITRE ATT&CK-mapped detections**, including Nmap service discovery, internal TCP SYN reconnaissance, and SSH password guessing.
- **Internal PKI** using a Vault root/intermediate hierarchy and service certificates.
- **Credentialed vulnerability management** with Greenbone/OpenVAS.
- **Failure and tuning narratives** covering IDS false positives, certificate-chain failures, firewall-discovered application behavior, and a Proxmox storage rebuild.

## Architecture at a Glance

```text
Internet / ONT
      |
      v
  OPNsense
      |
      v
Cisco 3560CX
      |
      +--> VLAN 10  Main / Admin
      +--> VLAN 20  WiFi
      +--> VLAN 30  Wired Desktop
      +--> VLAN 40  Security Services
      +--> VLAN 50  Windows Workstation
      +--> VLAN 60  Domain Controller
      +--> VLAN 70  Red Team
      +--> VLAN 99  Management

OPNsense Filebeat --TLS/5044--> Logstash --verified TLS--> Elasticsearch --> Kibana / Elastic Security
Windows Elastic Agent ---------------------------------------------> Fleet / Elasticsearch
Sysmon + Elastic Defend -------------------------------------------> Endpoint telemetry
```

| VLAN | Subnet | Purpose |
|---|---|---|
| 10 | `192.168.10.0/24` | Main / trusted administration |
| 20 | `192.168.20.0/24` | WiFi |
| 30 | `192.168.30.0/24` | Wired desktop |
| 40 | `192.168.40.0/24` | ELK, OpenVAS, Vault |
| 50 | `192.168.50.0/24` | Windows domain workstation |
| 60 | `192.168.60.0/24` | Domain controller |
| 70 | `192.168.70.0/24` | Kali red-team / validation host |
| 99 | `192.168.99.0/24` | Proxmox, switch, firewall management |

[Read the full architecture and trust-boundary design →](docs/architecture.md)

## Detection Evidence

Controlled lab activity is used to verify that security controls produce usable telemetry and alerts rather than only proving that the tools are installed.

![Elastic Security rules](docs/images/kibana/Kibana_Security_Rules_Updated.png)

The current validation workflow includes:

| Detection | Validation | ATT&CK |
|---|---|---|
| Nmap ACK scan from the red-team segment | Kali traffic → Suricata → ECS-normalized index → Elastic Security alert | `T1046` Network Service Discovery |
| Internal TCP SYN scan / possible insider reconnaissance | `HOME_NET` host → custom Suricata SID `9000001` → Elastic Security rule | `T1046` Network Service Discovery |
| Linux SSH password guessing | Three failed password-based SSH attempts from the same source/host within 60 seconds → EQL correlation alert | `T1110.001` Password Guessing |

[Read the detection engineering and tuning workflow →](docs/detection-engineering.md)

## Elastic Defend Endpoint Visibility

Elastic Defend is centrally managed through Fleet and provides endpoint-security telemetry from the Windows domain workstation and domain controller alongside Sysmon and Windows event logs. The current policy keeps Elastic Defend focused on malware detection and endpoint visibility while Microsoft Defender remains the primary Windows antivirus.

### Elastic Defend Policy

![Elastic Defend policy - protection settings](docs/images/kibana/DefendPolicyPrt1.png)

![Elastic Defend policy - event collection settings](docs/images/kibana/DefendPolicyPrt2.png)

The policy is centrally applied rather than configured host-by-host, making endpoint collection repeatable and easier to audit as the lab grows.

### Process-Level Telemetry

![Elastic Defend process telemetry](docs/images/kibana/Elastic-Defend-Process-Detection.png)

Elastic Defend process events expose host, process, parent process, user, action, and command-line context in Kibana. Controlled commands on the Windows workstation are used to verify that endpoint activity is searchable and attributable before building or tuning detections around it.

[Read the Elastic and endpoint telemetry implementation →](docs/elastic-and-endpoint-telemetry.md)

## Technical Documentation

The root README is intentionally kept skimmable. Detailed implementation notes, screenshots, rationale, and troubleshooting history are split into focused pages:

| Area | Documentation |
|---|---|
| Architecture & trust boundaries | [docs/architecture.md](docs/architecture.md) |
| Firewall policy & segmentation | [docs/firewall-and-segmentation.md](docs/firewall-and-segmentation.md) |
| Elastic, Fleet, Elastic Defend & Sysmon | [docs/elastic-and-endpoint-telemetry.md](docs/elastic-and-endpoint-telemetry.md) |
| Detection engineering & validation | [docs/detection-engineering.md](docs/detection-engineering.md) |
| Active Directory & GPO deployment | [docs/active-directory.md](docs/active-directory.md) |
| Physical infrastructure, Proxmox & switching | [docs/infrastructure.md](docs/infrastructure.md) |
| Vault PKI & certificate trust | [docs/pki.md](docs/pki.md) |
| Vulnerability management | [docs/vulnerability-management.md](docs/vulnerability-management.md) |
| Lessons learned & recovery | [docs/lessons-learned.md](docs/lessons-learned.md) |
| Documentation index | [docs/README.md](docs/README.md) |

## Configuration Artifacts

Sanitized reference copies of active security configuration are published in [`configs/`](configs/README.md). Secrets, passwords, private keys, Vault material, and service tokens are excluded.

**Direct links:** [Filebeat](configs/sanitized/filebeat-opnsense.yml) · [Logstash](configs/sanitized/logstash-opnsense.conf) · [Elasticsearch](configs/sanitized/elasticsearch.yml) · [Kibana](configs/sanitized/kibana.yml) · [Suricata rules](configs/sanitized/suricata-local.rules) · [Suricata ECS template](configs/sanitized/suricata-alerts-ecs-template.json)

## Technology Stack

| Area | Technologies |
|---|---|
| Firewall / Routing | OPNsense, pf, VLANs, NAT, DHCP relay, Unbound |
| Switching | Cisco Catalyst 3560CX, 802.1Q trunks |
| IDS / Network Security | Suricata, Emerging Threats rules |
| SIEM / Search | Elasticsearch, Logstash, Kibana, Elastic Security |
| Endpoint Security | Elastic Agent, Fleet, Elastic Defend, Sysmon, Windows Event Logs |
| Identity / Management | Windows Server 2025, Active Directory, Group Policy, DNS, Kerberos, LDAP |
| PKI | HashiCorp Vault, X.509, internal root/intermediate CA |
| Vulnerability Management | Greenbone / OpenVAS |
| Virtualization | Proxmox VE cluster |
| Offensive Validation | Kali Linux, Nmap, FreeRDP |

## Project Status

The lab is operational and continues to evolve around **detection quality, telemetry enrichment, segmentation validation, endpoint visibility, infrastructure resilience, and purple-team exercises** rather than adding services for their own sake.

Future work will continue to add validated endpoint detections, expand ECS normalization, document hardware/storage changes, improve certificate lifecycle automation, and deepen vulnerability-remediation workflows.
