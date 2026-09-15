# Lab 04 — TCP Data Transfer & Connection Termination

## Objective

The objective of this lab was to analyze what happens after a TCP connection has been established.

Using Wireshark, I generated an HTTP request from Win11Pro to `example.com` and examined:

- HTTP application data traveling over TCP
- HTTP GET requests
- HTTP 200 OK responses
- Client and server TCP ports
- TCP FIN and ACK flags
- Graceful TCP connection termination

This lab builds directly on Lab 03, where I analyzed the TCP three-way handshake.

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
- HTTP

---

# Generating HTTP Traffic

A fresh Wireshark capture was started on the Win11Pro Ethernet interface.

The following command was executed from PowerShell:

```powershell
curl.exe http://example.com
```

HTTP was intentionally used instead of HTTPS so the application-layer request and response could be directly inspected in Wireshark without encryption.

The command successfully returned HTML content from Example Domain.

---

# HTTP Traffic Analysis

The following Wireshark display filter was used:

```text
http
```

This isolated the HTTP request and response generated during the connection.

Two important packets were identified:

```text
GET / HTTP/1.1

HTTP/1.1 200 OK
```

These packets represent application-layer communication occurring after TCP established the connection.

---

# HTTP GET Request

Win11Pro sent an HTTP GET request to the web server.

The request contained information similar to:

```http
GET / HTTP/1.1
Host: example.com
```

The GET method requests a resource from a web server.

The `/` identifies the root resource of the website.

Therefore:

```text
GET /
```

can conceptually be understood as:

```text
"Send me the resource located at the root of this website."
```

The `Host` field identifies the website being requested:

```text
Host: example.com
```

The client is the system making the request. The `/` is the resource being requested; it is not the system making the request.

<img width="1024" height="768" alt="4 1http-get-request" src="https://github.com/user-attachments/assets/08867e33-2e9e-434c-a70d-aa73d1e35e95" />

---

# HTTP 200 OK Response

The web server responded to the GET request with:

```text
HTTP/1.1 200 OK
```

HTTP status code `200` indicates that the request was successfully processed.

Conceptually:

```text
Win11Pro                           Web Server

GET / HTTP/1.1 ──────────────────►
"Send me the root resource."

              ◄──── HTTP 200 OK
                    + HTML Content

              "Request successful.
               Here is the content."
```

The HTML returned by the server was the application data displayed in PowerShell by `curl.exe`.

<img width="1024" height="768" alt="4 2-http-200-response" src="https://github.com/user-attachments/assets/2bdc2742-f639-4dfc-925b-5bafc6e85040" />

---

# TCP and HTTP Relationship

TCP and HTTP perform different functions during the communication.

TCP provides the transport connection.

HTTP uses that TCP connection to exchange web requests and responses.

A simplified model is:

```text
TCP
│
├── Establishes the connection
├── Tracks communication
├── Provides reliable delivery mechanisms
└── Terminates the connection

        ↓

HTTP
│
├── GET request
└── 200 OK + Web Content
```

A useful way to visualize this relationship is:

```text
TCP = The transportation path

HTTP = The application data traveling across that path
```

The TCP connection must first be established before the HTTP communication observed in this lab can occur.

---

# Client and Server Ports

The HTTP server communicated using TCP port:

```text
80
```

Port 80 is the well-known port associated with HTTP.

Win11Pro used an ephemeral high-numbered client port.

Conceptually:

```text
Win11Pro                          Web Server

High/Ephemeral Port  ◄────────►  TCP Port 80
```

The server's well-known port identifies the HTTP service.

The client's ephemeral port allows the operating system to track the specific connection and return traffic to the correct process.

---

# TCP Connection Termination

After the HTTP data exchange completed, TCP began gracefully terminating the connection.

The following Wireshark display filter was used:

```text
tcp.flags.fin == 1
```

This revealed FIN/ACK packets traveling in both directions.

The observed traffic included:

```text
Client High Port ── FIN, ACK ──► Server Port 80

Client High Port ◄─ FIN, ACK ─── Server Port 80
```

The direction of the ports helped identify the client and server sides of the conversation.

<img width="1024" height="768" alt="4 3tcp-fin-ack" src="https://github.com/user-attachments/assets/9a5fd450-62bf-4a62-8a14-bf8d00683b67" />

---

# FIN and ACK Flags

The FIN and ACK flags perform different functions.

## FIN

FIN stands for **Finish**.

It indicates that a system has finished sending data and wants to close its side of the TCP connection.

Conceptually:

```text
"I am finished sending data."
```

## ACK

