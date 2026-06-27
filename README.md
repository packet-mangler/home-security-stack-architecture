# home-security-stack

Production-grade home network security stack, documented as a portfolio project. Built and maintained by a senior security engineer with 15+ years of blue team experience.

This repo contains architecture diagrams, firewall policy documentation, and deployment patterns. It is intentionally sanitized for public consumption — specific IPs, device names, and hardware models are genericized.
---

## Stack overview

| Layer | Component | Role |
|---|---|---|
| Edge firewall | OPNsense (Protectli Vault) | Stateful inspection, IPS, DNS, VPN |
| IDS/IPS | Suricata | Inline threat detection, EVE JSON logging |
| DNS | Unbound + Cloudflare DoT | DNSSEC validation, malware-blocking upstream |
| Flow analysis | ntopng Community | Traffic visibility, alert generation |
| SIEM | Splunk Free | Log aggregation, detection engineering |
| Network segmentation | 802.1Q VLANs (managed switch) | Microsegmentation — Office / Lab / Services / IoT |
| Secrets management | OpenBao | Container secret management with transit auto-unsel |
| Container orchestration | Portainer | Docker management |
| Compute | Proxmox VE | VM host for Splunk, Docker, Kali, Windows |
| Storage / syslog relay | Synology NAS | Primary storage, syslog relay, interim Docker host |
| VPN | ProtonVPN (WireGuard) | Selective egress routing |
---

## Network segmentation

Four isolated segments enforced by OPNsense:

| VLAN | Purpose | Trust level |
|---|---|---|
| VLAN 10 — Office | Trusted wired workstations | High |
| VLAN 20 — Lab | Proxmox host + AI inference workstation | Medium — isolated from Office |
| VLAN 30 — Services | NAS, syslog relay, Docker containers | Restricted egress allowlist |
| IoT | Smart home devices (direct OPNsense interface) | Internet only, fully isolated |

Inter-VLAN policy is documented in `docs/policy/VLAN_Policy_Matrix.xlsx`.
---
## Syslog pipeline
```
OPNsense (filterlog local0, Suricata EVE local5)
    │
    ├── UDP/514 ──► NAS (Synology Log Center)
    │                    │
    │               filtered relay ──► Splunk Free VM (500 MB/day cap managed at relay)
    │
ntopng ── UDP/1514 ──► NAS (separate port to avoid facility code conflicts)
```
---

## Key design decisions

- **Syslog port isolation**: OPNsense and ntopng use separate UDP ports to the NAS relay. Sharing a single port caused silent discard when facility codes conflicted.
- **AI inference in Lab VLAN**: High-end GPU workstation placed in Lab rather than Office — higher lateral movement risk surface treated accordingly. Firewall rules restrict it to NAS NFS access and internet only.
- **Services VLAN egress allowlist**: NAS outbound is explicitly allowlisted (DSM updates, NTP, ClamAV definitions, DNS). All other outbound denied.
- **Docker host migration complete — dedicated Ubuntu 26.04 VM on Proxmox now hosts Portainer and OpenBao. opnsense-mcp remains on NAS as a Services VLAN resident.
- **Cloud-init templates**: All Linux VMs provisioned from a single Ubuntu 26.04 cloud-init template VMID 9010 (template) and 9011 (gold standard clone). ISO installs reserved for Kali and Windows only.
---

## Docs

```
docs/
  network/
    Home_Network_Topology_Jun2026.drawio   — Full network topology (draw.io)
    Proxmox_Environment.drawio             — Proxmox compute environment detail
  policy/
    VLAN_Policy_Matrix.xlsx                — Inter-VLAN firewall policy matrix,
                                             Services VLAN egress rules,
                                             switch port assignments
```
---

## What's next

- DNSBL deployment (OISD on Unbound)
- Splunk ingestion gap detection (Suricata pipeline monitoring)
- ntopng false positive suppression (OPNsense:53 DNS resolver flows)
- OpenBao secrets engine configuration (AppRole auth for automated workloads)
- Splunk VM deployment
- Let's Encrypt cert automation
---

## Note on sanitization
This is the public version of a private repository. Specific RFC 1918 subnets, device hostnames, and hardware model numbers have been replaced with generic labels. The architecture, policy logic, and operational patterns are unmodified.
