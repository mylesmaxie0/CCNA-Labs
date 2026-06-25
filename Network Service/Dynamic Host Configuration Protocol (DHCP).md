

## How DHCP Automates IP Address Management

#### Dynamic Allocation & Parameters
###### DHCP assigns IPs and configurations (DNS, gateway) on demand, letting devices join/leave without manual setup.

#### Fewer Errors, Faster Onboarding
###### Automated provisioning can cut IP configuration mistakes up to 75%, improving reliability and uptime.

#### Lease Management & Utilization
###### Tunable lease durations optimize address reuse, maximizing IP pool efficiency from small offices to enterprises.

#### Scalability & Automation
###### Discovery, offer, request, and acknowledgment enable streamlined growth and centralized network management.

---

## DHCP Client

### What is a DHCP Client?
###### A DHCP client is a device that automatically requests and receives IP configuration information from a DHCP server. This information typically includes an IP address, subnet mask, default gateway, and DNS servers, allowing the device to communicate on the network without manual configuration.

---

### Network Lab Topology
<img width="971" height="524" alt="Screenshot 2026-06-12 at 12 23 25 PM" src="https://github.com/user-attachments/assets/ea5147e5-a8fe-4610-b847-db03fd4a5a95" />

---

### ISP & HQ-EDGE Router Configuration

This covers the initial WAN connectivity setup between a simulated ISP router and the HQ edge router. The goal is to bring up the WAN link, assign a public IP via DHCP, and verify end-to-end reachability.

#### ISP Router Configuration

The ISP router acts as the upstream provider, owning the `200.1.1.0/24` public subnet.

```
ISP-RTR> enable
ISP-RTR# configure terminal 
ISP-RTR(config)# interface e0/0
ISP-RTR(config-if)# ip address 200.1.1.1 255.255.255.0
ISP-RTR(config-if)# no shutdown
ISP-RTR(config)# ip dhcp excluded-address 200.1.1.1
ISP-RTR(config)# ip dhcp pool ISP-PUBLIC-POOL
ISP-RTR(dhcp-config)# network 200.1.1.0 255.255.255.0
ISP-RTR(dhcp-config)# default-router 200.1.1.1
ISP-RTR(dhcp-config)# dns-server 8.8.8.8
ISP-RTR(dhcp-config)# end
ISP-RTR# write
```

**What this does:**
- Assigns `200.1.1.1/24` as the ISP gateway IP on `e0/0` and brings the interface up.
- Excludes `200.1.1.1` from the DHCP pool so the ISP's own address is never handed out to a client.
- Creates a DHCP pool (`ISP-PUBLIC-POOL`) that will dynamically assign addresses from the `200.1.1.0/24` range, pointing clients to `200.1.1.1` as their default gateway and `8.8.8.8` (Google DNS) as their DNS server.
- Saves the configuration to NVRAM with `write`.

#### HQ-EDGE Router Configuration

The HQ edge router connects to the ISP on its `e0/0` WAN interface and requests an IP dynamically.

```
HQ-EDGE-RTR> enable
HQ-EDGE-RTR# configure terminal
HQ-EDGE-RTR(config)# interface e0/0
HQ-EDGE-RTR(config-if)# ip address dhcp
HQ-EDGE-RTR(config-if)# no shutdown
```

**What this does:**
- Sets `e0/0` to obtain its IP address via DHCP from the ISP router.
- Brings the interface up, triggering a DHCP Discover. The ISP router responds and assigns `200.1.1.2/24`, as confirmed by the log message:

```
*Jun 12 15:58:45.424: %DHCP-6-ADDRESS_ASSIGN: Interface Ethernet0/0 assigned DHCP address 200.1.1.2, mask 255.255.255.0, hostname HQ-EDGE-RTR
```

#### Verification

##### Interface Status

```
HQ-EDGE-RTR# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
Ethernet0/0            200.1.1.2       YES DHCP   up                    up      
Ethernet0/1            unassigned      YES TFTP   administratively down down    
Ethernet0/2            unassigned      YES TFTP   administratively down down    
Ethernet0/3            unassigned      YES TFTP   administratively down down
```

- `e0/0` is **up/up** with the DHCP-assigned address `200.1.1.2` — WAN link is live.
- `e0/1–e0/3` are still **administratively down** — LAN-facing interfaces, not yet configured.

##### Connectivity Test

