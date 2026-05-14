# DMVPN Troubleshooting Checklist — NovaTech Lab

## 🔎 Step-by-Step Diagnostic Flow

---

### Step 1 — Verify ISP Simulation First (SW1)
```
DC-CORE# ping 10.0.0.11   ← must reach WAW-R1
DC-CORE# ping 10.0.0.12   ← must reach LIS-R2
DC-CORE# ping 10.0.0.13   ← must reach CAI-R3
```
If ANY ping fails here, DMVPN will never work.
Check: cable connected to SW1, WAN interface up, correct IP on Eth0/0

---

### Step 2 — Tunnel Interface State
```
show interface tunnel0
show ip interface tunnel0
```
Must be: Tunnel0 is up, line protocol is up
If down:
  [ ] tunnel source Ethernet0/0 configured
  [ ] Ethernet0/0 is up/up
  [ ] tunnel mode gre multipoint configured
  [ ] ip address assigned on tunnel

---

### Step 3 — NHRP Registration
```
show ip nhrp
show ip nhrp summary
show ip nhrp nhs detail
```
On DC-CORE you must see all 3 spokes registered.
On spokes you must see DC-CORE as NHS.

If spokes not registering:
  [ ] NHRP password matches: nova@pass99
  [ ] network-id matches: 200 on all routers
  [ ] ip nhrp nhs 10.55.0.1 (hub TUNNEL IP — not WAN IP)
  [ ] ip nhrp map 10.55.0.1 10.0.0.1 (tunnel IP then WAN IP)
  [ ] ip nhrp map multicast 10.0.0.1 configured

---

### Step 4 — DMVPN State
```
DC-CORE# show dmvpn
DC-CORE# show dmvpn detail
```
Expected output on DC-CORE:
  WAW-R1  10.55.0.11  NBMA: 10.0.0.11  State: UP
  LIS-R2  10.55.0.12  NBMA: 10.0.0.12  State: UP
  CAI-R3  10.55.0.13  NBMA: 10.0.0.13  State: UP

---

### Step 5 — IPsec Sessions
```
show crypto session
show crypto isakmp sa
show crypto ipsec sa
```
Must show: Session status: UP-ACTIVE
Pkts encaps and decaps must be increasing

If DOWN:
  [ ] tunnel protection ipsec profile NOVA-IPSEC-PROF applied
  [ ] tunnel mode gre multipoint configured BEFORE tunnel protection
  [ ] Use: debug crypto isakmp to see negotiation

---

### Step 6 — EIGRP Neighbors
```
show ip eigrp neighbors
show ip eigrp interfaces detail
```
DC-CORE must see all 3 spokes as neighbors.
Each spoke must see DC-CORE as neighbor.

If neighbors missing:
  [ ] network 10.55.0.0 0.0.0.255 in router eigrp 200
  [ ] no ip split-horizon eigrp 200 on DC-CORE Tunnel0
  [ ] hello/hold timers match: hello 20, hold 60
  [ ] Use: debug eigrp packets hello

---

### Step 7 — Routing Table
```
show ip route eigrp
```
Every router must see all 4 LAN subnets:
  172.20.0.0/24  DC-CORE LAN
  172.21.0.0/24  WAW-R1 LAN
  172.22.0.0/24  LIS-R2 LAN
  172.23.0.0/24  CAI-R3 LAN

If routes missing: check EIGRP network statements and distribute-list

---

### Step 8 — Spoke-to-Spoke Verification
```
WAW-R1# traceroute 172.23.0.1 source ethernet0/1
```
Expected: 1 hop directly to 10.55.0.13 (CAI-R3 tunnel IP)
NOT 2 hops via 10.55.0.1 (DC-CORE)

If 2 hops:
  [ ] tunnel mode gre multipoint on SPOKE (not just hub)
  [ ] Try twice — first ping always via hub, direct from 2nd ping

---

### Step 9 — MTU Verification
```
WAW-R1# ping 172.23.0.1 size 1400 df-bit repeat 5
```
All 5 pings must succeed without fragmentation
If fails: verify ip mtu 1400 and ip tcp adjust-mss 1360 on tunnel

---

## ⚡ Quick Debug Commands

```
debug nhrp                  ← NHRP registration issues
debug nhrp authentication   ← password mismatch
debug dmvpn all             ← full DMVPN events
debug tunnel                ← tunnel key mismatch (silent drops)
debug eigrp packets hello   ← neighbor formation
debug crypto isakmp         ← IPsec phase 1
debug crypto ipsec          ← IPsec phase 2
undebug all                 ← ALWAYS run this after debugging
```

---

## 🔑 Most Common Mistakes in This Lab

| Mistake | Symptom | Fix |
|---------|---------|-----|
| NHRP map uses tunnel IP for NBMA | Spokes never register | Use WAN IP (10.0.0.1) as NBMA |
| tunnel mode point-to-point on spoke | No spoke-to-spoke | Change to gre multipoint |
| Split-horizon not disabled on hub | Spokes blind to each other | no ip split-horizon eigrp 200 |
| tunnel key mismatch | Silent packet drops | Verify key 200 on all routers |
| Wrong distribute-list direction | Routes missing | OUT on hub, IN on spokes |
| IPsec configured before tunnel mode | Crypto session DOWN | Reorder: mode first, then protection |
