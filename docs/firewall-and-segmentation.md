# Firewall Policy & Network Segmentation

[← Back to main README](../README.MD) · [Documentation index](README.md)

OPNsense is the central Layer 3 policy-enforcement point for the lab. The firewall design follows a consistent pattern: required services are explicitly permitted, internal trust is constrained, meaningful denies are logged, and Kali remains isolated in a dedicated red-team segment.

## Policy Pattern

The final rulesets generally follow this order:

1. **Specific required service exceptions**
2. **Block to `Protected_Networks`**
3. **Limited Internet services where required**
4. **Logged default deny**

OPNsense evaluates rules on the ingress interface. Because the firewall is stateful, allowed connections do not require redundant return-path rules on the destination VLAN.

Routine successful traffic is not logged unless it is useful to a specific security workflow. High-value deny rules are logged so unexpected cross-zone behavior can be investigated in Elastic.

## Protected Networks Alias

A `Protected_Networks` alias groups the internal routed VLAN subnets. This provides a maintainable way to express broad internal-deny behavior without duplicating individual destination networks in every ruleset.

The design goal is simple: **exceptions are narrow and intentional; broad internal access is not the default.**

## VLAN 40 — Security Services

The security segment contains ELK, OpenVAS, and Vault. OpenVAS receives explicit exceptions to scan authorized targets, while other security-service traffic is prevented from freely reaching protected internal segments.

![VLAN 40 firewall rules](images/opnsense/vlan40-firewall-rules.webp)

**Rule rationale:** Security tooling is not treated as implicitly trusted just because it resides in the security VLAN. OpenVAS receives narrowly scoped access to authorized scan targets, while ELK, Vault, and other services rely on service-specific flows rather than unrestricted east-west access. Broader traffic to `Protected_Networks` is denied, limiting the ability of a compromised security appliance to become a pivot into the rest of the lab.

## VLAN 50 — Windows Workstation

The Windows workstation network contains ZMH-WS1, a domain-joined Windows 11 system. It requires domain services and telemetry connectivity without broad lateral access to the rest of the environment.

![VLAN 50 firewall rules](images/opnsense/vlan50-firewall-rules.webp)

**Rule rationale:** Required AD/DNS communication to DC01 and Elastic Agent connectivity are explicitly allowed above broader deny rules. Other routed internal access falls through to the `Protected_Networks` block. This reduces unnecessary east-west reachability from a user workstation while preserving the domain and monitoring services it actually needs.

## VLAN 60 — Domain Controller

DC01 resides in a dedicated server segment rather than sharing a general-purpose network with clients or security tools.

![VLAN 60 firewall rules](images/opnsense/vlan60-firewall-rules.webp)

**Rule rationale:** DC01 is treated as a high-value server segment, not as a generally trusted source. Only required infrastructure and telemetry paths are allowed, including the OPNsense DNS/DHCP path and Elastic services. Unnecessary access to other protected networks remains denied, reducing lateral-movement options if the domain-controller segment were ever compromised.

## VLAN 70 — Red Team

Kali is deliberately placed in a separate attacker/validation zone. Offensive testing should exercise the security architecture without weakening it.

![VLAN 70 firewall rules](images/opnsense/vlan70-firewall-rules.webp)

**Rule rationale:** Test-specific exceptions are placed above the broad internal deny so approved workflows—such as scoped RDP validation against ZMH-WS1 and controlled detection tests—can occur without granting VLAN 70 general access to internal systems. Traffic outside those exceptions is denied and logged, making the firewall both a preventive control and a source of validation telemetry in Elastic.

## VLAN 99 — Management

VLAN 99 is reserved for infrastructure administration, including Proxmox, the Cisco switch, and OPNsense management.

![VLAN 99 firewall rules](images/opnsense/vlan99-firewall-rules.webp)

**Rule rationale:** Required administration and infrastructure flows are explicitly permitted before broader protected-network/default-deny rules. Unexpected management-plane egress is logged. This policy previously exposed direct SMTP/25 attempts from a Proxmox host, demonstrating that restrictive management rules can reveal unintended application behavior in addition to preventing access.

## Suricata IDS

