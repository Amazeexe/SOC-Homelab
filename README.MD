# Homelab SOC — Master Reference Document
**Last Updated:** July 27, 2026

## Owner
| Field | Value |
|---|---|
| Role | Systems Administrator |
| Certifications | CompTIA Security+, CySA+ |
| Clearance | T4 Public Trust |
| Goal | Transition to MSSP/SOC — Security Analyst or Detection Engineer |

---

## Network Architecture

| Device | Role | IP |
|---|---|---|
| OPNsense (bare-metal) | Firewall/Router | LAN: 192.168.1.1 / VLAN99: 192.168.99.1 |
| Cisco 3560CX | Managed Switch | 192.168.99.3 |
| pve1 — Dell Optiplex Micro | Hypervisor Node 1 | 192.168.99.2 |
| pve2 — Xeon E5-2690 v4 4U | Hypervisor Node 2 | 192.168.99.50 |
| Ubuntu VM (ELK Stack) | SIEM | 192.168.40.10 (VLAN40) — pve1 |
| OpenVAS VM | Vulnerability Scanner | 192.168.40.20 (VLAN40) — pve1 |
| HashiCorp Vault VM | PKI / Secret Management | 192.168.40.30 (VLAN40) — pve2 |
| Windows Server 2025 DC | Active Directory | 192.168.60.10 (VLAN60) — pve2 |
| Main PC | Workstation | 192.168.10.153 |

### VLAN Segmentation
| VLAN | Subnet | Purpose |
|---|---|---|
| VLAN 10 | 192.168.10.0/24 | Main LAN |
| VLAN 20 | 192.168.20.0/24 | WiFi |
| VLAN 30 | 192.168.30.0/24 | Wired Desktops |
| VLAN 40 | 192.168.40.0/24 | Security/SIEM Infrastructure |
| VLAN 50 | 192.168.50.0/24 | Windows 11 Domain VMs |
| VLAN 60 | 192.168.60.0/24 | Windows Domain Controller |
| VLAN 99 | 192.168.99.0/24 | Management Network |

### Cluster
- **Name:** ZMH-Homelab
- **pve1 hostname:** homelabprox
- **pve2 hostname:** pve2
- **OPNsense web UI:** https://opnsense.zmh.homelab (Unbound host override)

---

## Hardware

### pve2 — Xeon Build (Primary Workload Node)
| Component | Detail |
|---|---|
| CPU | Intel Xeon E5-2690 v4 (14 core) |
| RAM | 32GB ECC DDR4 |
| Boot/VM Drive | Samsung 990 Pro 1TB NVMe |
| Backup Drive | Seagate ST500LM021 500GB HDD |
| Case | Rosewill RSV-R4100U 4U Rackmount |

### pve1 — Optiplex Micro
| Component | Detail |
|---|---|
| Drive | 256GB SSD |
| Current VMs | ELK Stack (VM 100), OpenVAS (VM 101) |

### Drive Failure — July 26 2026
The original pve2 boot drive (Timetec 1TB NVMe, Realtek controller) failed in production. Key findings:

- 594 Media and Data Integrity Errors logged by the kernel
- Root filesystem remounted read-only; LVM thin pool metadata corrupted
- SMART telemetry was inaccurate due to budget controller firmware
- Full OS reinstall was required; all pve2 VMs rebuilt from scratch
- Resolution: Samsung 990 Pro installed as replacement

---

## VM Inventory

| VMID | Name | Node | IP | Role |
|---|---|---|---|---|
| 100 | UbuntuSuricata | pve1 | 192.168.40.10 | ELK Stack — Elasticsearch, Kibana, Logstash, Fleet Server, Elastic Agent |
| 101 | ELKFleetServer | pve1 | 192.168.40.20 | OpenVAS/Greenbone (Docker Compose) |
| 102 | HashiCorpVault | pve2 | 192.168.40.30 | HashiCorp Vault PKI |
| 103 | AD-DomainController | pve2 | 192.168.60.10 | Windows Server 2025 Domain Controller |

---

## ELK Stack — 192.168.40.10

### Services
| Service | Port | Protocol |
|---|---|---|
| Elasticsearch | 9200 | HTTPS |
| Kibana | 5601 | HTTPS |
| Logstash Beats input | 5044 | TLS |
| Fleet Server | 8220 | HTTPS |

