# Phase 1: Network Infrastructure & Static Routing

**Status:** ✅ Complete and Validated as Optimal Architecture  
**Duration:** ~8 hours implementation  
**Completion Date:** January 11, 2026  
**Updated:** January 13, 2026 (Phase 2 analysis added)

---

## 📋 Overview

Phase 1 establishes the foundational network infrastructure for the homelab, implementing **static routing** between separate networks to enable wireless access from a MacBook to wired lab equipment. This architecture addresses the physical constraint of WiFi-only ISP connectivity while maintaining enterprise-grade firewall protection.

**Key Achievement:** Successfully deployed dual-router architecture with pfSense firewall, Proxmox virtualization platform, and Pi-hole DNS filtering, all accessible wirelessly via properly configured static routing.

---

## 🎯 Objectives & Outcomes

### Primary Objectives
- ✅ Enable wireless access to wired lab equipment
- ✅ Implement static routing between GL.iNet (192.168.8.0/24) and pfSense LAN (192.168.1.0/24)
- ✅ Deploy pfSense as lab firewall and router
- ✅ Install Proxmox virtualization platform
- ✅ Configure Pi-hole for DNS-based ad blocking

### Learning Outcomes
- **Static Routing:** Inter-network routing using persistent static routes
- **Firewall Management:** pfSense configuration and rule creation
- **Network Segmentation:** Separating networks with different security zones
- **Systems Administration:** Linux configuration, service deployment
- **Troubleshooting Methodology:** Systematic debugging approach

---

## 🏗️ Architecture

### Network Topology
```
Internet (ISP WiFi)
        ↓ WiFi
   GL.iNet Router
   192.168.8.0/24
        ↓ Ethernet
   pfSense Firewall
   WAN: 192.168.8.196
   LAN: 192.168.1.0/24
        ↓ Ethernet
   Netgear Switch
        ├── Proxmox: 192.168.1.10
        └── Pi-hole: 192.168.1.20

MacBook (WiFi) ←→ GL.iNet ←→ Static Route ←→ Lab Network
```

### Component Roles

| Component | Role | IP Addresses | Purpose |
|-----------|------|--------------|---------|
| **GL.iNet Router** | WiFi bridge + upstream router | 192.168.8.1 | Connects to ISP via WiFi, routes to pfSense |
| **pfSense** | Lab firewall/router | WAN: 192.168.8.196<br>LAN: 192.168.1.1 | Firewall protection, DHCP, routing |
| **Proxmox** | Virtualization host | 192.168.1.10 | VM/container platform |
| **Pi-hole** | DNS filtering | 192.168.1.20 | Ad blocking, DNS server |
| **MacBook** | Admin workstation | 192.168.8.160 (DHCP) | Wireless management access |

### Network Details

**GL.iNet Network (192.168.8.0/24):**
- Gateway: 192.168.8.1
- DHCP: Managed by GL.iNet
- Static route to lab network (192.168.1.0/24 via 192.168.8.196)

**Lab Network (192.168.1.0/24):**
- Gateway: 192.168.1.1 (pfSense LAN)
- DHCP: Managed by pfSense
- Firewall: pfSense controls access from upstream network

---

## 🔧 Implementation Details

### 1. Static Route Configuration (GL.iNet)

**Challenge:** Enable wireless access to lab network (192.168.1.0/24) from MacBook on GL.iNet WiFi (192.168.8.0/24).

**Solution:** Configure persistent static route on GL.iNet using OpenWRT's UCI system.

**Implementation:**
```bash
# SSH to GL.iNet
ssh root@192.168.8.1

# Add temporary route (immediate effect)
ip route add 192.168.1.0/24 via 192.168.8.196

# Make persistent across reboots
uci add network route
uci set network.@route[-1].interface='lan'
uci set network.@route[-1].target='192.168.1.0'
uci set network.@route[-1].netmask='255.255.255.0'
uci set network.@route[-1].gateway='192.168.8.196'
uci commit network
/etc/init.d/network reload
```

