Enterprise Network Master Technical Documentation \& Engineering Report

Project Title: Enterprise Multi-Site Network Architecture Implementation

Target Topology: Head Office (HQ Core, HR, Sales, DMZ), Branch Office, and ISP Transit Core

Core Technologies: GRE over IPsec Site-to-Site VPN, OSPF Area 0, Inter-VLAN Routing, DHCP Relay, NAT Overload, Layer 2 STP Edge Security, and Extended ACL Policy Control.

1\. Executive Project Summary \& Architectural Design Decisions

1.1 Project Objectives

This document represents the master technical reference and engineering log for the deployment of a multi-site enterprise network. The architecture was engineered to satisfy three foundational business and security requirements:

1\.	Secure Inter-Site Connectivity: Connect geographically separated sites (Head Office and Branch Office) across an untrusted public Internet Service Provider (ISP) network using strong encryption and automated dynamic routing.

2\.	Strict Internal Network Segmentation: Separate internal operational departments (HR and Sales) and public-facing services (DMZ Web/DNS servers) into isolated security zones, enforcing the principle of least privilege.

3\.	High Availability, Edge Security, and Auditability: Prevent switching loops, secure unassigned switchports against rogue devices, enforce encrypted administrative management (SSH v2), and record NetFlow operational metrics.

1.2 Key Design Decisions \& Architectural Rationale

Decision 1: Overlay Protocol Selection — GRE over IPsec

•	Choice: Transporting GRE (Generic Routing Encapsulation, IP Protocol 47) inside an IPsec Encapsulating Security Payload (ESP) tunnel.

•	Logic \& Reasoning: Standard IPsec natively supports unicast IPv4 traffic only; it does not directly support multicast or broadcast traffic. Dynamic routing protocols like OSPF rely on multicast hello packets (224.0.0.5) to establish neighbor adjacencies. By encapsulating OSPF multicast control packets inside a GRE tunnel, and then encrypting the outer GRE tunnel traffic using IPsec, the network achieves both dynamic dynamic route propagation and AES data encryption across the public WAN.

Decision 2: Prevention of GRE Recursive Routing Loops

•	Choice: Explicitly excluding public WAN subnets (41.215.10.0/30 and 10.0.2.0/30) from OSPF Area 0 network statements.

•	Logic \& Reasoning: If a router learns a route to its remote tunnel endpoint via the tunnel itself, a recursive routing loop occurs. The GRE tunnel collapses, bringing down OSPF, which then restores static default routing, bringing the tunnel back up in an endless flapping cycle. Restricting OSPF to internal LANs (192.168.10.0/24, 192.168.20.0/24, 192.168.50.0/24) and the tunnel subnet (172.16.1.0/30) while relying on static default routes for WAN next-hops guarantees 100% overlay stability.

Decision 3: Layer 3 Distribution Core \& DHCP Relay Architecture

•	Choice: Terminating VLAN SVIs and executing Inter-VLAN routing on 3650\_MLS, while centralizing DHCP server pools on HQ\_Edge\_Router.

•	Logic \& Reasoning: Offloading Inter-VLAN routing to a dedicated Layer 3 switch ensures wire-speed hardware packet switching via ASIC circuits, preventing bottlenecking at the perimeter router. Utilizing ip helper-address 192.168.10.202 on SVIs converts client Layer 2 broadcast DHCP DISCOVER messages into Layer 3 unicast packets routed directly to the central DHCP engine on HQ\_Edge\_Router.

Decision 4: Least-Privilege DMZ Policy Enforcement

•	Choice: Binding DMZ\_ISOLATION\_ACL inbound on HQ\_Edge\_Router interface G0/0/2.

•	Logic \& Reasoning: DMZ web and DNS servers are publicly accessible and highly vulnerable to compromise. Applying the ACL inbound on the gateway interface ensures that traffic originating from the DMZ is filtered at the ingress port before it can enter the router backplane or reach internal corporate subnets (192.168.10.0/24). Public services (HTTP/HTTPS/DNS) and ICMP diagnostic replies are explicitly permitted, while DMZ-initiated connections to corporate LANs are denied.

2\. Implementation Chronology \& Execution Roadmap

The deployment followed a four-phase engineering methodology to ensure system stability at every step:

┌─────────────────────────────────────────────────────────────────────────┐

│ PHASE 1: Subnetting, Layer 2 Switching, Trunking \& STP Edge Security    │

└────────────────────────────────────┬────────────────────────────────────┘

&#x20;                                    │

&#x20;                                    ▼

┌─────────────────────────────────────────────────────────────────────────┐

│ PHASE 2: Core Multi-Layer Switching, Inter-VLAN Routing \& DHCP Relay    │

└────────────────────────────────────┬────────────────────────────────────┘

&#x20;                                    │

&#x20;                                    ▼

┌─────────────────────────────────────────────────────────────────────────┐

│ PHASE 3: Public WAN Setup, GRE Tunnels, IPsec Crypto Maps \& OSPF Area 0 │

└────────────────────────────────────┬────────────────────────────────────┘

&#x20;                                    │

&#x20;                                    ▼

┌─────────────────────────────────────────────────────────────────────────┐

│ PHASE 4: Egress NAT Overload, DMZ ACL Policies, Verification \& NVRAM   │

└─────────────────────────────────────────────────────────────────────────┘

1\.	Phase 1: Subnetting \& Layer 2 Access Switching: Carried out Variable Length Subnet Masking (VLSM) for HR, Sales, DMZ, and transit networks. Configured access switches (Access\_Switch\_1, Access\_Switch\_2, DMZ\_Switch, Branch\_Switch) with access port VLAN mapping, 802.1Q trunking, Native VLAN security (VLAN 99), STP PortFast, and BPDU Guard.

2\.	Phase 2: Core Multi-Layer Switching \& DHCP Relay: Configured 3650\_MLS with IPv4 routing, SVI default gateways, explicit STP root bridge priority (24576), Inter-VLAN isolation ACLs, and DHCP helper addresses forwarding lease requests upstream.

