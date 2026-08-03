# CCNP ENCOR 350-401 — Enterprise Network Lab & Automation

A hands-on enterprise networking lab built while preparing for the **Cisco CCNP ENCOR (350-401)** certification. This project pairs a full three-tier enterprise campus topology with a **Python network-automation layer**, so the same environment is used to both study the ENCOR blueprint and automate its deployment.

Built and documented by **Vikramjeet Singh** — M.Eng. Internetworking, Dalhousie University (Halifax, NS).

---

## What this project demonstrates

- Designing and building a hierarchical enterprise campus (core / distribution / access) plus a WAN edge
- Configuring the full CCNP ENCOR feature set: L2 switching, IGP/BGP routing, high availability, network services, and security hardening
- Automating device provisioning, configuration backup, and health checks with **Python (Netmiko, NAPALM, Nornir, Jinja2)**
- Documenting each lab with topology, configuration, verification output, and troubleshooting notes

---

## Two-phase build

### Phase 1 — Platform & Automation
Build one reusable master topology and an automation layer on top of it: a device inventory, a Jinja2-templated base-config deployment script, a Netmiko configuration-backup script, and a device health-check script. Test end to end so the whole network can be provisioned and verified from code.

### Phase 2 — CCNP Feature Labs
Run the full ENCOR lab set on that same topology — using the Phase 1 automation to push and back up configurations — grouped into L2 infrastructure, routing, services & high availability, and security.

---

## Skills covered

| Domain | Technologies |
|--------|-------------|
| **L2 Infrastructure** | VLANs & SVIs, 802.1Q trunking, EtherChannel (LACP / static), STP suite (RSTP, MST, BPDU Guard, Root Guard, UDLD, Loop Guard) |
| **Routing** | EIGRP, OSPFv2 multi-area, OSPF filtering & summarization, OSPFv3 (IPv6 + address families), BGP path selection, Policy-Based Routing |
| **Services & HA** | HSRP / VRRP / GLBP, NTP / PTP, multicast (PIM-SM, IGMP), Flexible NetFlow, IP SLA, RSPAN |
| **Security** | AAA with TACACS+, infrastructure ACLs (iACL), Control Plane Policing (CoPP), IPsec GRE tunnels |
| **Automation** | Python, Netmiko, NAPALM, Nornir, Jinja2, NETCONF / RESTCONF |

---

## Lab environment

- **Network emulation:** EVE-NG / PNETLab
- **Device images:** Cisco IOSv, IOSvL2, CSR1000v
- **Automation host:** Python 3.11 virtual environment (Netmiko, NAPALM, Nornir, ncclient, Jinja2)
- **Tooling:** VS Code, Git

---

## Repository structure
ccnp-encor-lab/
├── diagrams/ # Topology diagrams
├── 01-infrastructure-l2/ # VLANs, trunking, EtherChannel, STP
├── 02-routing/ # EIGRP, OSPF, BGP, PBR
├── 03-services/ # FHRP, NTP/PTP, multicast, NetFlow, IP SLA
├── 04-security/ # AAA/TACACS+, iACL, CoPP, IPsec GRE
├── 05-automation/ # Python automation (inventory, backup, deploy, health-check)
└── README.md

Each module folder contains its own notes: objective, topology, key configuration, verification commands, and lessons learned.

---

## About

**Vikramjeet Singh** — Networking · Automation · Enterprise infrastructure
M.Eng. Internetworking, Dalhousie University · Halifax, NS