**Verification:**
```bash
ip route show | grep 192.168.1
# Expected: 192.168.1.0/24 via 192.168.8.196 dev br-lan proto static
```

**How It Works:**

When MacBook (192.168.8.160) accesses Proxmox (192.168.1.10):

1. MacBook checks: Is 192.168.1.10 local? → No
2. MacBook sends packet to default gateway (GL.iNet at 192.168.8.1)
3. GL.iNet checks routing table: Found static route → 192.168.1.0/24 via 192.168.8.196
4. GL.iNet forwards packet to pfSense WAN (192.168.8.196)
5. pfSense firewall checks rules: Allows 192.168.8.0/24 → 192.168.1.0/24
6. pfSense forwards to Proxmox on LAN
7. Response returns via same path in reverse

**Key Learning:** This is fundamental inter-network routing - a core networking concept essential for network engineering roles.

---

### 2. pfSense Firewall Configuration

**Platform:** pfSense 2.7.2 (FreeBSD-based)  
**Hardware:** Lenovo ThinkCentre M910q

#### Critical Setting: Block Private Networks

**Issue:** pfSense blocked all traffic from GL.iNet network (192.168.8.0/24) by default.

**Error in Logs:**
```
Block private networks from WAN block 192.168/16
```

**Root Cause:** Default pfSense WAN security setting blocks RFC1918 private IP addresses (appropriate for internet-facing WAN, incorrect for trusted upstream private network).

**Solution:**

- Navigate: **Interfaces → WAN → Reserved Networks**
- Action: **Unchecked** "Block private networks and loopback addresses"
- Apply changes

**Security Context:** This setting protects against spoofed private IPs from the internet. In our architecture, pfSense WAN connects to GL.iNet (trusted private network), so this protection blocks legitimate traffic.

#### WAN Firewall Rule

**Purpose:** Allow GL.iNet network to access lab resources.

**Configuration:**

| Field | Value |
|-------|-------|
| Interface | WAN |
| Action | Pass |
| Protocol | Any (IPv4) |
| Source | 192.168.8.0/24 |
| Destination | 192.168.1.0/24 |
| Description | Allow GL.iNet network to lab |

**Security Improvement (Future):**

Current rule allows ALL protocols. Should be restricted to specific services:
- TCP 443 (pfSense HTTPS)
- TCP 8006 (Proxmox HTTPS)
- TCP 80 (Pi-hole HTTP)
- UDP 53 (Pi-hole DNS)

This implements the **principle of least privilege**.

---

### 3. Proxmox Deployment

**Platform:** Proxmox VE (Debian-based hypervisor)  
**Hardware:** Lenovo ThinkCentre M720q

**Network Configuration:**

