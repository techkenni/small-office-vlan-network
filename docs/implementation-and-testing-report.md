# Project 01: Small Office VLAN Network
## Implementation and testing report

**Author:** KenniTech  
**Date:** 7 November 2026  
**Platform:** Cisco Packet Tracer 9.x on macOS

## 1. Executive summary

This project implements a small-office network for three departments: Sales, IT and HR. Each department is placed in a separate VLAN and IPv4 /24 subnet. A Cisco 2960-24TT switch provides Layer 2 access switching. A Cisco 2911 router provides inter-VLAN routing through 802.1Q subinterfaces, DHCP address assignment and extended ACL filtering.

The supplied evidence shows active VLANs, an operational trunk, router subinterfaces in an up/up state, six DHCP leases, ACL definitions and interface attachment, successful configuration saves, and six final ICMP connectivity tests matching the stated departmental policy.

## 2. Requirements

| Source | Destination | Required result |
|---|---|---|
| Sales | IT | Block |
| Sales | HR | Block |
| IT | Sales | Allow |
| IT | HR | Allow |
| HR | IT | Block |
| HR | Sales | Allow |

The intended policy restricts new traffic initiated by Sales towards IT and HR, and by HR towards IT. IT is allowed to initiate communication with the other departments. HR is allowed to communicate with Sales.

## 3. Topology and equipment

- Cisco 2911 router: inter-VLAN routing, DHCP and ACL processing.
- Cisco 2960-24TT switch: VLAN membership, access switching and trunking.
- Six PC-PT endpoints: two for each department.
- One router-to-switch 802.1Q trunk.

See [the topology screenshot](../screenshots/network-topology.png) once the evidence images have been added.

## 4. Addressing plan

| Department | VLAN | Subnet | Gateway | Observed DHCP clients |
|---|---:|---|---|---|
| Sales | 10 | 192.168.10.0/24 | 192.168.10.1 | 192.168.10.11, .12 |
| IT | 20 | 192.168.20.0/24 | 192.168.20.1 | 192.168.20.11, .12 |
| HR | 30 | 192.168.30.0/24 | 192.168.30.1 | 192.168.30.11, .12 |

Subnet mask: 255.255.255.0. DHCP excludes .1 through .10 in each subnet. Six automatic DHCP bindings were observed, two per VLAN.

## 5. Switch implementation

| VLAN | Name | Access ports |
|---:|---|---|
| 10 | SALES | Fa0/1 and Fa0/2 |
| 20 | IT | Fa0/3 and Fa0/4 |
| 30 | HR | Fa0/5 and Fa0/6 |

GigabitEthernet0/1 is an IEEE 802.1Q trunk. The switch output showed VLANs 10, 20 and 30 active and forwarding across the trunk. VLAN 1 was also listed as native/default.

Verification commands:

```text
show vlan brief
show interfaces trunk
show running-config
copy running-config startup-config
```

The switch configuration save returned [OK].

## 6. Router implementation

| Subinterface | VLAN tag | Gateway | State in evidence |
|---|---:|---|---|
| G0/0.10 | 10 | 192.168.10.1/24 | up/up |
| G0/0.20 | 20 | 192.168.20.1/24 | up/up |
| G0/0.30 | 30 | 192.168.30.1/24 | up/up |

The router uses router-on-a-stick. Each subinterface is associated with the corresponding VLAN tag. The observed DHCP binding table contains the six client leases shown in the addressing plan.

Verification commands:

```text
show ip interface brief
show ip dhcp binding
show access-lists
show ip interface gigabitEthernet 0/0.10
show ip interface gigabitEthernet 0/0.30
copy running-config startup-config
```

The router configuration save returned [OK].

## 7. ACL design and placement

Two inbound extended ACLs are applied:

- SALES-IN on GigabitEthernet0/0.10.
- HR-IN on GigabitEthernet0/0.30.

### SALES-IN

| Sequence | Rule purpose |
|---:|---|
| 10 | Permit established TCP return traffic from Sales to IT |
| 20 | Permit ICMP echo replies from Sales to IT |
| 30 | Permit established TCP return traffic from Sales to HR |
| 40 | Permit ICMP echo replies from Sales to HR |
| 50 | Deny other IP traffic from Sales to IT |
| 60 | Deny other IP traffic from Sales to HR |
| 70 | Permit remaining IP traffic |

### HR-IN

| Sequence | Rule purpose |
|---:|---|
| 10 | Permit established TCP return traffic from HR to IT |
| 20 | Permit ICMP echo replies from HR to IT |
| 30 | Deny other IP traffic from HR to IT |
| 40 | Permit remaining IP traffic |

The specific return-traffic permits appear before the broad deny entries. The interface verification output explicitly reported SALES-IN as the inbound ACL on G0/0.10 and HR-IN as the inbound ACL on G0/0.30.

**Technical limitation:** Cisco ACL `established` is a TCP flag match, not stateful connection tracking. ICMP echo-reply permits cover ping replies only. This configuration does not automatically allow return traffic for arbitrary UDP or other application protocols. Such protocols require their own policy and testing.

## 8. Final connectivity testing

The final screenshots show the following results:

| Source | Destination | Expected | Observed |
|---|---|---|---|
| Sales-PC1 | IT-PC1, 192.168.20.11 | Block | Destination host unreachable; 100% loss |
| Sales-PC1 | HR-PC1, 192.168.30.11 | Block | Destination host unreachable; 100% loss |
| IT-PC1 | Sales-PC1, 192.168.10.11 | Allow | Four replies; 0% loss |
| IT-PC1 | HR-PC1, 192.168.30.11 | Allow | Four replies; 0% loss |
| HR-PC1 | IT-PC1, 192.168.20.11 | Block | Destination host unreachable; 100% loss |
| HR-PC1 | Sales-PC1, 192.168.10.11 | Allow | Four replies; 0% loss |

These results satisfy all six ICMP tests derived from the four original departmental requirements.

A first ping can occasionally time out while ARP resolves a destination MAC address. Repeated test output and final packet statistics should be used when recording evidence.

## 9. Verification evidence

The evidence set should include:

1. Network topology.
2. Switch VLAN membership.
3. Switch trunk status.
4. Router interface status and DHCP bindings.
5. ACL definitions and match counters.
6. ACL attachment on Sales and HR subinterfaces.
7. Sales blocked test results.
8. IT successful test results.
9. HR allowed and blocked test results.

The screenshot files are intended to be stored in `screenshots/`. Add the actual image files from the lab session before treating the evidence directory as complete.

## 10. Backup and persistence

Both router and switch running configurations were copied to startup configuration. Packet Tracer returned [OK] for both devices.

The Packet Tracer project should be stored under `packet-tracer/` as:

`Project 01 - Small Office Network.pkt`

After reopening the project, repeat the switch, router, DHCP and ACL verification commands to confirm persistence.

## 11. Conclusion

The lab demonstrates departmental VLAN segmentation, 802.1Q trunking, router-on-a-stick routing, DHCP and extended ACL-based access restrictions. The observed final ICMP tests match the stated policy: Sales cannot ping IT or HR, IT can ping Sales and HR, and HR cannot ping IT but can ping Sales.

The configuration saves were confirmed. The project meets its stated ICMP access-control objectives. Further TCP and application-level testing is recommended before claiming complete protocol-level validation.
