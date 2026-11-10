# 📝 Project Summary

| | |
|---|---|
| **Project** | Aruba 6300 and 2540 switches with an LACP trunk, two Ubuntu VMs and an Apache web server |
| **Focus** | LACP trunking, VLANs, inter-VLAN routing, DHCP, Apache and SSH |
| **Date** | May 2025 (documentation updated November 2026) |

---

## Overview

An Aruba 6300 and an Aruba 2540 switch are joined by a two-link LACP port-channel that carries VLAN 7 and VLAN 70. One Ubuntu virtual machine sits on each VLAN. The 6300 assigns each VM an IP address and default gateway from its own DHCP pools and routes between the VLANs, so the VMs communicate without relying on the campus network's addressing. The VLAN 7 VM runs an Apache web server that the VLAN 70 VM browses to across the trunk, and the VMs and the switch are also reached over SSH.

## Goals

- Build an LACP trunk between two Aruba switches and carry two VLANs across it.
- Configure static addressing, VLANs and default gateways on the 6300.
- Make sure both VMs connect without relying on the campus IP and gateway.
- Install an Apache web server and reach it from the other VLAN.
- Log in over SSH between the VMs, using key pairs generated with `ssh-keygen`.
- Demonstrate the working setup live and document it.

## Steps Followed

1. Configure the 6300 and the 2540 with the scripts in [`configs/`](../configs/).
2. List the interfaces and MAC addresses on each VM with `ip -brief link`.
3. Install Apache on the VLAN 7 VM and publish the test page.
4. After confirming the Aruba network, edit `/etc/ssh/sshd_config` and set `PermitRootLogin no`.
5. Restart SSH and create a key pair with `ssh-keygen` for each user.
6. Copy the keys across and test with `ssh <username>@<ip address>`.
7. Capture the HTTP and SSH traffic in Wireshark.

## Deliverables

| Deliverable | Location |
|-------------|----------|
| Network diagram | [`packet-tracer/network-diagram.pkt`](../packet-tracer/network-diagram.pkt), [`screenshots/01-network-diagram.png`](../screenshots/01-network-diagram.png) |
| Switch configuration scripts | [`configs/aruba-6300-switch.cfg`](../configs/aruba-6300-switch.cfg), [`configs/aruba-2540-switch.cfg`](../configs/aruba-2540-switch.cfg) |
| `sshd_config` change | [`configs/sshd_config.snippet`](../configs/sshd_config.snippet) |
| Wireshark SSH captures | `screenshots/02` – `05` |
| Wireshark HTTP capture and web page | `screenshots/06`, [`web/index.html`](../web/index.html) |
| `ip a`, `ip route` on both VMs | `screenshots/07` – `10` |
| `sh ip int br`, `sh spanning-tree`, `show vlan`, `sh dhcp-server leases` | `screenshots/13` – `16` |

## Conclusion

Two Aruba switches were joined with an LACP trunk, two Ubuntu systems were connected on separate VLANs and assigned addressing and gateways through DHCP, and the VLAN 70 VM was shown reaching an Apache web server on VLAN 7 and communicating with the other VM and the switch over SSH, supported by Wireshark captures. The project gave hands-on practice with trunking, VLANs, DHCP, routing, Apache and SSH.

## References

- [ArubaOS 8.x CLI Reference Guide](https://arubanetworking.hpe.com/techdocs/ArubaOS-8.x-Books/ArubaOS-8.x-CLI-Reference-Guide.pdf)
- [HPE Support – document a00021327en_us](https://support.hpe.com/hpesc/public/docDisplay?docId=a00021327en_us)
- [HPE Support – document a00055715en_us](https://support.hpe.com/hpesc/public/docDisplay?docId=a00055715en_us&docLocale=en_US)
