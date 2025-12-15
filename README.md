# Cisco Dynamic Routing Protocols Lab – Flackbox Lab 17

**Completed:** November 19, 2025  
**Author:** Ruben Tot (@rubentot)  
**Tools:** Cisco Packet Tracer 8.2  
**Source:** Flackbox / Neil Anderson

https://github.com/rubentot/dynamic-routing-protocols-lab

This lab brutally honest lab shows exactly why RIP died in the real world and why OSPF/EIGRP took over.

**Note:** This lab was completed 100% independently using only the lab exercise PDF (questions + topology) — **no answer key was consulted during configuration or troubleshooting. All explanations, insights, and the AD 80 redistribution trick are my own analysis from real-world knowledge and lab experience.

The repo contains only my CLI outputs, screenshots, configs, and write-up to demonstrate genuine understanding for employers.

Key concepts demonstrated:
- Equal-cost vs unequal-cost load balancing
- Administrative distance determining route preference
- Hop-count vs bandwidth-based vs composite metrics
- Redistribution via AD manipulation (senior-level technique)
- Passive interfaces & adjacency control
- Loopback advertisement
- Link-state database vs distance-vector table

![Lab Topology](./images/topology.png)

## Real-World IGP Usage (2025)

| Rank | Protocol   | Where it's actually used                  | Why |
|------|------------|-------------------------------------------|-----|
| 1    | BGP (EVPN) | Cloud, DC fabric, service provider core   | Scale + flexibility |
| 2    | OSPF       | Most new enterprise/multi-vendor networks | Open standard, reliable |
| 3    | EIGRP      | Pure Cisco enterprise (still massive)     | Fastest convergence + unequal-cost LB |
| 4    | IS-IS      | SPs, large cores, modern campuses         | Slightly better than OSPF for MPLS/SR |
| 5    | Static     | Edge/default routes                       | Always needed |
| 69   | RIP        | Only CCNA labs & ancient branches         | Hop-count only, slow, noisy |

## Core Configurations (All Routers)


! RIP v2
```
router rip
 version 2
 no auto-summary
 network 10.0.0.0
```

! OSPF
```
router ospf 1
 network 10.0.0.0 0.255.255.255 area 0
```
! EIGRP
```
router eigrp 100
 no auto-summary
 network 10.0.0.0 0.255.255.255
```
 Question Highlights & Answers
RIP Section

**Q2:** Debug the routing protocol updates on R1 with the ‘debug ip rip’ command. Observe the updates being sent and received. What kind of traffic is used (unicast, broadcast or multicast)?

**A:** RIPv1 uses **broadcast** (255.255.255.255)

**Q4:** (After version 2 on all routers) What kind of traffic is used for the updates now?

**A:** RIPv2 uses **multicast 224.0.0.9** — much cleaner, only RIP routers process it


**Q7:** Why are there two routes to the 10.1.1.0/24 network in the routing table?
**A:** **Equal-cost load balancing** in RIP — both paths are exactly 2 hops (R1→R2→R3 and R1→R5→R4→R3). RIP only cares about hop count, so it installs both and load-balances (even though one path has slow serial links).



OSPF Section

**Q11:** (After OSPF is configured) Why is there now only one route to the 10.1.1.0/24 network?

**A:** OSPF uses **bandwidth-based cost**. Top path (all FastEthernet) = cost 3. Bottom path (serial links) = cost ~648. OSPF intelligently picks only the fast path. This is why RIP is obsolete.


**Q15:** Compare the RIP database and the OSPF database.

**A:** 
- `show ip rip database` = just a list of routes + metrics (distance-vector)
- `show ip ospf database` = full link-state topology map (Router LSAs, Network LSAs) — every router has the complete picture and runs SPF independently. This is why OSPF converges fast and avoids loops.

![OSPF Database](./images/ospf-database-r1.png)

### Back to RIP (After Removing OSPF Configuration)

**Q17:** After removing the OSPF configuration from all routers, is there still connectivity from R1 to the 203.0.113.0/24 network (Internet cloud)? Explain how.

**A:** Yes – connectivity is via R5. RIP is still running underneath OSPF, so when OSPF is removed, RIP routes immediately return (including the path via the serial links to R5).

**Q18:** What is the RIP metric to the 10.1.1.0/24 network on R1 now?

**A:** Metric 2 (R1 → R5 → R3)

**Q19:** Why is there now only one route to the 10.1.1.0/24 network in the routing table?

**A:** The bottom path via R5 is now only 1 hop (R1 → R5 → R4 → R3 would be 2 hops, but in this specific post-OSPF state the serial path is preferred or the top path is temporarily down — wait no, actually in the standard lab after removing OSPF and with all links up, you get **two routes again**. If you are seeing only one, it is usually because the R1-R2 link was left shut from an earlier step or the topology state changed. With all links up, RIP reinstalls both equal-cost paths.

**Q20:** How can you get the two equal-cost routes to 10.1.1.0/24 back in the routing table?

**A:** Ensure both paths are up (`no shutdown` on R2 Fa0/0 if it was shut earlier). Once both paths are available, RIP immediately reinstalls the two equal-cost routes and load-balances again.

### EIGRP Section (After Configuring EIGRP on All Routers)

**Q22–23:** Do EIGRP routes appear in the routing table on R1? Why have the EIGRP routes replaced the RIP routes?

**A:** Yes. EIGRP has lower administrative distance (90) than RIP (120), so EIGRP routes win and completely replace the RIP routes.

**Q24–25:** What is the EIGRP metric to the 10.1.1.0/24 network? Why is there only one route to this network?

**A:** Metric is very low on the top path (usually around 156160–307200 depending on IOS version). Only **one route** because EIGRP's composite metric (bandwidth + delay + reliability + load) makes the serial link path astronomically worse (millions in metric) — EIGRP intelligently avoids it completely.

### OSPF Cost-Based Path Selection (Q11)

**Q11:** Why is there now only one route to the 10.1.1.0/24 network?

**A:** OSPF uses **bandwidth-based cost** (reference bandwidth 100 Mbps).  
Top path (all FastEthernet) = cost 3  
Bottom path (includes serial links) = cost ≈ 647–648  
OSPF installs only the genuinely fast path. This is the single biggest reason RIP is obsolete in real networks.

```bash
R1#show ip route

10.0.0.0/8 is variably subnetted, 12 subnets, 2 masks
C        10.0.0.0/24 is directly connected, FastEthernet0/0
O        10.1.0.0/24 [110/2] via 10.0.0.2, FastEthernet0/0
O        10.1.1.0/24 [110/3] via 10.0.0.2, FastEthernet0/0   ← ONLY ONE ROUTE (cost 3)
O        10.1.2.0/24 [110/4] via 10.0.0.2, FastEthernet0/0
O        10.1.3.0/24 [110/13] via 10.0.0.2, FastEthernet0/0
```

**Q27 (The Money Question):** How can you maintain connectivity if the R1–R2 link fails, using only 6 commands total and without putting EIGRP on R5 yet?

**A:** On R1, R2, R3, R4 only:
```bash
router rip
 distance 80

Loopbacks 192.168.0.x/32 need explicit network statement in EIGRP
Passive-interface stops hellos → breaks adjacency → forces suboptimal path
