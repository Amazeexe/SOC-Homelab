# Active Directory & Group Policy Security Deployment

[← Back to main README](../README.MD) · [Documentation index](README.md)

The Windows portion of the lab uses the `zmh.homelab` Active Directory domain with Windows Server 2025 as DC01 and a Windows 11 domain workstation, ZMH-WS1.

## Domain Controller

DC01 provides Active Directory Domain Services and DNS for the Windows lab.

![Active Directory domain controller OU](images/windows/aduc-domain-controller.webp)

## Domain Workstation

ZMH-WS1 is a domain-joined Windows 11 workstation on VLAN 50.

![Active Directory computer OU](images/windows/aduc-computer-ou.webp)

## Group Policy Security Deployment

Active Directory is also used as a security-management mechanism, not only for identity and authentication.

Separate Group Policy Objects deploy:

- **Elastic Agent / Elastic Defend** to scoped Windows systems.
- **Sysmon** to scoped Windows systems.

This provides repeatable endpoint instrumentation and reduces configuration drift compared with manual per-host installation. New scoped systems can inherit the same monitoring baseline through domain policy.

The resulting telemetry is centralized in Elastic alongside firewall, Suricata, Linux authentication, and detection data. See [Elastic SIEM & Endpoint Telemetry](elastic-and-endpoint-telemetry.md).

## Kali-to-Workstation Validation

The Kali red-team host is isolated on VLAN 70 and receives only explicit testing exceptions. Before beginning domain discovery, reachability to the approved Windows workstation target was validated from Kali.

An Nmap check confirmed TCP/3389 was reachable on ZMH-WS1, after which FreeRDP was used to establish the authorized session.

![Kali Nmap and FreeRDP command](images/kali/Kali-To-Win11-Nmap-andFreeRDP-Command.png)

*Connectivity validation to ZMH-WS1 followed by the FreeRDP command used to initiate the session.*

![Successful FreeRDP connection](images/kali/End_of_FreeRDP_Successful_Connection.png)

*Successful end state of the Kali-to-ZMH-WS1 RDP session.*

This validates both the intended firewall exception and the fact that the attacker VLAN is not given unrestricted access to the entire workstation network.

## Domain Discovery from ZMH-WS1

From the workstation session, `nltest` and DNS SRV lookups were used to validate Active Directory service discovery.

The workflow confirmed:

- workstation domain identity,
- discovery of `DC01.zmh.homelab`,
- DC address `192.168.60.10`,
- LDAP/Kerberos/GC/DNS roles,
- `_ldap._tcp.dc._msdcs.zmh.homelab` service discovery.

![RDP workstation domain enumeration](images/kali/rdp-ws1-domain-enumeration.webp)

*Domain discovery performed from ZMH-WS1 after the Kali-originated RDP session was established.*

The sequence provides a coherent validation chain:

```text
Kali on VLAN 70
    -> validate TCP/3389 to approved workstation
    -> establish FreeRDP session
    -> operate from domain-joined ZMH-WS1
    -> discover DC01 and AD services
```

## Security Role of Active Directory in the Lab

AD serves three purposes in the project:

1. **Identity and domain services** — AD DS, DNS, Kerberos, LDAP, and workstation membership.
2. **Centralized policy** — Group Policy deploys and standardizes endpoint security tooling.
3. **Purple-team validation target** — controlled discovery and authentication activity creates realistic endpoint/network telemetry for Elastic.

The AD environment is therefore integrated into the broader detection architecture rather than existing as an isolated Windows administration exercise.
