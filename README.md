# Lab: DMVPN Phase 2 — NovaTech Industries Enterprise WAN

## 🏢 Scenario
**Company:** NovaTech Industries  
**Situation:** NovaTech has one Data Center (DC) and three branch offices  
(Warsaw, Lisbon, Cairo). The WAN team must build a secure DMVPN overlay  
so all branches communicate with the DC and directly with each other  
without traffic going through the DC for branch-to-branch flows.  
An IPsec profile is already deployed on all routers by the security team.  
EIGRP AS 200 is pre-configured on DC-CORE only.

---

## 🗺️ Topology Diagram

```
   172.21.0.0/24      172.22.0.0/24      172.23.0.0/24
   [WAW-R1 LAN]       [LIS-R2 LAN]       [CAI-R3 LAN]
        |                   |                   |
     Eth0/1              Eth0/1              Eth0/1
    [WAW-R1]            [LIS-R2]            [CAI-R3]
  Tunnel:10.55.0.11  Tunnel:10.55.0.12  Tunnel:10.55.0.13
     Eth0/0              Eth0/0              Eth0/0
   10.0.0.11           10.0.0.12           10.0.0.13
        \                   |                   /
         \                  |                  /
          \                 |                 /
           +-------[SW1 — ISP Simulated]------+
                            |ping 10.0.0
                        10.0.0.1
                        Eth0/0
                       [DC-CORE]
                     Tunnel:10.55.0.1
                        Eth0/1
                    172.20.0.0/24
                   
```

---

## 📋 IP Addressing Table

| Device  | Role  | Eth0/0 WAN   | Eth0/1 LAN    | Tunnel0 IP    |
|---------|-------|--------------|---------------|---------------|
| DC-CORE | Hub   | 10.0.0.1/24  | 172.20.0.1/24 | 10.55.0.1/24  |
| WAW-R1  | Spoke | 10.0.0.11/24 | 172.21.0.1/24 | 10.55.0.11/24 |
| LIS-R2  | Spoke | 10.0.0.12/24 | 172.22.0.1/24 | 10.55.0.12/24 |
| CAI-R3  | Spoke | 10.0.0.13/24 | 172.23.0.1/24 | 10.55.0.13/24 |
| SW1     | ISP   | —            | —             | —             |

> **Why same subnet on WAN side?**  
> SW1 is a Layer 2 switch simulating the internet. All routers must be  
> in the same subnet (10.0.0.0/24) so they can reach each other through it.  
> In real life each site would have a different ISP-assigned public IP,  
> but the DMVPN logic is identical — the WAN IPs are the NBMA addresses.

---

## ⚙️ PNET Build Steps

```
Step 1 — Add devices in PNET:
  4x Cisco Router (IOSv or IOS 15.x)
  1x Ethernet Switch (built-in PNET node)
  Rename them: DC-CORE | WAW-R1 | LIS-R2 | CAI-R3 | ISP

Step 2 — Connect cables:
  DC-CORE Eth0/0 ──► ISP port 0
  WAW-R1  Eth0/0 ──► ISP port 1
  LIS-R2  Eth0/0 ──► ISP port 2
  CAI-R3  Eth0/0 ──► ISP port 3

  Eth0/1 on each router is the local LAN — no switch needed

Step 3 — Configure WAN interfaces first:
  DC-CORE(config-if)# ip address 10.0.0.1 255.255.255.0
  WAW-R1 (config-if)# ip address 10.0.0.11 255.255.255.0
  LIS-R2 (config-if)# ip address 10.0.0.12 255.255.255.0
  CAI-R3 (config-if)# ip address 10.0.0.13 255.255.255.0

Step 4 — Verify ISP simulation (must work BEFORE DMVPN):
  DC-CORE# ping 10.0.0.11  ✅ WAW-R1
  DC-CORE# ping 10.0.0.12  ✅ LIS-R2
  DC-CORE# ping 10.0.0.13  ✅ CAI-R3

Step 5 — Configure LAN interfaces (Eth0/1 on each router)

Step 6 — Build DMVPN following the tasks below
```

---

## 🔐 NHRP & Tunnel Parameters

| Parameter     | Value           |
|---------------|-----------------|
| NHRP Password | nova@p99     |
| Network-ID    | 200             |
| Tunnel Key    | 200             |
| Hold Time     | 360 sec (6 min) |
| Tunnel Subnet | 10.55.0.0/24    |
| Tunnel Mode   | GRE Multipoint  |
| IPsec Profile | NOVA-IPSEC-PROF |

---

##  Tasks

### Task 1 — Configure DC-CORE as DMVPN Hub
- Tunnel source: Ethernet0/0
- Tunnel IP: 10.55.0.1/24
- NHRP: authentication, network-id 200, multicast dynamic
- Disable EIGRP split-horizon on Tunnel0
- Apply IPsec profile NOVA-IPSEC-PROF
- Tunnel key 200

