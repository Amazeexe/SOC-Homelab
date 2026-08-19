# Network Topology Assets

This folder contains the source and exported versions of the Security Homelab network topology used in the portfolio documentation.

![Security Homelab Network Topology](FinalTopology.drawio.png)

- [`FinalTopology.drawio`](FinalTopology.drawio) — editable draw.io source.
- [`FinalTopology.drawio.svg`](FinalTopology.drawio.svg) — scalable vector export.
- [`FinalTopology.drawio.png`](FinalTopology.drawio.png) — raster export used for reliable GitHub rendering.

The diagram represents both **logical VLAN placement** and **Proxmox hypervisor placement**. VLAN 99 is the management plane; workloads shown inside the Proxmox cluster retain their logical placement on their respective VLANs.
