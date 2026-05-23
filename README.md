# Home-Lab-Setup
Personal cybersecurity home lab — multi-OS, dual firewall, SIEM integration
> hands-on learning in detection, offense/defense, and SIEM integration.

## Lab Diagram
![Home Lab Setup](diagrams/home_lab_Setup.gif)

## Components
| Component         | Role                          |
|-------------------|-------------------------------|
| Kali Linux        | Attack machine (red team)     |
| Windows Server 22 | Target / AD environment       |
| Ubuntu Server     | Linux target / log forwarding |
| Windows 10        | Endpoint simulation           |
| Ubuntu Desktop    | Linux desktop target          |
| OPNsense          | Firewall (Host A side)        |
| pfSense           | Firewall (Host B side)        |
| Splunk / ELK      | SIEM (coming soon)            |

## Status
- [x] Network topology designed
- [x] Dual firewall configured
- [ ] SIEM integration (Splunk + ELK)
- [ ] Attack scenarios documented