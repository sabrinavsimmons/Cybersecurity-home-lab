# Phase 3: Security Operations Center (SOC) Deployment

**Status:** ✅ Complete  
**Duration:** ~15 hours (across multiple sessions)  
**Completion Date:** January 26, 2026

---

## 📋 Overview

Phase 3 transforms the homelab from basic network infrastructure into a **fully operational Security Operations Center (SOC)** with comprehensive network monitoring, endpoint detection, and attack simulation capabilities.

---

## 🎯 Objectives & Outcomes

### Primary Objectives
- ✅ Deploy Security Information and Event Management (SIEM) platform
- ✅ Implement network traffic monitoring and intrusion detection
- ✅ Configure endpoint detection and response (EDR) agents
- ✅ Establish attack simulation infrastructure
- ✅ Validate full detection pipeline with live testing

### Learning Outcomes
- Software-based traffic mirroring using Open vSwitch (OVS)
- Wazuh SIEM deployment and agent management
- Security Onion configuration with Suricata and Zeek
- Attack detection across network and endpoint layers
- Enterprise storage troubleshooting and capacity planning

---

## 🏗️ Architecture

### Network Topology
```
                    Internet
                        ↓
                  GL.iNet Router
                  192.168.8.0/24
                        ↓
                  pfSense Firewall
                  192.168.1.1
                        ↓
        ┌───────────────┴───────────────┐
        ↓                               ↓
    vmbr0 (Management)           vmbr1 (Monitored)
    192.168.1.0/24              10.0.0.0/24
        ↓                               ↓
    ┌───┴────┬──────┬────────┐    ┌────┴────┬──────────┐
    │        │      │        │    │         │          │
Proxmox  Wazuh  Security    │   Kali   Metasploitable Windows
.1.10    .1.108  Onion      │  .0.10      .0.20       .0.30
                 .1.30       │
                 (mgmt)      │
                             │
                      OVS Bridge (span0)
                      Traffic Mirroring
                             ↓
                      Security Onion
                      (monitoring port)
```

### Component Roles

| Component | Role | IP Addresses | Purpose |
|-----------|------|--------------|---------|
| Security Onion | Network monitoring | 192.168.1.30 (mgmt)<br>enp6s19 (monitor) | Zeek/Suricata IDS, packet capture |
| Wazuh Server | SIEM/EDR platform | 192.168.1.108 | Agent management, event correlation |
| Kali Linux | Attack platform | 192.168.1.102 (mgmt)<br>10.0.0.10 (attack) | Penetration testing, agent monitoring |
| Metasploitable | Vulnerable target | 10.0.0.20 | Attack testing, vulnerability research |
| Windows 11 | Production endpoint | 10.0.0.30 | Realistic endpoint monitoring |
| OVS Bridge | Traffic mirror | vmbr1 (software bridge) | VM-to-VM traffic capture |

---

## 🔧 Implementation Details

### 1. Security Onion Deployment

**Platform:** Security Onion 2.4 (standalone deployment)  
**Resources:** 300GB disk, 16GB RAM, 4 CPU cores

**Network Configuration:**
- Management Interface (enp6s18): 192.168.1.30/24 on vmbr0
- Monitoring Interface (enp6s19): Promiscuous mode on vmbr1 (no IP)

**Services Deployed:**
- Zeek - Network traffic analysis and logging
- Suricata - Real-time intrusion detection
- Elasticsearch - Event indexing and storage
- Kibana/Hunt - Web-based analysis interface

**Key Learning:** Security Onion requires dual network interfaces. The monitoring interface must be in promiscuous mode with no IP address to passively capture traffic.

---

### 2. OVS Bridge Traffic Mirroring

**Challenge:** Physical switch port mirroring (Netgear GS308E) failed when mirroring 7 ports, causing network-wide packet loss.

**Solution:** Implemented software-based traffic mirroring using Open vSwitch in Proxmox.