### Task 2 — Configure WAW-R1 (Warsaw Spoke)
- Tunnel IP: 10.55.0.11/24
- NHRP: auth, network-id 200, holdtime 360
- NHRP NHS: 10.55.0.1
- NHRP map: 10.55.0.1 → 10.0.0.1
- ip mtu 1400 / ip tcp adjust-mss 1360
- Apply IPsec profile / Join EIGRP 200

### Task 3 — Configure LIS-R2 (Lisbon Spoke)
- Same as Task 2 but tunnel IP: 10.55.0.12/24

### Task 4 — Configure CAI-R3 (Cairo Spoke)
- Same as Task 2 but tunnel IP: 10.55.0.13/24

### Task 5 — ⭐ Extra Challenge: EIGRP Tuning
- Hello interval 20 / hold time 60 on all Tunnel0 interfaces
- no auto-summary on all routers
- Advertise only LAN subnets (172.x.x.x) into EIGRP
- Verify all 4 LANs visible on every router

### Task 6 — ⭐ Extra Challenge: Route Filtering
- DC-CORE: permit only 172.20.0.0/14 le 24 outbound to spokes
- WAW-R1: accept only routes with mask /24 or shorter inbound
- Verify: no tunnel or WAN subnets in spoke routing tables

### Task 7 — ⭐ Extra Challenge: Redundancy Test
- Shut Tunnel0 on LIS-R2
- Verify WAW-R1 still reaches CAI-R3 (fallback via DC-CORE)
- Bring LIS-R2 back — verify spoke-to-spoke re-establishes
- Note the convergence time

### Task 8 — Final Verification
- traceroute WAW-R1 → CAI-R3 Eth0/1 → must be 1 hop direct
- traceroute LIS-R2 → WAW-R1 Eth0/1 → must be 1 hop direct
- All 3 spokes in show dmvpn on DC-CORE

---

## 🔧 Full Configuration

### DC-CORE — Hub
```
interface Ethernet0/0
 ip address 10.0.0.1 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 172.20.0.1 255.255.255.0
 no shutdown

interface Tunnel0
 ip address 10.55.0.1 255.255.255.0
 ip nhrp authentication nova@pass99
 ip nhrp network-id 200
 ip nhrp map multicast dynamic
 no ip split-horizon eigrp 200
 ip hello-interval eigrp 200 20
 ip hold-time eigrp 200 60
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
 tunnel protection ipsec profile NOVA-IPSEC-PROF

router eigrp 200
 no auto-summary
 network 10.55.0.0 0.0.0.255
 network 172.20.0.0 0.0.0.255

ip prefix-list LAN-ONLY seq 10 permit 172.20.0.0/24 le 24
ip prefix-list LAN-ONLY seq 20 deny 0.0.0.0/0 le 32
route-map FILTER-TO-SPOKES permit 10
 match ip address prefix-list LAN-ONLY
router eigrp 200
 distribute-list route-map FILTER-TO-SPOKES out Tunnel0
```

### WAW-R1 — Warsaw
```
interface Ethernet0/0
 ip address 10.0.0.11 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 172.21.0.1 255.255.255.0
 no shutdown

interface Tunnel0
 ip address 10.55.0.11 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication nova@pass99
 ip nhrp network-id 200
 ip nhrp holdtime 360
 ip nhrp nhs 10.55.0.1
 ip nhrp map 10.55.0.1 10.0.0.1
 ip nhrp map multicast 10.0.0.1
 ip hello-interval eigrp 200 20
 ip hold-time eigrp 200 60
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
 tunnel protection ipsec profile NOVA-IPSEC-PROF

router eigrp 200
 no auto-summary
 network 10.55.0.0 0.0.0.255
 network 172.21.0.0 0.0.0.255

ip prefix-list MAX-24 seq 10 permit 0.0.0.0/0 le 24
router eigrp 200
 distribute-list prefix MAX-24 in Tunnel0
```

### LIS-R2 — Lisbon
```
interface Ethernet0/0
 ip address 10.0.0.12 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 172.22.0.1 255.255.255.0
 no shutdown

interface Tunnel0
 ip address 10.55.0.12 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication nova@pass99
 ip nhrp network-id 200
 ip nhrp holdtime 360
 ip nhrp nhs 10.55.0.1
 ip nhrp map 10.55.0.1 10.0.0.1
 ip nhrp map multicast 10.0.0.1
 ip hello-interval eigrp 200 20
 ip hold-time eigrp 200 60
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
 tunnel protection ipsec profile NOVA-IPSEC-PROF

router eigrp 200
 no auto-summary
 network 10.55.0.0 0.0.0.255
 network 172.22.0.0 0.0.0.255

ip prefix-list MAX-24 seq 10 permit 0.0.0.0/0 le 24
router eigrp 200
 distribute-list prefix MAX-24 in Tunnel0
```

