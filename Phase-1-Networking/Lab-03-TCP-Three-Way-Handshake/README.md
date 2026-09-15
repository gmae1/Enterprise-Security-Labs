# Lab 03 — TCP Three-Way Handshake Analysis

## Objective

The objective of this lab was to capture and analyze the TCP three-way handshake using Wireshark.

I generated a TCP connection between a Windows 11 workstation and the DNS service running on DC-01 and examined the SYN, SYN-ACK, and ACK packets used to establish the connection.

This lab demonstrates how TCP establishes a connection before application data is exchanged.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| Win11Pro | TCP Client / Domain Workstation | 192.168.50.101 |
| DC-01 | Windows Server / DNS Server | 192.168.50.10 |
| pfSense | Router / Default Gateway | 192.168.50.1 |

**Internal Network:** `SEC-LAB`

**Subnet:** `192.168.50.0/24`

**Active Directory Domain:** `corp.contoso.local`

---

## Tools Used

- Wireshark
- Windows PowerShell
- Windows 11
- Windows Server 2022
- Oracle VirtualBox

---

# Generating TCP Traffic

Wireshark was started on the Windows 11 Ethernet interface connected to the SEC-LAB network.

A TCP connection to DC-01 was generated using PowerShell:

```powershell
Test-NetConnection 192.168.50.10 -Port 53
```

The test returned:

```text
TcpTestSucceeded : True
```

This confirmed that Win11Pro successfully established a TCP connection to the DNS service running on DC-01.

---

# Wireshark Filter

The following display filter was used to isolate TCP port 53 traffic involving DC-01:

```text
tcp.port == 53 && ip.addr == 192.168.50.10
```

This allowed the TCP three-way handshake to be clearly identified.

The handshake consisted of:

```text
SYN → SYN-ACK → ACK
```

---

# TCP Three-Way Handshake

TCP is a connection-oriented protocol.

Before application data is exchanged, the communicating systems establish a TCP connection using a three-way handshake.

```text
Win11Pro                              DC-01

   SYN ──────────────────────────────►

       ◄──────────────────── SYN-ACK

   ACK ──────────────────────────────►

         CONNECTION ESTABLISHED
```

Each packet performs a specific function during connection establishment.

---

# Packet 1 — SYN

The first packet was sent from Win11Pro to DC-01 with the SYN flag set.

```text
Win11Pro ───── SYN ─────► DC-01
```

Conceptually, the client is saying:

```text
"Can we establish a TCP connection?"
```

The SYN flag is used to begin establishing the TCP connection and synchronize TCP sequence numbers between the two systems.

The traffic was sent from a temporary client source port to TCP port 53 on DC-01.

```text
Source:
Win11Pro
192.168.50.101
Ephemeral Client Port

Destination:
DC-01
192.168.50.10
TCP Port 53
```

<img width="1024" height="768" alt="3 1TCP-SYN" src="https://github.com/user-attachments/assets/23b66df9-a7ce-4993-a41e-8de00a836e90" />


---

# Packet 2 — SYN-ACK

DC-01 responded with both the SYN and ACK flags set.

```text
Win11Pro ◄──── SYN-ACK ──── DC-01
```

The two flags perform different functions.

**SYN**

DC-01 begins synchronizing its side of the TCP connection.

**ACK**

DC-01 acknowledges that it received the SYN from Win11Pro.

Conceptually, DC-01 is saying:

```text
"I received your connection request, and I am ready to establish the connection."
```

The source port in this packet was TCP port 53, while the destination was the temporary client port originally selected by Win11Pro.

<img width="1024" height="768" alt="3 2TCP-SYN-ACK" src="https://github.com/user-attachments/assets/1243bcdb-6ccf-4232-9402-c85c96a9f4b7" />

---

# Packet 3 — ACK

Win11Pro responded with an ACK packet.

```text
Win11Pro ───── ACK ─────► DC-01
```

This ACK confirms that Win11Pro received the SYN-ACK from DC-01.

Conceptually:

```text
"I received your SYN-ACK."
```

After this final acknowledgment, the TCP three-way handshake is complete and the connection is established.

<img width="1024" height="768" alt="3 3TCP-ACK" src="https://github.com/user-attachments/assets/c020b200-31c7-4e0a-8dd5-6b9ce2628339" />


---

# Complete Handshake

The complete communication observed during the lab was:

```text
Win11Pro                              DC-01

   SYN ──────────────────────────────►
   "Can we establish a connection?"

       ◄──────────────────── SYN-ACK
       "I received your SYN.
        I am ready too."

   ACK ──────────────────────────────►
   "I received your SYN-ACK."

         TCP CONNECTION ESTABLISHED
```

This process allows both systems to establish TCP state and synchronize sequence information before application communication proceeds.

---

# TCP Sequence and Acknowledgment Numbers

