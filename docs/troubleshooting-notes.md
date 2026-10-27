# Real-World Engineering Troubleshooting Notes & Edge-Case Post-Mortem

This document records the exact failure signatures, root-cause analyses, packet-level mechanics, and definitive remediation commands encountered during the deployment and integration of this multi-vendor EVE-NG lab (Cisco IOS-XE 17.12, Cisco IOSv 15.9, Cisco IOSv-L2, and Ubuntu Server 24.04 LTS).

---

## Issue 1: Router-on-a-Stick Sub-Interface Tagging Order
* **Layer:** Layer 2 / Layer 3 Boundary (802.1Q Encapsulation)
* **Affected Device:** `R1-DC-GW` (`GigabitEthernet0/0.100`)

### Symptom & Error Output
Attempting to assign an IPv4 address to a fresh sub-interface on a Cisco router fails immediately:
```ios
R1-DC-GW(config-subif)# ip address 10.100.100.1 255.255.255.0

% Configuring IP routing on a LAN subinterface is only allowed if that
subinterface is already configured as part of an IEEE 802.10, IEEE 802.1Q,
or ISL vLAN.
```

### Root Cause
Unlike a physical Layer 3 interface or switch SVI, an Ethernet LAN sub-interface has no default frame format. Cisco IOS parser logic strictly enforces that the 802.1Q encapsulation tag (`encapsulation dot1Q <vlan-id>`) must be committed first. This instructs the router's parser how to tag outbound frames and demultiplex inbound tagged frames before the Layer 3 IP engine binds an IP address and mask to that logical socket.

### Remediation
Always configure the encapsulation mode and VLAN ID prior to IP addressing:
```ios
interface GigabitEthernet0/0
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 10.100.100.1 255.255.255.0
!
interface GigabitEthernet0/0.200
 encapsulation dot1Q 200
 ip address 10.100.200.1 255.255.255.0
```

---

## Issue 2: Linux Link-Layer vs. Network-Layer Command Syntax
* **Layer:** Linux Kernel / Network Stack (`iproute2`)
* **Affected Device:** `SRV-2` / `SRV-1` (Ubuntu Server)

### Symptom & Error Output
```bash
root@ubuntu:~# ip link set addr 10.100.200.20/24 dev ens3
"10.100.200.20/24" is invalid lladdr.
```

### Root Cause
In Linux `iproute2`, the `ip link` subsystem exclusively manages **Layer 2 link-layer properties** (MAC address / link-layer address `lladdr`, MTU, administrative state UP/DOWN, virtual interfaces). Passing an IPv4 address with CIDR notation (`10.100.200.20/24`) to `ip link set addr` causes the kernel parser to reject it because it expects a 48-bit hexadecimal MAC string (e.g., `52:54:00:12:34:56`).

### Remediation
Use `ip addr add` for Layer 3 IPv4/IPv6 addresses and `ip link set` for Layer 2 interface state:
```bash
# Correct Layer 3 IP configuration
sudo ip addr add 10.100.200.20/24 dev ens3

# Correct Layer 2 administrative state toggle
sudo ip link set ens3 up

# Verification
ip -br addr
```

---

## Issue 3: Modern `vsftpd` Chroot Jail Security Panic
* **Layer:** Application Security (Linux PAM / FTP Daemon)
* **Affected Device:** `SRV-2` (`vsftpd` on Ubuntu 24.04 LTS)

### Symptom & Error Output
When a router or user attempts to authenticate via FTP into a chrooted account:
```text
500 OOPS: vsftpd: refusing to run with writable root inside chroot()
Login failed.
421 Service not available, remote server has closed connection.
```

### Root Cause
Following historical security vulnerabilities where an attacker could execute code inside a writable chroot mount point to escape the jail, modern `vsftpd` (v3.0+) checks file system write permissions on the user's root directory (`/home/<user>`). If `chroot_local_user=YES` is enabled AND the user has write privileges (`w`) on their own home directory root, `vsftpd` panics and abruptly drops the session.

### Remediation
Add `allow_writeable_chroot=YES` to `/etc/vsftpd.conf`, or lock the landing point directly to an explicit sub-folder via `local_root`:
```ini
# Add to /etc/vsftpd.conf
chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/home/eleven/ftp-vault
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
```
Then restart the service:
```bash
sudo systemctl restart vsftpd
```

---