```
HQ-EDGE-RTR# ping 200.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 200.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

- Five consecutive pings to the ISP gateway (`200.1.1.1`) all succeed with sub-millisecond round-trip times.
- **100% success rate** confirms Layer 3 reachability across the simulated WAN link.

---

## DHCP Server

### What is a DHCP Server?
###### A DHCP server dynamically provides IP configuration information to clients, eliminating the need for manual IP address assignment and reducing configuration errors.

---

### Network Topology
<img width="895" height="650" alt="Screenshot 2026-06-12 at 3 11 48 PM" src="https://github.com/user-attachments/assets/8022e42d-caa6-494c-829d-bfe88ca40171" />

---

### HQ-EDGE LAN Interface & DHCP Configuration

This covers configuring the LAN-facing interface on the HQ-EDGE router, setting up a DHCP pool for internal clients, and verifying connectivity from a connected PC.

#### HQ-EDGE Router — LAN Interface & DHCP Setup

```
HQ-EDGE-RTR> enable 
HQ-EDGE-RTR# configure terminal 
HQ-EDGE-RTR(config)# interface e0/0
HQ-EDGE-RTR(config-if)# ip address 192.168.1.1 255.255.255.0
HQ-EDGE-RTR(config-if)# no shutdown
HQ-EDGE-RTR(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
HQ-EDGE-RTR(config)# ip dhcp pool HQ-LAN
HQ-EDGE-RTR(dhcp-config)# network 192.168.1.0 255.255.255.0
HQ-EDGE-RTR(dhcp-config)# default-router 192.168.1.1
HQ-EDGE-RTR(dhcp-config)# dns-server 8.8.8.8
HQ-EDGE-RTR(dhcp-config)# lease 7
HQ-EDGE-RTR(dhcp-config)# end
HQ-EDGE-RTR# write 
```

**What this does:**
- Assigns `192.168.1.1/24` as the LAN gateway IP on `e0/0` and brings the interface up.
- Excludes `192.168.1.1–192.168.1.10` from the DHCP pool, reserving that range for the router and any future static assignments (servers, printers, etc.).
- Creates a DHCP pool (`HQ-LAN`) that dynamically hands out addresses from `192.168.1.0/24`, pointing clients to `192.168.1.1` as their default gateway and `8.8.8.8` as DNS.
- Sets the DHCP lease duration to **7 days** (`604800` seconds) before clients must renew.
- Saves the configuration to NVRAM with `write`.

#### PC1 — DHCP Request

##### Before DHCP

```
PC1:~$ ip addr
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:10:57:ea brd ff:ff:ff:ff:ff:ff
```

- `eth0` is up at Layer 2 but has no IP address yet — only a MAC address is present.

##### Requesting an Address

```
PC1:~$ sudo udhcpc -i eth0
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.1.12, server 192.168.1.1
udhcpc: lease of 192.168.1.12 obtained from 192.168.1.1, lease time 604800
```

- `udhcpc` is a lightweight DHCP client. Running it on `eth0` triggers the standard DORA process (Discover → Offer → Request → Acknowledge).
- The router responds from `192.168.1.1` and assigns `192.168.1.12` — the first available address outside the excluded range (`.1–.10`).
- Lease time of `604800` seconds confirms the 7-day lease configured on the router.

##### Packet Capture Analysis — DORA Exchange
<img width="1761" height="799" alt="Screenshot 2026-06-12 at 3 47 22 PM" src="https://github.com/user-attachments/assets/62200062-5133-4d0f-8aec-fc2a4cb3ac78" />

The `.pcap` confirms all four DORA steps at the wire level. All frames share transaction ID `0x30138215` and originate from PC1 MAC `52:54:00:10:57:ea`.

| Step | Direction | Key Details |
|---|---|---|
| **Discover** | `0.0.0.0` → `255.255.255.255` | Broadcast with no source IP; vendor class `udhcp 1.37.0` |
| **Offer** | `192.168.1.1` → `192.168.1.12` | Offers `192.168.1.12/24`, gateway `192.168.1.1`, DNS `8.8.8.8` |
| **Request** | `0.0.0.0` → `255.255.255.255` | PC1 formally selects the offer; still broadcast so other servers can withdraw |
| **Acknowledge** | `192.168.1.1` → `192.168.1.12` | Confirms 7-day lease; T1 renewal at ~3.5 days, T2 rebind at ~6.1 days |

> **Note:** The capture also contains background STP BPDUs (Rapid PVST+) and a CDP frame from `HQ-SW` running `IOS XE 17.16.1a` — normal control-plane traffic, not part of the DHCP exchange.

#### Gateway Connectivity Test

```
PC1:~$ ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1): 56 data bytes
64 bytes from 192.168.1.1: seq=0 ttl=42 time=1.800 ms
64 bytes from 192.168.1.1: seq=1 ttl=42 time=1.237 ms
64 bytes from 192.168.1.1: seq=2 ttl=42 time=1.380 ms
64 bytes from 192.168.1.1: seq=3 ttl=42 time=1.368 ms
64 bytes from 192.168.1.1: seq=4 ttl=42 time=1.369 ms
64 bytes from 192.168.1.1: seq=5 ttl=42 time=1.286 ms
^C
--- 192.168.1.1 ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max = 1.237/1.406/1.800 ms
```

- PC1 successfully pings the LAN gateway at `192.168.1.1` with **0% packet loss**.
- Consistent sub-2ms round-trip times confirm a healthy Layer 3 connection between the PC and the HQ-EDGE router.

---

## DHCP Relay

### What is a DHCP Relay?
###### A DHCP relay enables clients on a different subnet than the DHCP server to obtain IP addresses by forwarding DHCP broadcast messages as unicast packets to the server.

---

### Network Topology
<img width="1047" height="592" alt="Screenshot 2026-06-19 141050" src="https://github.com/user-attachments/assets/694f23da-2613-4ce8-b809-9a10fc23562b" />

The HQ Office network consists of a Cisco ISR4331 edge router, a Cisco 2960-24TT access switch, a dedicated DHCP server, two PCs, and two printers — all segmented across three VLANs.

---

### DHCP Server Configuration
<img width="927" height="167" alt="Screenshot 2026-06-19 141707" src="https://github.com/user-attachments/assets/55755914-7e69-4394-b1f8-9794f5736394" />

The DHCP Server is assigned a static IP address rather than obtaining one via DHCP. As a network service device, a static address ensures clients and the router always reach it at a predictable, fixed address.

| Field | Value |
|---|---|
| IP Configuration | Static |
| IPv4 Address | `192.168.2.2` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.2.1` |
| DNS Server | `8.8.8.8` |

