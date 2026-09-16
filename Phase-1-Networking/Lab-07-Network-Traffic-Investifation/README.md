# Lab 07 — Network Traffic Investigation

## Objective

The objective of this lab was to troubleshoot a failed TCP connection using a structured network investigation process.

Instead of immediately assuming the cause of the problem, connectivity tests, Wireshark packet analysis, and server-side inspection were used to determine why Win11Pro could not establish a TCP connection to DC-01 on port 4444.

This lab focused on distinguishing between:

- Host reachability
- Port/service reachability
- TCP connection establishment
- TCP retransmissions
- Listening services
- Evidence-based troubleshooting

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| Win11Pro | Client Workstation | 192.168.50.101 |
| DC-01 | Active Directory / DNS Server | 192.168.50.10 |
| pfSense | Router / Firewall / Default Gateway | 192.168.50.1 |

**Network:** `192.168.50.0/24`  
**Virtual Network:** `SEC-LAB`

---

## Tools Used

- Wireshark
- Windows PowerShell
- Test-NetConnection
- netstat
- ICMP Ping
- Windows Server 2022
- VirtualBox

---

## Investigation Scenario

Win11Pro was unable to establish a TCP connection to DC-01 on port 4444.

The investigation followed a structured troubleshooting process:

```text
Connection Problem
        |
        v
Test Host Reachability
        |
        v
Test Specific TCP Port
        |
        v
Analyze Packets
        |
        v
Check Listening Services
        |
        v
Perform Control Test
        |
        v
Determine Finding
```

---

## Step 1 — Verify Host Reachability

Before investigating the application port, basic connectivity between Win11Pro and DC-01 was tested.

The following command was executed from Win11Pro:

```powershell
ping 192.168.50.10
```

DC-01 successfully returned four ICMP Echo Replies.

This established that Win11Pro could reach DC-01 across the network.

### Finding

```text
Win11Pro → DC-01

ICMP Connectivity: PASS
```

The successful ping demonstrated basic IP reachability.

However, successful ICMP communication does not prove that every TCP service on the destination is available.

---

## Step 2 — Test TCP Port 4444

The specific TCP port was then tested from Win11Pro:

```powershell
Test-NetConnection 192.168.50.10 -Port 4444
```

The result was:

```text
TcpTestSucceeded : False
```

This demonstrated that although DC-01 was reachable, a TCP connection to port 4444 could not be established.

---

## Step 3 — Analyze the Failed Connection with Wireshark

Wireshark was used to isolate the TCP traffic associated with port 4444.

The following display filter was applied:

```text
tcp.port == 4444
```

The capture showed Win11Pro repeatedly transmitting TCP SYN packets toward DC-01.

Wireshark identified subsequent attempts as:

```text
[TCP Retransmission] [SYN]
```

A successful TCP three-way handshake would normally appear as:

```text
Client                      Server

SYN ------------------------>

    <---------------- SYN-ACK

ACK ------------------------>

Connection Established
```

Instead, the observed traffic resembled:

```text
Win11Pro                    DC-01

SYN ------------------------>

SYN ------------------------>  TCP Retransmission

SYN ------------------------>  TCP Retransmission

SYN ------------------------>  TCP Retransmission

No successful handshake observed
```

The expected SYN-ACK was not observed in the capture.

### Evidence

<img width="1024" height="768" alt="7 1Failed-TCP-444-Syn-Retransmission" src="https://github.com/user-attachments/assets/909ac38c-e0fd-4fc4-9fcd-51ac6294c91a" />

### Finding

The TCP three-way handshake was not completing.

The retransmissions showed that Win11Pro repeatedly attempted to establish the connection but did not receive the response required to complete the handshake.

At this stage, the packet capture demonstrated the connection failure but did not by itself establish the underlying cause.

---

## Step 4 — Check for a Listening Service

Because the host itself was reachable, the investigation moved to DC-01 to determine whether a service was listening on TCP port 4444.

The following command was executed on DC-01:

```powershell
netstat -ano | findstr :4444
```

The command returned no results.

### Evidence

<img width="1024" height="768" alt="7 2-no-service-listening-port-4444" src="https://github.com/user-attachments/assets/9a46832b-eb6e-4985-8a3c-46f73f076583" />

### Finding

