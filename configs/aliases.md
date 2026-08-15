# OPNsense firewall aliases

This document describes the aliases used throughout the OPNsense firewall configuration. Aliases were created to improve rule readability, simplify maintenance, and implement reusable security policies across multiple firewall rules.

## Purpose of aliases

Instead of hardcoding IP addresses and ports into individual firewall rules, aliases provide a centralized abstraction layer.

Advantages:

* improved readability,
* easier maintenance,
* reduced configuration errors,
* reusable security objects,
* simplified future expansion.

## Network aliases

| Alias                | Value             | Description                        |
| -------------------- | ----------------- | ---------------------------------- |
| `MANAGEMENT_NETWORK` | `192.168.10.0/24` | Administrative workstation network |
| `DMZ_NETWORK`        | `192.168.20.0/24` | DMZ servers                        |
| `EMPLOYEES_NETWORK`  | `192.168.30.0/24` | Internal employee network          |

## Host aliases

| Alias          | Value          | Description          |
| -------------- | -------------- | -------------------- |
| `WAZUH_SERVER` | `192.168.20.2` | Wazuh Manager        |
| `WEB_SERVER`   | `192.168.20.3` | Apache + ModSecurity |

## Service aliases

### Management ports

| Alias              | Ports           |
| ------------------ | --------------- |
| `MANAGEMENT_PORTS` | `22, 443, 5601` |

This alias restricts administrative access to:

* SSH,
* HTTPS,
* and the Wazuh web interface.

### Wazuh agent ports

| Alias         | Ports        |
| ------------- | ------------ |
| `WAZUH_PORTS` | `1514, 1515` |

These ports allow communication between Wazuh agents and the Wazuh Manager.

## Port scan protection alias

### PortScan

| Alias      | Type                                    |
| ---------- | --------------------------------------- |
| `PortScan` | External (advanced) / PF overload table |

The `PortScan` alias is used by the floating rule responsible for automatic port scan detection.

When the configured connection threshold is exceeded, OPNsense automatically inserts the offending source IP into this table.

Traffic from any address present in the `PortScan` table is immediately blocked by a dedicated floating rule.

This mechanism provides automatic mitigation of reconnaissance activity using PF state tracking and overload tables.

## Design considerations

The alias structure was intentionally designed to support:

* least-privilege firewall policies,
* simplified rule maintenance,
* readable documentation,
* and future scalability.

By separating networks, hosts, and services into dedicated aliases, the firewall policy becomes significantly easier to audit and modify.
