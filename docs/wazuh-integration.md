# Wazuh Integration

## Overview

The laboratory integrates OPNsense with Wazuh to provide centralized security monitoring of firewall activity.

OPNsense generates firewall events through `filterlog`. These events are forwarded to the Wazuh Manager using Syslog over UDP port `5514`.

The Wazuh Manager then processes the received events and applies custom detection logic.

The resulting architecture is:

```text
OPNsense
    |
    | filterlog
    v
Syslog / UDP 5514
    |
    v
rsyslog
    |
    v
Wazuh Manager
    |
    v
Wazuh Analysis Engine
    |
    v
Custom Detection Rule
```

This integration allows firewall-level activity to be monitored centrally alongside endpoint security telemetry.

---

# Wazuh Manager

The Wazuh Manager is located in the DMZ:

```text
192.168.20.2
```

It acts as the central security monitoring component of the laboratory.

The Manager receives telemetry from:

* Wazuh agents.
* OPNsense Syslog.
* Security-related firewall events.

The environment therefore combines endpoint monitoring with network-security telemetry.

---

# Why Syslog Was Used for OPNsense

The other endpoints in the laboratory use Wazuh agents.

OPNsense, however, is a FreeBSD-based firewall appliance and is not treated like a conventional Linux or Windows endpoint.

For this reason, the firewall logs are forwarded using Syslog instead of installing a Wazuh agent directly on OPNsense.

The resulting architecture is:

```text
Linux / Windows endpoints
        |
        | Wazuh Agent
        v
Wazuh Manager

OPNsense
        |
        | Syslog
        v
Wazuh Manager
```

This allows OPNsense to remain responsible for network enforcement while Wazuh provides centralized analysis.

---

# OPNsense Syslog Configuration

OPNsense is configured to forward firewall events to the Wazuh Manager.

The destination is:

```text
Host: 192.168.20.2
Port: 5514
Protocol: UDP
Application: filterlog
```

The important event source is `filterlog`.

This provides visibility into firewall decisions and network traffic matching logged firewall rules.

A screenshot of the OPNsense Syslog target configuration is included in the project documentation:

```text
docs/images/07-syslog-settings.png
```

---

# Syslog Receiver

The Wazuh Manager uses `rsyslog` as the Syslog receiver.

The receiver listens on:

```text
UDP 5514
TCP 5514
```

The custom configuration used for the laboratory is stored in:

```text
configs/rsyslog/opnsense-5514.conf
```

The relevant configuration is:

```text
module(load="imudp")
module(load="imtcp")

input(type="imudp" port="5514")
input(type="imtcp" port="5514")

action(type="omfile" file="/var/log/opnsense.log")
```

This configuration performs three main functions:

1. Loads UDP Syslog input support.
2. Loads TCP Syslog input support.
3. Writes received events to `/var/log/opnsense.log`.

The dedicated log file makes troubleshooting and Wazuh integration easier.

---

# Why a Dedicated Port Was Used

The standard Syslog port is commonly associated with UDP `514`.

The project instead uses:

```text
5514
```

This separates the OPNsense integration from other potential Syslog services and avoids conflicts with existing logging configurations.

The custom port also makes it easier to identify the traffic during troubleshooting.

For example:

```bash
tcpdump -ni any port 5514
```

can be used to verify whether Syslog traffic is reaching the Wazuh Manager.

---

# Troubleshooting the Syslog Pipeline

The Syslog integration required validation at multiple layers.

The complete troubleshooting process followed the path of the log:

```text
OPNsense
   |
   | 1. Generate event
   v
Network
   |
   | 2. UDP 5514
   v
Wazuh Manager
   |
   | 3. rsyslog
   v
/var/log/opnsense.log
   |
   | 4. Wazuh localfile
   v
Wazuh Analysis
   |
   v
Alert
```

This approach makes it possible to isolate problems at each stage.

---

## Packet-Level Validation

The Wazuh Manager was monitored using:

```bash
sudo tcpdump -ni any port 5514
```

This confirms whether OPNsense is actually transmitting Syslog packets.

Packet capture was particularly useful because a successful TCP/UDP port test alone does not prove that OPNsense is forwarding the expected logs.

---

## Socket Validation

The Syslog listener was checked on the Wazuh Manager using:

```bash
sudo ss -lunpt | grep 5514
```

This verifies that `rsyslogd` is actively listening on the configured port.

The process itself can also be checked using:

```bash
ps aux | grep rsyslog
```

---

## Log File Validation

The receiver writes OPNsense events to:

```text
/var/log/opnsense.log
```

The file can be monitored using:

```bash
sudo tail -f /var/log/opnsense.log
```

This verifies that packets received by `rsyslog` are actually being written to disk.

---

# Wazuh Local File Configuration

The Wazuh Manager monitors the OPNsense log file using a `<localfile>` configuration.

The relevant configuration is located in:

```text
/var/ossec/etc/ossec.conf
```

The OPNsense log source is configured as a Syslog-type local file.

Conceptually:

```xml
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/opnsense.log</location>
</localfile>
```

This tells Wazuh to read the OPNsense log file and process each entry through the Wazuh analysis engine.

---

# Configuration Validation

Before restarting the Wazuh Manager, the configuration can be validated with:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