No process was shown by netstat as using or listening on port 4444.

A TCP port does not provide a network service by itself. An application or service must bind to the port and listen for incoming connections.

Conceptually:

```text
DC-01

Port 53
   |
   └── DNS Service Listening
       ✅ Service Available


Port 4444
   |
   └── No Listening Service Found
       ❌ No Application Available
```

---

## Step 5 — Perform a Control Test

To verify the difference between host reachability and service availability, a known service on DC-01 was tested.

DC-01 provides DNS services on port 53.

The following command was executed from Win11Pro:

```powershell
Test-NetConnection 192.168.50.10 -Port 53
```

The result was:

```text
TcpTestSucceeded : True
```

### Evidence

<img width="1024" height="768" alt="7 3-working-tcp-port-53" src="https://github.com/user-attachments/assets/0fdfcb72-da03-4633-a7b5-8598e7d3524c" />

This demonstrated that Win11Pro could successfully establish TCP communication with a service available on DC-01.

---

## Comparison

The investigation produced the following results:

| Test | Result | Interpretation |
|---|---|---|
| Ping DC-01 | PASS | DC-01 was reachable |
| TCP Port 4444 | FAIL | TCP connection could not be established |
| Wireshark TCP/4444 | SYN retransmissions | TCP handshake did not complete |
| netstat Port 4444 | No result | No listening service was identified |
| TCP Port 53 | PASS | Known DNS service accepted TCP connection |

---

## Root Finding

The investigation determined that the network path between Win11Pro and DC-01 was operational.

DC-01 successfully responded to ICMP traffic, and a TCP connection to the known DNS service on port 53 succeeded.

However, the connection attempt to TCP port 4444 failed.

Server-side inspection showed that no service was listening on TCP port 4444.

Therefore, the investigation identified **the absence of a listening service on TCP port 4444 as a key reason that the requested application connection could not succeed**.

The packet capture also showed unanswered SYN retransmissions. Because the capture alone does not establish why those SYNs received no response, firewall or filtering behavior should not be claimed as proven without additional evidence.

---

## Troubleshooting Methodology

A major lesson from this lab was the importance of separating different layers of connectivity.

```text
Can I reach the host?
        |
       Ping
        |
       YES
        |
        v
Can I reach the required service?
        |
Test-NetConnection
        |
        NO
        |
        v
What does packet analysis show?
        |
    Wireshark
        |
        v
Is the service listening?
        |
      netstat
        |
        NO
        |
        v
Investigate the service/application
```

If a service were confirmed to be listening but remote connections still failed, additional investigation could include:

- Host firewall rules
- Network firewall rules
- Routing
- Access control policies
- Application configuration
- Network path issues

---

## Security Relevance

This troubleshooting process is useful in help desk, system administration, network engineering, and security operations.

A security analyst should avoid immediately assuming that a failed connection means:

```text
"The network is down."
```

or:

```text
"The firewall blocked it."
```

Instead, evidence should be collected to determine which part of the communication process is failing.

Wireshark can provide visibility into connection attempts and TCP behavior, while server-side tools such as netstat can determine whether expected services are available.

This evidence-based approach helps distinguish between:

- Network connectivity problems
- Service availability problems
- Firewall/filtering problems
- Application configuration problems

---

## Key Takeaways

1. Successful ping demonstrates basic IP reachability but does not prove that a specific service is available.
2. `Test-NetConnection` can test whether a TCP connection can be established to a specific destination port.
3. TCP SYN retransmissions indicate that the sender did not receive the expected response.
4. Packet captures show what happened on the network but may not independently prove why it happened.
5. `netstat` can help determine whether a service is listening on a specific port.
6. A port is only a numbered transport endpoint; an application or service must use/listen on it to accept connections.
7. A server can be reachable while a specific service remains unavailable.
8. Known-working services can be used as control tests during troubleshooting.
9. Troubleshooting should follow evidence instead of assumptions.
10. Host connectivity and service connectivity are separate concepts.

---

## Lab Result

**PASS**

The failed TCP connection to port 4444 was successfully investigated using ICMP testing, TCP connectivity testing, Wireshark packet analysis, server-side port inspection, and a known-working service comparison.

The lab demonstrated a structured method for determining whether a connectivity problem exists at the host, transport, or application/service level.