### Elasticsearch SSL
- Certificate: Vault-issued `elk-stack_crt.pem` (signed by Homelab-Intermediate-CA)
- CA fingerprint: stored in `/etc/kibana/kibana.yml` as `ca_trusted_fingerprint`
- Cert files: `/etc/elasticsearch/certs/` and `/etc/ssl/homelab/`

### Fleet Server
- Policy ID: `fleet-server-policy`
- Service token: generated via Elasticsearch API
- Re-enrollment is required any time the Elasticsearch HTTP certificate changes

---

## PKI Infrastructure — HashiCorp Vault

### Instance
| Field | Value |
|---|---|
| URL | http://192.168.40.30:8200 |
| Storage backend | File (/opt/vault/data) |
| Auth method | Shamir (5 shares, threshold 3) |
| Version | 2.0.3 |
| Auto-unseal | Not configured |

### CA Hierarchy
```
Homelab-Root-CA          (RSA-4096, 10 year)
  └── Homelab-Intermediate-CA   (RSA-4096, 5 year)
        └── Leaf certificates          (RSA-4096, 1 year)
```

### Vault PKI Paths
| Path | Purpose |
|---|---|
| `pki/` | Root CA |
| `pki_int/` | Intermediate CA |
| `pki_int/roles/homelab-role` | Leaf certificate issuance |

The homelab-role permits any common name, any IP SAN, and subdomains of `zmh.homelab`, `zmh.local`, and `homelab`.

### Root CA
- CN: Homelab-Root-CA
- Valid: Jul 26 2026 – Jul 23 2036
- Trust store: Windows `Cert:\LocalMachine\Root`

### Intermediate CA
- CN: Homelab-Intermediate-CA
- Valid: Jul 26 2026 – Jul 25 2031
- Trust store: Windows `Cert:\LocalMachine\CA`

### Bulk Certificate Issuance
```bash
export VAULT_ADDR='http://192.168.40.30:8200'
vault login

cat > /tmp/issue_certs.sh << 'EOF'
#!/bin/bash
export VAULT_ADDR='http://192.168.40.30:8200'
mkdir -p /tmp/certs

issue_cert() {
  NAME=$1; IP=$2; CN=$3
  vault write -format=json pki_int/issue/homelab-role \
    common_name="$CN" ip_sans="$IP" ttl="8760h" > /tmp/certs/${NAME}.json
  jq -r '.data.certificate' /tmp/certs/${NAME}.json > /tmp/certs/${NAME}_crt.pem
  jq -r '.data.private_key' /tmp/certs/${NAME}.json > /tmp/certs/${NAME}_key.pem
  jq -r '.data.issuing_ca' /tmp/certs/${NAME}.json > /tmp/certs/${NAME}_ca.pem
}

issue_cert "opnsense"     "192.168.1.1,192.168.99.1"  "opnsense.zmh.homelab"
issue_cert "proxmox-pve1" "192.168.99.2"               "pve1.zmh.homelab"
issue_cert "proxmox-pve2" "192.168.99.50"              "pve2.zmh.homelab"
issue_cert "elk-stack"    "192.168.40.10"              "elk.zmh.homelab"
issue_cert "vault"        "192.168.40.30"              "vault.zmh.homelab"
issue_cert "openvas"      "192.168.40.20"              "openvas.zmh.homelab"

vault read -field=certificate pki/cert/ca > /tmp/certs/homelab-root-ca.pem
EOF
chmod +x /tmp/issue_certs.sh
bash /tmp/issue_certs.sh
```

---

## TLS Certificate Deployment

### Current Status
| Service | Certificate | Status |
|---|---|---|
| Elasticsearch | elk-stack_crt.pem | Active |
| Kibana | elk-stack_crt.pem | Active |
| Logstash (Beats input) | elk-stack_crt.pem | Active |
| Fleet Server | Internal PKI (re-enrolled) | Active |
| OPNsense web UI | opnsense_crt.pem | Active |
| Proxmox pve1 | proxmox-pve1_crt.pem | Active |
| Proxmox pve2 | proxmox-pve2_crt.pem | Active |
| OpenVAS/nginx | openvas_crt.pem | Active |

