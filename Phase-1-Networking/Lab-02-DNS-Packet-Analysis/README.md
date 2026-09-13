# Lab 02 — DNS Packet Analysis

## Objective

The objective of this lab was to capture and analyze DNS traffic between a Windows 11 workstation and the DNS server within the SEC-LAB environment.

Using Wireshark, I observed how a client requests the IPv4 address associated with a hostname and how the DNS server responds with the corresponding DNS record.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| Win11Pro | DNS Client / Domain Workstation | 192.168.50.101 |
| DC-01 | Active Directory / DNS Server | 192.168.50.10 |
| pfSense | Router / Default Gateway | 192.168.50.1 |

**Internal Network:** `SEC-LAB`

**Subnet:** `192.168.50.0/24`

**Active Directory Domain:** `corp.contoso.local`

---

## Tools Used

- Wireshark
- Windows PowerShell
- Windows Server 2022 DNS
- Windows 11
- pfSense
- Oracle VirtualBox

---

# DNS Configuration

Win11Pro was configured to use DC-01 as its DNS server.

```text
Win11Pro
IP Address: 192.168.50.101
DNS Server: 192.168.50.10
```

This configuration allows the domain workstation to use DC-01 for both internal Active Directory DNS resolution and external DNS resolution.

---

# Traffic Capture

Wireshark was started on the Windows 11 Ethernet interface connected to the `SEC-LAB` network.

The following Wireshark display filter was applied:

```text
dns
```

A DNS lookup was then manually generated from PowerShell:

```powershell
nslookup www.google.com
```

This produced a DNS query from Win11Pro to DC-01 followed by a DNS response from DC-01 to Win11Pro.

---

# DNS Query Analysis

Win11Pro sent a DNS query to DC-01 requesting the A record associated with:

```text
www.google.com
```

The captured traffic followed this direction:

```text
192.168.50.101 → 192.168.50.10
Win11Pro          DC-01
```

The query contained:

- Source IP: `192.168.50.101`
- Destination IP: `192.168.50.10`
- Destination Port: `53`
- Query Name: `www.google.com`
- Record Type: `A`
- Class: `IN`

An **A record** maps a hostname to an IPv4 address.

The DNS query was therefore essentially asking:

```text
"What IPv4 address corresponds to www.google.com?"
```

<img width="1024" height="768" alt="2 1DestinationPort" src="https://github.com/user-attachments/assets/c16f08e3-daa3-4aed-b33e-0418ce0b62e5" />
<img width="1024" height="768" alt="2 2Queries" src="https://github.com/user-attachments/assets/2d78590c-64c3-484a-8b90-e370ad5fd3b2" />


---

# DNS Response Analysis

DC-01 responded to Win11Pro with the DNS information requested by the client.

The response traveled in the opposite direction:

```text
192.168.50.10 → 192.168.50.101
DC-01             Win11Pro
```

Within the DNS response, the Answer section contained an A record mapping the requested hostname to an IPv4 address.

Conceptually:

```text
www.google.com → A Record → IPv4 Address
```

The DNS response allowed Win11Pro to determine which IPv4 address should be used when communicating with the requested hostname.

<img width="1024" height="768" alt="2 3ResponseUDP" src="https://github.com/user-attachments/assets/acd8afdf-b49e-4e3a-8721-d0c21c21ec03" />
<img width="1024" height="768" alt="2 4ResponseARecord" src="https://github.com/user-attachments/assets/62585859-3804-47f7-81c1-66de279b5b4e" />

---

# UDP Port Analysis

The DNS query observed during the lab used UDP.

The client selected a temporary high-numbered source port and sent the request to the DNS service on UDP port 53.

```text
DNS QUERY

Win11Pro
Random High Port
      |
      | UDP
      v
DC-01
Port 53
```

When DC-01 returned the response, the source and destination ports were reversed.

```text
DNS RESPONSE

DC-01
Port 53
      |
      | UDP
      v
Win11Pro
Original Client Port
```

