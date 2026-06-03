# Home-Lab-Setup
Personal cybersecurity home lab — multi-OS, dual firewall, SIEM integration
> hands-on learning in detection, offense/defense, and SIEM integration.

---

> **Author:** Muqtaruddin
> **Lab Type:** Active Directory + SOC Detection Lab
> **Hypervisor:** Hyper-V (Windows Host)
> **Status:** 🟡 In Progress — Splunk Universal Forwarder & Attack Machine Pending

---

## 📋 Table of Contents

- [Lab Overview](#lab-overview)
- [Network Architecture](#network-architecture)
- [Virtual Machines](#virtual-machines)
- [Phase 1 — Active Directory Setup (Windows Server 2022)](#phase-1--active-directory-setup-windows-server-2022)
- [Phase 2 — Ubuntu AD Integration (Troubleshooting & Resolution)](#phase-2--ubuntu-ad-integration-troubleshooting--resolution)
- [Phase 3 — Splunk Deployment](#phase-3--splunk-deployment-in-progress)
- [Phase 4 — Attack Machine (Kali Linux)](#phase-4--attack-machine-kali-linux-pending)
- [Incident Reports](#incident-reports)
- [Pending Tasks](#pending-tasks)
- [Key Takeaways](#key-takeaways)

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



---

## Lab Overview

This home lab simulates an enterprise-grade on-premises environment for hands-on practice in:

- Active Directory administration and domain management
- Linux-Windows identity integration (Kerberos, SSSD, LDAP)
- Security monitoring and log ingestion via Splunk
- Offensive security and attack simulation using Kali Linux
- Network segmentation using pfSense and OPNsense firewalls

The environment mirrors real-world SOC and sysadmin workflows, designed to support a portfolio demonstrating both defensive and offensive security skills.

---

## Network Architecture

```
                        [ INTERNET ]
                             |
                         [ ROUTER ]
                             |
                    [ HYPER-V SWITCH ]
                    /                \
           [ OPNsense FW ]      [ pfSense FW ]
           (Internal Seg.)      (192.168.*.1)
                |                     |
        [ Kali Linux ]      ┌─────────┴──────────┐
        (Attack VM)         |                    |
                     [ WIN-Server ]     [ Windows 10 ]
                     (192.168.*.2)      (192.168.*.3)
                     [AD/DNS/KDC]
                            |
                    ┌───────┴────────┐
             [ Ubuntu-Server ]  [ Ubuntu-Desktop ]
             (192.168.*.6)       (192.168.*.5)
             [Domain Member]     [Domain Member]

HOST A (Laptop) ──── HYPER-V SWITCH ──── HOST B (Laptop)
                           |
                       [ SPLUNK ]
                    (Log Collection)
```

**Domain:** `MERCY.local`
**AD Server IP:** `192.168.*.2`

---

## Phase 1 — Active Directory Setup (Windows Server 2022)

**Status:** ✅ Completed without issues

### What Was Configured

- Promoted Windows Server 2022 to Domain Controller for `MERCY.local`
- Configured AD DS, DNS Server, and Kerberos KDC roles
- Created AD DNS Forward Lookup Zone (`MERCY.local`)
- Enabled AD-Integrated DNS zone for automatic SRV record registration
- Verified `_msdcs.MERCY.local` zone exists and contains KDC/LDAP SRV records
- Joined Windows 10 workstation to the domain successfully
- Created Organizational Units (OUs) and user accounts
- Configured Group Policy baseline

### Key Concepts Demonstrated

- Active Directory Domain Services (AD DS) promotion
- DNS zone types: Primary vs. AD-Integrated
- Kerberos Key Distribution Center (KDC) role
- Netlogon SRV record registration
- Group Policy Objects (GPO)

---

## Phase 2 — Ubuntu AD Integration (Troubleshooting & Resolution)

**Status:** ✅ Resolved after 3-day troubleshooting cycle

This phase involved integrating two Ubuntu machines (Server + Desktop) into the `MERCY.local` Active Directory domain. This was a multi-day effort that produced rich troubleshooting documentation.

---

### Day 1 — Kerberos & DNS Failures

**Core Problem:** Ubuntu could not obtain Kerberos tickets.

```
cannot contact any KDC for realm MERCY.local
```

**Root Cause:** Missing Kerberos SRV records in AD DNS — the `_msdcs.MERCY.local` zone did not exist, and the `MERCY.local` zone was not AD-Integrated, preventing Netlogon from publishing SRV records.

**Fixes Applied:**

| Area | Action |
|---|---|
| Time Sync | Installed `ntpdate`, synced with DC (192.168.1.2) |
| DNS Resolver | Disabled `systemd-resolved` stub (127.0.0.53), pointed to DC |
| `/etc/resolv.conf` | Replaced with static DC entry |
| Netplan | Persisted DNS configuration across reboots |
| AD DNS (DC-side) | Verified zone type → converted to AD-Integrated |
| AD DNS (DC-side) | Created `_msdcs.MERCY.local` zone |
| Netlogon | Ran `nltest /dsregdns`, restarted Netlogon service |

**Verification Commands:**
```bash
dig _kerberos._tcp.mercy.local SRV
dig _kerberos._tcp.dc._msdcs.mercy.local SRV
kinit administrator@MERCY.LOCAL
klist
```

---

### Day 2 — SSSD, Kerberos Config & Login Issues

**Core Problems:**
- Machine account vs. user account confusion
- Incorrect `krb5.conf` hostname/realm mapping
- Stale SSSD cache blocking identity resolution
- AD user missing valid UPN and password

**Fixes Applied:**

```bash
# Rejoin domain
sudo realm join mercy.local -U administrator

# Clear SSSD cache
sudo systemctl stop sssd
sudo rm -rf /var/lib/sss/db/* /var/lib/sss/mc/*
sudo systemctl start sssd

# Grant sudo rights to AD user
sudo visudo
# Add: UBUNTU-SER ALL=(ALL) ALL
```

**Key Lessons:**
- Machine accounts (`HOSTNAME$`) cannot be used for manual Kerberos auth
- Kerberos requires exact hostname and realm case-matching in `krb5.conf`
- SSSD caches must be cleared after any domain rejoin
- AD users must have a valid UPN set before `kinit` will succeed

---

### Day 3 — Incident Report: AD Identity Loss

**Symptom:** Previously working domain login broke intermittently.

```bash
id UBUNTU-SER          # → no such user
kinit -k HOSTNAME$     # → pre-authentication failed
sudo                   # → I'm afraid I can't do that
```

**Root Causes Identified:**

| # | Root Cause | Impact |
|---|---|---|
| 1 | Machine account trust broken | SSSD couldn't authenticate to AD |
| 2 | SSSD config missing/incorrect after rejoin | No AD users returned |
| 3 | AD user lacked sudo rights | Auth worked but no admin access |

**Final Working SSSD Config (`/etc/sssd/sssd.conf`):**

```ini
[sssd]
domains = mercy.local
config_file_version = 2
services = nss, pam

[domain/mercy.local]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad
ldap_id_mapping = True
fallback_homedir = /home/%u
default_shell = /bin/bash
use_fully_qualified_names = False
```

```bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
```

**Hardening Applied:**
```bash
# Lock DNS config from accidental overwrites
sudo chattr +i /etc/resolv.conf

# Enable NTP sync with Domain Controller
sudo apt install chrony -y
# Add to /etc/chrony.conf:
# server <DC-IP> iburst
sudo systemctl restart chrony
```

**Final State:**
- ✅ Ubuntu fully joined to `MERCY.local`
- ✅ Kerberos tickets obtained successfully
- ✅ SSSD resolving AD identities
- ✅ AD users can log in with home directory auto-creation
- ✅ Sudo access functional via sudoers

---

## Phase 3 — Splunk Deployment *(In Progress)*

**Status:** 🟡 Splunk Server installed on Host A — Universal Forwarders pending on VMs

### Planned Architecture

```
[ WIN-Server ]  ──┐
[ Windows 10 ]  ──┤  Splunk Universal      [ Splunk Server ]
[ Ubuntu-Srv  ]  ──┤  Forwarder (9997) ──►  (Host A / SIEM)
[ Ubuntu-Desk ]  ──┘
```

### Pending Tasks — Splunk

- [ ] Install Splunk Universal Forwarder on **Windows Server 2022**
- [ ] Install Splunk Universal Forwarder on **Windows 10**
- [ ] Install Splunk Universal Forwarder on **Ubuntu-Server**
- [ ] Install Splunk Universal Forwarder on **Ubuntu-Desktop**
- [ ] Configure `inputs.conf` for Windows Event Logs (Security, System, Application)
- [ ] Configure `inputs.conf` for Linux Syslog and Auth logs
- [ ] Configure `outputs.conf` on all forwarders to point to Splunk indexer
- [ ] Create Splunk indexes: `windows`, `linux`, `ad_events`
- [ ] Build detection dashboards for AD authentication events

---

## Phase 4 — Attack Machine (Kali Linux) *(Pending)*

**Status:** 🔴 VM created — attack configuration pending

### Planned Activities

- [ ] Configure Kali Linux networking (OPNsense segment)
- [ ] Verify attacker-to-internal network reachability
- [ ] Run AD enumeration with BloodHound / ldapdomaindump
- [ ] Perform credential attacks (Kerberoasting, AS-REP Roasting)
- [ ] Simulate lateral movement techniques
- [ ] Validate Splunk detection rules catch attack telemetry

---

## Incident Reports

| Report | Date | Summary | Status |
|---|---|---|---|
| Ubuntu-AD Kerberos Failure | Day 1 | Missing SRV records in AD DNS prevented KDC discovery | ✅ Resolved |
| SSSD Identity Loss | Day 2 | Machine account confusion + stale cache broke AD resolution | ✅ Resolved |
| Intermittent AD Login Failure | Day 3 | Machine trust broken + missing SSSD config after rejoin | ✅ Resolved |

Full incident documentation is available in `/docs/incident-reports/`.

---

## Pending Tasks

| Priority | Task | Phase |
|---|---|---|
| 🔴 High | Install Splunk UF on all internal VMs | Phase 3 |
| 🔴 High | Configure Kali attack machine | Phase 4 |
| 🟡 Medium | Build Splunk AD detection dashboards | Phase 3 |
| 🟡 Medium | Run BloodHound AD enumeration | Phase 4 |
| 🟢 Low | Auto-create home dirs for AD users on Linux | Phase 2 |
| 🟢 Low | Map AD groups to Linux sudoers groups | Phase 2 |
| 🟢 Low | Configure SSH login with AD credentials | Phase 2 |

---

## Key Takeaways

**Active Directory & DNS**
- AD-Integrated DNS zones are required for automatic SRV record registration via Netlogon
- `_msdcs.MERCY.local` zone must exist for Kerberos and DC locator to function
- Time synchronization (NTP) is a hard dependency for Kerberos — even a few minutes of drift causes failures

**Linux-Windows Identity Integration**
- SSSD is the enterprise-standard daemon for Linux AD integration
- Machine accounts and user accounts serve different purposes — never mix them for manual auth
- Always clear SSSD cache after domain rejoin or config changes
- `chmod 600` on `sssd.conf` is mandatory — SSSD refuses to start otherwise

**Security Operations**
- `chattr +i` on `/etc/resolv.conf` prevents accidental DNS override that can break domain auth
- Sudoers management via AD groups is the scalable enterprise approach vs individual user entries
- Log everything — SSSD and Kerberos logs are the first place to look during integration failures

---

## Tools & Technologies Used

`Windows Server 2022` `Active Directory` `Kerberos` `DNS` `SSSD` `realm` `kinit`
`Ubuntu Server` `Ubuntu Desktop` `Kali Linux` `pfSense` `OPNsense` `Hyper-V`
`Splunk` `Netplan` `chrony` `nltest` `dig` `BloodHound` *(planned)*

---

> 📁 **Docs:** `/docs/` — Incident reports, config backups, troubleshooting notes
> 🗺️ **Diagram:** `/diagrams/Home_Lab.drawio` — Full network topology
> 📅 **Last Updated:** June 2026