ACK stands for **Acknowledgment**.

It indicates that TCP is acknowledging previously received data or connection state.

Therefore, a packet containing:

```text
FIN, ACK
```

can conceptually mean:

```text
"I acknowledge what I received, and I am finished sending on my side."
```

---

# Why FIN Appears From Both Sides

TCP communication is bidirectional.

Each side can independently finish sending data.

Therefore, both the client and server may send FIN packets while gracefully terminating the session.

A simplified termination process looks like:

```text
Client                             Server

FIN, ACK ─────────────────────────►
"I'm finished sending."

         ◄──────────────── FIN, ACK
                    "I'm finished too."

         CONNECTION TERMINATED
```

The exact packet sequence may contain additional ACK packets because each side's TCP state is managed independently.

---

# Complete TCP/HTTP Session Lifecycle

Combining Labs 03 and 04 demonstrates the lifecycle of a TCP-based HTTP connection.

## Stage 1 — Connection Establishment

```text
Client                             Server

SYN ──────────────────────────────►

     ◄──────────────────── SYN-ACK

ACK ──────────────────────────────►

       CONNECTION ESTABLISHED
```

The TCP three-way handshake establishes communication state between the systems.

---

## Stage 2 — Application Data Transfer

```text
Client                             Server

HTTP GET / ───────────────────────►

           ◄────────── HTTP 200 OK
                        + HTML
```

The client requests the root web resource.

The server successfully processes the request and returns the requested content.

---

## Stage 3 — Connection Termination

```text
Client                             Server

FIN, ACK ─────────────────────────►

         ◄──────────────── FIN, ACK

          CONNECTION CLOSING
```

The systems indicate that they are finished sending data and gracefully close the TCP session.

---

# Full Communication Model

The entire process can be summarized as:

```text
             TCP HANDSHAKE
                  │
                  ▼
          SYN → SYN-ACK → ACK
                  │
                  ▼
        CONNECTION ESTABLISHED
                  │
                  ▼
           HTTP GET REQUEST
                  │
                  ▼
           HTTP 200 RESPONSE
                  │
                  ▼
             HTML CONTENT
                  │
                  ▼
          FIN / ACK EXCHANGE
                  │
                  ▼
          CONNECTION CLOSED
```

This demonstrates how multiple protocol layers work together during normal web communication.

---

# Security Relevance

Understanding the normal lifecycle of TCP and HTTP communication is important when analyzing network traffic.

Security analysts may inspect this traffic to identify:

- Suspicious HTTP requests
- Unexpected outbound web connections
- Connections to malicious infrastructure
- Unusual destination ports
- Failed or incomplete TCP sessions
- Unexpected connection resets
- Abnormal connection durations
- Suspicious data transfers
- Command-and-control traffic
- Cleartext application data

Because HTTP does not encrypt its application data, packet captures may expose information contained within HTTP requests and responses.

This is one reason HTTPS is preferred for modern web communication.

---

# HTTP vs HTTPS

The traffic in this lab intentionally used HTTP.

```text
HTTP
TCP Port 80
Application data can be visible in packet captures
```

HTTPS adds encryption using TLS.

```text
HTTPS
TCP Port 443
HTTP application content is encrypted by TLS
```

With HTTPS, a network analyst can still observe certain network and connection metadata, but the HTTP request and response content itself is normally protected by encryption.

A later lab will examine encrypted HTTPS/TLS traffic.

---

# Key Findings

During this lab, I observed that:

1. TCP establishes the transport connection before HTTP application data is exchanged.
2. Win11Pro generated an HTTP GET request for the root resource of `example.com`.
3. The web server returned an HTTP `200 OK` response.
4. The HTTP response contained the requested HTML content.
5. HTTP used TCP port 80 on the server side.
6. The client used an ephemeral high-numbered port.
7. FIN indicates that a system has finished sending data on its side of the connection.
8. ACK acknowledges previously received TCP information.
9. FIN/ACK packets were observed in both directions during connection termination.
10. Wireshark can be used to follow a TCP connection from establishment through application data transfer and termination.

---

# Skills Practiced

- Wireshark packet capture
- HTTP traffic analysis
- TCP session analysis
- HTTP GET request analysis
- HTTP status code analysis
- TCP port analysis
- Ephemeral port identification
- TCP FIN flag analysis
- TCP ACK flag analysis
- TCP connection termination
- PowerShell `curl.exe`
- Application-layer protocol analysis
- Network security analysis

---

## Lab Result

**PASS — Successfully captured and analyzed HTTP application data and TCP connection termination using Wireshark.**
