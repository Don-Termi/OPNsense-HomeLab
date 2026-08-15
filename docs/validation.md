# Validation and Testing

## Overview

The security controls implemented in this project were validated through practical network and security tests.

The objective was not only to configure the firewall and Wazuh integration, but to demonstrate that the implemented controls actually behaved as intended.

Validation was performed at multiple levels:

* Network connectivity.
* Firewall enforcement.
* Network segmentation.
* Stateful filtering.
* Logging.
* Syslog forwarding.
* Wazuh ingestion.
* Custom detection rules.
* Automated PortScan blocking.

The testing methodology followed the security-control lifecycle:

```text
Configuration
     |
     v
Generate Test Traffic
     |
     v
Observe Firewall Behaviour
     |
     v
Verify Logs
     |
     v
Verify Wazuh
     |
     v
Confirm Expected Security Outcome
```

---

# Test Environment

The main systems used during validation were:

| System               |             Address | Role                   |
| -------------------- | ------------------: | ---------------------- |
| Linux Mint           |      `192.168.10.2` | Management workstation |
| Wazuh Manager        |      `192.168.20.2` | SIEM                   |
| Apache + ModSecurity |      `192.168.20.3` | Web server / WAF       |
| Windows 10           |                DHCP | Employee workstation   |
| OPNsense             | Multiple interfaces | Firewall               |

The Management workstation was used extensively to validate administrative connectivity and the PortScan detection mechanism.

The Windows Employee workstation was used to validate segmentation and restricted user access.

---

# Validation Methodology

Testing was performed using a combination of:

* `ping`
* `nc`
* `tcpdump`
* `pfctl`
* `nmap`
* `systemctl`
* `journalctl`
* `tail`
* Wazuh Dashboard
* OPNsense live firewall logs

The tools were used at different stages to distinguish between:

1. Network connectivity problems.
2. Firewall policy problems.
3. Logging problems.
4. Wazuh ingestion problems.
5. Detection-rule problems.

---

# 1. Network Segmentation Validation

The first validation step was confirming that the different networks were actually isolated.

The primary security requirement was:

```text
Employees
    |
    X
    |
Management
```

A connection attempt from the Employee workstation to the Management workstation was generated.

The expected result was:

```text
Connection: BLOCKED
Firewall log: GENERATED
```

This confirmed that the firewall was enforcing the intended inter-zone restriction.

---

# 2. Management Connectivity

The Management workstation was tested against required infrastructure services.

The workstation:

```text
192.168.10.2
```

was able to reach the services required for administration and monitoring.

This included connectivity to the Wazuh Manager where explicitly permitted.

A TCP connectivity test was performed using:

```bash
nc -vz 192.168.20.2 1514
```

This was useful for verifying Wazuh agent communication.

When the corresponding firewall rule was missing or incorrectly configured, the connection failed.

After correcting the rule, the Wazuh agent successfully reconnected to the Manager.

This demonstrated that firewall policy changes directly affected service availability as expected.

---

# 3. Wazuh Agent Connectivity

The Linux Mint Management workstation already had a Wazuh agent installed.

The agent service was verified using:

```bash
systemctl status wazuh-agent
```

The service was running, but the endpoint initially appeared as disconnected in the Wazuh Dashboard.

The problem was traced to firewall policy.

Once the required Wazuh communication traffic was explicitly allowed, the endpoint appeared as connected again.

This test demonstrated an important operational principle:

> A running security agent does not guarantee connectivity to its management infrastructure.

Network-level controls must also permit the required communication.

---

# 4. OPNsense Syslog Validation

The Syslog integration was validated independently before testing Wazuh detection.

The Wazuh Manager was checked for a listening Syslog socket:

```bash
ss -lunpt | grep 5514
```

The `rsyslogd` process was also verified:

```bash
ps aux | grep rsyslog
```

The Syslog configuration listens on:

```text
UDP 5514
TCP 5514
```

OPNsense was configured to send `filterlog` events to:

```text
192.168.20.2:5514
```

---

# 5. Packet-Level Syslog Validation

Packet capture was used to verify whether OPNsense was actually transmitting Syslog traffic.

The following command was used on the Wazuh Manager:

```bash
sudo tcpdump -ni any port 5514
```

This was an important diagnostic step because a successful connection test does not necessarily mean that the expected log traffic is being generated.

The final configuration produced packets originating from the OPNsense firewall.

This confirmed that the network path was operational.

---

# 6. Syslog File Validation

The received events were written to:

```text
/var/log/opnsense.log
```

The file was monitored using:

```bash
sudo tail -f /var/log/opnsense.log
```

This confirmed whether `rsyslog` was receiving and writing the events.

The validation process therefore separated:

```text
Network reception
```

from:

```text
Log processing
```

This made troubleshooting significantly easier.

---

# 7. Wazuh Configuration Validation

After configuring the Wazuh Manager to monitor the OPNsense log file, the analysis configuration was validated using:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

The command completed successfully without configuration errors.

This confirmed that the Wazuh analysis configuration was syntactically valid before relying on it for detection.

---

# 8. Firewall Log Validation

The OPNsense live firewall logs were used to verify that security-relevant traffic was actually being logged.

For example, a blocked connection generated by an Employee workstation could be observed in OPNsense's live logging interface.

The important information included:

* Source address.
* Destination address.
* Protocol.
* Source port.
* Destination port.
* Action.
* Matching firewall rule.

This confirmed that the firewall itself was correctly generating the expected telemetry.

---

# 9. Filterlog Validation

The OPNsense `filterlog` application was enabled for Syslog forwarding.

This was important because simply configuring a generic Syslog destination does not guarantee that the desired firewall events will be forwarded.

The relevant event source was:

```text
filterlog
```

After enabling it, firewall traffic generated corresponding Syslog packets.

This allowed the project to connect the firewall decision with the centralized monitoring system.

---

# 10. PortScan Detection Validation

The PortScan mechanism was validated using a high rate of TCP connection attempts.

During laboratory testing, the threshold was intentionally reduced to:

```text
Max New Connections (c): 2
Max New Connections (s): 2
Time window: 5 seconds
```

This made the detection mechanism easy to reproduce.

The test generated sufficient connection attempts to trigger the PF overload mechanism.

The source address was then verified using:

```bash
pfctl -t PortScan -T show
```

The Management workstation appeared in the table:

```text
192.168.10.2
```

This confirmed that the detection stage was functioning.

---

# 11. Automated Blocking Validation

After the source address was inserted into `PortScan`, the corresponding floating block rule was expected to deny subsequent traffic.

The validation flow was:

```text
Connection burst
       |
       v
Threshold exceeded
       |
       v
IP added to PortScan
       |
       v
Floating block rule
       |
       v
Traffic blocked
```

This demonstrated that the detection and enforcement stages were both functioning.

---

# 12. Nmap Validation

Nmap was used during the laboratory to generate scanning behavior.

The important validation was not the Nmap output itself, but the firewall response.

The expected sequence was:

```text
Nmap scan
   |
   v
Multiple TCP connections
   |
   v
PF threshold exceeded
   |
   v
Source added to PortScan
   |
   v
Traffic blocked
```

The source IP was verified directly in the PF table.

This provided stronger evidence than simply observing the Nmap command fail.

---

# 13. Wazuh Detection Validation

The final stage was validating that the firewall event could be processed by Wazuh.

The project includes a custom Wazuh rule:

```text
Rule ID: 100100
```

The rule processes the relevant OPNsense event.

The complete expected chain was:

```text
PortScan
    |
    v
OPNsense block
    |
    v
filterlog
    |
    v
Syslog UDP/5514
    |
    v
rsyslog
    |
    v
/var/log/opnsense.log
    |
    v
Wazuh
    |
    v
Custom Rule 100100
```

This represents the final security monitoring workflow.

---

# 14. Troubleshooting Wazuh Alerts

During testing, not every firewall event immediately appeared as a Wazuh alert.

This highlighted an important distinction:

```text
Firewall event
      ≠
Wazuh alert
```

A firewall event must first:

1. Be generated.
2. Be logged.
3. Be forwarded.
4. Be received.
5. Be written to the monitored file.
6. Be parsed by Wazuh.
7. Match an applicable rule.

If the event does not match the custom detection rule, it can still exist as a log event without becoming the expected security alert.

This distinction was important when validating the custom Wazuh integration.

---

# 15. Clearing the PortScan State

During testing, the aggressive threshold occasionally caused the administrator's own workstation to be blocked.

The PF table was inspected with:

```bash
pfctl -t PortScan -T show
```

The test IP could be removed with:

```bash
pfctl -t PortScan -T delete 192.168.10.2
```

Alternatively, the complete table could be cleared:

```bash
pfctl -t PortScan -T flush
```

After clearing the state, normal connectivity could be restored.

This was useful during iterative testing of the PortScan configuration.

---

# 16. Stateful Firewall Considerations

During testing, existing PF states occasionally affected the observed behavior.

Removing or changing a firewall rule does not necessarily mean that an already-established state disappears immediately.

