# Lab 08 — pfSense Firewall Rule Fundamentals

## Objective

The objective of this lab was to configure, test, and analyze a custom pfSense firewall rule.

The lab demonstrated how pfSense evaluates firewall traffic based on:

- Action
- Protocol
- Source
- Destination
- Rule order
- First-match behavior
- Firewall logging

A custom firewall rule was created to block ICMP traffic originating from the Windows 11 workstation. The rule was then tested and verified using connectivity testing and pfSense firewall logs.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| pfSense | Router / Firewall / Default Gateway | 192.168.50.1 |
| Win11Pro | Windows Client | 192.168.50.101 |
| DC-01 | Active Directory / DNS Server | 192.168.50.10 |
| Kali Linux | Security Workstation | 192.168.50.104 |

**LAN Network:** `192.168.50.0/24`  
**Virtual Network:** `SEC-LAB`

---

## Existing LAN Firewall Rules

Before creating a custom rule, the existing pfSense LAN rules were examined.

The primary IPv4 rule was:

```text
Default allow LAN to any rule
```

The rule contained values similar to:

```text
Action:       Pass
Protocol:     IPv4
Source:       LAN subnets
Source Port:  *
Destination:  *
Dest. Port:   *
```

This can be interpreted as:

> Allow IPv4 traffic originating from the LAN subnet to any destination.

The `*` values represent any applicable value.

Because the SEC-LAB network is:

```text
192.168.50.0/24
```

systems on this network can match the broad default allow rule unless a higher-priority rule matches first.

---

## pfSense Rule Processing

An important concept demonstrated during the lab was pfSense rule order.

Interface firewall rules are evaluated from:

```text
TOP
 ↓
 ↓
BOTTOM
```

When traffic matches a rule, that rule's action is applied.

This creates a **first-match-wins** model.

For example:

```text
1. BLOCK Win11Pro ICMP
2. ALLOW LAN → ANY
```

ICMP traffic originating from Win11Pro matches Rule #1 first.

Therefore:

```text
Traffic
   |
   v
Rule #1
MATCH
   |
   v
BLOCK
   |
   v
STOP
```

The broad allow rule underneath is not used for that traffic.

---

## Custom Firewall Rule

A custom rule was created on the pfSense LAN interface.

The configuration was:

```text
Action:       Block
Protocol:     ICMP
Source:       192.168.50.101
Destination:  Any
Logging:      Enabled
Description:  LAB08 - Block Win11 ICMP
```

The source address `192.168.50.101` represented Win11Pro.

The rule was positioned above:

```text
Default allow LAN to any rule
```

This ensured that matching ICMP traffic from Win11Pro encountered the custom block rule before reaching the broad allow rule.

---

## Source and Destination Logic

The custom rule specifically identified:

```text
Source:
192.168.50.101

Destination:
Any
```

Therefore, it applied to matching ICMP traffic originating **from Win11Pro**.

For example:

```text
Win11Pro
192.168.50.101
      |
      | ICMP
      v
8.8.8.8
```

matched the rule.

However, traffic originating from another host such as Kali:

```text
Kali
192.168.50.104
      |
      | ICMP
      v
8.8.8.8
```

would not match the custom rule because the source IP was different.

pfSense could therefore continue evaluating later rules, including the Default Allow LAN rule.

This demonstrated that firewall rules can be scoped to specific systems rather than applying to an entire subnet.

---

## Firewall Test

After saving and applying the custom firewall rule, Win11Pro attempted to ping:

```text
8.8.8.8
```

using:

```powershell
ping 8.8.8.8
```

The result was:

```text
Request Timed Out
```

The traffic path was effectively:

```text
Win11Pro
192.168.50.101
      |
      | ICMP Echo Request
      v
pfSense
      |
      | LAB08 - Block Win11 ICMP
      |
      X
    BLOCK
```

The ICMP traffic matched the custom rule before reaching the Default Allow rule.

---

## Firewall Log Verification

A failed ping alone does not identify why communication failed.

