# ⚙️ Setup Guide

Step-by-step instructions to rebuild the Aruba 6300 and 2540 LACP trunk and Apache web server lab.

> Addressing and port assignments are in [topology.md](topology.md).

---

## 1. Configure the Aruba 6300

1. Connect to the switch console or SSH in, then enter configuration mode:

   ```
   Switch-7# configure terminal
   Switch-7(config)#
   ```

2. Paste the contents of [`configs/aruba-6300-switch.cfg`](../configs/aruba-6300-switch.cfg). It creates:
   - DHCP pools `vlan7` and `vlan70`
   - VLANs 7 and 70
   - Gateway interfaces `interface vlan 7` (`172.20.7.1/25`) and `interface vlan 70` (`172.20.70.1/25`)
   - `lag 1` with ports `1/1/1` and `1/1/2`, trunking VLANs 7 and 70
   - Access port `1/1/3` (VLAN 7)

3. Confirm the interfaces are up:

   ```
   Switch-7# sh ip int br
   vlan7             172.20.7.1/25         up/up
   vlan70            172.20.70.1/25        up/up
   ```

---

## 2. Configure the Aruba 2540

1. Connect to the switch console and enter configuration mode.
2. Paste the contents of [`configs/aruba-2540-switch.cfg`](../configs/aruba-2540-switch.cfg). It creates:
   - `trk1` from ports `1` and `2` using LACP
   - VLAN 7 and VLAN 70, both tagged on `trk1`
   - Port `3` as the untagged port for VLAN 70

---

## 3. Cable the lab

1. Run two cables between the switches: 6300 `1/1/1` to 2540 port `1`, and 6300 `1/1/2` to 2540 port `2`.
2. Connect the VLAN 7 VM's lab adapter to 6300 port `1/1/3`.
3. Connect the VLAN 70 VM's lab adapter to 2540 port `3`.
4. Check the VLANs and spanning tree on the 6300:

   ```
   Switch-7# show vlan
   Switch-7# sh spanning-tree
   ```

   Expected: `lag1` is listed in VLANs 7 and 70 and is `Designated Forwarding`.

---

## 4. Check the VM network settings

1. List the interfaces and MAC addresses:

   ```bash
   ip -brief link
   ```

2. Check that the lab adapter received an address from the 6300:

   ```bash
   ip a
   ```

   Expected (VLAN 7 VM):

   ```
   inet 172.20.7.5/25 brd 172.20.7.127 scope global dynamic noprefixroute ens37
   ```

3. Check the default gateway and the route to the other VLAN:

   ```bash
   ip route
   ```

   If the other VLAN is missing, add a route through the local gateway (VLAN 70 VM shown):

   ```bash
   sudo ip route add 172.20.7.0/25 via 172.20.70.1 dev ens33
   ```

   > The project report does not show how the routes were added. This is the standard command for it; use your own method (for example Netplan) if you need it to persist across reboots.

---

## 5. Install Apache on the VLAN 7 VM

1. Install and start the web server:

   ```bash
   sudo apt update
   sudo apt install -y apache2
   sudo systemctl enable --now apache2
   sudo systemctl status apache2
   ```

2. Publish the test page. Edit [`web/index.html`](../web/index.html) first and replace the placeholders, then copy it into place:

   ```bash
   sudo cp web/index.html /var/www/html/index.html
   ```

3. Test locally:

   ```bash
   curl http://localhost
   ```

---

## 6. Configure SSH on both VMs

1. Install the SSH server if it is not already present:

   ```bash
   sudo apt install -y openssh-server
   ```

2. Disable direct root login. Edit `/etc/ssh/sshd_config` (see [`configs/sshd_config.snippet`](../configs/sshd_config.snippet)):

   ```
   PermitRootLogin no
   ```

3. Restart the service:

   ```bash
   sudo systemctl restart ssh
   sudo systemctl status ssh
   ```

4. Create a key pair for the user on each VM:

   ```bash
   ssh-keygen
   ```

5. Copy the public key to the other VM:

   ```bash
   ssh-copy-id <username>@<other-vm-ip>
   ```

---

## 7. Test and capture in Wireshark

1. Start Wireshark on the lab adapter of the VLAN 70 VM (`ens33`) and set the display filter to `http`.
2. From the VLAN 70 VM, open the web server:

   ```bash
   links http://172.20.7.5
   ```

   Expected: the test page appears, and Wireshark shows `GET / HTTP/1.1` followed by `HTTP/1.1 200 OK (text/html)`.

3. Change the display filter to `ssh` and SSH from each VM to the other VM:

   ```bash
   ssh <username>@172.20.7.5        # from the VLAN 70 VM
   ssh <username>@172.20.70.108     # from the VLAN 7 VM
   ```

4. SSH from each VM to the switch gateway:

   ```bash
   ssh student@172.20.7.1       # from the VLAN 7 VM
   ssh student@172.20.70.1      # from the VLAN 70 VM
   ```

5. Confirm Wireshark shows the SSHv2 version exchange, `Key Exchange Init` and `Elliptic Curve Diffie-Hellman Key Exchange Init` packets.

---

## 8. Verify the switch

Run the show commands listed in [verification.md](verification.md): `sh ip int br`, `sh spanning-tree`, `show vlan` and `sh dhcp-server leases`.

---

## 🧯 Troubleshooting

- **The VLAN 70 VM has no `172.20.70.x` address** – check the trunk first: `sh spanning-tree` should list `lag1` as forwarding and `show vlan` should list `lag1` in VLAN 70. Then check that `dhcp-server` is `enable`d on the 6300.
- **The LAG or a port does not join the VLANs** – on AOS-CX, a LAG or port that should switch (not route) may need `no routing`, and an LACP bundle needs `lacp mode active` under `interface lag 1`. These lines are not in the captured configuration, so add them only if you see the problem.
- **Cannot reach the other VLAN** – verify the static route with `ip route` and that both `interface vlan` SVIs are `up/up`.
- **The web page does not load** – confirm `sudo systemctl status apache2` is running on the VLAN 7 VM.
- **SSH refuses the connection** – confirm `sudo systemctl status ssh` is running on the target VM.
