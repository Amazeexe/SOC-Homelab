# Infrastructure, Proxmox & Switching

[← Back to main README](../README.MD) · [Documentation index](README.md)

This page documents the physical and virtualization layer underneath the security architecture. The project intentionally keeps the root README focused on security outcomes while preserving the infrastructure context here.

## Physical Path

```text
ISP / ONT
    |
    v
Bare-metal OPNsense firewall
    |
    | 802.1Q trunk
    v
Cisco Catalyst 3560CX
    |
    +--> physical access ports
    +--> Proxmox trunk links
    |
    +--> pve1
    +--> pve2
```

OPNsense provides routed VLAN gateways and firewall enforcement. The Cisco switch provides physical Layer 2 segmentation and trunk/access-port assignments. Proxmox VLAN-aware bridges carry tagged VM networks without moving Layer 3 policy enforcement away from OPNsense.

## Infrastructure Systems

| System | Role | Management Address | Current Lab Role |
|---|---|---|---|
| OPNsense | Bare-metal firewall/router | `192.168.99.1` management | Routing, firewall policy, Suricata, DHCP relay, Filebeat |
| Cisco 3560CX | Managed switch | `192.168.99.3` | 802.1Q trunks and access VLANs |
| pve1 | Proxmox VE hypervisor | `192.168.99.2` | Hosts Kali and OpenVAS workloads |
| pve2 | Proxmox VE hypervisor | `192.168.99.50` | Hosts ELK, Vault, DC01, and ZMH-WS1 workloads |

Exact CPU/RAM/storage component inventories change as the lab is upgraded, so this documentation focuses on the stable architectural roles rather than freezing temporary hardware specifications into the security design.

## Proxmox Cluster

The environment runs across two Proxmox nodes. VLAN-aware Linux bridges carry tagged VM interfaces while VLAN 99 remains dedicated to infrastructure management.

![Proxmox cluster](images/proxmox/cluster-gui.webp)

The separation between VM VLANs and the management plane means a workload does not need to share the same trust zone as the hypervisor interface hosting it.

## VM Placement

Current workload placement distributes the lab across both nodes:

```text
pve1
  |-- Kali          VLAN 70
  `-- OpenVAS       VLAN 40

pve2
  |-- ELK           VLAN 40
  |-- Vault         VLAN 40
  |-- ZMH-WS1       VLAN 50
  `-- DC01          VLAN 60
```

This layout also makes service dependencies visible during maintenance or recovery: the ELK/PKI/identity stack spans separate logical security zones even when several workloads reside on the same physical node.

## Cisco 3560CX

The Cisco switch provides the physical VLAN trunks and access-port assignments between OPNsense, the Proxmox hosts, and endpoint networks.

![Cisco VLAN brief](images/cisco/vlan-brief.webp)

![Cisco trunk interfaces](images/cisco/trunk-interfaces.webp)

The switch remains Layer 2 for the segmented lab networks, allowing OPNsense to remain the central Layer 3 policy-enforcement point.

## Management Plane

VLAN 99 is reserved for infrastructure administration. Proxmox, switch, and OPNsense management interfaces are separated from user, server, security-service, and red-team workloads.

The management ruleset follows the same least-privilege model as the rest of the lab. Unexpected egress is logged, which has already exposed application behavior that would have been easy to miss on a permissive management network.

See [Firewall Policy & Segmentation](firewall-and-segmentation.md) for the VLAN 99 rule rationale.

## Backups & Recovery

The Proxmox environment uses scheduled VM backups so service recovery does not depend entirely on reconstructing every guest from memory.

A real storage failure on pve2 provided an unplanned recovery exercise. Rebuilding the node required more than restoring virtual disks: services such as Vault and Elastic depended on certificate trust, enrollment state, and inter-service relationships that also had to be validated after recovery.

The recovery process reinforced several infrastructure lessons:

- VM backups are necessary but do not capture every operational dependency by themselves.
- PKI and service trust relationships need documentation and validation after restore/rebuild events.
- Centralized monitoring should be checked after infrastructure recovery so a restored service is not assumed healthy simply because it boots.
- Separating management, workload, and security networks makes post-recovery connectivity tests more deliberate.

## Infrastructure Documentation Roadmap

As physical hardware changes, this page can be expanded with stable information such as:

- chassis/server models,
- CPU and memory configurations,
- NIC models and interface assignments,
- storage pools and backup destinations,
- UPS/power design,
- physical rack/cabling layout.

Those details will be added when they represent the current long-term build rather than temporary upgrade plans.
