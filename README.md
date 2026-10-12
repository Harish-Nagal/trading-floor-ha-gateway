High-Availability Dual-Homed Trading Floor GatewayAn enterprise-grade network edge architecture delivering fault-tolerant, deterministic internet connectivity for an ultra-low latency trading floor local area network. This design leverages Hot Standby Router Protocol (HSRPv2) with WAN interface state tracking and Dynamic Port Address Translation (Edge PAT) across redundant Tier-1 ISP links.Architecture OverviewFinancial trading environments demand zero single points of failure (SPOF) and sub-second deterministic gateway recovery. This implementation models a high-availability dual-homed internet gateway terminating redundant ISP uplinks:First-Hop Gateway Redundancy: HSRP version 2 provides a shared Virtual IP (10.10.10.1) serving as the persistent default gateway for trading floor workstations and low-latency servers.Interface State Tracking: The primary edge gateway (R1) actively monitors its WAN interface (GigabitEthernet0/0). If the primary uplink fails, priority decrements by 20 points, triggering immediate preemption by the standby router (R2).Symmetric Dynamic Edge NAT: Outbound sessions undergo Port Address Translation (Overload) directly on the active router's WAN IP interface, ensuring clean session tracking over whichever ISP circuit is operational.Rapid L2 Spanning-Tree Convergence: Switch access ports employ Spanning Tree PortFast to eliminate listening/learning state delay for edge clients.Logical Topology Diagram                           +---------------------------+
                           | Trading Floor Workstation |
                           |       10.10.10.50/24      |
                           |   Default GW: 10.10.10.1  |
                           +-------------+-------------+
                                         | Fa0
                                         |
                                         | Fa0/1
                           +-------------+-------------+
                           |       SW1 (Catalyst 2960) |
                           +------+-------------+------+
                                  |             |
                       Gi0/1      |             | Gi0/2
           +----------------------+             +----------------------+
           | Gi0/1                                                     | Gi0/1
+----------+-----------------------+         +-------------------------+---------+
|        R1 (Primary Edge)         |         |        R2 (Standby Edge)          |
|  LAN IP: 10.10.10.2/24           |         |  LAN IP: 10.10.10.3/24            |
|  HSRP VIP: 10.10.10.1 (Pri 110)  |  HSRPv2 |  HSRP VIP: 10.10.10.1 (Pri 100)   |
|  Track: Gi0/0 (Decrement 20)     |<------->|  Preemption: Enabled              |
|  Preemption: Enabled             |         |  Edge Dynamic PAT Enabled         |
|  Edge Dynamic PAT Enabled        |         |  WAN IP: 198.51.100.2/30          |
|  WAN IP: 203.0.113.2/30          |         +-------------------------+---------+
+----------+-----------------------+                                   | Gi0/0
           | Gi0/0                                                     |
           |                                                           |
           | Gi0/0                                                     | Gi0/0