### Certificate File Locations — ELK VM (192.168.40.10)
| File | Path | Owner | Mode |
|---|---|---|---|
| ELK cert | `/etc/ssl/homelab/elk-stack_crt.pem` | root:logstash | 640 |
| ELK key | `/etc/ssl/homelab/elk-stack_key.pem` | root:logstash | 640 |
| CA chain | `/etc/ssl/homelab/homelab-root-ca.pem` | root:logstash | 640 |
| Elasticsearch cert copy | `/etc/elasticsearch/certs/elk-stack_crt.pem` | root:elasticsearch | 640 |
| Elasticsearch key copy | `/etc/elasticsearch/certs/elk-stack_key.pem` | root:elasticsearch | 640 |
| Kibana cert | `/etc/kibana/kibana-cert.pem` | root:kibana | 640 |
| Kibana key | `/etc/kibana/kibana-key.pem` | root:kibana | 640 |
| Kibana CA | `/etc/kibana/homelab-root-ca.pem` | root:kibana | 640 |

### Certificate File Locations — OPNsense
| File | Path |
|---|---|
| CA chain (intermediate + root concatenated) | `/usr/local/etc/beats/certs/Homelab-CA_crt.pem` |

### Proxmox Certificate Deployment
```bash
# pve1
scp proxmox-pve1_crt.pem root@192.168.99.2:/etc/pve/nodes/homelabprox/pveproxy-ssl.pem
scp proxmox-pve1_key.pem root@192.168.99.2:/etc/pve/nodes/homelabprox/pveproxy-ssl.key
systemctl restart pveproxy

# pve2
scp proxmox-pve2_crt.pem root@192.168.99.50:/etc/pve/nodes/pve2/pveproxy-ssl.pem
scp proxmox-pve2_key.pem root@192.168.99.50:/etc/pve/nodes/pve2/pveproxy-ssl.key
systemctl restart pveproxy
```

### OpenVAS Certificate Deployment
```bash
CERT_VOL=/var/lib/docker/volumes/greenbone-community-edition_nginx_certificates_vol/_data
sudo cp openvas_crt.pem $CERT_VOL/server.cert.pem
sudo cp openvas_key.pem $CERT_VOL/server.key
sudo docker restart greenbone-community-edition-nginx-1
```

### Windows Trust Store
```powershell
Import-Certificate -FilePath "homelab-root-ca.pem" -CertStoreLocation Cert:\LocalMachine\Root
Import-Certificate -FilePath "elk-stack_ca.pem" -CertStoreLocation Cert:\LocalMachine\CA
```

### CA Chain File Creation
```powershell
Get-Content "elk-stack_ca.pem", "homelab-root-ca.pem" | Set-Content "homelab-ca-chain.pem"
```

### Fleet Server Re-Enrollment
```bash
# Generate new service token
curl -u elastic:<password> -X POST \
  "https://192.168.40.10:9200/_security/service/elastic/fleet-server/credential/token/<token-name>" -k

# Re-enroll
sudo elastic-agent enroll \
  --fleet-server-es=https://192.168.40.10:9200 \
  --fleet-server-service-token=<token> \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-ca-trusted-fingerprint=<sha256-fingerprint> \
  --fleet-server-port=8220 \
  --fleet-server-es-insecure \
  --force

sudo systemctl restart elastic-agent
```

### Get Elasticsearch Certificate Fingerprint
```bash
sudo openssl x509 -in /etc/elasticsearch/certs/elk-stack_crt.pem \
  -noout -fingerprint -sha256 | sed 's/://g' | cut -d= -f2 | tr '[:lower:]' '[:upper:]'
```

---

## Pipeline Architecture

```
OPNsense (Filebeat)
    |
    | TLS — port 5044
    v
Logstash (192.168.40.10)
    |-- suricata + alert      --> suricata-alerts-YYYY.MM.dd
    |-- suricata (other)      --> suricata-logs-YYYY.MM.dd
    |-- firewall              --> firewall-logs-YYYY.MM.dd
    |-- dns                   --> dns-logs-YYYY.MM.dd
    |-- dhcp                  --> dhcp-logs-YYYY.MM.dd
    |
    | HTTPS
    v
Elasticsearch (192.168.40.10:9200)
    |
    v
Kibana (https://192.168.40.10:5601)

Elastic Agent (Fleet Server policy — Ubuntu VM)
    |-- system logs, auth, syslog --> logs-*
    |-- system metrics            --> metrics-*
    v
Elasticsearch via Fleet Server (port 8220)

Elastic Agent (Windows-EndPoints policy — Main PC)
    |-- Windows Event Logs (Security, System, Application)
    |-- System metrics
    v
Fleet Server → Elasticsearch
```

