# Lab 05 — HTTPS & TLS Traffic Analysis

## Objective

The objective of this lab was to capture and analyze HTTPS traffic using Wireshark and understand how TLS protects HTTP application data.

This lab compared the plaintext HTTP traffic observed in Lab 04 with HTTPS traffic protected by TLS.

During the lab, I examined:

- TLS Client Hello
- TLS Server Hello
- TLS Application Data
- The relationship between TCP, TLS, and HTTP
- Why HTTPS application data is not visible in plaintext
- What information remains observable when network traffic is encrypted

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| Win11Pro | Client Workstation | 192.168.50.101 |
| DC-01 | DNS Server | 192.168.50.10 |
| pfSense | Router / Default Gateway | 192.168.50.1 |

**Internal Network:** `SEC-LAB`

**Subnet:** `192.168.50.0/24`

---

## Tools Used

- Wireshark
- Windows PowerShell
- Windows 11
- pfSense
- Oracle VirtualBox
- HTTPS
- TLS

---

# Generating HTTPS Traffic

A fresh Wireshark capture was started on the Win11Pro Ethernet interface.

The following Wireshark display filter was applied:

```text
tls
```

An HTTPS connection was then generated using PowerShell:

```powershell
curl.exe https://example.com
```

The request successfully returned the Example Domain webpage content to the client.

Unlike the HTTP traffic analyzed in Lab 04, the application-layer communication was protected by TLS while traveling across the network.

---

# HTTP vs HTTPS

In Lab 04, plain HTTP was used:

```text
http://example.com
```

Wireshark could directly inspect information such as:

```text
GET / HTTP/1.1

HTTP/1.1 200 OK

<html>...</html>
```

This information was visible because HTTP itself does not provide encryption.

In this lab, HTTPS was used:

```text
https://example.com
```

Instead of seeing the HTTP request and webpage content directly, Wireshark displayed TLS traffic such as:

```text
Client Hello

Server Hello

Application Data
```

The HTTP application data was protected by TLS encryption.

---

# TCP, TLS, and HTTP

TCP, TLS, and HTTP perform different jobs during an HTTPS connection.

A simplified model is:

```text
TCP
│
└── Establish the transport connection
        │
        ▼
TLS
│
└── Establish secure communication
        │
        ▼
HTTP
│
└── Exchange web requests and responses
```

A useful way to remember this relationship is:

```text
TCP  = Connect
TLS  = Secure
HTTP = Communicate
```

---

# Stage 1 — TCP Connection

Before the TLS handshake occurs, the underlying TCP connection must first be established.

As demonstrated in Lab 03:

```text
Client                              Server

SYN ──────────────────────────────►

     ◄──────────────────── SYN-ACK

ACK ──────────────────────────────►

       TCP CONNECTION ESTABLISHED
```

TCP provides the transport mechanism used to reliably move the TLS data between the endpoints.

---

# Stage 2 — TLS Client Hello

After the TCP connection is established, the client begins the TLS handshake.

The first TLS handshake message examined was:

```text
Client Hello
```

Conceptually, the client is saying:

```text
"I would like to establish secure communication.
Here are the TLS capabilities and cryptographic
options that I support."
```

The Client Hello may contain information such as:

- Supported TLS versions
- Supported cipher suites
- TLS extensions
- Other capabilities required during negotiation

The exact fields visible depend on the TLS version and implementation.

<img width="1024" height="768" alt="5 1tls-client-hello" src="https://github.com/user-attachments/assets/9e89d282-fff9-414c-aba3-827a6622ecdb" />

---

# Stage 3 — TLS Server Hello

The server responded with:

```text
Server Hello
```

Conceptually, the server is saying:

```text
"I received your Client Hello.
Here are the parameters selected for this TLS session."
```

The Server Hello may identify information such as:

- Negotiated TLS version
- Selected cipher suite
- TLS extensions
- Other negotiated parameters

The exact handshake behavior and visible fields depend on the TLS version being used.

<img width="1024" height="768" alt="5 2TLS-Server-Hello" src="https://github.com/user-attachments/assets/e35db75a-8129-4dcc-a24b-3efa43c9504b" />

---

# TLS Handshake

The Client Hello and Server Hello are part of a larger TLS handshake.

A simplified representation is:

```text
Client                               Server

Client Hello ──────────────────────►
"Here is what I support."

             ◄────── Server Hello
             "Here is what we selected."

                    │
                    ▼

        Additional TLS Handshake
             Communication

                    │
                    ▼

          SECURE TLS SESSION
```

The TLS handshake involves more than only the two Hello messages.

Depending on the TLS version, the handshake also performs functions related to:

- Cryptographic parameter negotiation
- Server authentication
- Key establishment
- Creation of shared session secrets

Once the necessary TLS handshake steps are completed, encrypted application communication can occur.

---

# Stage 4 — TLS Application Data

After the TLS session was established, Wireshark displayed packets identified as:

```text
Application Data
```

Unlike the HTTP traffic from Lab 04, the underlying HTTP request and response content was not directly readable in the normal passive packet capture.