File: `/etc/network/interfaces`
```bash
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

**Apply Configuration:**
```bash
systemctl restart networking
# OR restart Proxmox services
systemctl restart pveproxy
systemctl restart pvedaemon
```

**Access:** https://192.168.1.10:8006

**Initial Challenge:** Proxmox initially reported IP 192.168.8.207 despite being physically connected behind pfSense. Root cause was old network configuration from previous setup. Fixed by reconfiguring `/etc/network/interfaces` with correct IP and gateway.

**Lesson:** Always verify device configuration matches physical topology.

---

### 4. Pi-hole Deployment

**Platform:** Raspberry Pi OS Lite (64-bit)  
**Hardware:** Raspberry Pi (mounted behind rack)

**Installation:**
```bash
curl -sSL https://install.pi-hole.net | bash
```

**Configuration Method:** DHCP Reservation in pfSense (not static IP in OS)

**Why DHCP Reservation:**

Challenge encountered: Raspberry Pi OS network configuration proved inconsistent across multiple methods:
- `/etc/network/interfaces` - Ignored
- `/etc/dhcpcd.conf` - Conflicted with NetworkManager
- NetworkManager - Unreliable across updates

**Solution:** Centralized IP management via pfSense DHCP reservation.

**pfSense DHCP Reservation:**

- Location: **Services → DHCP Server → LAN → DHCP Static Mappings**
- MAC Address: `2c:cf:67:ef:51:3e`
- IP Address: `192.168.1.20`
- Hostname: `pihole`

**Benefits:**
- Centralized management in pfSense
- Reliable across OS updates
- Same result as static IP (consistent address)
- No OS-level configuration complexity

**Web Interface:** http://192.168.1.20/admin

**Lesson:** DHCP reservations are often more reliable than OS-level static IPs for servers/services.

---

## 🔍 Troubleshooting Journey

### Challenge 1: Initial Routing Confusion

**Problem:** Proxmox reported IP 192.168.8.207 despite physical connection behind pfSense.

**Investigation:**
```bash
ip addr show
# Showed: 192.168.8.207/24 (incorrect network)
```

**Root Cause:** Old network configuration from previous setup remained in `/etc/network/interfaces`.

**Solution:** Reconfigured with correct static IP (192.168.1.10) and gateway (192.168.1.1).

**Lesson:** Always verify device configuration matches physical topology. Past configurations can persist.

---

### Challenge 2: pfSense Blocking Private Networks

**Problem:** MacBook could not reach lab devices. Firewall logs showed blocks.

**Firewall Log:**
```
Block private networks from WAN block 192.168/16
Source: 192.168.8.160 → Destination: 192.168.1.10
```

**Investigation:**
1. Verified static route on GL.iNet ✅
2. Verified pfSense WAN interface up ✅
3. Checked pfSense firewall logs → Found blocks ❌

**Root Cause:** pfSense WAN default security setting blocks RFC1918 private IP ranges.

**Solution:** Disabled "Block private networks" on WAN interface (Interfaces → WAN → Reserved Networks).

**Lesson:** Security settings appropriate for internet-facing WAN must be adjusted for private upstream networks. Context matters in firewall configuration.

---

### Challenge 3: Pi-hole Network Configuration

**Problem:** Raspberry Pi refused to accept static IP through multiple configuration methods.

**Attempts:**
1. `/etc/network/interfaces` - Ignored by system
2. `/etc/dhcpcd.conf` - Conflicted with NetworkManager
3. NetworkManager via `nmcli` - Unreliable across reboots

**Root Cause:** Raspberry Pi OS has evolved through different network management systems (ifupdown → dhcpcd → NetworkManager). Multiple configuration files can conflict.

**Solution:** Used DHCP reservation in pfSense instead of OS-level static IP.

**Result:** Reliable, consistent IP address without OS configuration complexity.

**Lesson:** Centralized network management (via DHCP reservations) often provides better reliability than distributed OS-level configurations.

---

### Challenge 4: Raspberry Pi OS Selection

**Problem:** Initially flashed full Raspberry Pi OS with desktop environment.

**Impact:**
- Unnecessary disk space consumption
- Slower boot times
- Unused services consuming resources

**Solution:** Reflashed with **Raspberry Pi OS Lite** (no desktop environment).

**Benefits:**
- Minimal footprint
- Faster boot
- Headless server appropriate for Pi-hole

**Lesson:** Use minimal OS for headless servers. Desktop environments are unnecessary overhead for services.

---

## 🎯 Testing & Verification

### Connectivity Tests

**From MacBook on WiFi (192.168.8.160):**

| Test | Command | Result |
|------|---------|--------|
| Ping pfSense | `ping 192.168.1.1` | ✅ Success |
| Ping Proxmox | `ping 192.168.1.10` | ✅ Success |
| Ping Pi-hole | `ping 192.168.1.20` | ✅ Success |
| pfSense Web | http://192.168.1.1 | ✅ Accessible |
| Proxmox Web | https://192.168.1.10:8006 | ✅ Accessible |
| Pi-hole Web | http://192.168.1.20/admin | ✅ Accessible |

### Routing Verification

**GL.iNet Static Route:**
```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