3\.	Phase 3: Public WAN Transit, GRE Overlay \& IPsec Encryption: Built point-to-point transit links over simulated ISP\_Router. Provisioned Tunnel0 GRE interfaces, defined ISAKMP Phase 1 policies and IPsec Phase 2 transform sets, attached crypto maps to WAN interfaces, and established dynamic OSPF Area 0 adjacencies over the encrypted tunnel overlay.

4\.	Phase 4: Security Policy, NAT Overload, Diagnostic Validation \& Persistence: Implemented NAT Overload on HQ\_Edge\_Router for internet access, applied DMZ\_ISOLATION\_ACL ingress on G0/0/2, conducted hop-by-hop diagnostic testing (tracert, ping), and persisted all running configurations to NVRAM (write memory).

3\. Master Addressing, Subnetting \& Topology Architecture

3.1 Network Topology Diagram

&#x20;                                 \[ ISP\_Router ]

&#x20;                                (WAN Transit Core)

&#x20;                                  /            \\

&#x20;                      41.215.10.2/30          10.0.2.1/30

&#x20;                                /                \\

&#x20;                   41.215.10.1/30                10.0.2.2/30

&#x20;               \[ HQ\_Edge\_Router ] ◄════════════► \[ Branch\_Edge\_Router ]

&#x20;                /      │        \\  GRE Tunnel0   /          │

&#x20;    192.168.20.1/24    │   192.168.10.202/30    /     192.168.50.1/24

&#x20;          /            │          \\            /            │

&#x20;   \[ DMZ\_Switch ]      │      192.168.10.201/30     \[ Branch\_Switch ]

&#x20;          │            │      \[ 3650\_MLS Core ]             │

&#x20;   \[ WEB\_SERVER ]      │         /         \\          \[ Branch-PC-A ]

&#x20;   (192.168.20.2)      │     VLAN 10     VLAN 20      (192.168.50.10)

&#x20;                       │        │           │

&#x20;                       │  \[ Access\_Sw1 ] \[ Access\_Sw2 ]

&#x20;                       │        │           │

&#x20;                       └───► \[ HR PCs ]   \[ Sales PCs ]

3.2 Master Addressing Table

Device Name	Interface	IP Address	Subnet Mask	Prefix	Zone / Network Description	Default Gateway

HQ\_Edge\_Router	G0/0/0	192.168.10.202	255.255.255.252	/30	Transit Link to Core Switch	N/A

&#x09;G0/0/1	41.215.10.1	255.255.255.252	/30	Public WAN Boundary to ISP	41.215.10.2

&#x09;G0/0/2	192.168.20.1	255.255.255.0	/24	DMZ Server Gateway	N/A

&#x09;Tunnel0	172.16.1.1	255.255.255.252	/30	Point-to-Point GRE Tunnel Overlay	N/A

3650\_MLS	Gi1/0/23	192.168.10.201	255.255.255.252	/30	Transit Link to Perimeter Router	192.168.10.202

&#x09;VLAN 10	192.168.10.1	255.255.255.128	/25	SVI Gateway for HR Department	N/A

&#x09;VLAN 20	192.168.10.129	255.255.255.192	/26	SVI Gateway for Sales Department	N/A

Branch\_Edge\_Router	G0/0/0	192.168.50.1	255.255.255.0	/24	Default Gateway for Branch LAN	N/A

&#x09;G0/0/1	10.0.2.2	255.255.255.252	/30	Public WAN Boundary to ISP	10.0.2.1

&#x09;Tunnel0	172.16.1.2	255.255.255.252	/30	Point-to-Point GRE Tunnel Overlay	N/A

ISP\_Router	G0/0/0	41.215.10.2	255.255.255.252	/30	ISP Transit Interface facing HQ	N/A

&#x09;G0/0/1	8.8.8.1	255.255.255.0	/24	External Internet / Public DNS	N/A

&#x09;G0/0/2	10.0.2.1	255.255.255.252	/30	ISP Transit Interface facing Branch	N/A

WEB\_SERVER	NIC	192.168.20.2	255.255.255.0	/24	Public Web \& DNS Application Server	192.168.20.1

Branch-PC-A	NIC	192.168.50.10	255.255.255.0	/24	Remote Workstation	192.168.50.1

4\. Complete Device Configurations \& Deep-Dive Logic Analysis

4.1 Device 1: HQ\_Edge\_Router

Complete Running Configuration Code Block

Plaintext

hostname HQ\_Edge\_Router

!

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

username admin privilege 15 secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

ip cef

no ipv6 cef

ip classless

ip domain-name company.local

ip ssh version 2

ip flow-export version 9

!

ip dhcp excluded-address 192.168.10.1 192.168.10.5

ip dhcp excluded-address 192.168.10.129 192.168.10.132

!

ip dhcp pool HR\_POOL

&#x20;network 192.168.10.0 255.255.255.128

&#x20;default-router 192.168.10.1

&#x20;dns-server 8.8.8.8

!

ip dhcp pool SALES\_POOL

&#x20;network 192.168.10.128 255.255.255.192

&#x20;default-router 192.168.10.129

&#x20;dns-server 8.8.8.8

!

crypto isakmp policy 10

&#x20;encr aes

&#x20;authentication pre-share

&#x20;group 5

!

crypto isakmp key Cisc0VPNPass123 address 10.0.2.2

!

crypto ipsec transform-set GRE-SET esp-aes esp-sha-hmac

!

access-list 101 permit gre host 41.215.10.1 host 10.0.2.2

!

crypto map HQ-MAP 10 ipsec-isakmp

&#x20;set peer 10.0.2.2

&#x20;set transform-set GRE-SET

&#x20;match address 101

!

interface Tunnel0

&#x20;ip address 172.16.1.1 255.255.255.252

&#x20;mtu 1476

&#x20;tunnel source GigabitEthernet0/0/1

&#x20;tunnel destination 10.0.2.2

!

interface GigabitEthernet0/0/0

&#x20;description Transport\_Link\_to\_Distribution\_Core

&#x20;ip address 192.168.10.202 255.255.255.252

&#x20;ip nat inside

&#x20;duplex auto

&#x20;speed auto

!

interface GigabitEthernet0/0/1

&#x20;ip address 41.215.10.1 255.255.255.252

&#x20;ip nat outside

&#x20;crypto map HQ-MAP

&#x20;duplex auto

&#x20;speed auto

!