## Issue 4: Discrepancy Between FTP Uploads (`local_root`) and SCP Uploads (`sshd`)
* **Layer:** Application Daemon Scope (vsftpd vs. OpenSSH)
* **Affected Device:** `SRV-2`

### Symptom
When running `copy flash: ftp:` from `R4-BRANCH-1` with destination `hello.cfg`, the file landed directly in `/home/eleven/hello.cfg` instead of `/home/eleven/ftp-vault/`. Furthermore, running `copy run scp:` landed in `/home/eleven/` completely ignoring FTP configurations.

### Root Cause
1. **FTP Behavior:** By default, `vsftpd` treats the user's home folder `/home/eleven/` as the FTP root (`/`). Unless `ftp-vault/hello.cfg` was explicitly typed or `local_root=/home/eleven/ftp-vault` was set in `/etc/vsftpd.conf`, relative paths write to the user's root directory.
2. **SCP Behavior:** SCP does not communicate with `vsftpd` at all. It runs via OpenSSH (`sshd`, port 22), which reads `/etc/passwd` for the user's home directory. Therefore, changes to `vsftpd.conf` have zero effect on SCP transfers.

### Remediation
1. **For FTP:** Add `local_root=/home/eleven/ftp-vault` in `/etc/vsftpd.conf`.
2. **For SCP:** Change the default home path in Linux for the automation user, or supply the explicit subfolder in the transfer URI:
   ```bash
   sudo usermod -d /home/eleven/scp-vault eleven
   ```
   Or explicitly in Cisco IOS:
   ```ios
   copy running-config scp://eleven@10.100.200.20/scp-vault/r4-scp.cfg
   ```

---

## Issue 5: Interactive Terminal CRLF Parser Lockout in Cisco IOS
* **Layer:** Terminal / Console Emulation (EVE-NG Telnet/HTML5)
* **Affected Device:** `R4-BRANCH-1`

### Symptom & Error Output
```ios
R4-Branch-1# copy running-config ftp:
Address or name of remote host []?
?Host name or address not specified
%Error parsing filename (Bad address)
```
The router printed the prompt and immediately terminated before the engineer could type an IP address or filename.

### Root Cause
Telnet sessions or browser HTML5 consoles in virtualized lab environments frequently translate the `Enter` key as a compound Carriage Return + Line Feed (`\r\n`). Cisco IOS interprets the first `\r` as the command submission and the trailing `\n` as an instantaneous blank input to the first interactive prompt (`Address or name of remote host []?`), causing a bad-address parsing failure.

### Remediation
Three proven solutions:
1. **Bypass Interactive Prompts (Recommended):** Set `file prompt quiet` in global configuration, or pass the full inline target URI:
   ```ios
   configure terminal
   file prompt quiet
   exit
   copy running-config ftp://10.100.200.20/r4-ftp.cfg
   ```
2. **Send Raw Carriage Return:** Use `Ctrl + M` or `Ctrl + J` instead of the physical `Enter` key.
3. **PuTTY / SecureCRT Configuration:** Under Connection -> Telnet, uncheck *"Return key sends Telnet New Line instead of ^M"*.

---

## Issue 6: Cross-Generation SSHv2 MAC & Cipher Incompatibility
* **Layer:** Security / Transport Layer (SSHv2 Negotiation)
* **Affected Devices:** `R5-AUTO-PROVISIONED` (Cisco IOS-XE 17.12.1 Client) <-> `R1-DC-GW` (Cisco IOSv 15.9 Server)

### Symptom & Error Output
```ios
R5-AUTO-PROVISIONED# copy scp: scpfile
Address or name of remote host [172.16.10.1]? 
Source username [scp]? 
Source filename [running-config]? 
Destination filename [scpfile]? 
%Error opening scp://*@172.16.10.1/running-config (Undefined error)

*Oct 26 20:55:49.897: %SSH-3-NO_MATCH: No matching mac found: client hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com server hmac-sha2-256,hmac-sha2-512,hmac-sha1,hmac-sha1-96
```

### Root Cause
Modern Cisco IOS-XE (Release 17.12+) defaults to hardened **Encrypt-then-MAC (ETM)** message authentication algorithms (`*-etm@openssh.com`). Older Cisco IOS releases (such as IOSv 15.9) only support traditional non-ETM HMAC algorithms (`hmac-sha2-256`, `hmac-sha1`). Because the intersection of offered client MAC algorithms and supported server MAC algorithms was null, the SSH binary packet protocol handshake was torn down immediately with a `%SSH-3-NO_MATCH` fatal alert.

