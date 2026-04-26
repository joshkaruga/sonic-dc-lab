# SONiC BGP EVPN VXLAN Fabric Lab

A hands-on study lab built on NVIDIA Air, exploring the same 
network stack used by hyperscale data centers at Google, 
Microsoft, and Meta.

## What This Is

This lab runs a fully functional spine-leaf DC fabric using:
- **SONiC** (sonic-vs-202305) — the open-source NOS used by 
  major hyperscalers
- **BGP** — as the underlay routing protocol (eBGP, unique AS 
  per leaf)
- **EVPN** — as the overlay control plane
- **VXLAN** — for tenant traffic encapsulation and isolation

## Topology
[spine01 AS65199]    [spine02 AS65199]
      |      \          /      |
      |       \        /       |
[leaf01]    [leaf02]    [leaf03]
AS65101     AS65102     AS65103
   |            |           |
   - 2 spines, 3 leaves, 6 Ubuntu servers
- All fabric links: eBGP unnumbered point-to-point (/31)
- VTEP loopbacks: 10.0.0.1 (leaf01), 10.0.0.2, 10.0.0.3

## What I Verified

### Underlay (BGP)
- eBGP sessions confirmed on all leaf-spine links
- Spine routing table shows BGP-learned routes to all 3 leaf 
  loopbacks
- ECMP confirmed — leaf01 has 2 equal-cost paths to each remote 
  VTEP (via spine01 and spine02)

### Overlay (EVPN-VXLAN)
- EVPN Type-2 routes (MAC/IP) exchanged across all leaves
- EVPN Type-3 routes (BUM/multicast) active per VNI
- VNI map confirmed: Vlan10→VNI10, Vlan20→VNI20
- Remote VTEPs discovered via BGP EVPN (not manual config)

### VRF Tenant Separation
- Created Vrf_TenantA on leaf01
- Bound VNI 10 and Vlan10 interface to VRF
- Configured anycast gateway: 10.10.10.1/24
- Verified prefix 10.10.10.0/24 present in VRF BGP table
- Identified FRR 8.5.1 limitation: Type-5 EVPN export from VRF 
  requires L3VNI declaration via SONiC ConfigDB in this build. 
  Production fix documented in notes/frr-limitation.md

## End-to-End Connectivity Test

Verified the full L2-to-L3 path through the configured VRF works 
end to end by pinging from server01 to leaf01's anycast gateway.

**Setup:**
- Assigned server01 (eth1) an IP in the Vlan10 subnet: `10.10.10.100/24`
- Pinged leaf01's anycast gateway: `10.10.10.1` (Vrf_TenantA)

**Result:**
ubuntu@server01:~$ ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.962 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.409 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.385 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=0.630 ms
--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3078ms
rtt min/avg/max/mdev = 0.385/0.596/0.962/0.231 ms

![Successful ping from server01 to leaf01 anycast gateway](ping_success.png)

**What this proves:**

The ping packet traversed every layer of the stack:
- server01's Linux eth1 interface
- leaf01's physical port (member of Vlan10)
- leaf01's Vlan10 SVI (10.10.10.1/24, bound to Vrf_TenantA)
- The VRF's local routing table
- Return path through the same chain

4/4 packets received with sub-millisecond latency confirms the 
VLAN-to-VNI mapping, the SVI configuration, the VRF binding, 
and the gateway IP are all correctly wired together. This is 
the foundational building block — every more complex fabric 
behavior is this same pattern repeated.

## Fabric Health Script

`fabric_health.py` automates the verification steps above. It 
SSHes into leaf01 via the oob-mgmt-server jump host and runs 
5 checks, printing a PASS/FAIL report.

Checks:
1. BGP session health (both spines up, prefixes received)
2. EVPN session health (L2VPN peers active)
3. VNI map presence (VNI 10 and VNI 20)
4. Remote VTEP discovery (leaf02 and leaf03)
5. ECMP redundancy (2+ paths to each VTEP)

```bash
pip install paramiko
python fabric_health.py
```

## Why This Matters

This is the same technology stack that underlies AI data center 
fabrics — including NVIDIA DGX SuperPOD deployments and 
Microsoft's CO+I AI infrastructure. SONiC + BGP EVPN-VXLAN is 
the de facto standard for hyperscale DC networking.

Understanding it at this level — control plane verification, 
VRF tenant isolation, ECMP path redundancy — maps directly to 
L5/L6 network engineering roles.

## Platform

- NVIDIA Air (air.nvidia.com) — free browser-based simulation
- SONiC version: sonic-vs-202305
- FRR version: 8.5.1
