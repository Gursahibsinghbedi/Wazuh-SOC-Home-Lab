## 🏗️ Environment Setup — Building the Lab Infrastructure

### Objective
Stand up a small self-contained SOC lab in **VMware Workstation Pro**: one Ubuntu-based Wazuh manager and two monitored endpoints (Linux + Windows), all on the same NAT'd internal network, before any detection engineering work begins.

---

### Environment

| VM | Role | OS |
|---|---|---|
| `wazuh-server` | Wazuh Manager + Indexer + Dashboard | Ubuntu 24.04 LTS |
| `Ubuntu 64-bit` (`sahib-linux`) | Monitored Linux endpoint | Ubuntu 64-bit |
| `Windows 10 x64` (`sahib-windows`) | Monitored Windows endpoint | Windows 10 Pro |

All VMs were created from scratch through VMware Workstation's **New Virtual Machine Wizard** (NVMe disk, UEFI firmware, ISO-based install) and stored under a dedicated `D:\vms` working directory to keep them off the constrained system drive.

---

### Step 1 — Provisioning the VMs
Each VM was built manually through the wizard: virtual disk type (NVMe), firmware type (UEFI), VM naming, and ISO attachment (`Windows.iso` for the Windows endpoint, an Ubuntu ISO for the manager and Linux endpoint).

**Issue faced:** the default VM storage location was on the near-full `C:` drive (99% used, ~2.2 GB free), which caused install failures and slow disk I/O. VM storage was relocated via **Edit → Preferences → Workspace → Default location for virtual machines** to `D:\vms`, and existing VM folders were moved/reorganized there.

A snapshot (`Snapshot 1`) was taken on the manager VM immediately after a clean OS install, to provide a rollback point before any Wazuh configuration changes.

---

### Step 2 — Installing the Wazuh Manager
The Wazuh manager stack (manager, indexer, dashboard) was installed on the `wazuh-server` Ubuntu VM and verified with:

```bash
sudo systemctl status wazuh-manager
```

Output confirmed the service **active (running)**, with core daemons up (`wazuh-analysisd`, `wazuh-syscheckd`, `wazuh-logcollector`, `wazuh-monitord`, `wazuh-modulesd`, `wazuh-execd`, `wazuh-authd`, `wazuh-db`, plus the API workers).

Remote agent connectivity was confirmed by inspecting the `<remote>` block in `ossec.conf`:

```xml
<remote>
  <connection>secure</connection>
  <port>1514</port>
  <protocol>tcp</protocol>
  <queue_size>131072</queue_size>
</remote>
```

...and validating that `wazuh-remoted` was actually listening:

```bash
sudo ss -tulnp | grep 1514
# tcp LISTEN 0 128 0.0.0.0:1514 users:(("wazuh-remoted",pid=13225,fd=4))

sudo ufw status
# Status: inactive
```

With the host firewall inactive and `wazuh-remoted` bound to `0.0.0.0:1514`, the manager was reachable from both endpoint VMs on the internal NAT network.

---

### Step 3 — Enrolling the Linux Agent
The Wazuh agent was installed directly from the official `packages.wazuh.com` repo, pointing at the manager's IP and setting a friendly agent name in a single command:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.77.129' WAZUH_AGENT_NAME='sahib-linux' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

The agent enrolled successfully and later showed as **active** under agent ID `002` in the Wazuh **Endpoints** dashboard.

---

### Step 4 — Enrolling the Windows Agent
On the Windows 10 VM, the agent MSI was pulled and installed silently via PowerShell:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/... -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='192.168.77.129' ...
NET START Wazuh
```

**Issue faced:** the agent initially showed **"never connected"** in the Endpoints dashboard even after the service reported as started. The root cause was traced to the manager IP in the local `ossec.conf`:

```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

After confirming/correcting the `<address>` field to match the manager's actual IP (`192.168.77.129`) and restarting the service:

```powershell
Restart-Service -Name WazuhSvc
```

...the agent transitioned to **active** under agent ID `001` (`sahib-windows`), confirmed on the **Endpoints Summary** page (Agents (2): `sahib-windows` active, `sahib-linux` active/pending).

---

### Outcome
- ✅ Manager, Linux agent, and Windows agent all provisioned from scratch in VMware Workstation.
- ✅ VM storage relocated to avoid disk-space failures during install.
- ✅ Manager verified listening on `1514/tcp` with the host firewall inactive.
- ✅ Both agents successfully enrolled and confirmed **active** in the Endpoints dashboard, ready for the detection engineering work in the rest of this repo.

### Skills Demonstrated
- VM provisioning and resource planning in VMware Workstation
- Linux service administration (`systemctl`, `ss`, `ufw`)
- Wazuh manager installation and remote-connection verification
- Wazuh agent enrollment on both Linux (apt/dpkg) and Windows (MSI/PowerShell)
- Troubleshooting agent-to-manager connectivity via `ossec.conf`

### Screenshots
See [`screenshots/00-vm-provisioning/`](../screenshots/00-vm-provisioning/) and [`screenshots/01-manager-and-agent-setup/`](../screenshots/01-manager-and-agent-setup/) for the full walkthrough.