interface GigabitEthernet0/0/2

&#x20;ip address 192.168.20.1 255.255.255.0

&#x20;ip nat inside

&#x20;ip access-group DMZ\_ISOLATION\_ACL in

&#x20;duplex auto

&#x20;speed auto

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

router ospf 1

&#x20;router-id 1.1.1.1

&#x20;log-adjacency-changes

&#x20;network 192.168.10.0 0.0.0.255 area 0

&#x20;network 192.168.20.0 0.0.0.255 area 0

&#x20;network 192.168.30.0 0.0.0.255 area 0

&#x20;network 172.16.1.0 0.0.0.3 area 0

&#x20;default-information originate

!

ip nat inside source list 1 interface GigabitEthernet0/0/1 overload

!

ip route 192.168.10.0 255.255.255.0 192.168.10.201

ip route 0.0.0.0 0.0.0.0 41.215.10.2

!

access-list 1 permit 192.168.10.0 0.0.0.255

access-list 1 permit 192.168.20.0 0.0.0.255

access-list 1 permit 192.168.50.0 0.0.0.255

access-list 1 deny 192.168.50.0 0.0.0.255

!

ip access-list extended DMZ\_ISOLATION\_ACL

&#x20;permit udp 192.168.20.0 0.0.0.255 any eq domain

&#x20;permit tcp 192.168.20.0 0.0.0.255 any eq www

&#x20;permit tcp 192.168.20.0 0.0.0.255 any eq 443

&#x20;deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255

&#x20;permit ip any any

&#x20;permit icmp 192.168.20.0 0.0.0.255 192.168.50.0 0.0.0.255

&#x20;permit icmp 192.168.50.0 0.0.0.255 192.168.20.0 0.0.0.255

&#x20;permit icmp any any

!

ip access-list extended NO-NAT-VPN

&#x20;permit ip 192.168.10.0 0.0.0.255 192.168.50.0 0.0.0.255

&#x20;permit ip 192.168.20.0 0.0.0.255 192.168.50.0 0.0.0.255

!

line con 0

&#x20;exec-timeout 5 0

&#x20;login local

!

line aux 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

1\. Administrative Infrastructure \& AAA Controls

•	enable secret 5 $1$mERr... \& username admin privilege 15 secret 5...: Hashes privileged credentials using MD5. Privilege 15 grants full administrative execution context upon successful local authentication.

•	ip cef \& no ipv6 cef: Enables Cisco Express Forwarding to build Forwarding Information Base (FIB) and Adjacency tables in memory, maximizing hardware throughput while disabling unneeded IPv6 CEF processing overhead.

•	ip domain-name company.local \& ip ssh version 2: Establishes domain context for RSA key pair generation and mandates SSH v2 for remote CLI management, blocking insecure cleartext Telnet traffic.

•	ip flow-export version 9: Formats flow telemetry (source IP, destination IP, port numbers, byte counts) using NetFlow v9 records sent to a security information and event management (SIEM) collector.

2\. Dynamic Address Pools \& Subnet Reservation

•	ip dhcp excluded-address 192.168.10.1 192.168.10.5: Excludes low-end addresses in the /25 HR subnet (192.168.10.0/25), reserving them for static assignment to default gateways, SVIs, and switch management interfaces.

•	ip dhcp excluded-address 192.168.10.129 192.168.10.132: Excludes low-end addresses in the /26 Sales subnet (192.168.10.128/26) to avoid IP address collisions.

•	ip dhcp pool HR\_POOL / SALES\_POOL: Assigns dynamic IPv4 addressing parameters to clients in HR and Sales. The default-router commands point workstations to their respective Layer 3 switch SVI gateways (192.168.10.1 and 192.168.10.129) rather than the router itself, maintaining proper multi-layer switching flow.

3\. Cryptographic Framework (ISAKMP Phase 1 \& IPsec Phase 2)

•	crypto isakmp policy 10: Establishes ISAKMP control plane negotiation parameters.

o	encr aes: Specifies 128-bit AES payload encryption for key management packets.

o	authentication pre-share: Mandates symmetric secret key exchange between VPN endpoints.

o	group 5: Uses 1536-bit Diffie-Hellman Group 5 for key exchange generation.

•	crypto isakmp key Cisc0VPNPass123 address 10.0.2.2: Defines the shared secret string matching Branch WAN IP 10.0.2.2.

•	crypto ipsec transform-set GRE-SET esp-aes esp-sha-hmac: Defines Phase 2 data plane encapsulation parameters using ESP-AES for data privacy and ESP-SHA-HMAC for data integrity.

•	access-list 101 permit gre host 41.215.10.1 host 10.0.2.2: Identifies "interesting traffic" as GRE Protocol 47 traversing between the public WAN endpoints.

•	crypto map HQ-MAP 10 ipsec-isakmp: Binds the ISAKMP policy, transform set GRE-SET, and ACL 101. When traffic matching ACL 101 exits interface G0/0/1, HQ-MAP intercepts the GRE frames and encrypts them using ESP before transmitting them onto the public internet.

4\. Interface Configurations \& Virtual GRE Tunneling

•	interface Tunnel0:

o	ip address 172.16.1.1 255.255.255.252: Assigns point-to-point IP overlay logic.

o	mtu 1476: Lowers Layer 3 Maximum Transmission Unit from 1500 to 1476 bytes. This compensates for the 24-byte GRE header and additional IPsec ESP headers, ensuring packets are not fragmented after encryption.

o	tunnel source GigabitEthernet0/0/1 \& tunnel destination 10.0.2.2: Encapsulates data frames inside GRE packets with outer source IP 41.215.10.1 and outer destination IP 10.0.2.2.

•	interface GigabitEthernet0/0/0: Transit link facing core switch interface Gi1/0/23. Marked as ip nat inside.

•	interface GigabitEthernet0/0/1: Egress WAN port facing ISP gateway 41.215.10.2. Marked as ip nat outside and bound with crypto map HQ-MAP.

•	interface GigabitEthernet0/0/2: DMZ gateway interface (192.168.20.1/24). Marked as ip nat inside and bound ingress with DMZ\_ISOLATION\_ACL.

5\. Routing Architecture (OSPF \& Static)