---

## Key Configuration Files

### Logstash — `/etc/logstash/conf.d/suricata.conf`
**Input:**
```
beats {
  port => 5044
  host => "0.0.0.0"
  ssl_enabled => true
  ssl_certificate => "/etc/ssl/homelab/elk-stack_crt.pem"
  ssl_key => "/etc/ssl/homelab/elk-stack_key.pem"
  ssl_certificate_authorities => ["/etc/ssl/homelab/homelab-root-ca.pem"]
  ssl_verify_mode => "peer"
  ssl_supported_protocols => ["TLSv1.3"]
}
```

**Output:**
```
elasticsearch {
  hosts => ["https://192.168.40.10:9200"]
  user => "elastic"
  password => "<ELASTIC_PASSWORD>"
  ssl_certificate_verification => false
  index => "suricata-alerts-%{+YYYY.MM.dd}"
}
```

### OPNsense Filebeat — `/usr/local/etc/beats/filebeat.yml`
```yaml
output.logstash:
  hosts: ["192.168.40.10:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/usr/local/etc/beats/certs/Homelab-CA_crt.pem"]
```

### Kibana — `/etc/kibana/kibana.yml`
```yaml
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/kibana-cert.pem
server.ssl.key: /etc/kibana/kibana-key.pem
elasticsearch.hosts: [https://192.168.40.10:9200]
elasticsearch.ssl.certificateAuthorities: [/etc/kibana/homelab-root-ca.pem]
xpack.fleet.outputs: [{..., ca_trusted_fingerprint: <ES_CA_FINGERPRINT>}]
xpack.encryptedSavedObjects.encryptionKey: "<KIBANA_ENCRYPTION_KEY>"
```

### Elasticsearch — `/etc/elasticsearch/elasticsearch.yml`
```yaml
xpack.security.http.ssl:
  enabled: true
  certificate: /etc/elasticsearch/certs/elk-stack_crt.pem
  key: /etc/elasticsearch/certs/elk-stack_key.pem

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12

xpack.security.http.ssl.client_authentication: none
xpack.security.http.ssl.supported_protocols: [TLSv1.3]
```

---

## OPNsense Filebeat

### Update Survival
OPNsense updates regenerate the active Filebeat config from a template. All persistent changes must be made to the template, not the output file.

| File | Path |
|---|---|
| Template | `/usr/local/opnsense/service/templates/OPNsense/Beats/filebeat.yml` |
| Active config | `/usr/local/etc/beats/filebeat.yml` |

After editing the template:
```bash
configctl template reload OPNsense/Beats
```

The active config is locked with BSD immutable flags to prevent silent overwrites:
```bash
chflags schg /usr/local/etc/beats/filebeat.yml    # lock
chflags noschg /usr/local/etc/beats/filebeat.yml  # unlock
```

### Post-Update Recovery
```bash
chflags noschg /usr/local/etc/beats/filebeat.yml
vi /usr/local/etc/beats/filebeat.yml
chflags schg /usr/local/etc/beats/filebeat.yml
service filebeat restart
filebeat test output -c /usr/local/etc/beats/filebeat.yml
```

---

## Backup Infrastructure

### Hardware
| Component | Detail |
|---|---|
| Drive | Seagate ST500LM021 500GB HDD — pve2 `/dev/sda` |
| ZFS Pool | `backup-pool` (single disk) |
| Mount point | `/backup-pool` |
| NFS export | pve2 exports `/backup-pool/dump` to pve1 |
| pve1 mount | `/backup-pool/dump` (fstab) |

### Backup Jobs
| VMID | VM | Node | Schedule | Retention |
|---|---|---|---|---|
| 102 | HashiCorpVault | pve2 | 00:00 daily | Daily 7, Weekly 4 |
| 100 | UbuntuSuricata (ELK) | pve1 | 01:00 daily | Daily 7, Weekly 4 |