**Implementation:**
```bash
# Install OVS
apt install openvswitch-switch -y

# Create bridge
ovs-vsctl add-br vmbr1

# Configure port mirroring
PORT_UUID=$(ovs-vsctl get Port tap105i1 _uuid)
ovs-vsctl -- --id=@m create mirror name=span0 select-all=true \
  output-port=$PORT_UUID -- add bridge vmbr1 mirrors @m

# Add to /etc/network/interfaces
auto vmbr1
iface vmbr1 inet manual
    ovs_type OVSBridge
    ovs_ports none
```

**Validation:**
```bash
ovs-vsctl list mirror
# name: span0, output_port: Security Onion
# select_all: true, tx_packets: 9496
```

**Result:** Software-based mirroring provides reliable traffic capture without hardware limitations.

**Trade-off:** Mirror configuration doesn't persist across reboots (acceptable for homelab; would require automation in production).

---

### 3. Wazuh Server Deployment

**Platform:** Wazuh 4.9.2 (all-in-one installation)  
**Resources:** 60GB disk, 4GB RAM, 2 CPU cores  
**Storage:** Deployed on ssd-vg (931GB SSD) to avoid NVMe capacity issues

**Installation:**
```bash
wget https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

**Installation Time:** ~15 minutes

**Components:** Wazuh Manager (agent communication), Wazuh Indexer (event storage), Wazuh Dashboard (web interface)
---

## 🔍 Critical Challenges & Solutions

### Challenge 1: Physical Switch Limitations

**Problem:** Netgear GS308E switch failed when mirroring all 7 ports simultaneously.

**Symptoms:** Network-wide packet loss, services unreachable, switch overwhelmed.

**Solution:** Migrated to Open vSwitch software-based mirroring in Proxmox hypervisor.

**Lesson:** Budget hardware has real performance constraints. Software-defined networking can overcome physical limitations.

---

### Challenge 2: Disk Space Exhaustion

**Timeline:**
1. Initial Wazuh VM (50GB) deployed successfully
2. Ubuntu 24.04 upgrade initiated
3. Proxmox root filesystem filled to 100%
4. VM crashed mid-upgrade

**Root Cause:**
```bash
root@promox:~# df -h
/dev/mapper/pve-root   68G   68G     0 100% /

# Large VM disk consumed space
/var/lib/vz/images/120/vm-120-disk-0.qcow2: 56GB
```

**Resolution:**
- Deleted broken Wazuh VM (freed 56GB)
- Removed duplicate files (freed 3.6GB)
- Cleared ISO files and journal logs
- Redeployed on ssd-vg storage pool (535GB free)
- Allocated 60GB with monitoring

**Storage Architecture Discovery:**
```bash
pve (NVMe):  237GB total, 16GB free
ssd-vg (SSD): 931GB total, 535GB free
```

**Lessons Learned:**
- SIEM platforms require substantial storage (plan for 30-40% overhead)
- Understand hypervisor storage architecture before deployment
- Monitor capacity proactively during upgrades
- Delete installation media after VM deployment

**Capacity Planning Formula:**
```
(Base OS + Software) × 1.4 + (Daily Logs × Retention Days)

Example: Wazuh with 5 hosts, 30-day retention
- Base: (15GB + 20GB) × 1.4 = 49GB
- Logs: 250MB/day × 30 = 7.5GB
- Total: ~60GB minimum
```

---

### Challenge 3: Certificate Management

**Problem:** Wazuh dashboard crash-looping after Ubuntu upgrade.

**Error:**
```
ENOENT: no such file or directory
open '/etc/wazuh-dashboard/certs/dashboard-key.pem'
```

**Root Cause:** System upgrade replaced config but deleted SSL certificates.

**Investigation:**
```bash
sudo ls -la /etc/wazuh-dashboard/certs/
# Found: wazuh-dashboard.pem, wazuh-dashboard-key.pem
# Missing: dashboard.pem, dashboard-key.pem
```

**Solution:**
```bash
# Extract from backup
sudo tar -xf wazuh-install-files.tar
sudo cp wazuh-install-files/wazuh-dashboard* /etc/wazuh-dashboard/certs/

