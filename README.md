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
| Secrets management | OpenBao | Transit auto-unseal architecture; KV-v2 secrets engine; userpass and AppRole auth |
| Container orchestration | Portainer | Docker management |
| Compute | Proxmox VE | VM host for Splunk, Docker, Kali, Windows |
| Storage / syslog relay | Synology NAS | Primary storage, syslog relay, bao-transit unseal engine |
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
## Secrets management architecture
OpenBao is deployed in a dual-instance transit auto-unseal pattern across two physical hosts:

```
bao-transit (NAS, Services VLAN)
    │
    └── transit seal ──► bao-primary (docker-host VM, Lab VLAN)
                              │
                              ├── KV-v2 secrets engine (secret/)
                              ├── userpass auth (interactive)
                              └── AppRole auth (automated workloads)
```

**Unseal behavior**: bao-primary auto-unseals on every container or host restart by contacting bao-transit over the Services VLAN. bao-transit requires a single manual unseal only after a NAS reboot (rare). Recovery keys for bao-primary are stored offline for DR.

**Design rationale**: Avoids cloud KMS dependencies while eliminating the manual unseal ceremony on the primary vault. Transit instance separation across physical hosts means a docker-host compromise does not expose the unsealing capability.

---
## Key design decisions
- **Syslog port isolation**: OPNsense and ntopng use separate UDP ports to the NAS relay. Sharing a single port caused silent discard when facility codes conflicted.
- **AI inference in Lab VLAN**: High-end GPU workstation placed in Lab rather than Office — higher lateral movement risk surface treated accordingly. Firewall rules restrict it to NAS NFS access and internet only.
- **Services VLAN egress allowlist**: NAS outbound is explicitly allowlisted (DSM updates, NTP, ClamAV definitions, DNS). All other outbound denied.
- **Docker host migration complete**: Dedicated Ubuntu 26.04 VM on Proxmox (Lab VLAN) now hosts Portainer CE and OpenBao (bao-primary). The NAS retains opnsense-mcp and bao-transit as Services VLAN residents.
- **Cloud-init templates**: All Linux VMs provisioned from Ubuntu 26.04 cloud-init base template (VMID 9010), with a post-configuration gold standard clone (VMID 9011). ISO installs reserved for Kali and Windows only.
- **OpenBao over Infisical**: Infisical's open-core model gates path-level RBAC, automated secret rotation, and audit log streaming behind a paid license. OpenBao (MPLv2, Linux Foundation governed) provides these capabilities unrestricted in the self-hosted deployment.
---
## Docs
```
docs/
  network/
    Home_Network_Topology_Aug2026.drawio   — Full network topology (draw.io)
    Proxmox_Environment.drawio             — Proxmox compute environment detail
  policy/
    VLAN_Policy_Matrix.xlsx                — Inter-VLAN firewall policy matrix,
                                             Services VLAN egress rules,
                                             switch port assignments
```
---
## What's next
- AppRole auth configuration for automated workload secret injection
- Splunk VM deployment (dedicated Lab VLAN VM)
- Suricata ingestion gap detection (pipeline monitoring)
- ntopng false positive suppression (OPNsense:53 DNS resolver flows)
- Port mirror configuration on managed switch
- Let's Encrypt cert renewal automation (labtrash.duckdns.org)
---
## Note on sanitization
This is the public version of a private repository. Specific RFC 1918 subnets, device hostnames, and hardware model numbers have been replaced with generic labels. The architecture, policy logic, and operational patterns are unmodified.
