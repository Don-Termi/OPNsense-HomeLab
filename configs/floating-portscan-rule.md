# Floating Rule — Port Scan Detection

## Overview

The OPNsense firewall uses a floating rule with PF state tracking to detect an excessive rate of new TCP connections.

The purpose of this mechanism is to identify behavior consistent with network reconnaissance or port scanning and automatically place the source IP address into a PF overload table named:

```text
PortScan
```

Once an address is present in this table, a separate floating block rule prevents further traffic from that source.

This creates an automated detection and response mechanism directly at the firewall layer.

---

## Detection Architecture

The mechanism consists of two complementary floating rules:

```text
             New TCP connections
                     |
                     v
        +-------------------------+
        |  Port Scan Detection    |
        |  PF State Tracking      |
        +------------+------------+
                     |
                     | threshold exceeded
                     v
             +---------------+
             | PortScan      |
             | PF Table      |
             +-------+-------+
                     |
                     v
             +---------------+
             | Block Rule    |
             +---------------+
                     |
                     v
             Traffic blocked
```

The first rule detects excessive connection creation.

The second rule checks whether the source IP exists in the `PortScan` table and blocks the traffic.

---

## PortScan PF Table

The overload table is:

```text
PortScan
```

The table contains IP addresses that have exceeded the configured connection threshold.

It can be inspected directly from the OPNsense shell:

```bash
pfctl -t PortScan -T show
```

An address can be removed manually with:

```bash
pfctl -t PortScan -T delete <IP>
```

The complete table can be flushed during laboratory testing with:

```bash
pfctl -t PortScan -T flush
```

> These commands are intended for controlled laboratory testing and troubleshooting.

---

## Detection Rule

The detection rule uses PF state tracking and connection-rate limits.

The relevant parameters are:

| Parameter               | Purpose                                              |
| ----------------------- | ---------------------------------------------------- |
| Keep State              | Maintains state information for matching connections |
| Max States              | Limits the number of states associated with the rule |
| Max Source Nodes        | Limits tracked source nodes                          |
| Max Source States       | Limits states associated with each source            |
| Max Source Connections  | Limits concurrent source connections                 |
| Max New Connections (c) | Limits new client-side connections                   |
| Max New Connections (s) | Limits new server-side connections                   |
| Overload Table          | Adds offending sources to `PortScan`                 |

The exact thresholds are intentionally configurable because overly aggressive values can cause legitimate users to be blocked.

---

## Laboratory Validation

During testing, the thresholds were temporarily reduced to:

```text
Max New Connections (c): 2
Max New Connections (s): 2
Time window: 5 seconds
```

These values were used specifically to make the detection mechanism easy to reproduce in the isolated laboratory environment.

A high rate of TCP connection attempts from the test workstation caused the source IP to be added to:

```text
PortScan
```

The subsequent block rule then prevented further traffic from that address.

The resulting behavior confirmed that the complete PF detection and response chain was operational.

---

## Why the Thresholds Were Reduced

A threshold such as `2 new connections / 5 seconds` is **not suitable as a general production configuration**.

Modern applications and web browsers routinely create multiple simultaneous connections.

Using such a low threshold in a real environment could therefore result in false positives and accidental blocking of legitimate users.

The lower threshold was used only during controlled validation.

For normal operation, the threshold should be tuned according to:

* Expected user behavior.
* Number of concurrent applications.
* Server response patterns.
* Normal connection rates.
* Network size.
* Security requirements.

The final configuration should prioritize a balance between detection sensitivity and false-positive reduction.

---

## Management Network Consideration

During validation, the Management network was temporarily included in the detection rule so that the mechanism could be tested directly from the administrator workstation.

This resulted in the Management workstation being successfully added to the `PortScan` table.

After validating the mechanism, the Management network can be excluded from automatic port-scan blocking where appropriate.

This prevents an administrator from accidentally locking themselves out of the firewall or management infrastructure during normal administrative activity.

This distinction is important:

```text
Laboratory validation
        |
        v
Aggressive threshold
        |
        v
Easy reproduction
```

versus:

```text
Operational configuration
        |
        v
Tuned threshold
        |
        v
Reduced false positives
```

---

## Automatic Blocking

The second floating rule uses the `PortScan` table as its source.

Conceptually:

```text
IF source IP ∈ PortScan
THEN
    BLOCK traffic
    LOG event
```

This rule operates independently from the connection-rate detection mechanism.

That separation provides a clean two-stage architecture:

1. **Detection** — identify excessive connection behavior.
2. **Response** — block addresses already identified as abusive.

---

## Rule Ordering

The floating rules use the `Quick` option.

This is important because PF normally evaluates rules according to its rule-processing behavior. A quick rule terminates further evaluation when it matches.

The block rule therefore takes precedence when a source has already been identified as malicious or abusive.

The intended flow is:

```text
Traffic
   |
   v
Is source in PortScan?
   |
  YES --------------------> BLOCK
   |
   NO
   |
   v
Continue normal firewall evaluation
```

---

## Logging

The relevant rules have logging enabled.

This allows blocked traffic to generate OPNsense `filterlog` events.

The events can then be forwarded to the Wazuh Manager through Syslog:

```text
OPNsense
192.168.20.1
      |
      | UDP 5514
      v
Wazuh Manager
192.168.20.2
```

This means the firewall performs the immediate enforcement while Wazuh provides centralized visibility and detection.

---

## Wazuh Integration

The OPNsense `filterlog` event generated by the blocking rule is processed by Wazuh.

A custom Wazuh rule was created with ID:

```text
100100
```

The custom rule is designed to identify the relevant OPNsense firewall event and generate a higher-level security alert.

This creates the following detection pipeline:

```text
TCP connection burst
        |
        v
OPNsense PF
        |
        v
PortScan table
        |
        v
Traffic blocked
        |
        v
filterlog
        |
        v
Syslog :5514
        |
        v
Wazuh
        |
        v
Rule 100100
```

---

## Security Benefits

This mechanism provides several useful security controls:

### Automated Response

The firewall can automatically block a source without administrator intervention.

### Reduced Exposure

Repeated connection attempts are stopped at the network perimeter.

### Centralized Monitoring

Blocking events are forwarded to Wazuh.

### Stateful Detection

The mechanism uses PF state information rather than simply counting packets.

### Defence in Depth

The port-scan control complements the standard inter-zone firewall policies.

---

## Limitations

This mechanism should not be considered a complete network intrusion-detection system.

A connection-rate based control can detect behavior associated with certain types of scanning, but it can also produce false positives or miss slower reconnaissance techniques.

For a production environment, this mechanism could be complemented with:

* IDS/IPS such as Suricata.
* Network-flow analysis.
* Wazuh correlation rules.
* Threat-intelligence feeds.
* Rate limiting.
* Endpoint telemetry.

The purpose of this implementation is to demonstrate automated firewall-level detection and response using native PF functionality.

---

## Validation Evidence

The following evidence should be included in the project documentation:

1. OPNsense floating detection rule.
2. OPNsense floating block rule.
3. `PortScan` table containing the test IP.
4. OPNsense `filterlog` event.
5. Syslog reception on the Wazuh Manager.
6. Wazuh custom rule `100100`.

Together, these demonstrate the complete detection-to-response workflow.