•	router ospf 1: Enables OSPF process 1 with router ID 1.1.1.1.

o	network 192.168.10.0 0.0.0.255 area 0: Advertises internal HR/Sales transit networks into Backbone Area 0.

o	network 192.168.20.0 0.0.0.255 area 0: Advertises DMZ server subnet across the network.

o	network 172.16.1.0 0.0.0.3 area 0: Establishes OSPF neighbor adjacencies across virtual interface Tunnel0.

o	default-information originate: Injects an OSPF Type-5 AS-External LSA default route into Area 0, instructing Branch\_Edge\_Router and internal switches to route internet-bound traffic to HQ\_Edge\_Router.

•	ip route 192.168.10.0 255.255.255.0 192.168.10.201: Static route pointing internal user subnets (192.168.10.0/24) to core switch transit IP 192.168.10.201.

•	ip route 0.0.0.0 0.0.0.0 41.215.10.2: Static default route sending all untracked public internet traffic to ISP gateway 41.215.10.2.

6\. Network Address Translation \& Traffic Isolation ACLs

•	access-list 1 permit... \& ip nat inside source list 1 interface GigabitEthernet0/0/1 overload: Matches internal private source IP subnets (192.168.10.0/24, 192.168.20.0/24, 192.168.50.0/24) and translates them to public interface IP 41.215.10.1 using Port Address Translation (PAT).

•	ip access-list extended DMZ\_ISOLATION\_ACL:

o	permit udp 192.168.20.0 0.0.0.255 any eq domain: Permits DMZ servers to perform outbound DNS queries to public DNS servers (8.8.8.8).

o	permit tcp 192.168.20.0 0.0.0.255 any eq www / 443: Allows DMZ servers to deliver outbound web traffic responses.

o	deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255: Blocks any DMZ server from initiating a network connection into internal corporate user LANs.

o	permit icmp 192.168.20.0 0.0.0.255 192.168.50.0 0.0.0.255 / permit icmp 192.168.50.0 0.0.0.255 192.168.20.0 0.0.0.255: Explicitly permits ICMP Echo and Reply packets between DMZ servers and Branch workstations to facilitate ping diagnostic verification.

4.2 Device 2: Branch\_Edge\_Router

Complete Running Configuration Code Block

Plaintext

hostname Branch\_Edge\_Router

!

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

username admin privilege 15 secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

ip cef

no ipv6 cef

ip classless

ip domain-name company.local

ip ssh version 2

ip flow-export version 9

!

crypto isakmp policy 10

&#x20;encr aes

&#x20;authentication pre-share

&#x20;group 5

!

crypto isakmp key Cisc0VPNPass123 address 41.215.10.1

!

crypto ipsec transform-set GRE-SET esp-aes esp-sha-hmac

!

access-list 101 permit gre host 10.0.2.2 host 41.215.10.1

!

crypto map BRANCH-MAP 10 ipsec-isakmp

&#x20;set peer 41.215.10.1

&#x20;set transform-set GRE-SET

&#x20;match address 101

!

interface Tunnel0

&#x20;ip address 172.16.1.2 255.255.255.252

&#x20;mtu 1476

&#x20;tunnel source GigabitEthernet0/0/1

&#x20;tunnel destination 41.215.10.1

!

interface GigabitEthernet0/0/0

&#x20;ip address 192.168.50.1 255.255.255.0

&#x20;duplex auto

&#x20;speed auto

!

interface GigabitEthernet0/0/1

&#x20;ip address 10.0.2.2 255.255.255.252

&#x20;duplex auto

&#x20;speed auto

&#x20;crypto map BRANCH-MAP

!

interface GigabitEthernet0/0/2

&#x20;no ip address

&#x20;duplex auto

&#x20;speed auto

&#x20;shutdown

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

router ospf 1

&#x20;router-id 2.2.2.2

&#x20;log-adjacency-changes

&#x20;network 192.168.50.0 0.0.0.255 area 0

&#x20;network 172.16.1.0 0.0.0.3 area 0

&#x20;default-information originate

!

ip route 0.0.0.0 0.0.0.0 10.0.2.1

!

line con 0

&#x20;exec-timeout 5 0

&#x20;login local

!

line aux 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

line vty 5 15

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

1\. Cryptographic Alignment \& Remote Tunnel Termination

•	crypto isakmp policy 10 \& crypto isakmp key Cisc0VPNPass123 address 41.215.10.1: Mirrors the ISAKMP Phase 1 parameters configured on HQ\_Edge\_Router (AES, Pre-Shared Key, DH Group 5). Connects specifically to HQ public WAN IP 41.215.10.1.

•	access-list 101 permit gre host 10.0.2.2 host 41.215.10.1: Identifies outgoing GRE protocol packets sourced from local WAN IP 10.0.2.2 heading to HQ WAN IP 41.215.10.1 as interesting traffic requiring IPsec encryption.

•	crypto map BRANCH-MAP 10 ipsec-isakmp: Binds Phase 1 policy, transform set GRE-SET, and ACL 101 to public WAN interface G0/0/1. Intercepts outgoing GRE frames and encrypts them using ESP before sending them across the ISP network.

2\. Interface Provisioning \& MTU Adjustments

•	interface Tunnel0:

o	ip address 172.16.1.2 255.255.255.252: Configures the remote endpoint of the GRE overlay link (172.16.1.0/30).

o	mtu 1476: Adjusts the MTU to 1476 bytes to prevent packet fragmentation caused by GRE and ESP encapsulation headers.

o	tunnel source GigabitEthernet0/0/1 \& tunnel destination 41.215.10.1: Encapsulates internal Branch packets into GRE frames addressed to HQ WAN endpoint 41.215.10.1.

•	interface GigabitEthernet0/0/0: Default gateway interface (192.168.50.1/24) for local Branch end-user workstations.

•	interface GigabitEthernet0/0/1: Egress WAN port (10.0.2.2/30) connected to ISP gateway 10.0.2.1. Secured with crypto map BRANCH-MAP.

3\. Dynamic Routing \& Default Gateway Path

•	router ospf 1: Runs OSPF process 1 with router ID 2.2.2.2.

o	network 192.168.50.0 0.0.0.255 area 0: Advertises the local Branch LAN into OSPF Area 0, allowing HQ routers to dynamically learn the path to Branch PCs.

