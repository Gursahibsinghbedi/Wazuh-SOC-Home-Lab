## 🔬 Investigation: Deploying Sysmon for Enhanced Process & File Telemetry

### Objective
Deploy **Sysmon** on both monitored endpoints (Linux and Windows) using community-maintained detection-oriented configurations, and confirm the resulting telemetry flows into Wazuh via the agents' `localfile` log collection.

---

### Environment

| Component | Details |
|---|---|
| Linux Agent | `sahib-linux` (agent id `002`), Sysmon-for-Linux |
| Windows Agent | `sahib-windows` (agent id `001`), Sysmon v15.x |
| Linux Config Source | [`microsoft/MSTIC-Sysmon`](https://github.com/microsoft/MSTIC-Sysmon) — `collect-all.xml` |
| Windows Config Source | [`olafhartong/sysmon-modular`](https://github.com/olafhartong/sysmon-modular) |

---

### Step 1 — Installing Sysmon on Linux
Sysmon-for-Linux was already present on the agent as a package (`sysmon`); a detection-oriented ruleset was pulled from Microsoft's MSTIC repo and loaded:

```bash
wget https://raw.githubusercontent.com/microsoft/MSTIC-Sysmon/refs/heads/main/linux/configs/collect-all.xml
sudo sysmon -i collect-all.xml
```

Output confirmed the config validated against schema version 4.81/4.90, and the driver + service started cleanly:

```
Configuration file validated.
Created symlink /etc/systemd/system/multi-user.target.wants/sysmon.service → /etc/systemd/system/sysmon.service.
```

A manual `uname` command was then run and confirmed in Wazuh **Discover** (`wazuh-archives*`, filtered by `agent.name: sahib-linux`) as a Sysmon-tagged process-creation event (`Linux-Sysmon/Operational`, `EventID 1`), proving the pipeline end-to-end from OS → Sysmon → Wazuh agent → manager → indexer.

---

### Step 2 — Installing Sysmon on Windows
On the Windows endpoint, the community **sysmon-modular** ruleset (`olafhartong/sysmon-modular`) was downloaded and loaded via an elevated PowerShell session:

```powershell
sysmon64.exe -i sysmonconfig.xml
```

Output confirmed:

```
Sysmon schema version: 4.91
Configuration file validated.
Sysmon installed.
SysmonDrv installed.
Sysmon started.
```

The Wazuh service was restarted afterward (`Restart-Service -Name WazuhSvc`) so the agent would pick up the new Sysmon operational log channel via its `localfile` configuration in `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Windows **Event Viewer** was used to independently confirm the driver loaded correctly — Event ID `6` from `FilterManager` showing `File System Filter 'SysmonDrv' has successfully loaded and registered with Filter Manager`.

---

### Step 3 — Verifying Telemetry in Wazuh
Both endpoints were queried in **Discover** (`wazuh-archives*`) with a simple `sysmon` full-text search:

| Agent | Result |
|---|---|
| `sahib-linux` | 110 hits in a 24h window — process creation (`EventID 1`) and other Sysmon-tagged events |
| `sahib-windows` | 1,592 hits in a 24h window — process creation, network connection, and command-line telemetry (`data.win.eventdata.*` fields: `commandLine`, `parentImage`, `parentCommandLine`, `hashes`, etc.) |

The **Endpoints Summary** page for `sahib-windows` was also checked, showing the MITRE ATT&CK **Top Tactics** panel already populating from the new Sysmon-driven telemetry (Privilege Escalation, Defense Evasion, Persistence, Initial Access, Lateral Movement) — confirming Wazuh's built-in technique correlation was working against the richer event stream.

---

### Outcome
- ✅ Sysmon deployed on both Linux and Windows endpoints using community detection-oriented configs (not default/minimal configs).
- ✅ Sysmon service + driver load confirmed independently via Event Viewer (Windows) and `systemctl`/log output (Linux).
- ✅ End-to-end telemetry flow validated in Wazuh Discover on both platforms.
- ✅ MITRE ATT&CK tactic correlation confirmed active on the richer Sysmon event stream.

### Skills Demonstrated
- Sysmon deployment and configuration management (cross-platform)
- Use of community/industry-standard detection configs (MSTIC-Sysmon, sysmon-modular) over defaults
- Wazuh `localfile`/`eventchannel` log collection configuration
- Cross-referencing OS-level tooling (Event Viewer) against SIEM ingestion to validate pipelines
- MITRE ATT&CK-aware telemetry validation



