# Homelab Infrastructure Documentation

Personal homelab environment for learning defensive cybersecurity, network architecture, and systems administration.

[![Status](https://img.shields.io/badge/Status-Phase%201%20Complete-success)]()
[![Architecture](https://img.shields.io/badge/Architecture-Dual--Router-blue)]()
[![License](https://img.shields.io/badge/License-MIT-green)]()

---

## 📋 Project Overview

This repository documents the design, implementation, and evolution of my homelab infrastructure. The lab demonstrates practical networking, security, and systems administration skills through hands-on implementation.

**Current Architecture:** Dual-router design with static routing between networks, enabling wireless access to wired lab equipment.

**Key Learning Areas:**
- Inter-network routing and firewall configuration
- pfSense enterprise firewall deployment
- Network segmentation and security zones
- DNS-based ad blocking (Pi-hole)
- Virtualization platform (Proxmox)
- Professional troubleshooting methodology

---

## 🏗️ Current Architecture (Phase 1)

```
Internet (ISP WiFi)
        ↓
   GL.iNet Router
   192.168.8.0/24
        ↓ Static Route
   pfSense Firewall
   WAN: 192.168.8.196
   LAN: 192.168.1.0/24
        ↓
   Lab Network
   ├── Proxmox (192.168.1.10)
   └── Pi-hole (192.168.1.20)
```

**Status:** ✅ Fully operational and validated as optimal architecture for environment

---

## 📚 Documentation

### Main Documentation
- **[Phase 1 Complete Documentation](docs/Phase1-Complete-Documentation.md)** - Comprehensive guide including implementation, troubleshooting, and Phase 2 analysis

### Additional Resources
- **[Phase 2 Migration Analysis](docs/Phase2-Migration-Plan.md)** - Attempted migration and architectural constraint discovery
- **[Quick Start Guide](docs/Quick-Start.md)** - Essential commands and access information

---

## 💻 Hardware

| Device | Model | Role | IP Address |
|--------|-------|------|------------|
| Router | GL.iNet GL-BE3600 | ISP connectivity + WiFi | 192.168.8.1 |
| Firewall | Lenovo M910q | pfSense firewall/router | 192.168.1.1 |
| Hypervisor | Lenovo M720q | Proxmox VE virtualization | 192.168.1.10 |
| DNS/Ad-Block | Raspberry Pi | Pi-hole DNS filtering | 192.168.1.20 |
| Switch | Netgear | Network connectivity | - |

---

## 🎯 Key Accomplishments

### Phase 1 Implementation
- ✅ Static routing between separate networks (192.168.8.0/24 ↔ 192.168.1.0/24)
- ✅ pfSense firewall with custom rules for inter-network communication
- ✅ Wireless access to wired lab infrastructure
- ✅ Pi-hole DNS filtering deployment
- ✅ Proxmox virtualization platform setup

### Phase 2 Analysis
- ✅ Identified Layer 2 bridging constraints with WiFi-only ISP
- ✅ Root cause analysis of architectural limitations
- ✅ Validated Phase 1 as optimal design for physical environment
- ✅ Demonstrated professional decision-making (rollback vs. persistence)

---

## 🔧 Technologies & Skills

**Networking:**
- Static routing and route tables
- Firewall rule creation and management
- Network Address Translation (NAT)
- DHCP and DNS configuration
- OSI Layer 2/3 understanding

**Systems:**
- pfSense (FreeBSD-based firewall)
- Proxmox VE (Debian-based hypervisor)
- Pi-hole (DNS-based ad blocking)
- Raspberry Pi OS (Debian-based)
- OpenWRT (GL.iNet firmware)

**Security:**
- Network segmentation
- Firewall policy implementation
- Log analysis and monitoring
- Defense-in-depth principles

**Professional Skills:**
- Technical documentation
- Troubleshooting methodology
- Backup/restore procedures
- Architectural analysis
- Root cause analysis

---

## 📖 Learning Highlights

### Technical Discoveries

**WiFi Layer 2 Bridging Limitation:**
During Phase 2 migration attempt, discovered that consumer WiFi devices operating in client mode cannot provide true Layer 2 bridging required for pfSense WAN connectivity. This validated Phase 1 dual-router architecture as the correct solution for WiFi-only ISP environments.

**Static Routing Implementation:**
Successfully implemented persistent static routes using OpenWRT's UCI system, enabling seamless wireless access to separate wired network without VPN overhead.

**Firewall Context Awareness:**
Learned that pfSense's default "Block Private Networks" setting on WAN interface, while correct for internet-facing deployments, must be disabled when WAN connects to trusted upstream private network.

### Process Learnings

- **Always backup before major changes** - pfSense configuration backup enabled clean rollback
- **Validate assumptions early** - Testing Layer 2 connectivity requirements prevented wasted effort
- **Document failures, not just successes** - Phase 2 analysis demonstrates professional troubleshooting
- **Recognize physical constraints** - Not every problem is solvable with better configuration

---

## 🚀 Future Enhancements

### Immediate (Planned)
- [ ] Configure GL.iNet DNS to use Pi-hole (network-wide ad blocking)
- [ ] Update pfSense to latest version (2.8.1)
- [ ] Tighten firewall rules (least privilege principle)
- [ ] Implement monitoring and alerting

### Short-term
- [ ] WireGuard VPN for remote access
- [ ] Grafana dashboard for network monitoring
- [ ] Additional VMs on Proxmox (Docker, dev environment)
- [ ] Automated backup procedures

### Long-term
- [ ] VLAN segmentation on lab network
- [ ] IDS/IPS implementation (Suricata)
- [ ] Advanced logging and SIEM exploration
- [ ] High availability testing

---

## 🎓 Career Relevance

This homelab demonstrates skills directly applicable to:
- **Network Engineer** - Routing, firewalling, network design
- **Security Analyst** - Defense-in-depth, log analysis, threat detection
- **Systems Administrator** - Linux administration, service deployment
- **DevOps Engineer** - Infrastructure as code, automation potential

**Interview Talking Points:**
- Implemented inter-network routing with static routes
- Configured enterprise firewall (pfSense) with custom rule sets
- Identified and documented architectural constraints (Layer 2 bridging)
- Demonstrated professional troubleshooting and rollback procedures
- Created comprehensive technical documentation

---

## 📝 Architecture Decision Records

### Why Dual-Router Architecture?

**Context:** ISP connection is WiFi-only, physically distant from lab location.

**Decision:** Maintain dual-router architecture (GL.iNet + pfSense) rather than single-router design.

**Rationale:**
- WiFi client devices cannot provide Layer 2 bridging required for pfSense WAN
- Running ethernet cable to ISP not feasible (distance, permissions)
- Dual-router provides network segmentation and firewall protection
- Appropriate trade-off: slight complexity vs. physical impossibility

**Alternatives Considered:**
- pfSense as edge router (requires wired WAN - not available)
- VPN to lab (adds latency, doesn't solve management access)
- Move equipment to ISP location (defeats purpose of dedicated lab space)

**Status:** Validated as optimal solution. See [Phase 2 Analysis](docs/Phase1-Complete-Documentation.md#7-phase-2-attempt--analysis-critical-section) for details.

---

## 🤝 Contributing

This is a personal learning project, but feedback and suggestions are welcome! Feel free to:
- Open an issue for questions or discussion
- Share your own homelab experiences
- Suggest improvements to documentation

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Documentation and configuration examples are provided as-is for educational purposes.

---

## 🔗 Connect

- **GitHub:** https://github.com/sabrinavsimmons
- **LinkedIn:** https://www.linkedin.com/in/sabrina-simmons-830095a1


---

## 📊 Project Timeline

- **January 11, 2026** - Phase 1 implementation completed (~8 hours)
- **January 13, 2026** - Phase 2 attempt and architectural analysis (~3 hours)
- **January 13, 2026** - Documentation finalized and published

**Total Investment:** ~11 hours of hands-on implementation and learning

---

## 🙏 Acknowledgments

- pfSense community for excellent documentation
- OpenWRT project for flexible router firmware
- Pi-hole team for DNS-based ad blocking solution
- Various online resources and homelab communities

---

**Built with curiosity, documented with care, shared for learning.**