Port `53` is the well-known port associated with DNS.

Although ordinary DNS queries commonly use UDP port 53, DNS can also use TCP port 53 when required.

---

# A and AAAA Records

Two important DNS record types examined during this lab were A and AAAA records.

```text
A Record
Hostname → IPv4 Address

AAAA Record
Hostname → IPv6 Address
```

For example:

```text
www.example.com → A → IPv4 Address
www.example.com → AAAA → IPv6 Address
```

The DNS query captured during this lab requested an **A record**, meaning Win11Pro was requesting an IPv4 address.

---

# DNS Resolution Process

The observed communication can be summarized as:

```text
User/Application requests www.google.com
                |
                v
Win11Pro needs the destination IP
                |
                v
Win11Pro sends DNS query to DC-01
                |
                | UDP Port 53
                v
DC-01 DNS Server
                |
                v
DNS information is resolved
                |
                v
DC-01 sends DNS response
                |
                v
Win11Pro receives the IP address
                |
                v
Client can communicate with destination
```

DC-01 may answer using cached information or perform/forward DNS resolution using its configured DNS infrastructure when the requested external record is not already available locally.

---

# Relationship Between ARP and DNS

This lab also demonstrated how DNS builds upon the networking concepts examined in Lab 01.

Before Win11Pro can send a DNS query to DC-01, Ethernet communication with DC-01 must be possible.

If Win11Pro does not already know the MAC address associated with `192.168.50.10`, ARP can be used first.

```text
Need to resolve www.google.com
            |
            v
DNS Server = 192.168.50.10
            |
            v
Is DC-01 MAC address known?
            |
           NO
            |
            v
ARP Request
"Who has 192.168.50.10?"
            |
            v
ARP Reply
"192.168.50.10 is at [MAC]"
            |
            v
DNS Query → UDP 53
            |
            v
DNS Response
```

This demonstrates how multiple protocols work together during normal network communication.

---

# Active Directory DNS

DC-01 also provides DNS services for the Active Directory domain:

```text
corp.contoso.local
```

Domain-joined Windows systems rely heavily on DNS to locate Active Directory resources and services.

This is why Win11Pro is configured to use DC-01 as its DNS server rather than directly using a public DNS resolver.

---

# Security Relevance

DNS traffic can provide valuable information during security investigations because it reveals which domain names systems attempt to resolve.

Security analysts may examine DNS activity when investigating:

- Connections to suspicious domains
- Malware command-and-control infrastructure
- Phishing infrastructure
- Unexpected DNS servers
- Abnormally frequent DNS requests
- DNS spoofing or cache poisoning
- DNS tunneling
- Compromised endpoints attempting to locate malicious infrastructure

Understanding normal DNS behavior creates a baseline for recognizing abnormal DNS activity.

---

# Key Findings

During this lab, I observed that:

1. Win11Pro was configured to use DC-01 as its DNS server.
2. DNS queries were sent from Win11Pro to DC-01.
3. DNS traffic used port 53.
4. The client requested the A record for `www.google.com`.
5. An A record maps a hostname to an IPv4 address.
6. An AAAA record maps a hostname to an IPv6 address.
7. DC-01 returned the DNS response to the requesting workstation.
8. DNS queries and responses can be directly inspected using Wireshark.
9. DNS and ARP perform different functions but can work together during network communication.
10. DNS traffic is valuable evidence during security monitoring and incident investigation.

---

# Skills Practiced

- Wireshark packet capture
- DNS packet analysis
- Wireshark display filtering
- DNS query analysis
- DNS response analysis
- A record identification
- AAAA record identification
- UDP port analysis
- DNS port 53
- Windows DNS configuration
- Active Directory DNS concepts
- Network troubleshooting
- Protocol analysis
- Security traffic analysis

---

## Lab Result

**PASS — DNS query and response traffic successfully captured and analyzed between Win11Pro and DC-01.**
