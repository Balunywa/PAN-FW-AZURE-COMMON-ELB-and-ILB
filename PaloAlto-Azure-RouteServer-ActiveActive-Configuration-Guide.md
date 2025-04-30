This document explains how to configure a pair of Palo Alto firewalls in an active-active deployment between your internal VNet (Trust) and the Internet (Untrust), peering to Azure Route Server for dynamic route exchange.

1. Interfaces
Define two L3 interfaces:

ethernet1/2 → Trust-zone (inside)

ethernet1/1 → Untrust-zone (Internet)

xml
Copy
Edit
<!-- ========================= -->
<!--      Interface Config     -->
<!-- ========================= -->
<network>
  <interface>
    <ethernet>
      <!-- Trust-Zone Interface -->
      <entry name="ethernet1/2">
        <layer3>
          <ipv4>
            <!-- 
              YOUR_INSIDE_IP/32:
              • Firewall Trust-zone interface IP
              • Used as BGP router-ID & local‐address 
            -->
            <entry name="10.10.1.0/32"/>
          </ipv4>
          <interface-management-profile>HTTPS-ext</interface-management-profile>
        </layer3>
      </entry>
      
      <!-- Untrust-Zone Interface (Internet) -->
      <entry name="ethernet1/1">
        <layer3>
          <ipv4>
            <!-- YOUR_OUTSIDE_IP/32 -->
            <entry name="10.10.1.64/32"/>
          </ipv4>
          <interface-management-profile>HTTPS-ext</interface-management-profile>
        </layer3>
      </entry>
    </ethernet>
  </interface>
2. Interface-Management Profile
Create a profile to allow Azure health-probes and HTTPS/ICMP management from 168.63.129.16/32.

xml
Copy
Edit
<!-- ========================= -->
<!-- Interface-Mgmt Profile    -->
<!-- ========================= -->
<profiles>
  <interface-management-profile>
    <entry name="HTTPS-ext">
      <https>yes</https>
      <ping>yes</ping>
      <permitted-ip>
        <!-- Azure health-probe/load-balancer IP -->
        <entry name="168.63.129.16/32"/>
      </permitted-ip>
    </entry>
  </interface-management-profile>
</profiles>
3. BGP Configuration
Peer to Azure Route Server using a peer-group ars.

xml
Copy
Edit
<!-- ========================= -->
<!--      BGP Configuration    -->
<!-- ========================= -->
<protocol>
  <bgp>
    <!-- Must match your Trust-interface IP -->
    <router-id>10.10.1.0</router-id>

    <peer-group>
      <entry name="ars">
        <peer>
          <!-- Azure Route Server #1 -->
          <entry name="ars1">
            <peer-address><ip>10.10.1.132</ip></peer-address>
            <local-address>
              <ip>10.10.1.0/32</ip>
              <interface>ethernet1/2</interface>
            </local-address>
          </entry>

          <!-- Azure Route Server #2 -->
          <entry name="ars2">
            <peer-address><ip>10.10.1.133</ip></peer-address>
            <local-address>
              <ip>10.10.1.0/32</ip>
              <interface>ethernet1/2</interface>
            </local-address>
          </entry>
        </peer>
      </entry>
    </peer-group>
  </bgp>
</protocol>
4. Static Routes
Ensure reachability before and after BGP adjacency:

xml
Copy
Edit
<!-- ========================= -->
<!--      Static Routes        -->
<!-- ========================= -->
<routing-table>
  <ip>
    <!-- 1) Default Internet via Untrust -->
    <static-route>
      <entry name="DefaultRoute">
        <destination>0.0.0.0/0</destination>
        <nexthop><ip-address>10.10.1.64</ip-address></nexthop>
        <interface>ethernet1/1</interface>
      </entry>
    </static-route>

    <!-- 2) Internal VNet via Trust -->
    <static-route>
      <entry name="VNET">
        <destination>10.0.0.0/17</destination>
        <nexthop><ip-address>10.0.3.1</ip-address></nexthop>
        <interface>ethernet1/2</interface>
      </entry>
    </static-route>

    <!-- 3) Azure Health-Probe on Trust -->
    <static-route>
      <entry name="Probe">
        <destination>168.63.129.16/32</destination>
        <nexthop><ip-address>10.0.3.1</ip-address></nexthop>
        <interface>ethernet1/2</interface>
      </entry>
    </static-route>

    <!-- 4) Azure Health-Probe on Untrust -->
    <static-route>
      <entry name="ProbeExt">
        <destination>168.63.129.16/32</destination>
        <nexthop><ip-address>10.10.1.64</ip-address></nexthop>
        <interface>ethernet1/1</interface>
      </entry>
    </static-route>

    <!-- 5) Static for BGP Peer Subnet -->
    <static-route>
      <entry name="StaticForARS">
        <destination>10.10.1.128/25</destination>
        <nexthop><ip-address>10.0.3.1</ip-address></nexthop>
        <interface>ethernet1/2</interface>
      </entry>
    </static-route>
  </ip>
</routing-table>
5. Address Objects
Define reusable address objects for security and NAT policies:

xml
Copy
Edit
<!-- ========================= -->
<!--     Address Objects       -->
<!-- ========================= -->
<address>
  <entry name="IpForPalo">
    <ip-netmask>10.10.1.0/32</ip-netmask>
    <description>Firewall internal (Trust) IP</description>
  </entry>
  <entry name="IpForUntrust">
    <ip-netmask>10.10.1.64/32</ip-netmask>
    <description>Firewall external (Untrust) IP</description>
  </entry>
</address>
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

pgsql
Copy
Edit
> show routing protocol bgp summary
> show system log
This configuration provides a resilient, active-active Palo Alto firewall segment with dynamic routing via Azure Route Server, ensuring fast failover and load-sharing for both Internet and internal VNet traffic.