**Expected Output:**
```
192.168.1.0/24 via 192.168.8.196 dev br-lan proto static
```

**MacBook Network Configuration:**
```bash
# Verify MacBook IP
ifconfig | grep "inet 192"
# Expected: 192.168.8.160 (or similar)

# Verify default gateway
netstat -rn | grep default
# Expected: 192.168.8.1 (GL.iNet)
```

### pfSense Firewall Verification

**Check Firewall Logs:**
- Navigate: **Status → System Logs → Firewall**
- Verify: No blocks for traffic from 192.168.8.0/24 → 192.168.1.0/24
- Confirm: Pass rules are matching

---

## 📊 Phase 2 Analysis: Why This Architecture is Optimal

### Phase 2 Goal (Attempted)

**Objective:** Eliminate dual-router architecture by making pfSense the edge router.

**Desired Architecture:**
```
Internet → GL.iNet (Bridge Mode Only) → pfSense (Edge Router) → Lab Devices
```

**Expected Benefits:**
- Single router (no double NAT)
- Centralized management in pfSense
- Professional enterprise architecture
- Simplified DNS/DHCP

### Actions Taken During Phase 2 Attempt

**Step 1:** Backup pfSense configuration ✅  
**Step 2:** Configure GL.iNet in bridge/AP mode  
**Step 3:** Reconfigure pfSense WAN to DHCP  
**Step 4:** Test connectivity

### Critical Findings

**Finding 1: pfSense Never Received IPv4 Gateway**

**Symptoms:**
- Only IPv6 gateway appeared (WAN_DHCP6)
- No IPv4 gateway (WAN_DHCP)
- NAT table remained empty
- IPv4 internet traffic failed

**Finding 2: Root Cause - Layer 2 Limitation**

**The Problem:**

1. **GL.iNet connects to ISP via WiFi (Layer 3)**
   - Consumer WiFi devices cannot provide true Layer 2 bridging
   - WiFi client mode operates at Layer 3 (network layer)
   - This is a fundamental WiFi protocol limitation

2. **pfSense Requires Layer 2 WAN Connectivity**
   - pfSense WAN needs to see upstream router's Layer 2 frames
   - Required for proper DHCP negotiation and gateway discovery
   - WiFi bridges cannot provide this

3. **Physical Constraint**
   - ISP connection point is distant from lab
   - Running ethernet cable to ISP not feasible
   - WiFi is the only available WAN connection method

**Finding 3: This is NOT a Configuration Error**

**Verified Correct:**
- ✅ pfSense interfaces properly assigned
- ✅ NAT mode set to automatic
- ✅ Firewall rules unchanged from working config
- ✅ DHCP renewal successful
- ✅ No user configuration error

**This is an architectural limitation, not a mistake.**

### Why Phase 1 is the Correct Architecture

**Phase 1 is NOT a workaround - it's the appropriate solution for WiFi-only ISP connectivity.**

**Advantages of Phase 1:**
- ✅ Works reliably with WiFi-only ISP
- ✅ Provides network segmentation (upstream WiFi vs lab network)
- ✅ Enables wireless access to wired lab equipment
- ✅ pfSense protects lab with enterprise firewall
- ✅ Can expand with VLANs on pfSense LAN side
- ✅ ISP router compromise doesn't directly affect lab

**Disadvantages (Minor):**
- Double NAT (minimal impact for most use cases)
- Two configuration points (GL.iNet + pfSense)
- Slightly more complex troubleshooting

**Trade-off Analysis:**

The disadvantages are acceptable compared to the **impossibility** of Phase 2 with current physical constraints. This is sound engineering - choosing the solution that actually works.

### Alternative Solutions (Would Require)

