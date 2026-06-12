## What is Dynamic Host Configuration Protocol (DHCP)?

#### Topics Covered
- DHCP Fundamentals and Purpose
- DHCP Client
- DHCP Server
- DHCP Relay
---

## How DHCP Automates IP Address Management

#### Dynamic Allocation & Parameters
###### DHCP assigns IPs and configurations (DNS, gateway) on demand, letting devices join/leave without manual setup.

#### Fewer Errors, Faster Onboarding
###### Automated provisioning can cut IP configuration mistakes up to 75%, improving reliability and uptime.

#### Lease Management & Utilization
###### Tunable lease durations optimize address reuse, maximizing IP pool efficiency from small offices to enterprises. 

#### Scalability & Automation
###### Discovery, offer, request, and acknowledgment enable streamlined growth can centralized network management.
---

## DHCP Client
### What is a DHCP Client?
###### A DHCP client is a device that automatically requests and receives IP configuration information from a DHCP server. This information typically includes an IP address, subnet mask, default gateway, and DNS servers, allowing the device to communicate on the network without manual configuration.
#

#### Network Lab Topoogy
<img width="971" height="524" alt="Screenshot 2026-06-12 at 12 23 25 PM" src="https://github.com/user-attachments/assets/ea5147e5-a8fe-4610-b847-db03fd4a5a95" />

## ISP & HQ-EDGE Router Configuration Walkthrough

This covers the initial WAN connectivity setup between a simulated ISP router and the HQ edge router. The goal is to bring up the WAN link, assign a public IP via DHCP, and verify end-to-end reachability.

#

## ISP Router Configuration

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

---

## HQ-EDGE Router Configuration

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

---

## Verification

### Interface Status

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

### Connectivity Test

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