---

### VLAN DHCP Pool Configuration
<img width="881" height="104" alt="Screenshot 2026-06-19 142420" src="https://github.com/user-attachments/assets/66818273-1044-4acb-b59f-ad95c9b569f2" />

DHCP pools are configured on the DHCP Server for VLAN 1 (PCs) and VLAN 3 (Printers). VLAN 2 (Servers) is excluded — devices on that subnet use static IPs.

| Pool | Default Gateway | DNS Server | Start IP | Subnet Mask | Max Users |
|---|---|---|---|---|---|
| VLAN1 | `192.168.1.1` | `8.8.8.8` | `192.168.1.10` | `255.255.255.0` | 243 |
| VLAN3 | `192.168.3.1` | `8.8.8.8` | `192.168.3.10` | `255.255.255.0` | 243 |

Each pool starts at `.10`, reserving `.1–.9` for static assignments. Max 243 users per pool covers the remaining `/24` address space.

---

### Setting Up DHCP Relay on HQ-EDGE-RTR

This covers the DHCP relay (`ip helper-address`) configuration on the HQ-EDGE-RTR, enabling VLAN 1 and VLAN 3 clients to reach the centralized DHCP server on VLAN 2.

#### Configuration

```
HQ-EDGE-RTR(config)# interface g0/0/0.1
HQ-EDGE-RTR(config-subif)# ip helper-address 192.168.2.2
HQ-EDGE-RTR(config-subif)# exit
HQ-EDGE-RTR(config)# interface g0/0/0.3
HQ-EDGE-RTR(config-subif)# ip helper-address 192.168.2.2
```

#### What This Does

DHCP Discover messages are broadcasts — they don't cross router boundaries by default. `ip helper-address` tells the router to intercept those broadcasts on a sub-interface and forward them as unicast to the DHCP server at `192.168.2.2`.

- **`g0/0/0.1` (VLAN 1 — PCs)** — relays DHCP requests from PCs to the DHCP server on VLAN 2.
- **`g0/0/0.3` (VLAN 3 — Printers)** — relays DHCP requests from printers to the same DHCP server.
- **VLAN 2 is excluded** — the DHCP server itself sits on VLAN 2 with a static IP, so no relay is needed there.

---

### PC DHCP Verification — VLAN 1

DHCP address assignment for the Reception PC and Office PC on VLAN 1 (`192.168.1.0/24`).

#### Reception PC

##### Before DHCP

```
C:\>ipconfig

FastEthernet0 Connection:(default port)

   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: 0.0.0.0
```

- No IP address assigned yet — the PC has not yet requested one from the DHCP server.

##### After `ipconfig /renew`

```
C:\>ipconfig /renew

   IP Address......................: 192.168.1.10
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: 192.168.1.1
   DNS Server......................: 8.8.8.8
```

- Assigned `192.168.1.10` — the first address available from the VLAN 1 pool (start IP `.10`).

#### Office PC

##### Before DHCP

```
C:\>ipconfig

FastEthernet0 Connection:(default port)

   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: 0.0.0.0
```

- No IP assigned prior to renewal.

##### After `ipconfig /renew`

```
C:\>ipconfig /renew

   IP Address......................: 192.168.1.12
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: 192.168.1.1
   DNS Server......................: 8.8.8.8
```

- Assigned `192.168.1.12` — the next available address after `.10` and `.11`.

#### Summary

Both PCs successfully obtained addresses via the DHCP relay through HQ-EDGE-RTR to the centralized DHCP server at `192.168.2.2`. Gateway and DNS settings match the VLAN 1 pool configuration.

| Device | Assigned IP | Gateway | DNS |
|---|---|---|---|
| Reception PC | `192.168.1.10` | `192.168.1.1` | `8.8.8.8` |
| Office PC | `192.168.1.12` | `192.168.1.1` | `8.8.8.8` |