A timeout could potentially result from several conditions, including:

- Firewall filtering
- Routing problems
- Destination availability
- Packet loss
- Network path problems

Because firewall logging was enabled on the custom rule, the pfSense firewall logs were inspected.

The logs showed blocked traffic matching:

```text
Source:       192.168.50.101
Destination:  8.8.8.8
Protocol:     ICMP
Action:       Block
```

Four blocked entries corresponding with the ping attempts were observed.

### Evidence

<img width="1024" height="768" alt="1 1-pfSense-blocked-icmp-log" src="https://github.com/user-attachments/assets/9031d377-9fda-4c39-93bc-374cfef0fce0" />

This provided direct evidence that pfSense was blocking the ICMP traffic.

---

## Control Test

To verify that the custom firewall rule caused the failed connectivity test, the rule was disabled.

The configuration effectively became:

```text
LAB08 - Block Win11 ICMP
DISABLED

        ↓

Default allow LAN to any rule
ENABLED
```

The same command was then executed again from Win11Pro:

```powershell
ping 8.8.8.8
```

This time, ICMP Echo Replies were successfully received.

Conceptually:

```text
Win11Pro
      |
      | ICMP
      v
pfSense
      |
      | Custom Block Rule
      | DISABLED
      v
Default Allow LAN → Any
      |
      | PASS
      v
8.8.8.8
      |
      | Echo Reply
      v
Win11Pro
```

Changing only the firewall rule state and observing restored connectivity provided additional evidence that the custom rule was responsible for the earlier timeout.

---

## Blocked vs Allowed Comparison

| Configuration | Result |
|---|---|
| Custom ICMP Block Enabled | Ping failed |
| Firewall Logs | ICMP block confirmed |
| Custom ICMP Block Disabled | Ping succeeded |

This demonstrated how firewall configuration directly affected network behavior.

---

## Why the Default Allow Rule Did Not Override the Block

The broad rule:

```text
Default allow LAN to any rule
```

remained enabled during the initial test.

However, it did not override the custom block rule because the custom rule appeared first.

The processing sequence was:

```text
Win11Pro ICMP Traffic
        |
        v
LAB08 Block Rule
        |
     MATCH
        |
        v
     BLOCK
        |
       STOP
```

The Default Allow rule was never reached for matching traffic.

This demonstrated why firewall rule order is critical.

---

## Security Relevance

Firewall rules are a fundamental network security control.

They allow administrators and security engineers to define which systems may communicate, which protocols may be used, and which destinations or services may be accessed.

Understanding rule processing is also important during incident investigation.

For example, observing:

```text
Request Timed Out
```

does not prove that a firewall caused the failure.

Firewall logs provide additional evidence that can identify whether traffic was explicitly permitted or denied.

This distinction helps analysts avoid making conclusions based only on symptoms.

---

## Key Takeaways

1. pfSense firewall rules can control traffic based on protocol, source, destination, and other conditions.
2. Rules on an interface are evaluated from top to bottom.
3. The first matching rule determines how matching traffic is handled.
4. A specific block rule can take precedence over a broad allow rule when positioned above it.
5. `Source` identifies where matching traffic originates.
6. `Destination` identifies where matching traffic is headed.
7. `Any` or `*` represents an unrestricted value for that field.
8. Firewall logging provides evidence of traffic being blocked.
9. A network timeout alone does not prove that a firewall caused the failure.
10. Disabling the custom rule and repeating the same test provided a useful control comparison.
11. Firewall rules can target an individual host without blocking other systems on the subnet.
12. ICMP traffic can be controlled independently from TCP and UDP traffic.

---

## Lab Result

**PASS**

A custom pfSense firewall rule was successfully created to block ICMP traffic originating from Win11Pro.

The rule's behavior was verified through:

- ICMP connectivity testing
- Firewall rule ordering
- pfSense firewall logs
- A control test after disabling the rule

The lab demonstrated how pfSense uses source, destination, protocol, action, and rule order to make firewall decisions.
