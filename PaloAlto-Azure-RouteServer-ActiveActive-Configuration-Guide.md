Configuring Active-Active Palo Alto Firewalls with Azure Route Server
This document explains how to configure a pair of Palo Alto firewalls in an active-active deployment between your internal VNet (Trust) and the Internet (Untrust), peering to Azure Route Server for dynamic route exchange.

1. Interfaces
Define two L3 interfaces:

ethernet1/2 → Trust-zone (inside)
ethernet1/1 → Untrust-zone (Internet)

2. Interface-Management Profile
Create a profile to allow Azure health-probes and HTTPS/ICMP management from 168.63.129.16/32.


3. BGP Configuration
Peer to Azure Route Server using a peer-group ars.


4. Static Routes
Ensure reachability before and after BGP adjacency:


5. Address Objects
Define reusable address objects for security and NAT policies:


6. NAT & Security Policy
Use the above zones, addresses, and interfaces to build:

Health-Probe NAT allowing Azure LB checks
DNAT from ILB to internal VM
SNAT all Trust→Internet traffic
Security rules to permit web/SSH in and all outbound
(See full XML for examples.)

Key Tips
Profile names (e.g. HTTPS-ext) must match exactly when referenced.
BGP router-id must be a /32 on the Trust interface.
Static-route names must be unique and descriptive.
Always commit and verify with:

This configuration provides a resilient, active-active Palo Alto firewall segment with dynamic routing via Azure Route Server, ensuring fast failover and load-sharing for both Internet and internal VNet traffic.