Instead, Wireshark displayed encrypted TLS application data.

```text
Client                              Server

[Encrypted Application Data] ─────►

             ◄──── [Encrypted Application Data]
```

<img width="1024" height="768" alt="5 3tls-enccrypted-application-data" src="https://github.com/user-attachments/assets/434f9a38-db79-4ccd-ae9f-202632e9153b" />

---

# Why the Application Data Is Encrypted

HTTPS is HTTP protected using TLS.

Without TLS:

```text
HTTP

GET / HTTP/1.1
Host: example.com

HTTP/1.1 200 OK
<html>...</html>

        VISIBLE IN CAPTURE
```

With TLS:

```text
HTTPS

HTTP
 ↓
TLS Encryption
 ↓
Encrypted Application Data
 ↓
Network

        CONTENT PROTECTED
```

The client and server participating in the TLS session can process the protected communication.

A normal passive packet capture does not automatically possess the TLS session secrets required to decrypt the application content.

---

# Protocol Layering

The communication observed throughout the networking labs can be viewed as layers.

```text
+----------------------------------+
| HTTP / HTTPS Application Data    |
+----------------------------------+
| TLS Encryption                   |
+----------------------------------+
| TCP Transport                    |
+----------------------------------+
| IP Networking                    |
+----------------------------------+
| Ethernet / MAC Addressing        |
+----------------------------------+
```

Each layer performs a different function.

### Ethernet

Provides local Layer 2 frame delivery using MAC addresses.

### IP

Provides Layer 3 logical addressing and routing.

### TCP

Establishes and manages the transport connection.

### TLS

Provides cryptographic protection for the application communication.

### HTTP

Defines the web requests and responses exchanged by the application.

---

# Connecting Previous Labs

The first five networking labs demonstrate how multiple protocols work together rather than operating independently.

```text
ARP
│
└── Determine the local MAC address needed for delivery
        │
        ▼
DNS
│
└── Resolve a hostname to an IP address
        │
        ▼
TCP
│
└── Establish the transport connection
        │
        ▼
TLS
│
└── Establish secure communication
        │
        ▼
HTTP
│
└── Exchange web application data
```

This provides a more complete picture of what happens when a system communicates with a web service.

---

# What Can Still Be Observed?

Encryption protects the application content, but encrypted network traffic is not completely invisible.

Depending on the protocol, TLS version, and environment, network analysts may still be able to examine useful metadata such as:

- Source IP address
- Destination IP address
- Source port
- Destination port
- TCP flags
- Connection establishment
- Connection termination
- Packet sizes
- Packet timing
- Traffic volume
- Some TLS handshake information

This information can still be useful during network troubleshooting and security investigations.

---

# Security Relevance

TLS traffic analysis is important because most modern web traffic is encrypted.

Security analysts frequently investigate encrypted connections without automatically having access to the plaintext application content.

Observable network metadata can help identify:

- Unexpected outbound connections
- Connections to suspicious infrastructure
- Unusual destination IP addresses
- Unusual destination ports
- Abnormal traffic volumes
- Repeated connection attempts
- Suspicious connection timing
- TLS configuration issues
- Potential command-and-control communication

Encrypted traffic should not automatically be considered safe simply because its contents cannot be directly read.

Analysts must evaluate the surrounding network behavior and available metadata.

---

# HTTP vs HTTPS Summary

| Feature | HTTP | HTTPS |
|---|---|---|
| Typical TCP Port | 80 | 443 |
| TLS Protection | No | Yes |
| Application Data Encrypted | No | Yes |
| GET Request Normally Visible in Passive Capture | Yes | No |
| Response Content Normally Visible in Passive Capture | Yes | No |
| Provides Confidentiality for HTTP Content | No | Yes |

HTTPS can be summarized as:

```text
HTTP + TLS Protection = HTTPS
```

---

# Key Findings

During this lab, I observed that:

1. TCP establishes the underlying connection before the TLS handshake.
2. TLS establishes cryptographic protection over the communication channel.
3. The Client Hello begins TLS negotiation from the client side.
4. The Server Hello communicates parameters selected by the server.
5. The TLS handshake contains additional steps beyond Client Hello and Server Hello.
6. HTTP application data can be protected by TLS encryption.
7. Plain HTTP traffic can expose application content in packet captures.
8. HTTPS prevents a normal passive capture from directly revealing the protected HTTP content.
9. Encryption does not hide all network metadata.
10. Wireshark can still provide valuable information when analyzing encrypted network connections.

---

# Skills Practiced

- Wireshark packet capture
- TLS traffic analysis
- HTTPS traffic analysis
- TLS Client Hello analysis
- TLS Server Hello analysis
- Encrypted application data identification
- TCP/TLS/HTTP protocol layering
- HTTP vs HTTPS comparison
- Network metadata analysis
- Security traffic analysis
- Encrypted traffic investigation

---

## Lab Result

**PASS — Successfully generated, captured, and analyzed HTTPS/TLS traffic, including the TLS Client Hello, Server Hello, and encrypted Application Data.**