### CAI-R3 — Cairo
```
interface Ethernet0/0
 ip address 10.0.0.13 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 172.23.0.1 255.255.255.0
 no shutdown

interface Tunnel0
 ip address 10.55.0.13 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication nova@pass99
 ip nhrp network-id 200
 ip nhrp holdtime 360
 ip nhrp nhs 10.55.0.1
 ip nhrp map 10.55.0.1 10.0.0.1
 ip nhrp map multicast 10.0.0.1
 ip hello-interval eigrp 200 20
 ip hold-time eigrp 200 60
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
 tunnel protection ipsec profile NOVA-IPSEC-PROF

router eigrp 200
 no auto-summary
 network 10.55.0.0 0.0.0.255
 network 172.23.0.0 0.0.0.255

ip prefix-list MAX-24 seq 10 permit 0.0.0.0/0 le 24
router eigrp 200
 distribute-list prefix MAX-24 in Tunnel0
```

---

## ✅ Verification & Expected Output

```
DC-CORE# show dmvpn detail
Ident           NBMA Addr   State
10.55.0.11/32   10.0.0.11   UP   ← WAW-R1 ✅
10.55.0.12/32   10.0.0.12   UP   ← LIS-R2 ✅
10.55.0.13/32   10.0.0.13   UP   ← CAI-R3 ✅

WAW-R1# traceroute 172.23.0.1 source ethernet0/1
  1  10.55.0.13  ← DIRECT 1 hop, not via DC-CORE ✅

WAW-R1# show ip nhrp
10.55.0.1/32  NBMA: 10.0.0.1   Type: static
10.55.0.13/32 NBMA: 10.0.0.13  Type: dynamic ← after traceroute

WAW-R1# show crypto session
Session status: UP-ACTIVE ✅

DC-CORE# show ip eigrp neighbors
10.55.0.11  Tu0  ← WAW-R1
10.55.0.12  Tu0  ← LIS-R2
10.55.0.13  Tu0  ← CAI-R3
```

---

## 🐛 Troubleshooting Log

### Problem 1 — Spokes Not Registering
```
Symptom : show dmvpn on DC-CORE shows no spokes
Cause   : ip nhrp map used tunnel IP instead of WAN IP for hub
Fix     : ip nhrp map 10.55.0.1 10.0.0.1
          tunnel IP ──┘          └── WAN/NBMA IP (these are different!)
```

### Problem 2 — Spoke-to-Spoke Still Via Hub
```
Symptom : traceroute shows 2 hops through DC-CORE
Cause   : tunnel mode gre point-to-point on spoke (wrong)
Fix     : tunnel mode gre multipoint on ALL routers including spokes
Note    : First ping always via hub — direct forms from 2nd ping
```

### Problem 3 — EIGRP Neighbors Flapping
```
Symptom : Neighbors resetting repeatedly
Cause   : Hello/Dead timer mismatch between routers
Fix     : Verify ip hello-interval eigrp 200 20 AND
          ip hold-time eigrp 200 60 on all tunnel interfaces
```

### Problem 4 — IPsec Session Down
```
Symptom : show crypto session → DOWN
Cause   : tunnel protection applied before tunnel mode gre multipoint
Fix     : In IOS config order matters — always:
          1. tunnel mode gre multipoint
          2. tunnel protection ipsec profile NOVA-IPSEC-PROF
```

### Problem 5 — Routes Missing After Filtering
```
Symptom : Spokes not seeing DC LAN after distribute-list
Cause   : Direction wrong on distribute-list
Fix     : DC-CORE → distribute-list OUT Tunnel0
          Spokes  → distribute-list IN  Tunnel0
```

---

## 📚 Key Concepts Learned

1. SW1 (Layer 2 switch) simulates the ISP — all WAN interfaces must
   be in the same subnet (10.0.0.0/24) for the switch to forward frames

2. NHRP map takes TWO different IPs — never mix them up:
   ip nhrp map [tunnel-IP-of-hub] [WAN-IP-of-hub]
   10.55.0.1 is the overlay (tunnel) address
   10.0.0.1 is the underlay (NBMA/physical) address

3. no ip split-horizon eigrp 200 on Hub Tunnel0 is mandatory —
   without it spokes cannot learn each other's routes through the hub

4. Phase 2 spoke-to-spoke sequence:
   Pkt 1: WAW-R1 → DC-CORE → CAI-R3 (hub forwards + sends NHRP redirect)
   NHRP: WAW-R1 resolves CAI-R3 NBMA address
   Pkt 2+: WAW-R1 → CAI-R3 directly via dynamic tunnel

5. ip mtu 1400 + ip tcp adjust-mss 1360 prevents fragmentation
   after encryption (GRE ~24B + IPsec ~60B overhead on 1500B MTU)

6. tunnel key must match on ALL routers — causes silent packet drops

---

## 🧠 Interview Questions This Lab Covers

- What is the difference between DMVPN Phase 1, 2, and 3?
- Why is no split-horizon needed on the hub?
- What happens on the first ping between two spokes?
- Why set ip mtu 1400 on DMVPN tunnels?
- What are the two IPs in ip nhrp map and what does each represent?
- What breaks if a spoke uses point-to-point GRE instead of multipoint?
- How does distribute-list direction affect route filtering in EIGRP?
- How do you confirm spoke-to-spoke is truly direct?
=======
# Lab-dmvpn-phase2-hub-spoke
>>>>>>> ff5bf631cef131c0530999e9716a0341335567f0
