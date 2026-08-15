# Management firewall rules

The Management network (`192.168.10.0/24`) contains the administrative workstation used to manage the OPNsense firewall, Wazuh SIEM, and infrastructure services.

This security zone is intentionally treated as a **privileged administrative network**, with explicit access only to authorized management services.

## Security objectives

The Management network was designed with the following goals:

* administrative access to the firewall,
* secure management of DMZ services,
* access to the Wazuh SIEM,
* internet connectivity for administration,
* and controlled communication with infrastructure services.

## Management security model

The Management zone operates under a **least-privilege model**.

Administrative systems are allowed to access only the services required for infrastructure management, while unnecessary access to the DMZ is restricted.

## Rule summary

| Priority | Action | Source     | Destination  | Ports         | Purpose                        |
| -------- | ------ | ---------- | ------------ | ------------- | ------------------------------ |
| 1        | Pass   | Management | WAZUH_SERVER | 1514, 1515    | Wazuh agent communication      |
| 2        | Pass   | Management | DMZ          | 22, 443, 5601 | Administrative access          |
| 3        | Pass   | Management | Internet     | Any           | Administrative internet access |
| 4        | Pass   | Management | Internet     | ICMP          | Connectivity testing           |
| 5        | Block  | Management | DMZ          | Any           | Block unauthorized DMZ access  |

## Administrative access

Management hosts are explicitly permitted to access DMZ management services through the `MANAGEMENT_PORTS` alias.

This includes:

* SSH (22),
* HTTPS (443),
* and the Wazuh web interface (5601).

Restricting administrative access through a dedicated alias simplifies maintenance and future expansion.

## Wazuh communication

A dedicated rule allows communication with the Wazuh Manager on ports **1514 and 1515**.

This rule is required for:

* agent registration,
* agent authentication,
* and log transmission.

Without this rule, Management endpoints would be unable to communicate with the SIEM.

## Internet access

Management hosts require internet connectivity for:

* software updates,
* package installation,
* security research,
* and infrastructure administration.

ICMP is explicitly permitted from Management to support troubleshooting and connectivity validation.

## Default deny behavior

The final blocking rule prevents unrestricted communication from Management to the DMZ.

Only explicitly authorized services remain accessible.

This prevents administrative workstations from having unnecessary network access to DMZ systems and reinforces the principle of least privilege.

## Security considerations

Although Management is a trusted network, it is **not unrestricted**.

Administrative networks frequently become high-value targets during post-compromise activity. Restricting Management access to only essential services reduces lateral movement opportunities and limits the impact of workstation compromise.

## Validation

Management connectivity was validated by confirming:

* successful access to the Wazuh web interface,
* successful Wazuh agent communication,
* administrative access to DMZ services,
* internet connectivity,
* ICMP functionality,
* and blocking of unauthorized DMZ connections.

These rules establish the Management network as a controlled administrative zone rather than a fully trusted network.