**To implement Phase 2, we would need:**

1. **Wired ethernet to ISP** (not feasible - distance/permissions)
2. **Move pfSense to ISP location** (defeats homelab purpose)
3. **Cellular/fiber WAN connection** (expensive, may not be available)
4. **Different physical location** (not an option)

**Conclusion:** Phase 1 dual-router architecture is the **architecturally correct solution** for WiFi-only ISP environments.

---

## 💡 Key Technical Learnings

### 1. Static Routing Fundamentals

**Concept:** Static routes tell routers how to reach networks they're not directly connected to.

**Implementation:** Manual configuration of routing tables with persistent entries.

**Real-World Application:** Essential for:
- Multi-site networks
- Network segmentation
- Legacy system integration
- Specific traffic routing requirements

**Career Relevance:** Core networking skill for network engineering and security roles.

---

### 2. OSI Layer Understanding

**Layer 2 (Data Link):**
- Ethernet bridging, MAC addresses, switches
- Requires physical or true virtual connectivity
- WiFi in client mode **does not** provide Layer 2 bridging

**Layer 3 (Network):**
- IP routing, subnets, routers
- Where static routing operates
- WiFi operates here in client/router mode

**Application:** Understanding layer limitations prevents wasted effort on impossible architectures.

---

### 3. Firewall Context Awareness

**Principle:** Security settings are context-dependent.

**Example:** "Block Private Networks" on WAN
- **Correct for:** Internet-facing WAN (prevents spoofed private IPs)
- **Incorrect for:** Private upstream WAN (blocks legitimate traffic)

**Application:** Always evaluate default security settings against actual network topology.

---

### 4. Centralized vs. Distributed Management

**DHCP Reservations vs. Static IPs:**

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| DHCP Reservation | Centralized, reliable | Requires DHCP server | Most servers |
| Static IP in OS | No dependency | OS-specific, conflicts | Edge cases only |

**Lesson:** Centralized management (DHCP reservations) often provides better reliability than distributed configurations.

---

### 5. Hardware Limitations are Real

**Key Insight:** Not every problem is solvable with better configuration.

**Examples:**
- WiFi cannot provide Layer 2 bridging for pfSense WAN
- Consumer switches have port mirroring limitations (Phase 3)
- Physical distance constrains cable runs

**Application:** Recognize when constraints are architectural vs. configurational. Adapt architecture to constraints rather than fighting physics.

---

## 🎓 Skills Demonstrated

### Networking
- Static routing configuration and troubleshooting
- Inter-network routing implementation
- Understanding of OSI layers and limitations
- Network topology design and documentation

### Security
- Firewall rule creation and management
- Network segmentation implementation
- Security context evaluation
- Defense-in-depth principles

### Systems Administration
- Linux network configuration
- Service installation (Pi-hole, Proxmox)
- SSH administration
- DHCP/DNS management

### Professional Practices
- Systematic troubleshooting methodology
- Backup/restore procedures
- Architectural constraint analysis
- Professional decision-making (knowing when to roll back)
- Comprehensive technical documentation

---

## 🚀 Future Enhancements

### Immediate

**1. Network-Wide Ad Blocking**

Configure GL.iNet to use Pi-hole as DNS server:
- Log into GL.iNet: http://192.168.8.1
- Navigate: Network → DNS
- Set DNS: 192.168.1.20
- Apply changes

**Result:** All devices on WiFi receive ad-blocking.

**2. Change Pi-hole Admin Password**
```bash
ssh sabri@192.168.1.20
pihole -a -p
# Enter new password when prompted
```

### Short-Term

**1. Update pfSense**
- Current: 2.7.2
- Available: Check for updates
- Backup configuration first!

**2. Tighten Firewall Rules**

Replace broad WAN rule with specific services:
- TCP 443 (pfSense HTTPS)
- TCP 8006 (Proxmox HTTPS)
- TCP 80 (Pi-hole HTTP)
- UDP 53 (Pi-hole DNS)

