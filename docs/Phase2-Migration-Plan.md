# HOMELAB PHASE 2 - MIGRATION PLAN
## pfSense as Main Router/Firewall

Date Created: January 11, 2026
Status: PLANNING - Not Yet Executed

═══════════════════════════════════════════════════════════════════

## EXECUTIVE SUMMARY

Phase 2 transforms your homelab from a double-router architecture into a professional enterprise-grade network with pfSense as the primary router and firewall. This migration provides centralized network management, eliminates double-NAT, and enables advanced features like VLANs and proper network segmentation.

**Estimated Time:** 2-3 hours
**Difficulty:** Moderate
**Risk Level:** Medium (requires physical changes and can cause temporary network outage)
**Reversibility:** High (detailed rollback plan included)

═══════════════════════════════════════════════════════════════════

## WHAT CHANGES IN PHASE 2

### Current Architecture (Phase 1):
```
Internet (ISP WiFi)
        ↓
   GL.iNet Router (WiFi-to-Ethernet Bridge)
   Network: 192.168.8.0/24 ← GL.iNet manages WiFi
   Gateway: 192.168.8.1
        ↓ Ethernet
   pfSense M910q (Lab Firewall Only)
   WAN: 192.168.8.196
   LAN: 192.168.1.1 ← pfSense manages lab only
        ↓
   Lab Devices (Proxmox, Pi-hole)
```

**Problems with Current Setup:**
- Two separate DHCP servers (GL.iNet and pfSense)
- Double NAT (performance impact, complicates port forwarding)
- Split management (configure DNS/firewall in two places)
- MacBook on different network than lab
- Not scalable or professional

### Target Architecture (Phase 2):
```
Internet (ISP WiFi)
        ↓
   GL.iNet (Bridge/AP Mode ONLY)
   Just passes traffic, no routing
        ↓ Ethernet
   pfSense M910q (MAIN ROUTER/FIREWALL)
   WAN: Gets IP from ISP (via GL.iNet bridge)
   LAN: 192.168.1.1 ← Single network for everything
        ↓
   ALL Devices (MacBook via WiFi, Proxmox, Pi-hole)
   Network: 192.168.1.0/24
```

**Benefits:**
- ✅ Single network (192.168.1.0/24) for all devices
- ✅ One DHCP server (pfSense)
- ✅ One DNS configuration point
- ✅ No double NAT
- ✅ Enterprise-grade firewall for everything
- ✅ Professional architecture
- ✅ Easy to add VLANs later
- ✅ Better performance
- ✅ Centralized management

═══════════════════════════════════════════════════════════════════

## PRE-MIGRATION CHECKLIST

### 1. Information Gathering

**Record Current Settings:**

**GL.iNet:**
- Admin URL: http://192.168.8.1
- WiFi SSID: _______________
- WiFi Password: _______________
- Admin Password: _______________

**pfSense:**
- Current WAN IP: 192.168.8.196
- Current LAN IP: 192.168.1.1
- Admin Username: admin
- Admin Password: _______________

**ISP Connection:**
- Connection Type: WiFi
- ISP Router Location: _______________
- Can run ethernet cable to ISP? _______________

### 2. Backup Procedures

**CRITICAL: Backup pfSense Configuration**

**Steps:**
1. Log into pfSense: http://192.168.1.1
2. Navigate to: Diagnostics → Backup & Restore
3. Click "Download configuration as XML"
4. Save file as: `pfsense-backup-phase1-YYYY-MM-DD.xml`
5. Store on MacBook AND external location
6. Verify file downloaded successfully

**Why This Matters:**
If Phase 2 goes wrong, you can restore this backup and revert to Phase 1 architecture.

### 3. Tools & Materials Needed

- [ ] Ethernet cable (already have - connecting GL.iNet to pfSense)
- [ ] Monitor/keyboard for pfSense (for console access if needed)
- [ ] MacBook (for configuration)
- [ ] Phone with internet (backup internet if main network goes down)
- [ ] These documentation files