+----------+-----------------------+         +-------------------------+---------+
|          ISP-1 Router            |         |          ISP-2 Router             |
|    WAN IP: 203.0.113.1/30        |         |    WAN IP: 198.51.100.1/30        |
|    Target Loopback: 8.8.8.8/32   |         |    Target Loopback: 8.8.8.8/32    |
+----------------------------------+         +-----------------------------------+
IP Addressing & Hardware ProfileDeviceDevice ModelInterfaceIP AddressSubnet MaskDescription / RolePC0Generic PCFastEthernet010.10.10.50255.255.255.0Trading Client (GW: 10.10.10.1)SW1Cisco 2960-24TTVLAN 1Unmanaged / L2—Floor Access Switch (PortFast on Fa0/1)R1Cisco 2911 ISRGi0/1Gi0/010.10.10.2203.0.113.2255.255.255.0255.255.255.252Trading LAN Gateway (Active)Primary ISP-1 WAN CircuitR2Cisco 2911 ISRGi0/1Gi0/010.10.10.3198.51.100.2255.255.255.0255.255.255.252Trading LAN Gateway (Standby)Secondary ISP-2 WAN CircuitISP-1Cisco 2911 ISRGi0/0Loopback0203.0.113.18.8.8.8255.255.255.252255.255.255.255Primary Tier-1 ProviderSimulated Market ServiceISP-2Cisco 2911 ISRGi0/0Loopback0198.51.100.18.8.8.8255.255.255.252255.255.255.255Secondary Tier-1 ProviderSimulated Market ServiceProtocol & Configuration Details1. Hot Standby Router Protocol (HSRPv2)Group ID: 10Virtual IP (VIP): 10.10.10.1Virtual MAC Address: 0000.0C9F.F00A (Standard HSRPv2 format: 0000.0C9F.Fxxx)Primary Priority (R1): 110Secondary Priority (R2): 100Failover Logic: R1 tracks its WAN interface Gi0/0. When Gi0/0 drops to down/down, priority decrements by 20 (leaving R1 at priority 90). R2 detects this drop and triggers preemption (standby 10 preempt), assuming the Active role in under 3 hello intervals.2. Edge Dynamic PAT (Overload)Standard Access Control List 1 permits internal traffic (10.10.10.0/24) across both gateways:access-list 1 permit 10.10.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
During steady-state, traffic exits R1 translated to 203.0.113.2. During primary link degradation or failure, outbound sessions fail over to R2 and are dynamically translated to 198.51.100.2.Verification & Failover Validation ProceduresTest 1: Steady-State Active/Standby StateRun show standby brief across both routers to confirm role election:On R1 (Primary Gateway):R1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/1       10   110 P Active  local           10.10.10.3      10.10.10.1
On R2 (Secondary Gateway):R2# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Gi0/1       10   100 P Standby 10.10.10.2      local           10.10.10.1
Test 2: Active Dynamic NAT TranslationFrom the Trading PC (10.10.10.50), send ICMP echo requests to the external host:PC> ping 8.8.8.8
Inspect the translation table on R1:R1# show ip nat translations
Pro  Inside global         Inside local          Outside local      Outside global
icmp 203.0.113.2:1         10.10.10.50:1         8.8.8.8:1          8.8.8.8:1
Test 3: Uplink Failure Simulation (WAN Blackhole Avoidance)Simulate a fiber cut on ISP-1 by shutting down R1's WAN link:R1(config)# interface GigabitEthernet0/0
R1(config-if)# shutdown
Expected Transition Behavior:R1 triggers an interface track decrement: Priority drops from 110 $\to$ 90.R2 (Priority 100) preempts R1 as Active gateway:%HSRP-5-STATECHANGE: GigabitEthernet0/1 Grp 10 state Standby -> Active
Outbound ping from PC0 (ping 8.8.8.8 -t) maintains connectivity with minimal packet loss.Active NAT translations automatically re-establish through R2:R2# show ip nat translations
Pro  Inside global         Inside local          Outside local      Outside global
icmp 198.51.100.2:1        10.10.10.50:1         8.8.8.8:1          8.8.8.8:1
Test 4: Primary Link Restoration & PreemptionRe-enable R1's WAN link:R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
R1's interface track restores its priority to 110.With standby 10 preempt configured, R1 reclaims the Active role seamlessly.Repository Structuretrading-floor-ha-gateway/
├── README.md                          # Project documentation and test procedures
├── .gitignore                         # Git exclusion rules for temp files
├── topology/
│   ├── dual_homed_hsrp_nat.pkt        # Cisco Packet Tracer lab file
│   └── network_topology_diagram.png   # Logical topology schematic
├── configs/
│   ├── R1-Primary-Gateway.ios         # Cisco IOS configuration for R1
│   ├── R2-Secondary-Gateway.ios       # Cisco IOS configuration for R2
│   ├── SW1-Trading-Switch.ios         # Cisco IOS configuration for SW1
│   ├── ISP1-Edge.ios                  # Configuration for ISP-1 stub router
│   └── ISP2-Edge.ios                  # Configuration for ISP-2 stub router
└── docs/
    └── failover-test-results.md       # CLI logs during simulated failure tests
How to Run in Cisco Packet TracerClone this repository:git clone https://github.com/Harish-Nagal/trading-floor-ha-gateway.git
Open Cisco Packet Tracer (v8.0 or later recommended).Navigate to File > Open... and select topology/dual_homed_hsrp_nat.pkt.Open the PC Command Prompt and issue ping 8.8.8.8 -t to test real-time failover.