This can produce confusing test results.

For controlled validation, existing states were cleared when necessary so that traffic would be evaluated against the current firewall configuration.

This is an important consideration when testing stateful firewalls.

---

# 17. False-Positive Testing

The aggressive PortScan threshold demonstrated how legitimate traffic can accidentally trigger security controls.

Normal browsing traffic generated enough connections to cause blocking when the threshold was set extremely low.

This produced:

```text
Legitimate traffic
       |
       v
Threshold exceeded
       |
       v
False positive
       |
       v
Source blocked
```

This behavior was expected and demonstrated why production thresholds must be tuned carefully.

The laboratory threshold was intentionally aggressive for demonstration purposes.

---

# 18. Validation Matrix

The main validation scenarios can be summarized as follows:

| Test                              | Expected Result                                 | Result |
| --------------------------------- | ----------------------------------------------- | ------ |
| Management → required DMZ service | Allowed                                         | Pass   |
| Employee → Management             | Blocked                                         | Pass   |
| Employee → permitted web service  | Allowed                                         | Pass   |
| Wazuh agent → Manager             | Allowed                                         | Pass   |
| OPNsense → Syslog `5514`          | Allowed                                         | Pass   |
| OPNsense `filterlog` generation   | Generated                                       | Pass   |
| Syslog → `/var/log/opnsense.log`  | Written                                         | Pass   |
| Wazuh log ingestion               | Processed                                       | Pass   |
| PortScan threshold exceeded       | IP added to table                               | Pass   |
| IP in `PortScan` → traffic        | Blocked                                         | Pass   |
| Firewall block → `filterlog`      | Generated                                       | Pass   |
| Custom Wazuh detection            | Alert generated when matching event is produced | Pass   |

---

# 19. Security Control Verification

The project successfully demonstrates the following controls:

### Network Segmentation

Separate security zones enforce different access policies.

### Least Privilege

Users are prevented from accessing services that are not required for their role.

### Stateful Filtering

PF tracks connection state and applies connection limits.

### Automated Blocking

Suspicious sources can be dynamically added to the `PortScan` table.

### Centralized Logging

Firewall events are forwarded to Wazuh.

### Detection Engineering

A custom Wazuh rule processes the relevant firewall telemetry.

### Security Monitoring

Network security events can be investigated centrally.

---

# Lessons Learned

Several practical lessons emerged during implementation.

## Firewall Rules Are State-Aware

Changing a firewall rule does not necessarily invalidate existing connection states.

Testing must therefore account for state tracking.

## Connectivity and Logging Are Separate Problems

A port can be reachable while the expected application traffic is not being generated.

Packet capture was therefore essential for validating the Syslog pipeline.

## A Running Agent Can Still Be Disconnected

The Wazuh agent service can be running correctly while firewall policy prevents communication with the Manager.

## Logging Must Be Enabled at the Source

Creating a Syslog destination is not enough.

The relevant OPNsense event source, such as `filterlog`, must also be configured to generate and forward the desired events.

## Detection Thresholds Require Tuning

A highly sensitive security control can become unusable if legitimate traffic constantly triggers it.

This is particularly important for connection-rate based detection.

---

# Final Validation Result

The final implementation successfully demonstrated an end-to-end network security workflow:

```text
                    NETWORK TRAFFIC
                          |
                          v
                  OPNsense Firewall
                          |
              +-----------+-----------+
              |                       |
              v                       v
       Normal Filtering        PortScan Detection
                                      |
                                      v
                                PortScan Table
                                      |
                                      v
                                Block Traffic
                                      |
                                      v
                                  filterlog
                                      |
                                      v
                               Syslog UDP/5514
                                      |
                                      v
                                  rsyslog
                                      |
                                      v
                              Wazuh Manager
                                      |
                                      v
                              Custom Detection
```

The validation confirms that the project is not limited to configuration screenshots.

The security controls were actively tested through real network traffic, firewall state inspection, packet capture, logging validation, and SIEM monitoring.

---

# Conclusion

The laboratory demonstrates a complete practical security-engineering workflow:

**Design → Configure → Test → Detect → Block → Log → Monitor → Validate**

The combination of OPNsense, PF, Syslog, rsyslog, Wazuh, Apache, and ModSecurity provides multiple defensive layers.

More importantly, the project demonstrates the ability to troubleshoot security infrastructure when individual components do not initially behave as expected.

This validation methodology provides evidence that the implemented security controls operate as intended and forms the basis for future extensions such as Suricata IDS/IPS integration, more advanced Wazuh correlation, and automated incident response.
