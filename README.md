# IPv4 Packet Flow & Default Gateway Troubleshooting

A Cisco Packet Tracer lab focused on understanding IPv4 packet flow, ARP, MAC address learning, default gateway behavior, connectivity verification, and evidence-based troubleshooting.

## Objective

The purpose of this lab is to understand how a host communicates with devices on the same subnet and with devices on a remote subnet, then use network evidence to identify and troubleshoot a default gateway misconfiguration.

The lab follows a practical troubleshooting workflow:

**Define → Gather Evidence → Hypothesis → Test → Root Cause → Fix → Verify → Document**

---

## Topology

The network consists of two IPv4 LANs connected through a Cisco router.

```text
                 LAN 1
        192.168.10.0/24
                 
   PC-A                    PC-B
192.168.10.10          192.168.10.20
      |                      |
      +-------- SW1 ---------+
                   |
              R1 G0/0
           192.168.10.1
                   |
                   |
              R1 G0/1
           192.168.20.1
                   |
                  SW2
                   |
                Server
           192.168.20.50

                 LAN 2
        192.168.20.0/24
```

---

## IP Addressing

| Device | Interface | IP Address    | Subnet Mask   | Default Gateway |
| ------ | --------- | ------------- | ------------- | --------------- |
| PC-A   | NIC       | 192.168.10.10 | 255.255.255.0 | 192.168.10.1    |
| PC-B   | NIC       | 192.168.10.20 | 255.255.255.0 | 192.168.10.1    |
| R1     | G0/0      | 192.168.10.1  | 255.255.255.0 | —               |
| R1     | G0/1      | 192.168.20.1  | 255.255.255.0 | —               |
| Server | NIC       | 192.168.20.50 | 255.255.255.0 | 192.168.20.1    |

---

## Router Configuration

The router provides connectivity between the two directly connected networks.

```text
enable
configure terminal

hostname R1

interface gigabitEthernet 0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit

interface gigabitEthernet 0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit

end
```

No static routes were required because both networks are directly connected to R1.

---

## Baseline Verification

Before introducing a fault, connectivity and device state were verified.

### Router Interface Status

```text
show ip interface brief
```

Both router interfaces were verified as:

```text
up/up
```

### Routing Table

```text
show ip route
```

The router learned both networks as directly connected:

```text
192.168.10.0/24
192.168.20.0/24
```

### ARP Table

```text
show ip arp
```

The router had ARP entries for devices on both directly connected networks.

### Connectivity Tests

From PC-A:

```text
ping 192.168.10.20
ping 192.168.10.1
ping 192.168.20.50
```

Connectivity was successful.

The first ping to the remote server initially showed one lost packet. Repeating the test resulted in successful connectivity. This was treated as initial ARP/neighbor resolution rather than a persistent network fault.

---

## Local vs Remote Communication

A key part of the lab was comparing communication within the same subnet with communication to a remote subnet.

### Local Communication

PC-A:

```text
192.168.10.10
```

PC-B:

```text
192.168.10.20
```

Both devices are in:

```text
192.168.10.0/24
```

Therefore, PC-A can communicate directly with PC-B at Layer 2 after resolving PC-B's MAC address using ARP.

The frame is addressed directly to PC-B's MAC address.

### Remote Communication

The server is:

```text
192.168.20.50
```

This is outside PC-A's local subnet.

Therefore, PC-A must send the frame to its default gateway:

```text
192.168.10.1
```

PC-A uses ARP to resolve the gateway's MAC address.

The first Ethernet frame has:

```text
Source MAC      = PC-A
Destination MAC = R1 G0/0
```

while the IP packet keeps:

```text
Source IP       = 192.168.10.10
Destination IP  = 192.168.20.50
```

When the router forwards the packet to the second network, the Layer 2 frame is rebuilt for that network.

This demonstrates the difference between:

* Final destination IP
* Current Layer 2 destination MAC
* Default gateway as the next hop

---

## Troubleshooting Scenario

A fault was intentionally introduced by changing PC-A's default gateway from:

```text
192.168.10.1
```

to:

```text
192.168.10.254
```

### Symptoms

After the change:

```text
ping 192.168.10.20
```

was successful.

```text
ping 192.168.10.1
```

was successful.

However:

```text
ping 192.168.20.50
```

failed completely.

This indicated that local communication was still working while remote-subnet communication was failing.

---

## Evidence Collection

### 1. Verify Host Configuration

```text
ipconfig
```

PC-A showed:

```text
IP Address      192.168.10.10
Subnet Mask     255.255.255.0
Default Gateway 192.168.10.254
```

The gateway did not match the actual router interface:

```text
R1 G0/0 = 192.168.10.1
```

### 2. Check ARP

```text
arp -a
```

The ARP table contained entries for reachable local devices, including:

```text
192.168.10.1
192.168.10.20
```

There was no valid ARP entry for:

```text
192.168.10.254
```

### 3. Compare Local and Remote Tests

The results were:

| Test           | Result     |
| -------------- | ---------- |
| PC-A → PC-B    | Successful |
| PC-A → R1 G0/0 | Successful |
| PC-A → Server  | Failed     |

This narrowed the problem to communication beyond the local network rather than a complete loss of LAN connectivity.

---

## Root Cause

The root cause was an incorrect default gateway configured on PC-A.

Configured:

```text
192.168.10.254
```

Correct gateway:

```text
192.168.10.1
```

Because the server is on a remote subnet, PC-A needs to forward traffic to its default gateway.

With the incorrect gateway configured, PC-A could communicate with devices on its local subnet but could not correctly forward traffic toward the remote network.

---

## Remediation

The default gateway on PC-A was changed back to:

```text
192.168.10.1
```

---

## Verification After the Fix

The configuration was verified using:

```text
ipconfig
```

The default gateway was:

```text
192.168.10.1
```

Connectivity to the remote server was then tested:

```text
ping 192.168.20.50
```

Result:

```text
4/4 replies
0% packet loss
```

This confirmed that the connectivity issue had been resolved.

---

## Packet Flow Observation

Cisco Packet Tracer Simulation Mode was used to observe the packet journey.

A packet sent from the Server:

```text
Source IP      = 192.168.20.50
Destination IP = 192.168.10.10
```

initially used the router interface on the server's local network as the Layer 2 next hop.

The first Ethernet frame showed:

```text
Source MAC      = Server MAC
Destination MAC = R1 G0/1 MAC
```

The server did not need to know PC-A's MAC address directly.

Instead, it sent the frame to its default gateway.

This demonstrates an important networking principle:

> The destination IP identifies the final destination, while the destination MAC identifies the next Layer 2 hop.

---

## Troubleshooting Workflow

The troubleshooting process used in this lab was:

```text
1. Define the problem
2. Gather evidence
3. Form a hypothesis
4. Test the hypothesis
5. Identify the root cause
6. Apply the fix
7. Verify the result
8. Document the incident
```

The workflow was applied instead of immediately changing network settings.

---

## Key Lessons

### 1. Same-subnet communication does not require the default gateway

PC-A could reach PC-B even when the default gateway was incorrect because both devices were on the same subnet.

### 2. Remote communication requires a valid next hop

To reach the server on another subnet, PC-A needed to send traffic to R1 through its configured default gateway.

### 3. ARP is used to resolve Layer 2 addresses

ARP maps an IPv4 address to a MAC address on the local network.

### 4. MAC addresses change across routed networks

The Layer 2 frame is rebuilt at each routed hop.

The destination IP remains the final destination IP while the Layer 2 destination changes according to the next hop.

### 5. Ping results must be interpreted in context

A successful ping to the local gateway does not prove that communication with a remote network or server will work.

Testing different destinations helps isolate where the problem exists.

---

## Verification Commands

The following commands were used during the investigation:

### PC Commands

```text
ipconfig
arp -a
ping <destination>
```

### Router Commands

```text
show ip interface brief
show ip route
show ip arp
```

### Packet Tracer

Simulation Mode was used to inspect packet movement and Layer 2/Layer 3 information.

---

## Tools

* Cisco Packet Tracer
* Cisco IOS CLI
* Windows-style host networking commands available in Packet Tracer
* ARP
* ICMP
* IPv4
* Ethernet
* Packet Tracer Simulation Mode

---

## Project Structure

```text
ipv4-packet-flow-default-gateway-troubleshooting/
│
├── README.md
│
├── topology/
│   └── ipv4_default_gateway_troubleshooting.pkt
│
└── screenshots/
    ├── 01-topology.png
    ├── 02-baseline-verification.png
    ├── 03-default-gateway-fault.png
    ├── 04-troubleshooting-evidence.png
    ├── 05-fixed-connectivity.png
    └── 06-packet-flow-pdu.png
```

---

## Scope

This lab focuses on:

* IPv4 addressing
* Subnetting fundamentals
* Local vs remote communication
* ARP
* MAC address learning
* Default gateway behavior
* Basic routing
* ICMP connectivity testing
* Evidence-based troubleshooting
* Packet flow analysis

Layer 2 switching topics such as VLANs, trunks, and STP are intentionally outside the main troubleshooting scope of this lab and are covered in later networking labs.

---

## References

* Cisco Networking Academy — Networking fundamentals and Cisco Packet Tracer
* Cisco IOS documentation — ARP and IP networking
* IETF RFC 826 — Address Resolution Protocol (ARP)
* IETF RFC 791 — Internet Protocol Version 4 (IPv4)
* IETF RFC 792 — Internet Control Message Protocol (ICMP)