TCP uses sequence and acknowledgment numbers to track communication.

Wireshark displays relative sequence numbers by default, which makes the handshake easier to analyze.

A simplified handshake may appear as:

```text
Client → Server

SYN
SEQ = 0


Server → Client

SYN-ACK
SEQ = 0
ACK = 1


Client → Server

ACK
SEQ = 1
ACK = 1
```

The acknowledgment numbers indicate which sequence information has been successfully received and what is expected next.

These mechanisms later allow TCP to provide reliable and ordered data delivery.

---

# Source and Destination Ports

During the connection, Win11Pro selected an ephemeral high-numbered source port.

DC-01 was listening for DNS connections on TCP port 53.

```text
CLIENT                               SERVER

Random/Ephemeral Port                TCP 53
        │                               │
        └──────── TCP Connection ───────┘
```

The client source port allows the operating system to identify the specific connection and deliver returned traffic to the correct process.

---

# Why TCP Uses a Handshake

TCP is connection-oriented.

The handshake establishes communication state on both systems before application data is exchanged.

A useful way to visualize the process is:

```text
Client: "Are you there?"

Server: "Yes. I received you. Are you there?"

Client: "Yes. I received you."

Connection established.
```

Once the connection is established, TCP provides mechanisms for features such as:

- Reliable delivery
- Ordered delivery
- Sequence tracking
- Acknowledgments
- Retransmission of missing data
- Flow control

---

# TCP vs UDP

One major difference between TCP and UDP is connection establishment.

```text
TCP

SYN
 ↓
SYN-ACK
 ↓
ACK
 ↓
Connection Established
 ↓
Data Transfer
```

TCP is connection-oriented and provides mechanisms for reliable, ordered communication.

UDP does not perform a three-way handshake before sending data.

```text
UDP

Send Datagram
      ↓
No connection establishment required
```

UDP has less protocol overhead but does not provide TCP's built-in guarantees for reliable and ordered delivery.

---

# Failed TCP Connections

A TCP handshake does not always complete successfully.

For example:

```text
Client ───── SYN ─────► Server

Client ───── SYN ─────► Server

Client ───── SYN ─────► Server

           No Response
```

Repeated SYN packets without a SYN-ACK indicate that the TCP connection is not being successfully established.

However, the packet capture alone does not automatically explain why.

Possible causes could include:

- Destination system is offline
- Network connectivity problem
- Firewall silently dropping traffic
- Routing problem
- Return traffic not reaching the client

Additional investigation would be required to determine the actual cause.

---

# TCP Reset

Another possible response is a TCP RST packet.

```text
Client ───── SYN ─────► Server

       ◄──── RST ───── Server
```

RST stands for **Reset**.

A reset indicates that the TCP connection is being rejected or terminated rather than successfully established.

For example, a system may return a reset when no service is listening on the requested destination port.

---

# Security Relevance

TCP handshake behavior is extremely useful during network troubleshooting and security investigations.

Security analysts can examine TCP flags and connection patterns to identify:

- Successful connections
- Failed connection attempts
- Repeated connection attempts
- Closed or unavailable services
- Firewall behavior
- Port scanning activity
- Suspicious outbound connections
- Abnormal connection patterns

For example, repeated SYN packets sent to many destination ports may be associated with network reconnaissance or SYN scanning.

A large volume of incomplete TCP handshakes can also be relevant when investigating SYN flood denial-of-service activity.

Packet behavior must be analyzed in context before determining whether traffic is malicious.

---

# Key Findings

During this lab, I observed that:

1. TCP uses a three-way handshake before establishing a connection.
2. The handshake consists of SYN, SYN-ACK, and ACK.
3. SYN begins the TCP connection establishment process.
4. SYN-ACK acknowledges the client's SYN while synchronizing the server side of the connection.
5. The final ACK acknowledges the server's SYN-ACK.
6. Completion of the handshake establishes the TCP connection.
7. Clients commonly use ephemeral source ports when connecting to server services.
8. DC-01 accepted the test connection on TCP port 53.
9. TCP sequence and acknowledgment numbers help track communication.
10. Repeated SYN packets without a response indicate an incomplete connection attempt but do not, by themselves, reveal the cause.
11. TCP handshake analysis can assist with network troubleshooting and security investigations.

---

# Skills Practiced

- Wireshark packet capture
- TCP packet analysis
- TCP three-way handshake analysis
- SYN flag identification
- ACK flag identification
- TCP sequence number analysis
- TCP acknowledgment analysis
- Source and destination port analysis
- PowerShell network testing
- TCP troubleshooting
- Wireshark display filtering
- Network traffic interpretation
- Security traffic analysis

---

## Lab Result

**PASS — Successfully generated, captured, and analyzed a complete TCP three-way handshake between Win11Pro and DC-01.**
