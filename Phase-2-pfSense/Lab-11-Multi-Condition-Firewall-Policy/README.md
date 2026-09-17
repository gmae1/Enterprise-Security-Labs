# Lab 11 — Multi-Condition Firewall Policy

## Objective

This lab focused on creating and validating a targeted firewall policy in pfSense using multiple matching conditions.

The policy was designed to block HTTP traffic only when all of the following conditions were present:

- Protocol: TCP
- Source: Kali Linux (`192.168.50.104`)
- Destination: Any
- Destination Port: TCP/80

The lab also demonstrated what happens when traffic fails to match one of those conditions: pfSense continues evaluating subsequent firewall rules.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| pfSense | Router / Firewall | `192.168.50.1` |
| Windows 11 Pro | Client Workstation | `192.168.50.101` |
| Kali Linux | Security Workstation | `192.168.50.104` |
| SEC-LAB | Internal Network | `192.168.50.0/24` |

The external test destination used during the lab was:

```text
104.20.23.154
```

TCP port `80` was used to test HTTP connectivity.

---

## Establishing a Baseline

Before applying the new firewall policy, TCP/80 connectivity was verified from both Kali Linux and Windows 11.

### Kali Linux

Kali successfully connected to TCP/80 on the external destination.

Example command:

```bash
nc -vz example.com 80
```

The result included:

```text
(http) open
```

A reverse DNS lookup warning also appeared. This warning was independent of the TCP connection result and did not mean that the TCP connection had failed.

To prevent DNS lookups during future tests, the following command can be used:

```bash
nc -nvz -w 5 104.20.23.154 80
```

### Windows 11

The Active Directory DNS server was powered off during this lab. Because Windows 11 normally used that server for DNS, the test was performed against the destination IP address directly so DNS would not affect the result.

```powershell
Test-NetConnection 104.20.23.154 -Port 80
```

Result:

```text
TcpTestSucceeded : True
```

At this point, both Kali and Windows 11 had confirmed TCP/80 connectivity.

---

## Creating the Firewall Policy

The new rule was created under:

```text
Firewall → Rules → LAN
```

The following configuration was applied:

| Setting | Value |
|---|---|
| Action | Block |
| Protocol | TCP |
| Source | `192.168.50.104` |
| Destination | Any |
| Destination Port | HTTP / TCP 80 |
| Logging | Enabled |
| Description | `LAB11 - Block Kali HTTP` |

The custom block rule was placed above the broad default allow rule.

This ordering matters because pfSense processes interface rules from top to bottom and acts on the first rule that matches the traffic.

---

## Testing Kali Linux

After enabling the policy, Kali attempted another connection to TCP/80:

```bash
nc -vz -w 5 104.20.23.154 80
```

This time, the connection timed out.

The pfSense firewall logs were checked to determine whether the firewall had actually caused the failure rather than assuming that the timeout was firewall-related.

The logs showed traffic originating from:

```text
192.168.50.104
```

with TCP port:

```text
80
```

as the destination port.

pfSense blocked the traffic, providing firewall-side evidence that the configured policy caused the timeout.

### Evidence

<img width="1024" height="768" alt="4 1Kali-tcp80-blocked-firewall-log" src="https://github.com/user-attachments/assets/5cb30494-1a38-460b-8ef2-7ae6e9a280fa" />

---

## Testing Windows 11

The `LAB11 - Block Kali HTTP` rule remained enabled while Windows 11 tested the same destination and TCP port.

```powershell
Test-NetConnection 104.20.23.154 -Port 80
```

Result:

```text
TcpTestSucceeded : True
```

Windows 11 continued to connect successfully because its source IP was:

```text
192.168.50.101
```

The custom firewall rule required:

```text
192.168.50.104
```

as the source.

Because Windows 11 did not satisfy the source condition, the custom block rule did not match. pfSense continued evaluating the rule set until the traffic reached the existing default allow rule.

### Evidence

<img width="1024" height="768" alt="4 2-win11-tcp80-allowed" src="https://github.com/user-attachments/assets/fa1a6f38-9e7e-4853-ac49-4ec28c55ec6b" />

---

## Understanding the Rule Logic

The policy should not be interpreted simply as:

```text
Block Kali
```

Instead, its logic can be represented as:

```text
IF
    Protocol = TCP
AND Source = 192.168.50.104
AND Destination = Any
AND Destination Port = 80
THEN
    BLOCK
```

The required conditions work together. The traffic must satisfy all of them before this particular rule applies.

| Traffic | Custom Rule Match | Result |
|---|---|---|
| Kali `.104` → TCP/80 | Yes | Blocked |
| Kali `.104` → TCP/443 | No | Continues to subsequent rules |
| Windows `.101` → TCP/80 | No | Continues to subsequent rules |

This allows firewall policies to target specific types of communication without unnecessarily blocking every connection from a host.

---

## Why the Test Used an External Destination

