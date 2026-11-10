# ✅ Verification

Each check below lists the command, the expected output (taken from the project screenshots) and the matching screenshot. All captures are from 16 May 2025.

---

## Ubuntu VMs

### `ip a` – address from the switch

```bash
ip a
```

VLAN 70 VM (hostname `ubuntuvm`, lab adapter `ens33`):

```
inet 172.20.7.108/25 brd 172.20.7.127 scope global dynamic noprefixroute ens33
inet 172.20.70.108/25 brd 172.20.70.127 scope global dynamic noprefixroute ens33
```

The first line is a short-lived leftover lease (see the design notes in [topology.md](topology.md)). The VLAN 70 address is the one in use.

VLAN 7 VM (hostname `csp450`, lab adapter `ens37`):

```
inet 172.20.7.5/25 brd 172.20.7.127 scope global dynamic noprefixroute ens37
```

Screenshots: [VLAN 70 VM](../screenshots/09-ip-a-vlan70-vm.png) · [VLAN 7 VM](../screenshots/10-ip-a-vlan7-vm.png)

### `ip route` – default gateway and route to the other VLAN

VLAN 70 VM:

```
default via 10.33.35.1 dev ens38 proto dhcp metric 100
default via 172.20.70.1 dev ens33 proto dhcp metric 20101
10.33.35.0/24 dev ens38 proto kernel scope link src 10.33.35.77 metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
172.20.7.0/25 via 172.20.70.1 dev ens33
172.20.70.0/25 dev ens33 proto kernel scope link src 172.20.70.108 metric 101
```

VLAN 7 VM:

```
default via 10.33.35.1 dev ens33 proto dhcp metric 100
default via 172.20.7.1 dev ens37 proto dhcp metric 20101
10.33.35.0/24 dev ens33 proto kernel scope link src 10.33.35.62 metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
172.20.7.0/25 dev ens37 proto kernel scope link src 172.20.7.5 metric 101
172.20.70.0/25 via 172.20.7.1 dev ens37
```

Screenshots: [VLAN 70 VM](../screenshots/07-ip-route-vlan70-vm.png) · [VLAN 7 VM](../screenshots/08-ip-route-vlan7-vm.png)

### `sshd_config` – root login disabled

```
PermitRootLogin no
```

Screenshots: [VLAN 7 VM](../screenshots/11-sshd-config-vlan7-vm.png) · [VLAN 70 VM](../screenshots/12-sshd-config-vlan70-vm.png)

---

## Apache Web Server (Wireshark filter: `http`)

```bash
links http://172.20.7.5        # from the VLAN 70 VM
```

Expected: the test page loads. Wireshark on the VLAN 70 VM's `ens33` shows two packets:

```
No.  Source          Destination    Protocol  Info
19   172.20.70.108   172.20.7.5     HTTP      GET / HTTP/1.1
21   172.20.7.5      172.20.70.108  HTTP      HTTP/1.1 200 OK  (text/html)
```

The Ethernet destination of the request is `bc:d7:a5:d3:cd:80`, the 6300's MAC address. This matches the bridge MAC in `sh spanning-tree` and shows the traffic was routed by the 6300.

Screenshot: [06-apache-http-request-wireshark.png](../screenshots/06-apache-http-request-wireshark.png)

---

## SSH Tests (Wireshark filter: `ssh`)

### VM to VM across both switches

```bash
ssh <username>@172.20.7.5        # from the VLAN 70 VM
ssh <username>@172.20.70.108     # from the VLAN 7 VM
```

Expected: an Ubuntu 22.04.5 LTS welcome banner and a shell prompt on the remote VM. Wireshark shows `SSHv2` packets with the client and server protocol banners (`SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.13`) followed by `Key Exchange Init`.

Screenshots: [VLAN 70 → VLAN 7](../screenshots/04-ssh-vm-to-vm-vlan70-to-vlan7.png) · [VLAN 7 → VLAN 70](../screenshots/05-ssh-vm-to-vm-vlan7-to-vlan70.png)

### VM to switch

```bash
ssh student@172.20.70.1        # from the VLAN 70 VM
ssh student@172.20.7.1         # from the VLAN 7 VM
```

Expected: the HPE login banner, a password prompt, then the switch prompt:

```
student@172.20.7.1's password:
Last login: 2025-05-16 18:46:10 from 172.20.7.5
Switch-7#
```

Wireshark shows the full handshake: protocol exchange, `Key Exchange Init`, `Elliptic Curve Diffie-Hellman Key Exchange Init`/`Reply`, `New Keys`, then encrypted packets.

Screenshots: [VLAN 70 VM](../screenshots/02-ssh-to-switch-vlan70-wireshark.png) · [VLAN 7 VM](../screenshots/03-ssh-to-switch-vlan7-wireshark.png)

---

## Aruba 6300

### `sh ip int br`

```
Switch-7# sh ip int br
Interface         IP Address            Interface Status
                                        link/admin
vlan1             No Address            down/up
vlan7             172.20.7.1/25         up/up
vlan70            172.20.70.1/25        up/up
```

Screenshot: [13-show-ip-interface-brief.png](../screenshots/13-show-ip-interface-brief.png)

### `sh spanning-tree`

```
Switch-7# sh spanning-tree
Spanning tree status      : Enabled Protocol: MSTP

MST0
  Root ID    Priority   : 32768
             MAC-Address: bc:d7:a5:d3:cd:80
             This bridge is the root

Port     Role         State       Cost    Priority  Type
1/1/3    Designated   Forwarding  20000   128       P2P
1/1/4    Disabled     Blocking    20000   128       P2P
...
1/1/28   Disabled     Blocking    20000   128       P2P
lag1     Designated   Forwarding  20000   64        P2P
```

Screenshot: [14-show-spanning-tree.png](../screenshots/14-show-spanning-tree.png)

### `show vlan`

```
Switch-7# show vlan
VLAN  Name             Status  Reason                Type     Interfaces
1     DEFAULT_VLAN_1   down    no_member_forwarding  default  1/1/4-1/1/24,1/1/27-1/1/28
7     VLAN7            up      ok                    static   1/1/3,lag1
70    VLAN70           up      ok                    static   lag1
```

Screenshot: [15-show-vlan.png](../screenshots/15-show-vlan.png)

### `sh dhcp-server leases`

```
Switch-7# sh dhcp-server leases
IP-Address      Client-Hostname     VRF-Name
172.20.7.5      csp450              default
172.20.7.118    NHK1270-19-E063     default
172.20.70.84    NHK1270-20-E1C2     default
172.20.70.108   ubuntuvm            default
```

Screenshot: [16-show-dhcp-server-leases.png](../screenshots/16-show-dhcp-server-leases.png)

---

## Switch configuration scripts

The configuration screenshot for both switches: [17-switch-configuration.png](../screenshots/17-switch-configuration.png). The text versions are in [`configs/`](../configs/).

---

## Optional: scripted VM-side checks

[`scripts/verify-connectivity.bash`](../scripts/verify-connectivity.bash) runs the address, route, ping, SSH-port and web checks from a VM in one go:

```bash
./scripts/verify-connectivity.bash 172.20.7.1 172.20.70.108 172.20.7.5
```
