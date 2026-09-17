# Lab 10 — Firewall Rule Specificity & Source Control

## Objective

The objective of this lab was to demonstrate how pfSense can apply different firewall policies to individual endpoints based on their source IP addresses.

Two endpoints on the same LAN were used:

- Win11Pro — `192.168.50.101`
- Kali Linux — `192.168.50.104`

Both systems attempted to send ICMP traffic to the same external destination.

A source-specific firewall rule was configured so that Win11Pro's ICMP traffic was blocked while Kali Linux remained allowed.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| pfSense | Router / Firewall / Default Gateway | 192.168.50.1 |
| Win11Pro | Windows Endpoint | 192.168.50.101 |
| Kali Linux | Security Workstation | 192.168.50.104 |

**LAN Network:** `192.168.50.0/24`  
**Virtual Network:** `SEC-LAB`

Only pfSense, Win11Pro, and Kali Linux were required for this lab.

---

## Initial Connectivity Baseline

Before applying the firewall policy, connectivity was verified from both endpoints.

### Kali Linux

The following command was executed:

```bash
ping -c 4 8.8.8.8
```

Kali successfully received ICMP Echo Replies.

```text
Kali (.104) → pfSense → 8.8.8.8
                         ALLOWED
```

### Win11Pro

Win11Pro also tested connectivity using:

```powershell
ping 8.8.8.8
```

Win11Pro successfully received ICMP Echo Replies.

The baseline therefore established:

```text
Win11Pro (.101) → 8.8.8.8 → ALLOWED
Kali     (.104) → 8.8.8.8 → ALLOWED
```

Both endpoints had working ICMP connectivity before the source-specific firewall policy was enabled.

---

## Source-Specific Firewall Rule

The previously created Lab 08 firewall rule was reused for this experiment.

The rule configuration was:

```text
Action:       Block
Protocol:     ICMP
Source:       192.168.50.101
Destination:  Any
Description:  LAB08 - Block Win11 ICMP
```

The important field for this lab was:

```text
Source: 192.168.50.101
```

This address belongs specifically to Win11Pro.

The rule therefore did not mean:

```text
Block ICMP from every LAN device
```

Instead, it meant:

```text
Block matching ICMP traffic
ONLY when the source is 192.168.50.101
```

---

## Testing Win11Pro

With the source-specific firewall rule enabled, Win11Pro executed:

```powershell
ping 8.8.8.8
```

The result was:

```text
Request Timed Out
```

Conceptually:

```text
Win11Pro
192.168.50.101
      |
      | ICMP
      v
pfSense
      |
      | Source = 192.168.50.101?
      |
     YES
      |
      v
LAB08 - Block Win11 ICMP
      |
      X
    BLOCK
```

### Evidence
<img width="1024" height="768" alt="3 1-win11-icmp-blocked" src="https://github.com/user-attachments/assets/139dd68f-5571-4241-81a8-e531bc30d10e" />


This confirmed that Win11Pro's ICMP traffic was unsuccessful while the source-specific block rule was active.

---

## Testing Kali Linux

The firewall configuration was left unchanged.

Kali Linux then executed:

```bash
ping -c 4 8.8.8.8
```

Kali successfully received ICMP Echo Replies.

Conceptually:

```text
Kali Linux
192.168.50.104
      |
      | ICMP
      v
pfSense
      |
      | Source = 192.168.50.101?
      |
      NO
      |
      v
Continue Evaluating Rules
      |
      v
Default Allow LAN to Any
      |
     MATCH
      |
      v
    ALLOW
```

### Evidence
<img width="958" height="946" alt="3 2-kali-icmp-allowed" src="https://github.com/user-attachments/assets/e5c52690-3066-4ae5-af5e-0910dc4a2823" />


Kali remained able to communicate even though Win11Pro was blocked by the firewall.

---

## Comparing the Results

Both endpoints used:

```text
Protocol:    ICMP
Destination: 8.8.8.8
Network:     192.168.50.0/24
Firewall:    pfSense
```

The major difference was their source IP addresses.

| Source | Protocol | Destination | Result |
|---|---|---|---|
| Win11Pro `192.168.50.101` | ICMP | `8.8.8.8` | Blocked |
| Kali `192.168.50.104` | ICMP | `8.8.8.8` | Allowed |

The firewall policy specifically matched:

```text
192.168.50.101
```

and therefore did not match:

```text
192.168.50.104
```

---

## Understanding Source Matching

Firewall rules can use the Source field to determine where matching traffic must originate.

For this rule:

```text
Source: 192.168.50.101
```

pfSense effectively evaluates:

```text
Does this traffic originate from 192.168.50.101?
```

For Win11Pro:

```text
Source = 192.168.50.101

MATCH
  |
  v
BLOCK
```

