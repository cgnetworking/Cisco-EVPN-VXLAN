# Cisco-EVPN-VXLAN

Ansible automation for a Cisco NX-OS leaf/spine EVPN-VXLAN fabric (eBGP underlay + eBGP EVPN overlay).

## Before You Deploy (Required Changes)

Update these files before running the playbook in your own environment.

### 1) Inventory and device access (`inventory.yml`)

- Replace lab hostnames (`clab-nxos-*`) with your real device names.
- Add `ansible_host` per device if DNS does not resolve hostnames.
- Change device login credentials:
  - `ansible_user`
  - `ansible_password`
  - `ansible_become_password`
- Set the correct BGP ASNs:
  - Spine ASN under `spines.vars.asn`
  - Unique leaf ASNs under each leaf host.
- Keep `id` values unique per node (used to build loopback and VTEP IPs).

### 2) Fabric-wide variables (`group_vars/all.yml`)

- Set non-overlapping addressing for your environment:
  - `loopback_base`
  - `vtep_loopback_base`
  - `links[*].leaf_ip` / `links[*].spine_ip`
- Set tenant objects for your design:
  - `vrfs` (name, L3 VLAN, L3 VNI)
  - `vlans` (VLAN, L2 VNI, anycast gateway, VRF mapping)
- Validate uniqueness:
  - No duplicate VLAN IDs, VNIs, or subnets.
  - `vrfs[*].l3_vlan` must not overlap tenant L2 VLANs.
- Set local management user/key:
  - `ssh_local_user`
  - `ssh_local_user_role`
  - `ssh_local_user_pubkey`
- Set `anycast_gateway_mac` to your standard value.
- Confirm `fabric_mtu` is supported end-to-end on your links.

### 3) Physical/virtual link mapping (`group_vars/all.yml` -> `links`)

- Match each `leaf_intf` / `spine_intf` pair to your actual cabling.
- Ensure each link IP pair is a valid /31 (or adjust design and playbooks if using a different mask).

### 4) Optional containerlab topology files

If you are using containerlab, update:

- `topology.clab.yml` and/or `topology-with-hosts.clab.yml`
  - `mgmt.ipv4-subnet`
  - `nodes[*].mgmt-ipv4`
  - NX-OS image tag under `kinds.cisco_n9kv.image`
- If using host endpoints (`topology-with-hosts.clab.yml`), change host IP/gateway commands under `host10`/`host20` as needed.

### 5) Playbook choice and host port assumptions

- Use `playbook.yml` for fabric-only deployment.
- Use `playbook-with-hosts.yml` when also configuring host-facing ports.
- In `playbook-with-hosts.yml`, adjust the host access port assumptions if needed:
  - `Ethernet1/3` on `leaf01` -> VLAN 10
  - `Ethernet1/3` on `leaf02` -> VLAN 20

## Deploy

Install required collection and run one playbook:

```bash
ansible-galaxy collection install cisco.nxos
ansible-playbook playbook.yml
# or
ansible-playbook playbook-with-hosts.yml
```