A successful validation confirms that the Wazuh analysis configuration does not contain syntax errors.

This is preferable to blindly restarting the service after modifying `ossec.conf`.

---

# Filterlog Events

OPNsense's `filterlog` contains structured information about firewall activity.

Depending on the rule and traffic, events can include:

* Interface.
* Action.
* Direction.
* Protocol.
* Source IP.
* Source port.
* Destination IP.
* Destination port.
* IP version.
* Firewall rule information.

For example, a blocked connection may contain information similar to:

```text
action: block
protocol: TCP
source: 192.168.30.x
destination: 192.168.10.x
```

The exact fields depend on the event generated by OPNsense.

This structured information is what allows Wazuh to identify security-relevant firewall events.

---

# Custom Wazuh Detection Rule

A custom Wazuh rule was created for the project.

The rule ID is:

```text
100100
```

The rule is designed to identify the relevant OPNsense firewall event and elevate it into a security alert.

The custom rule is stored in:

```text
configs/wazuh/local_rules.xml
```

This demonstrates that the project does not simply collect logs but also implements custom detection logic.

---

# Detection Pipeline

The complete OPNsense-to-Wazuh detection chain is:

```text
             Network Event
                   |
                   v
             OPNsense PF
                   |
                   v
              filterlog
                   |
                   v
          Syslog UDP 5514
                   |
                   v
                rsyslog
                   |
                   v
       /var/log/opnsense.log
                   |
                   v
            Wazuh localfile
                   |
                   v
         Wazuh Analysis Engine
                   |
                   v
             Rule 100100
                   |
                   v
             Security Alert
```

This is one of the main security workflows demonstrated by the project.

---

# Firewall Logging Requirements

For a firewall event to reach Wazuh, several conditions must be satisfied.

### 1. The traffic must match a relevant firewall rule.

### 2. Logging must be enabled for that rule.

### 3. OPNsense must generate the corresponding `filterlog` event.

### 4. OPNsense must forward the event to `192.168.20.2:5514`.

### 5. `rsyslogd` must receive the event.

### 6. The event must be written to `/var/log/opnsense.log`.

### 7. Wazuh must monitor the file.

### 8. The event must match an applicable Wazuh rule.

If any stage fails, the event may not appear as a Wazuh alert.

This layered model was also useful during troubleshooting.

---

# Testing the Integration

The integration was tested using both synthetic and real firewall events.

A synthetic Syslog message can be generated for basic receiver testing:

```bash
logger -n 127.0.0.1 -P 5514 "TEST OPNsense syslog"
```

Network-level verification can then be performed using:

```bash
sudo tcpdump -ni any port 5514
```

Real OPNsense firewall events were subsequently used to validate the complete pipeline.

---

# Real Firewall Event Validation

The final validation used a real blocked traffic event.

For example:

```text
Employees
    |
    | blocked traffic
    v
OPNsense
```

The firewall generated a `filterlog` event.

The event was forwarded to Wazuh and processed by the analysis engine.

This provided a realistic validation rather than relying exclusively on manually generated test messages.

---

# Relationship Between OPNsense and Wazuh

The two platforms have different responsibilities.

### OPNsense

Responsible for:

* Traffic enforcement.
* Network segmentation.
* Connection state tracking.
* Automatic source blocking.
* Firewall logging.

### Wazuh

Responsible for:

* Centralized event collection.
* Log analysis.
* Detection rules.
* Alert generation.
* Security monitoring.

The architecture therefore follows a useful separation of responsibilities:

```text
OPNsense
    =
Prevention + Enforcement

Wazuh
    =
Detection + Monitoring
```

---

# Security Benefits

The integration provides several advantages.

### Centralized Visibility

Firewall events can be investigated from the SIEM instead of requiring manual inspection of the firewall.

### Detection Engineering

Custom Wazuh rules can identify specific firewall behaviors.

### Correlation

Firewall events can potentially be correlated with endpoint events from Wazuh agents.

### Incident Investigation

Historical firewall events provide additional context when investigating suspicious activity.

### Separation of Functions

The firewall performs immediate enforcement while Wazuh performs centralized analysis.

---

# Limitations

The implementation uses Syslog over UDP.

UDP provides low overhead but does not guarantee delivery.

For a production environment, additional considerations could include:

* Reliable Syslog transport.
* TLS-protected Syslog.
* Dedicated log aggregation.
* High availability.
* Persistent log storage.
* Centralized time synchronization.
* Additional Wazuh correlation rules.

The laboratory intentionally keeps the architecture simple enough to demonstrate the complete security workflow.

---

# Future Improvements

Possible improvements include:

* TLS-secured Syslog.
* Dedicated Wazuh decoders for OPNsense.
* More granular detection rules.
* Correlation between firewall and endpoint telemetry.
* Automated Wazuh active response.
* Suricata integration.
* GeoIP or threat-intelligence enrichment.
* Automated incident-response workflows.

---

# Summary

The OPNsense-Wazuh integration transforms firewall events into centrally monitored security telemetry.

The complete workflow is:

```text
Firewall Event
      |
      v
OPNsense filterlog
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
      |
      v
Security Alert
```

This demonstrates the integration of network security enforcement with SIEM-based monitoring and detection engineering.
