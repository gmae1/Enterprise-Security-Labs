# Lab 06 — UDP Traffic Analysis

## Objective

The objective of this lab was to capture and analyze UDP traffic using Wireshark and compare UDP communication with the TCP communication examined in previous labs.

A DNS query was generated from the Windows 11 workstation to the lab's DNS server, DC-01. The UDP header was analyzed to identify source and destination ports and demonstrate that UDP does not establish a connection using a handshake before transmitting data.

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
- Windows Server 2022 DNS
- pfSense
- VirtualBox

---

## Traffic Generation

Wireshark was started on the Windows 11 workstation and initially filtered for UDP traffic.

```text
udp
```

The following command was then executed from Win11Pro:

```powershell
nslookup www.cloudflare.com
```

Win11Pro was configured to use DC-01 (`192.168.50.10`) as its preferred DNS server.

The DNS lookup therefore generated UDP communication between Win11Pro and DC-01.

---

## UDP DNS Query

The captured DNS query showed Win11Pro communicating with DC-01 using UDP.

The traffic followed the general pattern:

```text
Win11Pro
192.168.50.101:<ephemeral-port>
        |
        | UDP DNS Query
        v
DC-01
192.168.50.10:53
```

The client selected a temporary high-numbered **ephemeral source port**, while the destination port was UDP port **53**, the well-known port used by DNS.

### Evidence

<img width="1024" height="768" alt="6 1-udp-dns-query" src="https://github.com/user-attachments/assets/517472e1-79b0-4171-890e-e3c04b073c7a" />

---

## UDP DNS Response

DC-01 responded to the DNS query using UDP.

The source and destination ports were reversed:

```text
DC-01
192.168.50.10:53
        |
        | UDP DNS Response
        v
Win11Pro
192.168.50.101:<ephemeral-port>
```

The DNS server used source port 53, while the response was returned to the same ephemeral port originally selected by the client.

### Evidence

<img width="1024" height="768" alt="6 2-udp-dns-response" src="https://github.com/user-attachments/assets/4f1f842e-d971-4963-9bae-69ec8686e209" />

---

## UDP Header Analysis

Unlike TCP, UDP has a very small transport-layer header.

The primary UDP header fields observed in Wireshark were:

```text
Source Port
Destination Port
Length
Checksum
```

The standard UDP header is only **8 bytes**.

This is considerably simpler than TCP, which contains additional fields used for connection management, sequencing, acknowledgments, flow control, and other reliability mechanisms.

---

## TCP vs UDP

One of the major observations from this lab was the absence of a TCP-style handshake.

TCP communication previously observed in the lab required:

```text
SYN
 ↓
SYN-ACK
 ↓
ACK
```

UDP did not perform this process.

The DNS query was transmitted without establishing a transport-layer connection first.

### TCP

TCP is connection-oriented and provides mechanisms for:

- Reliable delivery
- Ordered delivery
- Acknowledgments
- Retransmission
- Connection establishment
- Connection termination

### UDP

UDP is connectionless and does not provide built-in:

- Connection establishment
- Delivery guarantees
- Packet ordering
- Retransmission
- TCP-style acknowledgments

This reduces transport-layer overhead but places additional responsibility on the application when reliability is required.

---

## Important Observation: DNS Response vs UDP Acknowledgment

The DNS response observed in Wireshark should not be confused with a TCP acknowledgment.

```text
DNS Query
Client -----------------> DNS Server

DNS Response
Client <----------------- DNS Server
```

The response exists because the **DNS application protocol** uses a request/response model.

UDP itself did not acknowledge that the original datagram was successfully delivered.

If reliability or retries are required, the application using UDP can implement those mechanisms itself.

---

## Why Applications Use UDP

UDP can be useful when minimizing latency and transport overhead is more important than guaranteeing delivery of every individual packet.

Examples can include:

- DNS
- Voice over IP
- Real-time audio/video
- Online gaming
- Other latency-sensitive applications

For real-time communication, retransmitting old data may sometimes be less useful than continuing to process newer data.

The appropriate transport protocol depends on the application's requirements.

---

## Security Relevance

Understanding UDP behavior is important when analyzing network traffic because UDP communication looks fundamentally different from TCP communication.

A security analyst should recognize that the absence of a TCP handshake does not automatically indicate abnormal traffic when the application legitimately uses UDP.

UDP is also relevant when investigating activity involving:

- DNS traffic
- UDP port scanning
- Reflection and amplification attacks
- Unusual UDP services
- Unexpected source or destination ports
- Abnormal traffic volume

Packet analysis can help determine which systems are communicating, which ports are being used, and whether the observed behavior matches the expected network baseline.

---

## Key Takeaways

This lab demonstrated that:

1. UDP is a connectionless transport protocol.
2. UDP does not use the TCP three-way handshake.
3. UDP does not guarantee packet delivery or ordering.
4. UDP does not provide TCP-style acknowledgments or retransmission.
5. DNS commonly uses UDP port 53 for standard queries.
6. Clients commonly use temporary ephemeral source ports.
7. A DNS response is an application-layer response, not a UDP acknowledgment.
8. Applications may choose UDP when lower transport overhead and timely delivery are important.
9. Wireshark can be used to distinguish UDP communication from TCP communication.

---

## Lab Result

**PASS**

UDP DNS traffic was successfully generated, captured, filtered, and analyzed using Wireshark. The relationship between UDP, DNS, port numbers, and connectionless communication was successfully demonstrated.
