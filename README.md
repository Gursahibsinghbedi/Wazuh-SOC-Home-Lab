# 🛡️ Wazuh SOC Home Lab

A self-built Security Operations Center (SOC) lab using **Wazuh** (SIEM + XDR), **Sysmon**, and simulated attacker activity across Windows and Linux endpoints — built end-to-end in VMware Workstation to practice detection engineering, log analysis, and blue-team response.

This repo documents the full build: from standing up the infrastructure, to deploying agents and Sysmon, to writing custom detection rules, configuring automated response, building SOC dashboards, and generating/analyzing real telemetry.

---

## 🎯 Objective

Simulate a small enterprise environment (1 Windows endpoint + 1 Linux endpoint monitored by a central Wazuh manager) to:

- Provision and enroll agents from scratch, including realistic setup/connectivity troubleshooting
- Deploy Sysmon with community detection-oriented configs for richer telemetry
- Detect and automatically respond to brute-force attacks
- Monitor file and registry integrity in real time
- Build custom SOC dashboards for authentication/account-activity monitoring
- Generate and interpret Windows Security, Sysmon, and SSH telemetry
- Practice log correlation, triage, and MITRE ATT&CK mapping

---

## 🖥️ Lab Architecture

| Component | Role | OS / Details |
|---|---|---|
| **wazuh-server** | Wazuh Manager + Indexer + Dashboard (SIEM) | Ubuntu 24.04 LTS, `192.168.77.129` |
| **sahib-linux** | Monitored Linux endpoint (agent `002`) | Ubuntu, `192.168.77.131` |
| **sahib-windows** | Monitored Windows endpoint (agent `001`) | Windows 10 Pro, `192.168.77.132` |
| **Attacker machine** | Simulated brute-force source | Windows host (PowerShell / RDP) |

All VMs run under **VMware Workstation Pro** on a NAT'd internal network so the endpoints can reach the manager on port `1514` (agent comms), and the dashboard is reachable at `https://192.168.77.129`.

```
                     ┌───────────────────────────────┐
                     │     wazuh-server (Manager)     │
                     │  Ubuntu 24.04 · 192.168.77.129 │
                     │  Wazuh Indexer + Dashboard      │
                     └───────────────┬─────────────────┘
                                     │ port 1514 (agent comms, TCP)
                     ┌───────────────┴─────────────────┐
                     │                                  │
        ┌────────────▼────────────┐      ┌──────────────▼─────────────┐
        │   sahib-linux (002)      │      │   sahib-windows (001)      │
        │   Ubuntu · 192.168.77.131│      │   Win10 Pro · 192.168.77.132│
        │   Wazuh Agent + Sysmon   │      │   Wazuh Agent + Sysmon     │
        └───────────────────────────┘      └─────────────────────────────┘
```

> **Note:** IPs shown are internal, NAT'd lab addresses (RFC 1918 range) with no external exposure — kept as-is here for documentation clarity.

---

## 📚 Contents

| # | Investigation | Summary |
|---|---|---|
| [00](docs/00-environment-setup.md) | **Environment Setup** | VM provisioning, Wazuh manager install, agent enrollment (Linux + Windows), connectivity troubleshooting |
| [01](docs/01-sysmon-deployment.md) | **Sysmon Deployment** | Deploying Sysmon on both endpoints with community detection configs (MSTIC-Sysmon, sysmon-modular) |
| [03](docs/03-active-response.md) | **Active Response** | Custom SSH brute-force detection rule + automated `firewall-drop` blocking |
| [04](docs/04-file-integrity-monitoring.md) | **File Integrity Monitoring** | Real-time file/registry change detection on Linux and Windows |
| [05](docs/05-soc-dashboard.md) | **SOC Dashboard** | Custom Wazuh dashboard for authentication failures + account-management activity |
| [06](docs/06-telemetry.md) | **Telemetry Generation & Analysis** | Windows Security events, Sysmon, and SSH auth log interpretation |

> Numbering above follows the logical build order — start at `00` if you want the full story, or jump straight to whichever investigation interests you.

---

## 🔧 Configs

The [`configs/`](configs/) folder contains the actual XML snippets used in this lab (sanitized of anything environment-specific beyond the private lab IPs already shown above):

- [`local_rules.xml`](configs/local_rules.xml) — custom SSH brute-force correlation rule (`100102`)
- [`ossec.conf.active-response.xml`](configs/ossec.conf.active-response.xml) — Active Response block wired to that rule
- [`ossec.conf.syscheck.xml`](configs/ossec.conf.syscheck.xml) — File Integrity Monitoring config for the Linux endpoint

---

## 🖼️ Screenshots

Full screenshot walkthroughs live under [`screenshots/`](screenshots/), organized by phase:

```
screenshots/
├── 01-deploy agents +sysmon/         Sysmon install + verification (both platforms)
├── 02-active-response/           (add SSH brute-force attack/block screenshots here)
├── 03-file-integrity-monitoring/ (add FIM event screenshots here)
├── 04-soc-dashboard/             (add dashboard build screenshots here)

```

The first three folders are populated. Folders `03`–`06` are scaffolded and ready — drop in screenshots from those investigations if/when you want the visuals alongside the write-ups (the markdown log tables already tell the full story on their own).

---

## 🧠 Skills Demonstrated (Overall)

- **SIEM administration:** Wazuh manager/agent install, service management, connectivity troubleshooting
- **Detection engineering:** custom correlation rules (`local_rules.xml`), Active Response automation
- **Endpoint monitoring:** Sysmon deployment with community detection configs, File Integrity Monitoring
- **Dashboarding & visualization:** custom Wazuh/OpenSearch Dashboards for SOC triage
- **Log analysis:** Windows Security Event Log, Sysmon, SSH auth logs, MITRE ATT&CK correlation
- **Infrastructure:** VM provisioning, network configuration, systemd service administration
- **Blue-team workflow:** attack simulation → detection → automated response → verification

---

## 🚀 Reproducing This Lab

1. Provision 3 VMs in VMware Workstation (or any hypervisor): one Ubuntu manager, one Ubuntu agent, one Windows 10 agent, all on the same internal/NAT network.
2. Follow [`docs/00-environment-setup.md`](docs/00-environment-setup.md) to install the Wazuh manager and enroll both agents.
3. Follow [`docs/01-sysmon-deployment.md`](docs/01-sysmon-deployment.md) to deploy Sysmon on both endpoints.
4. Apply the configs in [`configs/`](configs/) on the manager/agent as noted in each file's header comment.
5. Work through investigations `03`–`06` to reproduce the detection, FIM, dashboard, and telemetry exercises.

---

## 📌 Notes

- All internal IPs shown are private (RFC 1918) lab addresses with no external routing.
- This lab was built iteratively, including realistic troubleshooting (config typos, disk-space issues, agent connectivity problems) — those are documented rather than edited out, since debugging is part of the actual SOC skillset being demonstrated.
