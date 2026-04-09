# Playbook Review

This document reviews [playbook.yml](playbook.yml) as a Cisco Nexus 9300 VXLAN EVPN fabric build that uses:

- eBGP for the routed underlay
- eBGP for the EVPN overlay
- BGP ingress replication instead of multicast
- manual route-target configuration for eBGP

## Review Summary

The current playbook is structured like a non-vPC leaf/spine EVPN fabric. It follows the Cisco 10.5(x) NX-OS VXLAN guide for the eBGP underlay and the eBGP EVPN overlay, while intentionally not implementing the iBGP, OSPF, IS-IS, multicast-underlay, TRM, or vPC-specific branches of the guide.

## Current Policy And Tag Names

- Loopback redistribution route-map: `LOOPBACK-ROUTES`
- EVPN next-hop preservation route-map: `NEXT-HOP-UNCHANGED`
- Host SVI redistribution route-map: `HOST-SVI-ROUTES`
- Loopback route tag variable: `bgp_loopback_tag` = `11111`
- Host SVI route tag variable: `host_svi_route_tag` = `22222`

## Source Key

- `UG`: Cisco 10.5(x) VXLAN Guide, [Configuring the Underlay](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_the_underlay.html)
- `EG-VLAN`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), section "Configure VLAN and VXLAN VNI"
- `EG-VRF`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), section "Configure VRF for VXLAN routing"
- `EG-CoreSVI`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), section "Configure SVI for core-facing VXLAN routing"
- `EG-HostSVI`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), section "Configure SVI for host-facing VXLAN routing"
- `EG-NVE`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), sections "Configure the NVE interface and VNIs" and "Configure VXLAN EVPN ingress replication"
- `EG-BGP`: Cisco 10.5(x) VXLAN Guide, [Configure VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html), sections "Configure BGP on the VTEP" and the eBGP sample configuration in the same chapter
- `ANS`: Ansible docs, [cisco.nxos.nxos_config module](https://docs.ansible.com/ansible/latest/collections/cisco/nxos/nxos_config_module.html)

## Leaf Play

| Line | Play or Task | Reason | Source |
| --- | --- | --- | --- |
| 8 | Configure leafs (VXLAN EVPN on NX-OS) | Applies all VTEP-side routing, VXLAN, EVPN, and gateway configuration to the leaf switches. | `UG`, `EG-BGP`, `EG-NVE` |
| 22 | Enable required NX-OS features | Turns on the NX-OS feature gates required before BGP, SVIs, VNIs, and EVPN configuration can exist. | `UG`, `EG-VLAN`, `EG-VRF` |
| 33 | Configure Loopback0 | Creates the stable router-id and overlay peering source used by BGP EVPN. | `UG`, `EG-BGP` |
| 40 | Configure Loopback1 for VTEP | Creates the VTEP source address that NVE uses for VXLAN encapsulation. | `EG-NVE` |
| 47 | Configure tenant VRFs | Creates a tenant routing context and binds each VRF to its L3VNI. | `EG-VRF` |
| 55 | Configure tenant VRF route-targets | Uses explicit RT import/export because Cisco requires manual RTs for eBGP fabrics instead of `auto`. | `EG-VRF` |
| 65 | Configure L3VNI VLAN mappings | Maps each L3VNI to a VLAN so NX-OS can instantiate routed VXLAN services. | `EG-VLAN`, `EG-VRF` |
| 72 | Configure L3VNI SVIs | Builds the core-facing SVI used for inter-subnet routing inside the tenant VRF. | `EG-CoreSVI` |
| 83 | Configure leaf underlay interfaces | Creates routed point-to-point uplinks with fabric MTU toward the spines. | `UG` |
| 95 | Configure EVPN next-hop route-map | Creates `NEXT-HOP-UNCHANGED`, which preserves the original VTEP next hop when EVPN routes are re-advertised in eBGP. | `EG-BGP` |
| 101 | Configure leaf loopback redistribution route-map | Creates `LOOPBACK-ROUTES`, which restricts underlay redistribution to loopbacks tagged with `bgp_loopback_tag`. | `UG`, `EG-BGP` |
| 107 | Configure host SVI redistribution route-map | Creates `HOST-SVI-ROUTES`, which restricts tenant VRF redistribution to host gateway SVIs tagged with `host_svi_route_tag`. | `EG-HostSVI`, `EG-BGP` |
| 113 | Configure base BGP on leafs | Instantiates the leaf BGP process and sets a deterministic router-id. | `UG`, `EG-BGP` |
| 119 | Configure leaf BGP underlay address-family | Enables ECMP and redistributes `LOOPBACK-ROUTES` so overlay and VTEP addresses are reachable across the underlay. | `UG`, `EG-BGP` |
| 128 | Configure leaf BGP EVPN address-family | Enables the EVPN control plane and applies `NEXT-HOP-UNCHANGED` at the EVPN address-family level. | `EG-BGP` |
| 137 | Configure leaf underlay BGP neighbors | Creates the eBGP sessions from each leaf to its directly connected spines. | `UG` |
| 146 | Configure leaf EVPN neighbors | Creates multihop eBGP EVPN sessions from each leaf to the spine loopbacks. | `EG-BGP` |
| 155 | Configure leaf EVPN neighbor address-family | Sends communities and applies the per-neighbor `NEXT-HOP-UNCHANGED` policy required by the eBGP overlay. | `EG-BGP` |
| 167 | Configure BGP VRF routing for leafs | Exports tenant IPv4 routes from the VRF RIB into EVPN and redistributes only `HOST-SVI-ROUTES` connected routes. | `EG-VRF`, `EG-BGP` |
| 179 | Configure VLAN to VNI mappings | Binds each bridged tenant VLAN to its L2 VNI. | `EG-VLAN` |
| 186 | Configure SVIs for tenant gateways | Creates the distributed anycast default gateway for local endpoints in each VLAN and tags the connected route with `host_svi_route_tag`. | `EG-HostSVI` |
| 196 | Configure NVE interface base | Brings up the VTEP interface, points it at Loopback1, and enables BGP-based host reachability. | `EG-NVE` |
| 205 | Configure leaf L2 VNIs on NVE | Attaches each bridged VNI to `nve1` and enables BGP ingress replication for BUM traffic. | `EG-NVE` |
| 214 | Configure leaf L3 VNIs on NVE | Attaches each routed VNI to `nve1` for tenant routing across the fabric. | `EG-NVE` |
| 221 | Configure EVPN L2 VNI attributes | Uses explicit EVPN RD/RT overrides for L2 VNIs instead of relying on `auto`. | `EG-VLAN` |
| 232 | Save leaf configuration when modified | Saves the running configuration to startup configuration only when Ansible changed the device. | `ANS` |

## Spine Play

| Line | Play or Task | Reason | Source |
| --- | --- | --- | --- |
| 240 | Configure spines (VXLAN EVPN on NX-OS) | Applies the routed-underlay and EVPN-control-plane configuration to the transit spines. | `UG`, `EG-BGP` |
| 252 | Enable required NX-OS features on spines | Enables the BGP and EVPN control-plane features required on the spines. | `UG`, `EG-BGP` |
| 260 | Configure spine Loopback0 | Creates the stable router-id and EVPN overlay peering source on each spine. | `UG`, `EG-BGP` |
| 267 | Configure spine underlay interfaces | Creates routed point-to-point links toward the leaves with the fabric MTU. | `UG` |
| 279 | Configure EVPN next-hop route-map on spines | Creates `NEXT-HOP-UNCHANGED`, which preserves the originating leaf VTEP as next hop when the spine re-advertises EVPN routes. | `EG-BGP` |
| 285 | Configure spine loopback redistribution route-map | Creates `LOOPBACK-ROUTES`, which restricts underlay redistribution to the spine loopback tagged with `bgp_loopback_tag`. | `UG`, `EG-BGP` |
| 291 | Configure base BGP on spines | Instantiates the spine BGP process and sets the router-id. | `UG`, `EG-BGP` |
| 297 | Configure spine BGP underlay address-family | Enables ECMP and redistributes `LOOPBACK-ROUTES` into the eBGP underlay. | `UG`, `EG-BGP` |
| 306 | Configure spine BGP EVPN address-family | Enables EVPN, applies `NEXT-HOP-UNCHANGED`, and retains all RTs so the spine can relay EVPN routes without owning local VNIs. | `EG-BGP` |
| 315 | Configure spine underlay BGP neighbors | Creates the eBGP point-to-point underlay sessions from each spine to the leaves. | `UG` |
| 324 | Configure spine EVPN neighbors | Creates the multihop eBGP EVPN sessions from each spine to the leaf loopbacks. | `EG-BGP` |
| 333 | Configure spine EVPN neighbor address-family | Sends communities and applies the per-neighbor `NEXT-HOP-UNCHANGED` policy required for eBGP route propagation. | `EG-BGP` |
| 345 | Save spine configuration when modified | Saves the running configuration only when the play changed the device. | `ANS` |

## Review Notes

- The playbook is aligned to the eBGP branches of the Cisco guide, not the iBGP, OSPF, or IS-IS branches.
- The playbook uses BGP ingress replication, so it intentionally does not configure multicast underlay, PIM, or Anycast-RP.
- The playbook uses manual route-targets because Cisco documents `auto` route-targets as iBGP-only.
- The playbook is not a vPC design. Because of that, it does not try to build the vPC-specific backup sessions or same-AS leaf pair exceptions shown elsewhere in the guide.