### 4. Time & Environment

**Best Time to Execute:**
- When you have 3-4 hours uninterrupted
- When you're well-rested
- When you don't need internet for anything critical
- Daytime (not late night) for clearer thinking

**Expected Downtime:**
- Initial downtime: 30-60 minutes (during reconfiguration)
- Intermittent connectivity issues: Possible during testing phase

═══════════════════════════════════════════════════════════════════

## MIGRATION PROCEDURE

### PHASE 2.1: GL.iNet Configuration (Bridge/AP Mode)

**Goal:** Change GL.iNet from router mode to bridge/AP mode

**Current State:** GL.iNet acts as router (192.168.8.0/24 network)
**Target State:** GL.iNet acts as wireless access point only

---

**Step 1: Access GL.iNet Admin Interface**

From MacBook (while still on WiFi):
```
http://192.168.8.1
```
Login with admin credentials

---

**Step 2: Configure Bridge/AP Mode**

**Navigation varies by GL.iNet firmware, but generally:**

**Option A: If "Bridge Mode" or "AP Mode" Setting Exists:**
1. Look for: Network → Network Mode (or similar)
2. Change from "Router" to "Bridge" or "Access Point"
3. Save and apply
4. GL.iNet will reboot

**Option B: If No Bridge Mode Setting:**
1. Navigate to: Network → LAN
2. Disable DHCP Server
3. Change LAN IP to: 192.168.1.2 (must be on pfSense LAN subnet)
4. Navigate to: Network → Firewall
5. Disable NAT/firewall features if possible
6. Save and reboot