# Create symlinks
sudo bash -c 'cd /etc/wazuh-dashboard/certs/ && \
  ln -sf wazuh-dashboard.pem dashboard.pem && \
  ln -sf wazuh-dashboard-key.pem dashboard-key.pem'

# Restart service
sudo systemctl restart wazuh-dashboard
```

**Lessons Learned:**
- Always preserve installation backup files
- SSL certificates require special handling during upgrades
- Test service functionality after system updates
- Document certificate locations for troubleshooting

---

### Challenge 4: Multi-OS Network Connectivity

**Problem:** Windows could ping Kali, but Kali could not ping Windows (one-way connectivity).

**Investigation:**
```bash
# Kali: Confirmed eth1 configured (10.0.0.10)
ip addr show eth1

# Proxmox: Verified both VMs on vmbr1
ovs-vsctl list-ports vmbr1
# Output: tap100i1 (Kali), tap110i1 (Windows)

# Found stale ARP entry
ip neigh show | grep 10.0.0.30
# 10.0.0.30 dev eth1 ... STALE
```

**Root Cause:** Windows Firewall blocking inbound ICMP on "Public" network profile.

**Solution:**
```powershell
# Change to Private profile
Set-NetConnectionProfile -NetworkCategory Private

# Allow ICMP
New-NetFirewallRule -DisplayName "Allow ICMPv4" `
  -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow

# Re-enable firewall for realistic monitoring
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

**Lesson:** Windows network profiles significantly affect firewall behavior. "Public" profile is highly restrictive by default.

---

## 🔧 Endpoint Agent Deployment

### Kali Linux Agent

**Configuration:** Dual-NIC (eth0: management, eth1: attack network)
```bash
# Install agent
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.2-1_amd64.deb
sudo dpkg -i wazuh-agent_4.9.2-1_amd64.deb

# Configure server
sudo sed -i 's|MANAGER_IP|192.168.1.108|g' /var/ossec/etc/ossec.conf

# Start agent
sudo systemctl enable wazuh-agent --now
```

**Network Persistence Issue:** NetworkManager attempted DHCP on eth1 (no DHCP server on attack network).

**Solution:**
```bash
# Disable NetworkManager control
sudo nmcli device set eth1 managed no

# Configure static IP in /etc/network/interfaces
auto eth1
iface eth1 inet static
    address 10.0.0.10
    netmask 255.255.255.0
```

---

### Windows 11 Agent

**Installation:**
```powershell
# Download and install (PowerShell as Administrator)
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi `
  -OutFile $env:tmp\wazuh-agent

msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='192.168.1.108' WAZUH_AGENT_NAME='windows11'

NET START WazuhSvc
```

**Firewall Configuration:**
```powershell
# Allow ICMP for testing
New-NetFirewallRule -DisplayName "Allow ICMPv4" `
  -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow

# Set network to Private
Set-NetConnectionProfile -NetworkCategory Private
```

---

### Metasploitable 2 Decision

**Challenge:** 32-bit system (i386) from 2010 with outdated dependencies.

**Issues:**
- Modern Wazuh agent (amd64) incompatible
- 32-bit agent had symbol lookup errors on ancient libraries
- No systemd (uses SysVinit)

**Decision:** Deploy without agent - serves as attack target only.

**Rationale:**
- Metasploitable is intentionally vulnerable
- Network monitoring (Security Onion) captures all attack traffic
- Kali agent monitors attacker side
- Realistic: attackers don't install monitoring agents on compromised systems

**Network Configuration:**
```bash
sudo ifconfig eth1 10.0.0.20 netmask 255.255.255.0 up
```

---

## 🎯 Detection Pipeline Testing

### Test 1: Nmap Reconnaissance Scan

**Attack Vector:** Aggressive nmap scan from Kali to Metasploitable

```bash
# From Kali (10.0.0.10)
nmap -A -T4 10.0.0.20
```
---

## 🎯 Detection Pipeline Testing

### Test 1: Nmap Reconnaissance (Kali → Metasploitable)

**Attack Command:**
```bash
nmap -A -T4 10.0.0.20
```

**Security Onion Detection:**
- ✅ Alert: "ET SCAN Nmap Scripting Engine User-Agent Detected"
- ✅ Zeek connection logs captured
- ✅ Full packet capture available

**Traffic Evidence:**
```
[sabri@so-defender ~]$ sudo tcpdump -i enp6s19 host 10.0.0.20 -c 20
19:58:43.865512 ARP, Request who-has 10.0.0.20 tell 10.0.0.10
19:58:44.037347 IP 10.0.0.10.63688 > 10.0.0.20.ssh: Flags [S]
19:58:44.037354 IP 10.0.0.10.63688 > 10.0.0.20.http: Flags [S]
19:58:44.037367 IP 10.0.0.10.63688 > 10.0.0.20.mysql: Flags [S]
```

**Wazuh Monitoring:**
- System audit events logged on Kali agent
- PAM authentication events captured
- File integrity monitoring active

**Key Insight:** Defense-in-depth validated - network layer detects attacks in transit, endpoint layer monitors system changes.

---

### Test 2: Windows 11 Reconnaissance (Kali → Windows)

**Attack Command:**
```bash
nmap -A 10.0.0.30
```

**Security Onion Detection:**
- ✅ Alert: "ET SCAN NMAP OS Detection Probe"
- ✅ SYN scans to multiple Windows ports captured
- ✅ Suricata signature match

**Wazuh Monitoring:**
- Windows 11 agent connected and reporting
- System information collected
- Security baseline established

**Validation:** Full detection pipeline operational with multi-OS visibility.

---

## 📊 System Specifications

### Virtual Machine Resources

| VM | vCPUs | RAM | Disk | Storage | Network |
|----|-------|-----|------|---------|---------|
| Security Onion | 4 | 16 GB | 300 GB | ssd-vg | vmbr0 + vmbr1 |
| Wazuh Server | 2 | 4 GB | 60 GB | ssd-vg | vmbr0 |
| Kali Linux | 2 | 4 GB | 32 GB | ssd-vg | vmbr0 + vmbr1 |
| Metasploitable | 1 | 512 MB | 8 GB | pve | vmbr1 |
| Windows 11 | 2 | 4 GB | 64 GB | ssd-vg | vmbr0 + vmbr1 |

**Total:** 11 vCPUs, 28.5 GB RAM, 464 GB disk

---

## 💡 Key Technical Learnings

### 1. Defense in Depth Architecture

**Layers Implemented:**
- **Network:** Security Onion captures all traffic, detects attack patterns
- **Endpoint:** Wazuh monitors system changes, authentication, file integrity
- **Perimeter:** pfSense firewall controls inter-network access

**Application:** Layered defenses ensure that if one layer is bypassed, others still provide visibility.

---

### 2. Software vs. Hardware Trade-offs

| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| Physical Switch | Simple, hardware-based | Limited capacity | ❌ Failed |
| OVS Software | Unlimited capacity, flexible | Configuration complexity | ✅ Implemented |

**Lesson:** Software-defined networking provides flexibility that budget hardware cannot match.

---

### 3. Legacy System Integration

**Metasploitable Constraints:**
- 32-bit architecture (i386)
- No systemd (SysVinit only)
- Outdated SSL/TLS libraries
- Ancient kernel (2.6.x era)

**Decision Framework:**
1. Attempt modern tooling
2. Identify compatibility barriers
3. Evaluate necessity
4. Accept limitations when network monitoring provides sufficient visibility

**Application:** Not every system can or should run modern security tools. Focus monitoring where it provides value.

---

### 4. Enterprise Tool Complexity

**Wazuh Deployment Insights:**
- All-in-one installer abstracts significant complexity
- Multiple services must coordinate (Manager, Indexer, Dashboard)
- Certificate management critical for service-to-service communication
- Troubleshooting requires understanding underlying architecture

**Professional Skill:** Work with abstraction layers while understanding what's underneath. When automation fails, manual intervention requires architectural knowledge.

---

## 🎓 Skills Demonstrated

### Security Operations
- SIEM deployment and configuration (Wazuh)
- IDS/IPS implementation (Security Onion, Suricata)
- Endpoint detection and response (EDR agents)
- Log analysis and event correlation
- Incident detection and validation

### Network Engineering
- Software-defined networking (OVS)
- Traffic mirroring and packet capture
- Network segmentation for security
- Dual-interface configurations

### Systems Administration
- Linux administration (Ubuntu, Debian-based)
- Windows Server management
- Certificate management
- Service troubleshooting
- Storage capacity planning

### Professional Practices
- Root cause analysis methodology
- Backup and disaster recovery
- Troubleshooting complex multi-system issues
- Knowing when to rebuild vs. repair
- Comprehensive technical documentation

---

## 📈 Performance Metrics

**Network Monitoring:**
- Packet capture: 1,825 packets during testing
- IDS detection rate: 100% (all test attacks detected)
- Mirror reliability: 9,496 packets mirrored via OVS

**Endpoint Monitoring:**
- Agents deployed: 2 (Kali Linux, Windows 11)
- Agent connectivity: 100% uptime
- Event collection: Active on both platforms

---

## 🚀 Future Enhancements

### Short-Term
- [ ] pfSense log forwarding to Security Onion
- [ ] Pi-hole DNS logging integration
- [ ] Custom Suricata rules for lab-specific threats
- [ ] Automated attack scenarios
- [ ] Wazuh vulnerability detection

### Medium-Term
- [ ] Add Splunk or ELK stack for comparison
- [ ] YARA rules for malware detection
- [ ] File integrity monitoring baselines
- [ ] Custom Wazuh decoders and rules
- [ ] Network behavior anomaly detection (Zeek scripts)

### Long-Term
- [ ] Threat intelligence feed integration
- [ ] Automated incident response playbooks
- [ ] Red team vs. blue team scenarios
- [ ] Compliance reporting (CIS benchmarks)
- [ ] Machine learning-based anomaly detection

---

## 📚 Resources & References

**Documentation:**
- [Wazuh Official Documentation](https://documentation.wazuh.com/)
- [Security Onion Documentation](https://docs.securityonion.net/)
- [Open vSwitch Manual](http://www.openvswitch.org/support/dist-docs/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)

**Community:**
- Wazuh Community Forums
- Security Onion Discord
- r/homelab subreddit

**Frameworks:**
- CIS Benchmarks for system hardening
- MITRE ATT&CK Framework for threat modeling
- NIST Cybersecurity Framework

---

## 📊 Project Statistics

**Time Investment:** ~15 hours across 3 sessions
- Session 1 (Jan 22): Security Onion, OVS bridge (~6 hours)
- Session 2 (Jan 25-26): Wazuh deployment, troubleshooting (~6 hours)
- Session 3 (Jan 26): Rebuild, agents, testing (~3 hours)

**Infrastructure:**
- 5 virtual machines deployed
- 464GB storage consumed
- 2 successful attack simulations validated
- 50+ pages technical documentation

---

## ✅ Success Criteria

- ✅ Network monitoring operational (Security Onion)
- ✅ SIEM platform deployed (Wazuh)
- ✅ Multi-OS endpoint coverage (Linux + Windows)
- ✅ Attack detection validated (IDS alerts confirmed)
- ✅ Full documentation complete
- ✅ Professional-grade implementation

---

## 🎓 Conclusion

Phase 3 successfully transformed a basic homelab into a **production-grade Security Operations Center** with comprehensive visibility across network and endpoint layers.

The implementation demonstrates technical competency with enterprise security tools, professional troubleshooting methodology, capacity planning, and architectural decision-making.

The challenges encountered - disk exhaustion, certificate management, network connectivity - were not setbacks but **learning opportunities** that demonstrate real-world problem-solving skills highly valued in security operations roles.

The infrastructure now supports advanced security research, attack simulation, and continuous learning in defensive cybersecurity.

**Phase 3 Status: Complete and Operational** ✅

---

*Last Updated: January 26, 2026*  
*Author: Sabrina Simmons*  
*Repository: [github.com/sabrinavsimmons/cybersecurity-home-lab](https://github.com/sabrinavsimmons/cybersecurity-home-lab)*