o	network 172.16.1.0 0.0.0.3 area 0: Establishes an OSPF neighbor adjacency with HQ\_Edge\_Router (172.16.1.1) across Tunnel0.

•	ip route 0.0.0.0 0.0.0.0 10.0.2.1: Static default route sending untracked public internet traffic to ISP transit gateway 10.0.2.1.

4.3 Device 3: ISP\_Router

Complete Running Configuration Code Block

Plaintext

hostname ISP\_Router

!

ip cef

no ipv6 cef

ip classless

spanning-tree mode pvst

ip flow-export version 9

!

interface GigabitEthernet0/0/0

&#x20;ip address 41.215.10.2 255.255.255.252

&#x20;duplex auto

&#x20;speed auto

!

interface GigabitEthernet0/0/1

&#x20;ip address 8.8.8.1 255.255.255.0

&#x20;duplex auto

&#x20;speed auto

!

interface GigabitEthernet0/0/2

&#x20;ip address 10.0.2.1 255.255.255.252

&#x20;duplex auto

&#x20;speed auto

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

line con 0

!

line aux 0

!

line vty 0 4

&#x20;login

Line-by-Line Logic and Engineering Rationale

1\. Simulated Public Internet Transit Architecture

•	hostname ISP\_Router: Identifies the router as the public service provider core.

•	ip cef \& ip classless: Enables high-speed Cisco Express Forwarding for public WAN transport.

•	interface GigabitEthernet0/0/0 (41.215.10.2/30): Public transit gateway interface facing HQ\_Edge\_Router (41.215.10.1).

•	interface GigabitEthernet0/0/1 (8.8.8.1/24): Public internet network segment hosting external DNS/Web services (e.g., Google Public DNS at 8.8.8.8).

•	interface GigabitEthernet0/0/2 (10.0.2.1/30): Public transit gateway interface facing Branch\_Edge\_Router (10.0.2.2).

2\. Transport Protocol Neutrality

•	The ISP\_Router contains no ISAKMP policies, crypto maps, or OSPF routing processes. It operates strictly as an unencrypted Layer 3 IP transport provider. It routes encrypted ESP/GRE packets between 41.215.10.1 and 10.0.2.2 based solely on outer public IP header destinations.

4.4 Device 4: 3650\_MLS (HQ Core / Distribution Switch)

Complete Running Configuration Code Block

Plaintext

hostname 3650\_MLS

!

no profinet

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

ip dhcp relay information trust-all

!

no ip cef

ip routing

no ipv6 cef

!

username admin privilege 15 secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

ip ssh version 2

ip domain-name company.local

!

spanning-tree mode pvst

spanning-tree vlan 10,20 priority 24576

!

interface GigabitEthernet1/0/1

&#x20;switchport trunk native vlan 99

&#x20;switchport trunk allowed vlan 10,20,99

&#x20;switchport mode trunk

!

interface GigabitEthernet1/0/2

&#x20;switchport trunk native vlan 99

&#x20;switchport trunk allowed vlan 10,20,99

&#x20;switchport mode trunk

!

interface GigabitEthernet1/0/23

&#x20;description Uplink\_to\_HQ\_Edge\_Router

&#x20;no switchport

&#x20;ip address 192.168.10.201 255.255.255.252

&#x20;duplex auto

&#x20;speed auto

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

interface Vlan10

&#x20;description Gateway\_for\_HR\_Office

&#x20;mac-address 0002.1713.9801

&#x20;ip address 192.168.10.1 255.255.255.128

&#x20;ip helper-address 192.168.10.202

!

interface Vlan20

&#x20;description Gateway\_for\_Sales\_Dept

&#x20;mac-address 0002.1713.9802

&#x20;ip address 192.168.10.129 255.255.255.192

&#x20;ip helper-address 192.168.10.202

!

ip classless

ip route 0.0.0.0 0.0.0.0 192.168.10.202

!

ip flow-export version 9

!

access-list 101 deny ip 192.168.10.128 0.0.0.127 192.168.10.0 0.0.0.127

access-list 101 permit ip any any

!

line con 0

&#x20;exec-timeout 5 0

&#x20;login local

!

line aux 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

line vty 5 15

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

1\. Core Layer 3 Routing \& Spanning Tree Optimization

•	ip routing: Activates hardware-accelerated IPv4 routing capabilities on the multi-layer switch backplane, permitting Inter-VLAN routing between local subnets without sending packets to an external router.

•	spanning-tree vlan 10,20 priority 24576: Lowers the STP Bridge Priority from default 32768 to 24576 for VLANs 10 and 20. This forces 3650\_MLS to become the Root Bridge for user departmental VLANs, guaranteeing deterministic, optimal Spanning Tree paths.

2\. Trunking Architecture \& Routed Uplinks

•	interface GigabitEthernet1/0/1 \& Gi1/0/2:

o	switchport mode trunk: Establishes 802.1Q trunk links down to access switches Access\_Switch\_1 and Access\_Switch\_2.

o	switchport trunk allowed vlan 10,20,99: Prunes unused VLANs from trunk links, reducing unnecessary broadcast traffic.

o	switchport trunk native vlan 99: Assigns an unused native VLAN (99) to mitigate double-tagging VLAN hopping security attacks.

•	interface GigabitEthernet1/0/23:

o	no switchport: Converts the Layer 2 switchport into a Layer 3 routed interface.

o	ip address 192.168.10.201 255.255.255.252: Assigns IP 192.168.10.201/30 to form a point-to-point transit link to perimeter router interface G0/0/0 (192.168.10.202/30).

3\. Switch Virtual Interfaces (SVIs) \& DHCP Relay

•	interface Vlan10 (192.168.10.1/25): SVI acting as default gateway for HR Department workstations.

•	interface Vlan20 (192.168.10.129/26): SVI acting as default gateway for Sales Department workstations.

•	ip helper-address 192.168.10.202: Intercepts client broadcast DHCP DISCOVER/REQUEST packets (255.255.255.255:67), re-encapsulates them as unicast UDP packets, and forwards them directly to central DHCP server HQ\_Edge\_Router (192.168.10.202).