Kali Linux, Windows 11, and DC01 all reside inside:

```text
192.168.50.0/24
```

Devices on the same subnet can normally communicate directly without sending the traffic through their default gateway.

For example:

```text
Kali 192.168.50.104
        ↓
ARP / Layer 2 communication
        ↓
DC01 192.168.50.10
```

Kali can resolve DC01's MAC address using ARP and then communicate directly with DC01 across the local network.

Because pfSense is not normally in the forwarding path for this same-subnet traffic, using DC01 would not properly test the pfSense LAN firewall policy.

The external destination is outside `192.168.50.0/24`, so Kali must send the traffic toward its default gateway:

```text
Kali
192.168.50.104
        ↓
Default Gateway
192.168.50.1
        ↓
pfSense Firewall
        ↓
Internet
        ↓
104.20.23.154
```

This places pfSense directly in the traffic path and allows the firewall to enforce the rule.

---

## Separating DNS Problems from TCP Problems

DC01 was powered off during the lab.

Windows 11 was configured to use DC01 (`192.168.50.10`) for DNS. As a result, hostname-based connectivity tests could fail because DNS was unavailable even though routing and TCP connectivity were still functioning.

Using the destination IP directly removed DNS from the test:

```powershell
Test-NetConnection 104.20.23.154 -Port 80
```

This reinforces an important troubleshooting technique:

> Test individual layers and services independently whenever possible.

A DNS resolution problem does not automatically mean that the underlying TCP connection is unavailable.

---

## Firewall Enforcement vs Logging

The custom firewall rule was configured to log matching packets.

During the first test, Kali's connection was blocked, but no corresponding firewall log appeared because packet logging had not yet been enabled on the rule.

After enabling:

```text
Log packets that are handled by this rule
```

the blocked Kali traffic became visible in the pfSense firewall logs.

This demonstrates that firewall enforcement and firewall logging are separate functions.

A rule can successfully block traffic without creating a log entry when logging is disabled.

---

## Key Concepts Demonstrated

### Rule Specificity

Firewall policies can evaluate several traffic characteristics, including:

- Source IP address
- Destination IP address
- Protocol
- Source port
- Destination port

Combining these characteristics allows administrators to create narrowly scoped access-control policies.

### Multiple Conditions

For the custom rule to apply:

```text
Source must be Kali
AND
Protocol must be TCP
AND
Destination port must be 80
```

If one required condition does not match, the rule does not apply.

### First-Match Processing

pfSense evaluates interface firewall rules from top to bottom.

Once a packet matches a rule, that rule determines how the traffic is handled.

### Routing and the Default Gateway

Traffic destined for a network outside `192.168.50.0/24` is sent to:

```text
192.168.50.1
```

which is the pfSense default gateway.

This causes the external traffic to traverse the firewall.

### Same-Subnet Traffic

Hosts on the same subnet can communicate directly at Layer 2 after resolving the appropriate MAC address through ARP.

The default gateway is not normally involved.

### Logging and Enforcement

A firewall can enforce a rule regardless of whether logging is enabled.

Logging adds evidence that administrators and security analysts can use for troubleshooting and investigations.

### DNS and TCP Are Separate

DNS name resolution and TCP connectivity are different functions.

Testing against a known IP address can help isolate whether a failure originates from DNS or from network connectivity.

---

## Security Relevance

Production firewall policies are commonly more specific than broad configurations such as:

```text
Allow everything
```

or:

```text
Block everything
```

Administrators can instead build policies around particular systems, networks, protocols, and services.

For example:

```text
Allow user subnet → DNS servers → TCP/UDP 53
Allow web servers → database servers → required database port
Block workstation subnet → management network → administrative ports
```

This type of specificity reduces unnecessary access while still permitting required communication.

Firewall logs add another layer of value by showing which traffic was evaluated and blocked, making them useful for both troubleshooting and security investigations.

---

## Lab Result

**PASS**

The multi-condition pfSense policy successfully:

- Blocked Kali Linux (`192.168.50.104`) from TCP/80
- Allowed Windows 11 (`192.168.50.101`) to reach the same destination and port
- Produced firewall logs confirming the Kali block
- Demonstrated source-specific and port-specific enforcement
- Reinforced pfSense first-match rule processing
- Demonstrated why the network path matters during firewall testing
- Distinguished DNS resolution problems from TCP connectivity problems

---

## Conclusion

Lab 11 demonstrated how pfSense can apply precise firewall controls based on several traffic characteristics at the same time.

Rather than denying every connection from Kali Linux, the policy denied only TCP/80 traffic that satisfied the configured conditions. Windows 11 remained unaffected because it did not match the source requirement.

The lab also connected several networking concepts from earlier work: ARP, Layer 2 communication, subnet boundaries, routing, default gateways, DNS, firewall placement, and logging.

Together, these concepts provide the foundation for more advanced firewall administration and network segmentation.