### Remediation
Explicitly expand the client MAC, Key Exchange (KEX), and cipher algorithms on the newer IOS-XE router to maintain backwards interoperability with legacy IOS devices:
```ios
configure terminal
! Enable standard non-ETM HMAC algorithms
ip ssh client algorithm mac hmac-sha2-256 hmac-sha2-512
! Enable legacy Diffie-Hellman groups if requested
ip ssh client algorithm kex diffie-hellman-group14-sha1 diffie-hellman-group-exchange-sha1
! Enable standard AES ciphers
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr aes128-cbc
end
```

---

## Issue 7: Stateful Packet Dropping of TFTP/FTP Ephemeral Data Ports
* **Layer:** Layer 4 Transport / Stateful Firewalling
* **Affected Device:** `R2-FW-NAT`

### Symptom
When applying a naive Extended ACL permitting only destination port `UDP 69` (TFTP) or `TCP 21` (FTP) on `R2-FW-NAT`, transfers from branch routers timed out with `%Error opening tftp://... (Timed out)`. Syslog logs revealed:
```text
%SEC-6-IPACCESSLOGP: list NAIVE_FIREWALL denied udp 10.100.100.10(49821) -> 10.4.4.4(55102)
```

### Root Cause
1. **TFTP (RFC 1350):** The initial Read/Write Request (`RRQ`/`WRQ`) is sent to well-known destination port `UDP 69`. However, the server dynamically allocates an **ephemeral Transfer Identifier (TID)** (e.g., `UDP 49821`) as its source port for all subsequent `DATA` and `ACK` blocks. A static firewall rule matching only destination `UDP 69` blocks these server replies.
2. **Active FTP (RFC 959):** The client issues a `PORT` command over `TCP 21`, after which the FTP server initiates a brand-new outbound connection from `TCP 20` to the client's dynamically negotiated high port.
3. **Passive FTP (PASV):** The server tells the client to initiate a connection to an ephemeral port range (e.g., `40000–40100`).

### Remediation
In stateless/extended ACL environments, you must explicitly permit the negotiated ephemeral port spaces:
```ios
ip access-list extended LAB_FIREWALL_ACL
 permit ospf any any
 permit icmp any any
 ! Permit DHCP Relay for ZTP
 permit udp any any eq bootps
 permit udp any any eq bootpc
 ! Permit TFTP Control & Ephemeral TIDs
 permit udp any host 10.100.100.10 eq tftp
 permit udp host 10.100.100.10 gt 1023 any
 permit udp any gt 1023 host 10.100.100.10 gt 1023
 ! Permit FTP Control, Active Data (20), & Defined Passive Range (40000-40100)
 permit tcp any host 10.100.200.20 eq ftp
 permit tcp any host 10.100.200.20 eq ftp-data
 permit tcp any host 10.100.200.20 range 40000 40100
 permit tcp host 10.100.200.20 any established
 permit tcp host 10.100.200.20 eq ftp-data any
 permit tcp any host 10.100.200.20 eq 22
 deny   ip any any log
```

---

## Issue 8: Dual-NIC Netplan Metric Routing Conflict
* **Layer:** Layer 3 Host Routing (Linux Kernel Route Table)
* **Affected Device:** `SRV-1`

### Symptom
When connecting a second virtual interface (`ens4`) to EVE-NG `Management(Cloud0)` for package installation via `apt update`, `SRV-1` either could not reach the internet or lost connectivity to the internal lab router (`10.100.100.1`).

### Root Cause
Both `ens3` (internal lab static gateway `10.100.100.1`) and `ens4` (external DHCP gateway) attempted to install default routes (`0.0.0.0/0`) into the main Linux routing table with equal metrics. If the lab router gateway won the route selection, internet traffic was forwarded to `10.100.100.1` (which had no NAT/WAN route), breaking `apt`.

### Remediation
Use `dhcp4-overrides` with explicit route metrics in `/etc/netplan/01-netcfg.yaml`:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens3:
      dhcp4: false
      addresses:
        - 10.100.100.10/24
      routes:
        - to: default
          via: 10.100.100.1
          metric: 200
    ens4:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 50
```
With `ens4` having metric `50`, outbound internet traffic prioritizes the Cloud0 DHCP gateway. When `ens4` is unplugged, the kernel seamlessly falls back to metric `200` (`10.100.100.1`) without manual reconfiguration.

---