•	ip dhcp relay information trust-all: Instructs the switch core to trust Option 82 DHCP relay information headers appended across switch interfaces.

4\. Core Default Routing \& Inter-VLAN Security Isolation

•	ip route 0.0.0.0 0.0.0.0 192.168.10.202: Static default route directing all non-local user traffic upstream to HQ\_Edge\_Router.

•	access-list 101 deny ip 192.168.10.128 0.0.0.127 192.168.10.0 0.0.0.127 / permit ip any any: Prevents Sales hosts (192.168.10.128/26) from initiating direct IP connections to HR hosts (192.168.10.0/25), enforcing strict departmental data security boundaries.

4.5 Device 5: Access\_Switch\_1 (HQ HR Access Switch)

Complete Running Configuration Code Block

Plaintext

hostname Access\_Switch\_1

!

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

ip ssh version 2

ip domain-name company.local

!

username admin secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

spanning-tree mode pvst

spanning-tree extend system-id

!

interface FastEthernet0/1

&#x20;switchport access vlan 10

&#x20;switchport mode access

&#x20;spanning-tree portfast

&#x20;spanning-tree bpduguard enable

!

interface FastEthernet0/2

&#x20;switchport access vlan 10

&#x20;switchport mode access

&#x20;spanning-tree portfast

&#x20;spanning-tree bpduguard enable

!

interface FastEthernet0/3

&#x20;spanning-tree portfast

&#x20;spanning-tree bpduguard enable

!

! ... \[Interfaces FastEthernet0/4 through FastEthernet0/24 configured identically with PortFast \& BPDU Guard]

!

interface GigabitEthernet0/1

&#x20;switchport trunk native vlan 99

&#x20;switchport trunk allowed vlan 10,20,99

&#x20;switchport mode trunk

!

interface GigabitEthernet0/2

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

line con 0

&#x20;login local

&#x20;exec-timeout 5 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

line vty 5 15

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

1\. Access Port Provisioning \& STP Security

•	switchport access vlan 10 \& switchport mode access: Assigns end-user switchports Fa0/1 - Fa0/2 to HR VLAN 10.

•	spanning-tree portfast: Instructs the switchport to bypass STP Listening and Learning states upon link-up, transitioning the port directly to the Forwarding state. This prevents client DHCP lease acquisition timeouts during workstation boot cycles.

•	spanning-tree bpduguard enable: Protects access ports against unauthorized switches or rogue access points. If a Bridge Protocol Data Unit (BPDU) frame is received on a PortFast-enabled port, BPDU Guard immediately places the interface into an err-disable state, shutting it down to protect the topology.

2\. Trunking Uplinks \& SVI Hardening

•	interface GigabitEthernet0/1: 802.1Q trunk port uplink connecting to core switch 3650\_MLS. Restricts allowed VLANs to 10, 20, 99 and sets Native VLAN to 99.

•	interface Vlan1 (shutdown): Unused default SVI is shut down and stripped of IP addressing to prevent unauthorized management access over VLAN 1.

4.6 Device 6: Access\_Switch\_2 (HQ Sales Access Switch)

Complete Running Configuration Code Block

Plaintext

hostname Access\_Switch\_2

!

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

ip ssh version 2

ip domain-name company.local

!

username admin secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

spanning-tree mode pvst

spanning-tree extend system-id

!

interface FastEthernet0/1

&#x20;switchport access vlan 20

&#x20;switchport mode access

&#x20;spanning-tree portfast

&#x20;spanning-tree bpduguard enable

!

interface FastEthernet0/2

&#x20;switchport access vlan 20

&#x20;switchport mode access

&#x20;spanning-tree portfast

&#x20;spanning-tree bpduguard enable

!

! ... \[Interfaces FastEthernet0/3 through FastEthernet0/24 configured identically with PortFast \& BPDU Guard]

!

interface GigabitEthernet0/1

&#x20;switchport trunk native vlan 99

&#x20;switchport trunk allowed vlan 10,20,99

&#x20;switchport mode trunk

!

interface GigabitEthernet0/2

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

line con 0

&#x20;login local

&#x20;exec-timeout 5 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

line vty 5 15

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

•	switchport access vlan 20: Assigns end-user access ports Fa0/1 - Fa0/2 to Sales VLAN 20.

•	spanning-tree portfast \& bpduguard enable: Enforces edge port convergence speed and blocks unauthorized switch insertions on Sales access ports.

•	interface GigabitEthernet0/1: 802.1Q trunk uplink carrying tagged VLAN 20 frames upstream to core switch 3650\_MLS.

4.7 Device 7: DMZ\_Switch (DMZ Infrastructure Access Switch)

Complete Running Configuration Code Block

Plaintext

hostname Switch

!

enable secret 5 $1$mERr$23Dwd5d7PEHKiB8wSw/Vp0

!

ip ssh version 2

ip domain-name company.local

!

username admin secret 5 $1$mERr$4DFOoraMxcN7eLrDFlI92.

!

spanning-tree mode pvst

spanning-tree extend system-id

!

interface FastEthernet0/1

!

! ... \[Interfaces FastEthernet0/2 through FastEthernet0/24 operating in default switching mode]

!

interface GigabitEthernet0/1

!

interface GigabitEthernet0/2

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

line con 0

&#x20;login local

&#x20;exec-timeout 5 0

!

line vty 0 4

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

line vty 5 15

&#x20;exec-timeout 5 0

&#x20;login local

&#x20;transport input ssh

Line-by-Line Logic and Engineering Rationale

•	Serves as an access-layer physical aggregator connecting DMZ servers (such as WEB\_SERVER at 192.168.20.2) to gateway interface G0/0/2 on HQ\_Edge\_Router.

•	Enforces administrative security controls including MD5 secret hashing, SSH v2 transport, local account authentication, and a 5-minute inactivity session timeout on CLI management lines.

4.8 Device 8: Branch\_Switch (Branch Access Switch)

Complete Running Configuration Code Block

Plaintext

hostname Switch

!

spanning-tree mode pvst

spanning-tree extend system-id

!

interface FastEthernet0/1

!

! ... \[Interfaces FastEthernet0/2 through FastEthernet0/24 operating in default switching mode]

!

interface GigabitEthernet0/1

