# Commands Reference — NovaTech DMVPN Lab

## 📋 IP Addressing Table

| Device  | Role  | Eth0/0 WAN   | Eth0/1 LAN    | Tunnel0 IP    |
|---------|-------|--------------|---------------|---------------|
| DC-CORE | Hub   | 10.0.0.1/24  | 172.20.0.1/24 | 10.55.0.1/24  |
| WAW-R1  | Spoke | 10.0.0.11/24 | 172.21.0.1/24 | 10.55.0.11/24 |
| LIS-R2  | Spoke | 10.0.0.12/24 | 172.22.0.1/24 | 10.55.0.12/24 |
| CAI-R3  | Spoke | 10.0.0.13/24 | 172.23.0.1/24 | 10.55.0.13/24 |
| SW1     | ISP   | all connected to it via Eth0/0 | | |

---

## 🔑 The Most Important IP Relationship to Remember

```
ip nhrp map  [HUB-TUNNEL-IP]  [HUB-WAN-IP]
             10.55.0.1         10.0.0.1
             (overlay)         (underlay/NBMA)

ip nhrp nhs  [HUB-TUNNEL-IP]
             10.55.0.1

ip nhrp map multicast  [HUB-WAN-IP]
                       10.0.0.1
```

---

## 🔧 DMVPN Show Commands

```
show dmvpn
show dmvpn detail
show ip nhrp
show ip nhrp nhs
show ip nhrp nhs detail
show ip nhrp summary
show interface tunnel0
show ip interface tunnel0
```

## 🔒 IPsec / Crypto Show Commands

```
show crypto session
show crypto session detail
show crypto isakmp sa
show crypto ipsec sa
show crypto map
```

## 🔀 EIGRP Show Commands

```
show ip eigrp neighbors
show ip eigrp neighbors detail
show ip eigrp interfaces
show ip eigrp interfaces detail
show ip eigrp topology
show ip eigrp topology all-links
show ip eigrp traffic
```

## 🗺️ Routing Show Commands

```
show ip route
show ip route eigrp
show ip route 10.55.0.0
show ip route 172.20.0.0
show ip protocols
```

---

## 🎯 Key Verification One-Liners

| What to verify              | Command                                         |
|-----------------------------|-------------------------------------------------|
| All spokes registered       | DC-CORE# show dmvpn                             |
| IPsec encrypting traffic    | show crypto ipsec sa (pkts encaps/decaps > 0)   |
| EIGRP neighbors up          | show ip eigrp neighbors                         |
| Spoke-to-spoke direct       | WAW-R1# traceroute 172.23.0.1 src eth0/1        |
| MTU correct                 | show interface tunnel0 (MTU 1400)               |
| No WAN routes in table      | show ip route eigrp (only 172.x.x.x/24 routes) |
| Dynamic NHRP entry exists   | show ip nhrp (dynamic entry for remote spoke)   |

---

## 📝 Spoke Config Template (copy and adjust tunnel IP per spoke)

```
interface Ethernet0/0
 ip address [WAN-IP] 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address [LAN-IP] 255.255.255.0
 no shutdown

interface Tunnel0
 ip address [TUNNEL-IP] 255.255.255.0
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
 network [LAN-NETWORK] [WILDCARD]

ip prefix-list MAX-24 seq 10 permit 0.0.0.0/0 le 24
router eigrp 200
 distribute-list prefix MAX-24 in Tunnel0
```

---

## 🖥️ Tunnel IP per Spoke

| Spoke  | Replace [TUNNEL-IP] with |
|--------|--------------------------|
| WAW-R1 | 10.55.0.11               |
| LIS-R2 | 10.55.0.12               |
| CAI-R3 | 10.55.0.13               |