For Kali:

```text
Source = 192.168.50.104

NO MATCH
   |
   v
Continue evaluating firewall rules
```

This allows firewall administrators to create policies that affect individual endpoints without necessarily affecting the entire network.

---

## Host-Specific vs Subnet-Wide Rules

This lab also demonstrated the difference between specifying an individual host and specifying an entire network.

### Host-Specific Source

```text
Source: 192.168.50.101
```

This targets one specific IP address.

Conceptually:

```text
192.168.50.101 → BLOCK
192.168.50.102 → Not matched
192.168.50.103 → Not matched
192.168.50.104 → Not matched
```

### Subnet-Wide Source

A broader rule could instead specify:

```text
Source: LAN subnets
```

In this lab environment, the LAN is:

```text
192.168.50.0/24
```

A rule targeting the LAN subnet could therefore match traffic originating from multiple systems within that network.

Conceptually:

```text
192.168.50.101 ─┐
192.168.50.102 ─┤
192.168.50.103 ─┼──► Matching subnet-wide policy
192.168.50.104 ─┘
```

This demonstrates why correctly defining the Source field is important when designing firewall policies.

---

## Rule Processing

The source-specific block rule was positioned before the broad Default Allow rule.

For Win11Pro:

```text
ICMP from 192.168.50.101
          |
          v
LAB08 - Block Win11 ICMP
          |
        MATCH
          |
          v
        BLOCK
          |
         STOP
```

For Kali:

```text
ICMP from 192.168.50.104
          |
          v
LAB08 - Block Win11 ICMP
          |
       NO MATCH
          |
          v
Continue evaluating
          |
          v
Default allow LAN to any
          |
        MATCH
          |
          v
        ALLOW
```

The same firewall configuration therefore produced different outcomes based on the source of the traffic.

---

## Rule Specificity

A firewall rule can contain several matching conditions.

Examples include:

```text
Action
Protocol
Source
Source Port
Destination
Destination Port
```

Traffic must satisfy the relevant conditions before the configured action is applied.

A rule such as:

```text
Action:       Block
Protocol:     ICMP
Source:       192.168.50.101
Destination:  Any
```

is more specific than a rule that simply applies to the entire LAN.

This specificity allows administrators to create targeted security policies.

---

## Security Relevance

Source-based firewall rules are commonly used to restrict network access according to the role or trust level of a system.

Examples could include:

```text
Allow administrator workstation → management network

Allow DNS server → required DNS destinations

Block untrusted endpoint → sensitive network

Allow monitoring server → managed systems

Block specific endpoint → external service
```

The Source field allows the firewall to distinguish between systems even when they are connected to the same network.

However, IP-based rules are only one component of network security. Address assignment, segmentation, identity controls, endpoint security, and other mechanisms may also be required depending on the environment.

---

## Relationship to Previous Labs

### Lab 08

Lab 08 demonstrated:

```text
Can pfSense block ICMP traffic?
```

### Lab 09

Lab 09 demonstrated:

```text
Can pfSense block a specific TCP destination port?
```

### Lab 10

Lab 10 demonstrated:

```text
Can pfSense apply a policy to one source endpoint
without applying that same policy to another endpoint?
```

The result was:

```text
YES
```

The Source field allowed the firewall to distinguish between Win11Pro and Kali Linux.

---

## Key Takeaways

1. pfSense rules can target individual source IP addresses.
2. The Source field identifies where matching traffic originates.
3. A host-specific rule can affect one endpoint without affecting another endpoint.
4. Win11Pro and Kali can receive different firewall treatment even while using the same LAN.
5. A rule targeting `192.168.50.101` does not automatically apply to `192.168.50.104`.
6. If a rule does not match, pfSense continues evaluating subsequent rules.
7. Kali's traffic eventually matched the Default Allow LAN rule.
8. A source defined as `LAN subnets` would be broader than a single-host source.
9. Firewall rule specificity determines which traffic matches a policy.
10. Source, destination, protocol, and ports can be combined to create precise security controls.
11. Baseline testing provides a known-good state before firewall changes are made.
12. Comparing two endpoints under the same firewall configuration helps demonstrate the effect of source-based filtering.

---

## Lab Result

**PASS**

A source-specific pfSense firewall policy was successfully tested using two endpoints.

With the same firewall configuration active:

```text
Win11Pro (.101) → ICMP → 8.8.8.8 → BLOCKED
Kali     (.104) → ICMP → 8.8.8.8 → ALLOWED
```

The experiment demonstrated that the firewall's Source field can be used to selectively apply security policy to individual endpoints.

The lab reinforced practical understanding of:

- Source-based filtering
- Host-specific firewall rules
- Rule specificity
- Rule ordering
- Default allow behavior
- Baseline testing
- Comparative firewall analysis