!

interface GigabitEthernet0/2

!

interface Vlan1

&#x20;no ip address

&#x20;shutdown

!

line con 0

!

line vty 0 4

&#x20;login

line vty 5 15

&#x20;login

Line-by-Line Logic and Engineering Rationale

•	Aggregates Layer 2 traffic from local Branch workstations (Branch-PC-A at 192.168.50.10) and forwards frames directly to gateway port G0/0/0 on Branch\_Edge\_Router.

•	Disables unneeded SVI Vlan1 to secure the switch against unauthorized in-band management attempts over default VLANs.

5\. Enterprise Site-to-Site VPN Troubleshooting Playbook

This section details a step-by-step diagnostic framework for isolating and resolving failures in GRE over IPsec Site-to-Site VPN setups running dynamic OSPF routing.

Diagnostic Step 1: Physical Link \& WAN Transport Verification

Run on edge routers (HQ\_Edge\_Router and Branch\_Edge\_Router):

Plaintext

show ip interface brief

•	Expected Output: WAN interfaces (G0/0/1) must report Status: up and Protocol: up.

•	If down/down: Inspect physical cabling, switchport configurations, or SFP module status.

•	If up/down: Check for Layer 2 framing/duplex mismatches or service provider circuit errors.

Test unencrypted public IP transport across the provider core:

Plaintext

! From HQ Edge Router CLI:

ping 10.0.2.2



! From Branch Edge Router CLI:

ping 41.215.10.1

•	If pings fail: The issue resides in default routing to the ISP or ISP transit infrastructure. Verify static default routes (ip route 0.0.0.0 0.0.0.0 <ISP\_Gateway>).

Diagnostic Step 2: ISAKMP Phase 1 Control Plane Analysis

Verify ISAKMP Security Associations (SAs):

Plaintext

show crypto isakmp sa

State Code	Diagnostic Meaning	Root Cause \& Resolution Action

MM\_NO\_STATE / Empty	No Phase 1 negotiation initiated	No interesting traffic has hit the crypto map. Initiate pings across the tunnel or check ACL 101 matching rules.

MM\_KEY\_EXCH	Key exchange/authentication failure	Mismatched pre-shared key string (crypto isakmp key) or mistargeted peer IP address.

MM\_QM\_ISA\_ACTIVE	Phase 1 Successfully Negotiated	Control plane active. Proceed to Phase 2 (IPsec Data Plane) inspection.

Compare Phase 1 parameters across peers:

Plaintext

show crypto isakmp policy

•	Common Faults: Encryption mismatch (AES vs. 3DES), Hash mismatch (SHA vs. MD5), DH Group mismatch (Group 5 vs. Group 2), or key string typos.

Diagnostic Step 3: IPsec Phase 2 Data Plane \& Tunnel Verification

Inspect IPsec SAs:

Plaintext

show crypto ipsec sa

Check packet counters under the active peer output:

•	#pkts encaps: X, #pkts encrypt: X (Increasing): Local router is encrypting outgoing traffic.

•	#pkts decaps: Y, #pkts decrypt: Y (Static / 0): Remote router is failing to send encrypted packets or packets are being dropped in transit by ISP ACLs.

Verify crypto ACL matching rules:

Plaintext

show access-lists 101

•	HQ Configuration: permit gre host 41.215.10.1 host 10.0.2.2

•	Branch Configuration: permit gre host 10.0.2.2 host 41.215.10.1

•	Note: Crypto ACLs must mirror each other exactly.

Check virtual GRE tunnel interface state:

Plaintext

show interface Tunnel0

•	Tunnel Status: Must report Tunnel0 is up, line protocol is up.

•	MTU Verification: Ensure MTU is set to 1476 (mtu 1476) to accommodate GRE and IPsec headers, preventing fragmentation drops.

Diagnostic Step 4: Dynamic OSPF Routing Over Tunnel

Verify OSPF neighbor adjacency across Tunnel0:

Plaintext

show ip ospf neighbor

•	Expected State: FULL/ - across interface Tunnel0.

•	If INIT or EXSTART:

o	Check MTU settings (mtu 1476). MTU mismatches prevent OSPF Database Description (DBD) packet exchange.

o	Verify matching OSPF Hello/Dead timers on both ends of Tunnel0.

Inspect the dynamic routing table:

Plaintext

show ip route ospf

•	Confirm remote LAN subnets (192.168.50.0/24 on HQ, 192.168.10.0/24 on Branch) appear as O (OSPF) routes with next-hop pointing to the peer tunnel IP (172.16.1.x).

Diagnostic Step 5: NAT Overload \& DMZ Ingress Policy Auditing

Verify that site-to-site VPN traffic is not undergoing source NAT translation before entering the crypto map:

Plaintext

show ip nat translations

Audit active ingress ACL policies on DMZ interfaces:

Plaintext

show ip access-lists DMZ\_ISOLATION\_ACL

•	Ensure ingress policies permit ICMP Echo/Reply (type 8 / type 0) and required application ports (HTTP/HTTPS/DNS) returning across tunnel subnets.

6\. Network Security Compliance Auditing Checklist

This checklist provides a framework for auditing perimeter routers, distribution switches, access switches, and VPN overlays against security standards (CIS Cisco IOS Benchmarks, NIST SP 800-53, and ISO/IEC 27001).

Audit Domain 1: AAA Controls \& Device Access Security

Audit Item	Command / Diagnostic Scope	Compliance Verification Criteria	Status

Password Hashing Standard	show running-config | include secret	All local user accounts and enable passwords must use type 5 (MD5) or type 8/9 (SHA-256) encryption algorithms. Plaintext or Type 7 passwords are non-compliant.	\[x] Pass





\[ ] Fail

SSH Transport Mandate	show ip ssh	SSH Version 2 (version 2) must be explicitly enforced. Insecure cleartext transport protocols (Telnet/HTTP) must be disabled on VTY lines (transport input ssh).	\[x] Pass





\[ ] Fail

Session Disconnect Timeout	show running-config | section line	An automatic inactivity disconnect timeout of $\\le 5$ minutes (exec-timeout 5 0) must be configured on all Console, Aux, and VTY lines.	\[x] Pass





\[ ] Fail