Suricata runs on OPNsense in **IDS mode** and forwards alert data into Elastic for investigation and Security detection rules.

![Suricata settings](images/opnsense/suricata-settings.webp)

*Suricata configuration used for network intrusion-detection visibility.*

![Suricata alerts](images/opnsense/suricata-alerts.webp)

*Live Suricata alerts generated by lab traffic.*

The current Suricata deployment is not treated as an inline IPS. Prevention remains the responsibility of OPNsense firewall policy, while Suricata supplies additional network-level evidence and signatures.

### Capture Interfaces vs. `HOME_NET`

The Suricata interface list and `HOME_NET` serve different purposes:

- **Capture interfaces** determine where Suricata observes packets.
- **`HOME_NET`** classifies which addresses signatures should treat as internal.

This distinction matters on a VLAN-aware firewall. The parent trunk interface already sees tagged traffic for several VLANs, so an individual VLAN does not need to be selected as a separate Suricata capture interface simply for its traffic to be visible. Validation showed VLAN 50, VLAN 60, and VLAN 70 traffic arriving through the trunk with the VLAN tag preserved.

The current internal classification includes the routed production/lab networks used by VLANs 10, 20, 30, 40, 50, 60, and 99. VLAN 70 is intentionally excluded from `HOME_NET` so the Kali validation host is treated as external to the protected environment:

```text
HOME_NET     -> trusted/internal lab networks
EXTERNAL_NET -> !$HOME_NET
VLAN 70      -> outside HOME_NET
```

This preserves a useful attacker-versus-target model even though Kali physically resides inside the homelab.

### External-to-Internal ACK Scan Detection

The Emerging Threats rule `ET SCAN NMAP -sA (1)` (`SID 2000538`) is defined for traffic from `EXTERNAL_NET` to `HOME_NET`. A controlled ACK scan from Kali therefore matches when sent from VLAN 70 toward an internal target such as the ELK host on VLAN 40.

The same signature does **not** represent HOME_NET-to-HOME_NET reconnaissance. A scan sourced from an internal workstation toward another internal network falls outside the signature's directionality even if the packet pattern otherwise resembles an Nmap ACK scan.

That behavior led to an explicit split between red-team/external-style reconnaissance and internal reconnaissance instead of trying to make one rule represent both cases.

### Internal Reconnaissance Detection

A custom Suricata rule was added for concentrated TCP SYN scanning between internal hosts:

```text
alert tcp $HOME_NET any -> $HOME_NET any (msg:"HOMELAB Internal TCP SYN Scan - Possible Insider Recon"; flow:stateless; flags:S,CE; dsize:0; threshold:type both, track by_src, count 40, seconds 10; classtype:attempted-recon; sid:9000001; rev:10;)
```

The rule is designed to identify rapid SYN-based service discovery originating from a host that is already inside `HOME_NET`, which makes it useful for insider-threat and lateral-discovery validation. It is mapped in Elastic Security to **MITRE ATT&CK T1046 — Network Service Discovery**.

The active sanitized rule set is published at [Suricata local rules](../configs/sanitized/suricata-local.rules), and the Elastic-side validation workflow is documented in [Detection Engineering & Validation](detection-engineering.md).

## Filebeat Export

OPNsense Filebeat forwards firewall and Suricata telemetry to Logstash over TLS.

![OPNsense Filebeat settings](images/opnsense/filebeat-settings.webp)

The downstream Logstash pipeline separates Suricata alerts, general Suricata events, firewall logs, DNS, DHCP, and OPNsense authentication data into purpose-built indices. See [Elastic & Endpoint Telemetry](elastic-and-endpoint-telemetry.md).

## Firewall Evidence in Elastic

A controlled Kali-to-security-network connection attempt is visible in Kibana with the source, destination, interface, and block action preserved.

![Firewall block VLAN70](images/kibana/firewall-block-vlan70.webp)

![Firewall block VLAN70 to VLAN40](images/kibana/firewall-block-vlan70-to-vlan40.webp)

This demonstrates the intended closed loop: **policy blocks the traffic, logging preserves the event, and Elastic makes the enforcement visible for investigation.**