### Compressed Backup Sizes (ZSTD)
| VM | Provisioned | Compressed |
|---|---|---|
| Vault | 32GB | ~3.9GB |
| ELK | 112GB | ~23GB |

### ZFS Mirror Upgrade
When 4TB drives are added, convert the single-disk pool to a mirror without data loss:
```bash
zpool attach backup-pool sda /dev/sdb
```

### Backup Verification
```bash
zpool list backup-pool
zpool status backup-pool
du -sh /backup-pool/dump/*
```

---

## Active Directory

### Domain Controller — VM 103
| Field | Value |
|---|---|
| Domain | zmh.homelab |
| NetBIOS | ZMH |
| DC hostname | WIN-HEKTRP8ODQR.zmh.homelab |
| IP | 192.168.60.10 |
| VLAN | 60 |
| OS | Windows Server 2025 |
| Forest/Domain level | Windows Server 2016 |

### FSMO Roles
All five FSMO roles held by WIN-HEKTRP8ODQR:
- Schema Master
- Domain Naming Master
- PDC Emulator
- RID Pool Manager
- Infrastructure Master

### DNS
- DC serves as DNS for the `zmh.homelab` zone
- DNS forwarder configured to OPNsense (192.168.60.1)
- NTP synced to `time.windows.com`

### DC Health Check Commands
```powershell
dcdiag /q
nslookup -type=srv _ldap._tcp.dc._msdcs.zmh.homelab
w32tm /query /status
netdom query fsmo
```

---

## Elasticsearch Index Management

### ILM Policy
- Policy: `homelab-30-day`
- Retention: 30 days
- Applied to: `suricata-alerts-template`, `suricata-logs-template`, `firewall-logs-template`

### Index Patterns
| Index | Source | Ingest Path |
|---|---|---|
| `suricata-alerts-*` | OPNsense Suricata | Filebeat → Logstash |
| `suricata-logs-*` | OPNsense Suricata | Filebeat → Logstash |
| `firewall-logs-*` | OPNsense filter | Filebeat → Logstash |
| `dns-logs-*` | OPNsense DNS | Filebeat → Logstash |
| `dhcp-logs-*` | OPNsense DHCP | Filebeat → Logstash |
| `logs-*` | Ubuntu system | Elastic Agent |
| `metrics-*` | Ubuntu metrics | Elastic Agent |

---

## Pending Tasks

| Priority | Task |
|---|---|
| High | Migrate ELK (VM 100) from pve1 to pve2 |
| High | Vault auto-unseal configuration |
| Medium | Windows 11 client VM — domain join, Sysmon, Elastic Agent |
| Medium | OpenVAS backup job |
| Medium | 4TB HDD pair — ZFS mirror upgrade |
| Low | SPAN port configuration on Cisco 3560CX |
| Low | Logstash deprecation warnings — replace ssl_verify_mode |
| Low | GitHub README update |

---

## Reference Commands

### Pipeline Health
```bash
curl -u elastic:<password> https://192.168.40.10:9200 -k
curl -u elastic:<password> "https://192.168.40.10:9200/_cat/indices?v&s=index" -k
sudo systemctl status logstash
sudo tail -50 /var/log/logstash/logstash-plain.log
sudo /opt/Elastic/Agent/elastic-agent status
filebeat test output -c /usr/local/etc/beats/filebeat.yml
```

### Service Restart Order
```bash
sudo systemctl restart elasticsearch
sudo systemctl restart logstash
sudo systemctl restart kibana
sudo systemctl restart elastic-agent
```

### Logstash — Force Kill
```bash
sudo pkill -9 -f logstash && sudo systemctl start logstash
```

### OPNsense Filebeat
```bash
service filebeat status
service filebeat restart
filebeat test output -c /usr/local/etc/beats/filebeat.yml
```

### Vault
```bash
export VAULT_ADDR='http://192.168.40.30:8200'
vault status
vault login
vault operator unseal
```

### Proxmox Cluster
```bash
pvecm status
pvecm nodes
```

### pve2 Bridge VLAN Persistence
```bash
# /etc/network/interfaces — post-up entries under vmbr0
post-up bridge vlan add vid 40 dev nic0
post-up bridge vlan add vid 40 dev vmbr0 self
post-up bridge vlan add vid 60 dev nic0
post-up bridge vlan add vid 60 dev vmbr0 self
```