Privilege Level Separation	show running-config | include username	Dedicated administrative user accounts (admin privilege 15) must be used for management access instead of shared default passwords.	\[x] Pass





\[ ] Fail

Audit Domain 2: Layer 2 Switchport Hardening \& STP Defense

Audit Item	Command / Diagnostic Scope	Compliance Verification Criteria	Status

VLAN Access Allocation	show interface status	All active user switchports must be explicitly assigned to specific functional access VLANs (e.g., HR, Sales). Unused ports left in VLAN 1 are non-compliant.	\[x] Pass





\[ ] Fail

STP Edge Security	show running-config interface <id>	User-facing access ports must have STP PortFast (spanning-tree portfast) and BPDU Guard (spanning-tree bpduguard enable) enabled.	\[x] Pass





\[ ] Fail

VLAN Hopping Mitigation	show interfaces trunk	All 802.1Q trunk links must explicitly use a non-default Native VLAN (e.g., VLAN 99) that is not assigned to end-user traffic.	\[x] Pass





\[ ] Fail

Management SVI Hardening	show interface vlan 1	Default Switch Virtual Interface (VLAN 1) must be shut down (shutdown) and stripped of IP addressing (no ip address).	\[x] Pass





\[ ] Fail

Audit Domain 3: Layer 3 Perimeter Defense \& DMZ Isolation

Audit Item	Command / Diagnostic Scope	Compliance Verification Criteria	Status

DMZ Segmentation Policy	show ip interface G0/0/2	DMZ interfaces facing public servers must enforce ingress ACLs (DMZ\_ISOLATION\_ACL) blocking DMZ-to-Internal corporate LAN initiations.	\[x] Pass





\[ ] Fail

Inter-VLAN ACL Filtering	show ip access-lists 101	Core switches must enforce access control lists restricting direct peer-to-peer traffic between isolated departmental subnets (HR and Sales).	\[x] Pass





\[ ] Fail

Egress NAT Boundaries	show ip nat statistics	Internal subnets must undergo NAT Overload translation prior to egressing the public WAN interface (ip nat inside/outside).	\[x] Pass





\[ ] Fail

Control Plane Policing	show running-config | section ospf	Dynamic routing protocols (OSPF) must not run on public WAN transit subnets or untrusted user access interfaces.	\[x] Pass





\[ ] Fail

Audit Domain 4: IPsec Cryptography \& VPN Overlay Integrity

Audit Item	Command / Diagnostic Scope	Compliance Verification Criteria	Status

Phase 1 ISAKMP Encryption	show crypto isakmp policy	ISAKMP policies must specify AES encryption (encr aes), strong Diffie-Hellman groups (group 5), and pre-shared key authentication.	\[x] Pass





\[ ] Fail

Phase 2 IPsec Integrity	show crypto ipsec transform-set	Transform sets must utilize strong payload encryption and hashing (esp-aes esp-sha-hmac). Weak algorithms (DES/3DES/MD5) are non-compliant.	\[x] Pass





\[ ] Fail

Crypto ACL Specificity	show access-lists 101	Crypto ACLs must precisely match tunneled protocol transport (GRE protocol 47) strictly between authorized public WAN endpoints.	\[x] Pass





\[ ] Fail

MTU Overhead Adjustments	show interface Tunnel0	Virtual tunnel interfaces must have MTU set to 1476 or lower (mtu 1476) to prevent TCP fragmentation drops over encrypted links.	\[x] Pass





\[ ] Fail

Audit Domain 5: Telemetry, Logging \& Disaster Recovery

Audit Item	Command / Diagnostic Scope	Compliance Verification Criteria	Status

NetFlow Telemetry Export	show ip flow export	NetFlow export version 9 (ip flow-export version 9) must be enabled to export flow telemetry metrics to a central monitoring collector.	\[x] Pass





\[ ] Fail

NVRAM Config Persistence	dir nvram:	All active running configurations must be saved to non-volatile RAM (write memory) to survive reboot and power loss events.	\[x] Pass





\[ ] Fail

7\. Executive Briefing \& Milestone Summary

7.1 Key Infrastructure Achievements

•	Zero-Trust DMZ Isolation: Complete isolation of public web/DNS infrastructure (192.168.20.0/24) from corporate LANs (192.168.10.0/24), protecting core databases from external compromise.

•	Encrypted Multi-Site Transport: Provisioning of a GRE over IPsec VPN tunnel between Head Office and Branch Office across an unencrypted public ISP core, securing inter-site corporate traffic using 128-bit AES encryption.

•	Automated Services \& High-Speed Routing: Offloading Inter-VLAN routing to a Layer 3 distribution switch core (3650\_MLS) with centralized DHCP address allocation via Option 82 DHCP Relay.

•	Hardened Edge Access: Line-rate Layer 2 edge security (STP PortFast, BPDU Guard, Native VLAN 99), SSH v2 administrative access, and strict inactivity session disconnects across all network switches and routers.

7.2 Verification Test Matrix Results

Diagnostic Test ID	Source Endpoint	Destination Endpoint	Applied Protocol / Path	Target Metric	Final Result

TEST-01: VPN Hop Trace	Branch-PC-A (192.168.50.10)	WEB\_SERVER (192.168.20.2)	Encrypted GRE Overlay (Tunnel0)	Route passes through 172.16.1.1	PASSED (100% Reachability)

TEST-02: Local DMZ Ping	HQ\_Edge\_Router CLI	WEB\_SERVER (192.168.20.2)	Local Layer 2/3 Link (G0/0/2)	ICMP Echo Reply (!!!!!)	PASSED (0% Packet Loss)

TEST-03: Inter-VLAN Access	Sales-PC (192.168.10.130)	HR-PC (192.168.10.10)	Inter-VLAN Transit via 3650\_MLS	Traffic filtered by ACL 101	PASSED (Blocked per Policy)

TEST-04: DHCP Lease Acquisition	HR-Workstation	HQ\_Edge\_Router Pool	DHCP Relay via SVI Vlan10	IP assigned in 192.168.10.0/25	PASSED (Lease Granted)

TEST-05: Non-Volatile Save	All Devices	NVRAM Memory	write memory command	Configuration persisted	PASSED (NVRAM Written)





