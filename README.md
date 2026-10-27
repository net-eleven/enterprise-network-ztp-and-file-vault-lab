# Enterprise Network Zero-Touch Provisioning (ZTP) & Configuration Vault Lab

![Topology Diagram](docs/topology-diagram.jpg)

## Overview
An end-to-end enterprise networking and Linux systems integration lab built inside **EVE-NG** over **KVM**, combining **Cisco IOS-XE (IOL 17.12) / IOSv (15.9)** routers and switches with **Ubuntu Server 24.04 LTS** infrastructure nodes.

This project demonstrates how modern enterprise networks automate router deployment, enforce deterministic management source routing, secure unencrypted and encrypted file transfers across firewall boundaries, and perform zero-downtime configuration rollbacks.

## Key Features & Architecture
1. **802.1Q VLAN Segmentation & OSPF Area 0 Core:**
   - Router-on-a-Stick (`R1-DC-GW`) isolating TFTP/DHCP Provisioning (`VLAN 100`) from FTP/SCP Vault services (`VLAN 200`).
   - Full OSPF Area 0 routing across Data Center, Firewall, WAN Core, and Branch routers with deterministic Loopback sourcing (`ip tftp/ftp source-interface Loopback0`).
2. **Zero-Touch Provisioning (Cisco AutoInstall via DHCP Option 150 + TFTP):**
   - Unconfigured branch router (`R5-ZERO-TOUCH`) boots with an empty NVRAM, sends a DHCP broadcast across `R3-ISP-WAN` (`ip helper-address 10.100.100.10`), receives DHCP Option 150 + Option 67 (`ztp/r5-confg`) from `isc-dhcp-server`, and self-provisions hostname, credentials, and OSPF adjacencies over TFTP with zero console intervention.
3. **Automated Configuration Archiving & Atomic Rollback (`R4-BRANCH-1`):**
   - Configured Cisco IOS `archive` to version running configurations to the Data Center vault on every `write memory` while masking secrets (`hidekeys`).
   - Verified live configuration diffs (`show archive config differences`) and zero-reload atomic rollbacks using `configure replace`.
4. **Hardened Multi-Protocol Linux Vaults (`SRV-1` & `SRV-2`):**
   - **`tftpd-hpa`:** Chrooted `/srv/tftp` hierarchy (`backups/`, `ztp/`, `images/`) with RFC 2348 `blksize 8192` throughput optimization.
   - **`vsftpd` & `OpenSSH`:** Jailed FTP root (`chroot_local_user=YES`, constrained Passive FTP port range `40000-40100`) and SSHv2 SCP/SFTP compatibility between modern Linux/IOS-XE and legacy IOS 15.9 (`hmac-sha2-256`, `diffie-hellman-group14-sha1`).
5. **Extended Firewall ACL Filtering (`R2-FW-NAT`):**
   - Explicit transit filtering permitting OSPF, ICMP, BootP/DHCP relay, TFTP ephemeral TID data ports (`>1023`), Active/Passive FTP data channels, and SSH/SCP (`TCP/22`) while logging unauthorized access attempts.
6. **Cisco Router as an ACL-Protected TFTP & AAA SCP Server (`R1-DC-GW`):**
   - Configured local AAA (`aaa new-model`), SSHv2 RSA keys, `ip scp server enable`, and ACL-restricted `tftp-server` file sharing.

## IP Addressing & Topology Matrix
| Node | OS / Image | Interfaces & IPs | Role |
| :--- | :--- | :--- | :--- |
| `SRV-1` | Ubuntu 24.04 LTS | `ens3`: `10.100.100.10/24` | TFTP (`tftpd-hpa`) & DHCP (`isc-dhcp-server`) |
| `SRV-2` | Ubuntu 24.04 LTS | `ens3`: `10.100.200.20/24` | FTP (`vsftpd`) & SCP/SFTP (`openssh-server`) |
| `SW1-DC-CORE` | Cisco vIOS-L2 | Trunk `Gi0/0`, Access `Gi0/1-2` | Data Center L2 Switch (`VLAN 100`, `VLAN 200`) |
| `R1-DC-GW` | Cisco IOSv 15.9 | `Gi0/0.100`: `10.100.100.1/24`<br>`Gi0/0.200`: `10.100.200.1/24`<br>`Gi0/1`: `172.16.10.1/30` | DC Gateway, ROAS, Cisco TFTP & SCP Server |
| `R2-FW-NAT` | Cisco IOSv 15.9 | `Gi0/0`: `172.16.10.2/30`<br>`Gi0/1`: `172.16.20.1/30` | WAN Perimeter Firewall Router (Extended ACLs) |
| `R3-ISP-WAN` | Cisco IOSv 15.9 | `Gi0/0`: `172.16.20.2/30`<br>`Gi0/1`: `172.16.30.1/30`<br>`Gi0/2`: `172.16.40.1/30` | WAN Core & DHCP Relay Agent (`ip helper-address`) |
| `R4-BRANCH-1` | Cisco IOSv 15.9 | `Gi0/0`: `172.16.30.2/30`<br>`Lo0`: `4.4.4.4/32` | Branch 1 Edge (Auto-Archive & Config Rollback) |
| `R5-AUTO-PROVISIONED` | Cisco IOL 17.12.1 | `Et0/0`: `172.16.40.2/30` (DHCP)<br>`Lo0`: `5.5.5.5/32` | Branch 2 Router (Provisioned 100% via ZTP) |