**3. Document in GitHub**

Create repository with:
- Phase 1 complete documentation
- Phase 2 analysis
- Network diagrams
- Professional README

### Long-Term

**1. WireGuard VPN (pfSense)**
- Remote access to lab
- Secure external connectivity
- Mobile device access

**2. Network Monitoring**
- pfSense logging and analysis
- Grafana dashboard
- Pi-hole statistics
- Alerting for anomalies

**3. VLAN Segmentation (pfSense LAN)**

Even with Phase 1 architecture, implement VLANs:
- VLAN 10: Management (pfSense, Proxmox)
- VLAN 20: Services (Pi-hole, future)
- VLAN 30: Testing/sandbox

**4. IDS/IPS (Suricata on pfSense)**
- Intrusion detection
- Threat monitoring
- Security event logging

**5. Additional Proxmox VMs**
- Docker host
- Development environment
- Database server
- Web server

---

## 📚 Resources & References

**Documentation:**
- [pfSense Official Documentation](https://docs.netgate.com/pfsense/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [OpenWRT Documentation](https://openwrt.org/docs/)

**Community:**
- pfSense subreddit and forum
- Proxmox community forum
- Pi-hole Discourse
- r/homelab

**Networking Fundamentals:**
- RFC1918 - Private IP Address Allocation
- OSI Model reference
- TCP/IP networking concepts

---

## 📊 Project Statistics

**Time Investment:** ~8 hours implementation (Phase 1)

**Infrastructure Deployed:**
- 1 router (GL.iNet)
- 1 firewall (pfSense)
- 1 hypervisor (Proxmox)
- 1 DNS server (Pi-hole)
- 1 managed switch

**Skills Applied:**
- Static routing
- Firewall configuration
- Network troubleshooting
- Linux administration
- Service deployment

**Documentation:** 40+ pages technical documentation

---

## ✅ Success Criteria

- ✅ Wireless access to lab network operational
- ✅ Static routing configured and persistent
- ✅ pfSense firewall protecting lab
- ✅ Proxmox accessible and operational
- ✅ Pi-hole installed and accessible
- ✅ Architecture validated as optimal for constraints
- ✅ Comprehensive documentation complete

---

## 🎓 Conclusion

Phase 1 successfully established the foundational network infrastructure for the homelab, implementing proper static routing to enable wireless access to wired lab equipment. The subsequent Phase 2 analysis validated that this dual-router architecture is not a temporary compromise, but rather the **architecturally correct solution** for WiFi-only ISP connectivity.

**Key Achievements:**

**Technical Implementation:**
- Static routing between separate networks (192.168.8.0/24 ↔ 192.168.1.0/24)
- pfSense firewall with proper rule configuration
- Proxmox virtualization platform deployment
- Pi-hole DNS filtering service

**Professional Skills:**
- Systematic troubleshooting methodology
- Architectural constraint analysis
- Backup/restore procedures
- Knowing when to roll back vs. persist
- Comprehensive technical documentation

**Career Relevance:**

This homelab demonstrates skills directly applicable to network engineering, security operations, and systems administration roles:
- Inter-network routing implementation
- Firewall management and security policy
- Understanding of OSI layers and limitations
- Professional decision-making under constraints
- Clear technical communication

**Looking Forward:**

Phase 1 provides a solid foundation for:
- Network-wide ad blocking via Pi-hole
- VPN access for remote administration
- VLAN segmentation on lab network
- Additional services on Proxmox
- IDS/IPS implementation
- Continued learning and expansion

**This is not the end - it's a strong beginning.**

---

*Last Updated: January 13, 2026*  
*Author: Sabrina Simmons*  
*Repository: [github.com/sabrinavsimmons/cybersecurity-home-lab](https://github.com/sabrinavsimmons/cybersecurity-home-lab)*