**Option C: Factory Reset + Setup as AP:**
(Use only if Options A/B don't work)
1. Factory reset GL.iNet (hold reset button 10 seconds)
2. During setup wizard, choose "Access Point" or "Bridge" mode
3. Configure WiFi SSID/password same as before
4. Set static LAN IP: 192.168.1.2

---

**Step 3: Verify GL.iNet Bridge Mode**

After GL.iNet reboots:

**Test 1: Can you still access GL.iNet?**
- If yes at http://192.168.8.1: Bridge mode NOT active, try again
- If yes at http://192.168.1.2: Bridge mode active ✅
- If no access: May need to connect via ethernet to reconfigure

**Test 2: Does GL.iNet show as DHCP server?**
- If GL.iNet admin shows DHCP server active: Not in bridge mode
- If DHCP server disabled/not present: Good ✅

---

**Expected Behavior After GL.iNet Bridge Mode:**
- GL.iNet provides WiFi
- GL.iNet does NOT assign IP addresses
- GL.iNet passes all traffic to pfSense
- pfSense becomes DHCP server for WiFi clients

**IMPORTANT:** After this step, your MacBook will lose WiFi connectivity temporarily. This is expected. Continue to Step 2.2.

═══════════════════════════════════════════════════════════════════

### PHASE 2.2: pfSense WAN Reconfiguration

**Goal:** Change pfSense WAN from static IP (192.168.8.196) to DHCP from ISP

**You'll need:** Physical access to pfSense (monitor + keyboard) OR ethernet connection from MacBook to switch

---

**Step 1: Connect to pfSense**

**Option A: Via Ethernet (Recommended)**
1. Unplug MacBook from WiFi
2. Connect ethernet cable: MacBook → Switch
3. MacBook will get IP from pfSense DHCP (192.168.1.x)
4. Access pfSense: http://192.168.1.1

**Option B: Via pfSense Console**
1. Connect monitor and keyboard to pfSense M910q
2. Use console menu to configure

---

**Step 2: Change pfSense WAN to DHCP**

**Via Web Interface:**
1. Navigate to: Interfaces → WAN
2. Find "IPv4 Configuration Type"
3. Change from "Static IPv4" to "DHCP"
4. Remove static IP (192.168.8.196)
5. Scroll to bottom, click "Save"
6. Click "Apply Changes"

**Via Console:**
1. Select option 2: Set interface(s) IP address
2. Select WAN interface
3. Choose DHCP
4. Confirm changes

---

**Step 3: Verify WAN Gets IP from ISP**

pfSense WAN should now receive IP address from your ISP (via GL.iNet bridge).

**Check WAN IP:**
- Navigate to: Status → Interfaces
- Look at WAN interface
- Should show IP address from ISP (likely 192.168.x.x or 10.x.x.x)
- Should NOT be 192.168.8.196

**If WAN shows no IP or 169.254.x.x:**
- GL.iNet might not be in proper bridge mode
- ISP might require MAC address cloning
- Check physical cable connections

═══════════════════════════════════════════════════════════════════

### PHASE 2.3: pfSense Firewall Rules Update

**Goal:** Update firewall rules for new architecture

---

**Step 1: Remove Old WAN Rule**

The Phase 1 rule allowing 192.168.8.0/24 → 192.168.1.0/24 is no longer needed.

1. Navigate to: Firewall → Rules → WAN
2. Find rule: "Allow GL.iNet network to access pfSense LAN network"
3. Click the trash icon to delete
4. Click "Apply Changes"

---

**Step 2: Verify Default WAN Rules**

pfSense should have default WAN rules that block all incoming traffic by default.

**This is GOOD for security.**

You'll add specific allow rules only when needed (e.g., VPN access later).

---

**Step 3: Keep "Block Private Networks" Disabled**

Since GL.iNet is now just a bridge (not a router), and pfSense WAN connects to it:

1. Navigate to: Interfaces → WAN
2. Verify "Block private networks and loopback addresses" is UNCHECKED
3. If checked, uncheck it and save

═══════════════════════════════════════════════════════════════════

### PHASE 2.4: Device Reconnection & Testing

**Goal:** Reconnect all devices to new unified network

---

**Step 1: Reconnect MacBook via WiFi**

1. Disconnect ethernet from MacBook (if connected)
2. Connect to GL.iNet WiFi (same SSID/password)
3. MacBook should receive IP from pfSense DHCP: 192.168.1.x
4. Check IP: `ifconfig | grep "inet 192"`
5. Expected: 192.168.1.x (not 192.168.8.x)

**Verify MacBook DNS:**
```bash
scutil --dns | grep nameserver
```

Expected: `nameserver[0] : 192.168.1.20` (Pi-hole) ✅

If you see other DNS servers, renew DHCP:
- System Settings → Network → Details → TCP/IP → Renew DHCP Lease

---

**Step 2: Verify Lab Devices**

**Proxmox (192.168.1.10):**
- From MacBook: `ping 192.168.1.10`
- Should work ✅
- Access web interface: https://192.168.1.10:8006

**Pi-hole (192.168.1.20):**
- From MacBook: `ping 192.168.1.20`
- Should work ✅
- Access web interface: http://192.168.1.20/admin

**pfSense (192.168.1.1):**
- From MacBook: `ping 192.168.1.1`
- Should work ✅
- Access web interface: http://192.168.1.1

---

**Step 3: Test Internet Connectivity**

From MacBook:
```bash
ping 8.8.8.8          # Test internet via IP
ping google.com       # Test DNS resolution
```

Both should work ✅

---

**Step 4: Verify Pi-hole Ad Blocking**

1. Open browser on MacBook
2. Go to: http://192.168.1.20/admin
3. Dashboard should show queries being logged
4. Visit a website with ads (any news site)
5. Check Pi-hole dashboard - should show blocked queries

**If ad blocking works, Phase 2 is COMPLETE! 🎉**

═══════════════════════════════════════════════════════════════════

## POST-MIGRATION VERIFICATION

### Comprehensive Network Check

**1. Network Architecture Verification**

From MacBook, run:
```bash
# Check your IP (should be 192.168.1.x)
ifconfig | grep "inet 192"

# Check your gateway (should be 192.168.1.1 - pfSense)
netstat -rn | grep default

# Check your DNS (should be 192.168.1.20 - Pi-hole)
scutil --dns | grep nameserver
```

**Expected Output:**
```
inet 192.168.1.xxx
default            192.168.1.1
nameserver[0] : 192.168.1.20
```

---

**2. Connectivity Test Matrix**

From MacBook WiFi (192.168.1.x):

| Test | Command | Expected Result |
|------|---------|----------------|
| Ping pfSense | `ping 192.168.1.1` | Success ✅ |
| Ping Proxmox | `ping 192.168.1.10` | Success ✅ |
| Ping Pi-hole | `ping 192.168.1.20` | Success ✅ |
| Ping Internet | `ping 8.8.8.8` | Success ✅ |
| DNS Resolution | `ping google.com` | Success ✅ |
| pfSense Web | http://192.168.1.1 | Loads ✅ |
| Proxmox Web | https://192.168.1.10:8006 | Loads ✅ |
| Pi-hole Web | http://192.168.1.20/admin | Loads ✅ |

---

**3. pfSense Status Verification**

In pfSense web interface:

**Check WAN Status:**
- Status → Interfaces → WAN
- Should show IP from ISP (not 192.168.8.196)
- Status: UP
- Gateway: Should have gateway IP from ISP

**Check DHCP Leases:**
- Status → DHCP Leases
- Should show:
  - Your MacBook (192.168.1.x)
  - Any other devices on WiFi

**Check System Logs:**
- Status → System Logs → System
- Look for any errors or warnings
- Should be mostly clean

---

**4. Pi-hole Functionality Check**

**Access Pi-hole Dashboard:**
http://192.168.1.20/admin

**Verify:**
- Status: Active ✅
- Queries today: Should be increasing
- Blocked: Should show some percentage
- Clients: Should show your MacBook

**Test Ad Blocking:**
1. Open browser
2. Visit: https://canyoublockit.com/testing/
3. Run the test
4. Should show high block rate ✅

═══════════════════════════════════════════════════════════════════

## TROUBLESHOOTING GUIDE

### Issue 1: MacBook Won't Get IP from pfSense

**Symptoms:**
- MacBook connects to WiFi but shows "No Internet"
- MacBook has 169.254.x.x IP (APIPA/self-assigned)

**Diagnosis:**
```bash
ifconfig | grep inet
```

**Causes:**
- GL.iNet not in proper bridge mode (still acting as DHCP server)
- pfSense DHCP server not running
- Physical cable issue between GL.iNet and pfSense

**Solutions:**

**A. Verify GL.iNet Bridge Mode:**
- Try accessing http://192.168.8.1 (if this works, bridge mode NOT active)
- Try accessing http://192.168.1.2 (if this works, bridge mode IS active)
- If still in router mode, repeat Phase 2.1

**B. Check pfSense DHCP Status:**
- Connect via ethernet to pfSense
- Access http://192.168.1.1
- Status → Services → check if "dhcpd" is running
- If not running: Services → DHCP Server → LAN → verify it's enabled

**C. Restart DHCP Service:**
- Services → DHCP Server → LAN
- Save (even without changes)
- Status → Services → restart dhcpd

---

### Issue 2: Can Access Lab but No Internet

**Symptoms:**
- Can ping/access 192.168.1.x devices
- Cannot ping 8.8.8.8 or google.com
- pfSense reachable but no external connectivity

**Diagnosis:**
```bash
ping 192.168.1.1    # Works
ping 8.8.8.8        # Fails
```

**Causes:**
- pfSense WAN not getting IP from ISP
- pfSense WAN gateway not set
- NAT not configured properly

**Solutions:**

**A. Check pfSense WAN Status:**
- Interfaces → WAN
- Status should be UP
- IP address should be from ISP (not blank)
- Gateway should be populated

**B. If WAN has no IP:**
- Interfaces → WAN
- Verify "IPv4 Configuration Type" is DHCP
- Click "Save" → "Apply Changes"
- Status → Interfaces → click "Release" then "Renew" for WAN

**C. Check Default Gateway:**
- System → Routing → Gateways
- Should show WAN gateway as default
- Status should be "Online"

**D. Restart WAN Interface:**
- Status → Interfaces
- Click "Release" next to WAN interface
- Wait 10 seconds
- Click "Renew"

---

### Issue 3: Pi-hole Not Blocking Ads

**Symptoms:**
- Ads still showing on websites
- Pi-hole dashboard shows 0 queries

**Diagnosis:**
```bash
scutil --dns | grep nameserver
```

If nameserver is NOT 192.168.1.20, DNS not going through Pi-hole.

**Solutions:**

**A. Verify pfSense DNS Configuration:**
- Services → DHCP Server → LAN
- Check "DNS servers" field
- Should have 192.168.1.20
- Save → Apply Changes

**B. Renew DHCP on MacBook:**
- System Settings → Network
- Details → TCP/IP → Renew DHCP Lease

**C. Manually Set DNS (Temporary):**
- System Settings → Network
- Details → DNS tab
- Add 192.168.1.20
- Click OK

**D. Check Pi-hole Status:**
- SSH to Pi-hole: `ssh sabri@192.168.1.20`
- Run: `pihole status`
- Should show "Pi-hole blocking is enabled"
- If disabled: `pihole enable`

---

### Issue 4: GL.iNet Won't Enter Bridge Mode

**Symptoms:**
- GL.iNet still acts as router
- Can access http://192.168.8.1
- MacBook still gets 192.168.8.x IP

**Solutions:**

**A. Check GL.iNet Documentation:**
- Your specific GL.iNet model might have different bridge mode process
- Google: "[Your GL.iNet model] bridge mode setup"

**B. Alternative: Disable DHCP + Change IP:**
Even if no "bridge mode" setting exists:
1. GL.iNet admin → Network → LAN
2. Change IP to: 192.168.1.2
3. Network → DHCP → Disable DHCP server
4. Firewall → Disable NAT if possible
5. Save and reboot

**C. Factory Reset + AP Setup:**
1. Factory reset GL.iNet (hold reset button 10 seconds)
2. During initial setup, look for "Access Point" mode
3. Choose AP mode instead of router mode
4. Configure WiFi SSID/password

**D. Last Resort - Skip GL.iNet:**
If GL.iNet absolutely won't cooperate:
- Connect pfSense WAN directly to ISP router via ethernet
- Use a separate WiFi access point if needed
- This requires running ethernet cable to ISP location

---

### Issue 5: Complete Loss of Network Access

**Symptoms:**
- Cannot access pfSense
- Cannot access any lab devices
- No internet

**This is the "oh crap" scenario. Stay calm.**

**Solution: Physical Console Access**

**Step 1: Connect Monitor/Keyboard to pfSense**

**Step 2: Access pfSense Console Menu**

**Step 3: Reset WAN to Static IP (Temporary Rollback)**
- Select option 2: Set interface(s) IP address
- Select WAN interface
- Choose Static
- IP: 192.168.8.196
- Subnet: 24
- Gateway: 192.168.8.1

**Step 4: From Another Device:**
- Connect ethernet to switch
- Should get 192.168.1.x IP
- Access pfSense: http://192.168.1.1
- Restore backup configuration (see Rollback Plan)

═══════════════════════════════════════════════════════════════════

## ROLLBACK PLAN

If Phase 2 fails and you need to revert to Phase 1 architecture:

### Quick Rollback Procedure

**Step 1: Restore pfSense Backup**

1. Connect to pfSense (via console or ethernet)
2. Access: http://192.168.1.1
3. Navigate to: Diagnostics → Backup & Restore
4. Click "Choose File" and select your Phase 1 backup
5. Click "Restore Configuration"
6. Click "Yes" to confirm
7. pfSense will reboot

**After reboot, pfSense will have:**
- WAN: Static IP 192.168.8.196
- LAN: 192.168.1.1
- All Phase 1 firewall rules

---

**Step 2: Reconfigure GL.iNet to Router Mode**

1. Access GL.iNet admin (might be http://192.168.1.2 or reset needed)
2. Change from Bridge/AP mode back to Router mode
3. OR factory reset and set up as router with:
   - LAN IP: 192.168.8.1
   - DHCP enabled
   - WiFi configured

---

**Step 3: Verify Phase 1 Architecture Restored**

From MacBook WiFi:
- Should get IP: 192.168.8.x
- Can access GL.iNet: http://192.168.8.1
- Can access pfSense: http://192.168.1.1 (via static route)
- Can access Proxmox: https://192.168.1.10:8006
- Can access Pi-hole: http://192.168.1.20/admin

**If all above work, you're back to Phase 1. ✅**

═══════════════════════════════════════════════════════════════════

## COMMON QUESTIONS & ANSWERS

**Q: Will I lose internet access during migration?**
A: Yes, expect 30-60 minutes of downtime during GL.iNet and pfSense reconfiguration. Plan accordingly.

**Q: Can I still access my lab devices during migration?**
A: During certain steps, no. Once Phase 2.2 is complete, yes. Plan for temporary loss of access.

**Q: What if I mess something up?**
A: That's why we have the backup and rollback plan. Worst case: restore Phase 1 backup.

**Q: Do I need to reconfigure Proxmox or Pi-hole?**
A: No! They stay at 192.168.1.10 and 192.168.1.20. No changes needed.

**Q: Will my WiFi SSID/password change?**
A: No, GL.iNet keeps the same WiFi settings even in bridge mode.

**Q: What if GL.iNet won't go into bridge mode?**
A: See troubleshooting section. Worst case: connect pfSense directly to ISP and use different AP.

**Q: How long should each phase take?**
- Phase 2.1 (GL.iNet): 20-30 minutes
- Phase 2.2 (pfSense WAN): 15-20 minutes  
- Phase 2.3 (Firewall rules): 5-10 minutes
- Phase 2.4 (Testing): 20-30 minutes
- Total: 60-90 minutes if everything goes smoothly

**Q: What if I run into issues not covered here?**
A: Document the error messages, check pfSense logs, and we can troubleshoot. Having the Phase 1 backup means you can always roll back.

═══════════════════════════════════════════════════════════════════

## SUCCESS CRITERIA

Phase 2 is complete when ALL of these are true:

**Network Architecture:**
- [ ] MacBook on WiFi receives IP from pfSense DHCP (192.168.1.x)
- [ ] pfSense WAN receives IP from ISP (not 192.168.8.196)
- [ ] GL.iNet in bridge/AP mode (not routing)
- [ ] All devices on single network (192.168.1.0/24)

**Connectivity:**
- [ ] MacBook can ping pfSense (192.168.1.1)
- [ ] MacBook can ping Proxmox (192.168.1.10)
- [ ] MacBook can ping Pi-hole (192.168.1.20)
- [ ] MacBook can ping Internet (8.8.8.8)
- [ ] MacBook can resolve DNS (ping google.com)

**Services:**
- [ ] pfSense web interface accessible
- [ ] Proxmox web interface accessible
- [ ] Pi-hole web interface accessible
- [ ] Pi-hole blocking ads (check dashboard)

**DNS Configuration:**
- [ ] MacBook uses Pi-hole for DNS (192.168.1.20)
- [ ] Pi-hole dashboard shows queries from MacBook
- [ ] Ad blocking verified on test websites

**Performance:**
- [ ] No double NAT (check router status)
- [ ] Internet speed reasonable
- [ ] No unusual latency

═══════════════════════════════════════════════════════════════════

## NEXT STEPS AFTER PHASE 2

Once Phase 2 is complete and stable:

**Immediate (Within 1 Week):**
1. Update pfSense to latest version (2.8.1)
   - System → Update
   - Backup first!

2. Change Pi-hole admin password
   - SSH: `pihole -a -p`
   - Or web interface

3. Document new architecture
   - Update your homelab documentation
   - Screenshot new network topology
   - Update GitHub repository

**Short Term (Within 1 Month):**
1. Tighten firewall rules
   - Remove unnecessary WAN rules
   - Implement least privilege
   - Document rules

2. Set up VPN access
   - WireGuard on pfSense
   - Remote access to lab

3. Implement monitoring
   - Enable pfSense logging
   - Review Pi-hole statistics
   - Check for anomalies

**Long Term (Phase 3 Ideas):**
1. VLAN segmentation
   - Guest network VLAN
   - IoT device VLAN
   - Lab VLAN

2. Additional services
   - Network monitoring (Zabbix, Grafana)
   - IDS/IPS (Suricata on pfSense)
   - Reverse proxy (Nginx Proxy Manager)

3. High availability
   - pfSense backup/failover
   - Network redundancy

═══════════════════════════════════════════════════════════════════

## EXECUTION CHECKLIST

Print or reference this checklist during migration:

**Pre-Migration:**
- [ ] Read entire Phase 2 plan
- [ ] Backup pfSense configuration
- [ ] Record all current settings
- [ ] Confirm you have 3+ hours available
- [ ] Ensure phone/backup internet available
- [ ] Well-rested and clear-headed

**Phase 2.1: GL.iNet Bridge Mode**
- [ ] Access GL.iNet admin
- [ ] Change to bridge/AP mode
- [ ] Verify GL.iNet no longer routes
- [ ] Document new GL.iNet IP (if changed)

**Phase 2.2: pfSense WAN Reconfiguration**
- [ ] Connect to pfSense (ethernet or console)
- [ ] Change WAN from Static to DHCP
- [ ] Verify WAN gets IP from ISP
- [ ] Verify WAN gateway set

**Phase 2.3: Firewall Rules**
- [ ] Remove old WAN rule (192.168.8.0/24)
- [ ] Verify "Block private networks" still unchecked
- [ ] Apply changes

**Phase 2.4: Device Testing**
- [ ] MacBook reconnects to WiFi
- [ ] MacBook gets 192.168.1.x IP
- [ ] MacBook uses Pi-hole DNS
- [ ] Can access pfSense
- [ ] Can access Proxmox
- [ ] Can access Pi-hole
- [ ] Internet works
- [ ] Ad blocking works

**Post-Migration:**
- [ ] Run full verification checklist
- [ ] Test from multiple devices
- [ ] Document any issues encountered
- [ ] Update documentation with new topology
- [ ] Celebrate! 🎉

═══════════════════════════════════════════════════════════════════

## FINAL NOTES

**Be Patient:**
Network migrations take time. Don't rush through steps.

**Document Everything:**
Take screenshots. Record IPs. Note any errors. This helps troubleshooting.

**Don't Panic:**
If something breaks, you have a rollback plan. Worst case: restore Phase 1 backup.

**Ask for Help:**
If you get stuck, stop and ask for help rather than making it worse.

**Celebrate Success:**
When Phase 2 works, you've built a professional-grade network. That's a real accomplishment.

═══════════════════════════════════════════════════════════════════

## CONCLUSION

Phase 2 transforms your homelab from a functional setup into a professional enterprise architecture. This migration:

- Eliminates double-NAT
- Centralizes network management
- Enables network-wide services (Pi-hole, VPN, etc.)
- Provides better learning experience for cybersecurity career
- Creates foundation for advanced features (VLANs, IDS/IPS, etc.)

The migration is moderately complex but well-documented. With proper preparation, backup procedures, and this plan, you have everything needed for success.

**You've got this. See you tomorrow for Phase 2 execution! 💪**

═══════════════════════════════════════════════════════════════════

Document End
─────────────────────────────────────────────────────────────
Created: January 11, 2026, 11:45 PM
Status: PLANNING - Ready for execution when you are
