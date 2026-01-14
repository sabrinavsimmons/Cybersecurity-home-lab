# Quick Start Guide

Essential commands and access information for homelab infrastructure.

---

## 🌐 Access URLs

| Service | URL | Purpose |
|---------|-----|---------|
| GL.iNet Admin | http://192.168.8.1 | Router configuration |
| pfSense Web GUI | http://192.168.1.1 | Firewall management |
| Proxmox Web GUI | https://192.168.1.10:8006 | VM management |
| Pi-hole Admin | http://192.168.1.20/admin | DNS filtering |

**Note:** Access to pfSense, Proxmox, and Pi-hole requires being on GL.iNet WiFi or connected via ethernet to the lab network.

---

## 🔐 SSH Access

```bash
# GL.iNet Router
ssh root@192.168.8.1

# pfSense (if SSH enabled)
ssh admin@192.168.1.1

# Proxmox
ssh root@192.168.1.10

# Pi-hole
ssh sabri@192.168.1.20
```

---

## 🔍 Network Testing

### From MacBook on WiFi

```bash
# Check your IP (should be 192.168.8.x)
ifconfig | grep "inet 192"

# Check gateway (should be 192.168.8.1)
netstat -rn | grep default

# Test connectivity to lab
ping 192.168.1.1    # pfSense
ping 192.168.1.10   # Proxmox
ping 192.168.1.20   # Pi-hole

# Test internet
ping 8.8.8.8
ping google.com
```

### Verify Static Route on GL.iNet

```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

Expected: `192.168.1.0/24 via 192.168.8.196 dev br-lan proto static`

---

## 🔧 Common Tasks

### Restart GL.iNet Network

```bash
ssh root@192.168.8.1
/etc/init.d/network reload
```

### Restart Proxmox Network

```bash
ssh root@192.168.1.10
systemctl restart networking
systemctl restart pveproxy
systemctl restart pvedaemon
```

### Pi-hole Management

```bash
ssh sabri@192.168.1.20

# Check status
pihole status

# Update Pi-hole
pihole -up

# Change admin password
pihole -a -p

# Restart DNS
pihole restartdns

# Live statistics
pihole -c
```

### pfSense Backup

1. Access http://192.168.1.1
2. Navigate to: Diagnostics → Backup & Restore
3. Click "Download configuration as XML"
4. Save as: `pfsense-backup-YYYY-MM-DD.xml`

---

## 🚨 Troubleshooting

### Can't Access Lab from WiFi

**Check static route:**
```bash
ssh root@192.168.8.1
ip route show | grep 192.168.1
```

**If missing, re-add:**
```bash
ip route add 192.168.1.0/24 via 192.168.8.196
uci add network route
uci set network.@route[-1].interface='lan'
uci set network.@route[-1].target='192.168.1.0'
uci set network.@route[-1].netmask='255.255.255.0'
uci set network.@route[-1].gateway='192.168.8.196'
uci commit network
/etc/init.d/network reload
```

### Pi-hole Not Responding

1. Check if Pi is powered on
2. Check ethernet cable connected
3. Reboot Pi (unplug power, wait 10 seconds, plug back in)
4. Check DHCP lease in pfSense: Status → DHCP Leases

### Lost pfSense Access

**Connect via ethernet:**
1. Plug MacBook directly into switch
2. Get IP from pfSense DHCP (192.168.1.x)
3. Access http://192.168.1.1

**Or use console:**
1. Connect monitor/keyboard to pfSense M910q
2. Use pfSense console menu

---

## 📊 Network Information

### IP Addressing

| Network | Subnet | Gateway | DHCP Server |
|---------|--------|---------|-------------|
| WiFi/Home | 192.168.8.0/24 | 192.168.8.1 | GL.iNet |
| Lab | 192.168.1.0/24 | 192.168.1.1 | pfSense |

### Static Assignments

| Device | IP | Assignment Method |
|--------|----|--------------------|
| GL.iNet | 192.168.8.1 | Static (device default) |
| pfSense WAN | 192.168.8.196 | Static (pfSense config) |
| pfSense LAN | 192.168.1.1 | Static (pfSense config) |
| Proxmox | 192.168.1.10 | Static (OS config) |
| Pi-hole | 192.168.1.20 | DHCP Reservation (pfSense) |

---

## 🔄 After Reboot Checklist

### GL.iNet Reboot
- [ ] Verify static route: `ip route show | grep 192.168.1`
- [ ] Test lab connectivity: `ping 192.168.1.1`

### pfSense Reboot
- [ ] Verify WAN interface up (192.168.8.196)
- [ ] Verify LAN interface up (192.168.1.1)
- [ ] Check firewall logs for issues

### Pi-hole Reboot
- [ ] Verify gets IP 192.168.1.20
- [ ] Test: `ping 192.168.1.20`
- [ ] Access web interface: http://192.168.1.20/admin

### Proxmox Reboot
- [ ] Verify IP: `ssh root@192.168.1.10 'ip addr'`
- [ ] Access web interface: https://192.168.1.10:8006

---

## 📖 Full Documentation

For complete implementation details, troubleshooting guide, and architectural analysis:
- [Phase 1 Complete Documentation](Phase1-Complete-Documentation.md)
- [Phase 2 Migration Analysis](Phase2-Migration-Plan.md)

---

## 💡 Quick Tips

**Working wirelessly from MacBook:**
- Always verify you're on GL.iNet WiFi (192.168.8.x IP)
- Lab access requires static route on GL.iNet
- Use ethernet for most reliable access during troubleshooting

**pfSense firewall logs are your friend:**
- Status → System Logs → Firewall
- Shows blocked/allowed traffic in real-time
- Essential for troubleshooting connectivity issues

**Keep backups current:**
- Backup pfSense before any major changes
- Store backups in multiple locations
- Test restore procedure when system is working

**Document changes:**
- Note any configuration changes
- Keep a log of issues and resolutions
- Update this guide as environment evolves
