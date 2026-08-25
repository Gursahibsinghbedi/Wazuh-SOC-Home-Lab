## 🗂️ Investigation: File Integrity Monitoring (FIM) — Detecting Unauthorized File & Registry Changes

### Objective
Configure Wazuh's File Integrity Monitoring (Syscheck) module to detect real-time changes (creation, modification, deletion) to sensitive files and Windows registry keys on both a Linux and a Windows endpoint, and validate detection through simulated changes.

---

### Environment
| Component | Details |
|---|---|
| Wazuh Manager | `192.168.77.132` |
| Linux Agent | `sahib-linux` (agent id `002`) |
| Windows Agent | `sahib-windows` (agent id `001`) |
| Monitored Paths (Linux) | `/opt/company-data`, `/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/boot` |
| Monitored Paths (Windows) | `C:\company data`, system binaries, and key registry hives |

---

### Step 1 — Enabling Syscheck (Linux)
`/var/ossec/etc/ossec.conf` was edited to enable real-time monitoring on a custom "sensitive data" directory:

```xml
<!-- File integrity monitoring -->
<syscheck>
  <disabled>no</disabled>

  <!-- Frequency that syscheck is executed default every 12 hours -->
  <frequency>43200</frequency>

  <directories realtime="yes">/opt/company-data</directories>

  <scan_on_start>yes</scan_on_start>

  <!-- Directories to check (perform all possible verifications) -->
  <directories>/etc,/usr/bin,/usr/sbin</directories>
  <directories>/bin,/sbin,/boot</directories>
</syscheck>
```

A test directory and file were created to simulate a sensitive data store:
```bash
sudo su -
mkdir /opt/company-data
echo "payroll data" > /opt/company-data/test.txt
```

---

### Step 2 — Enabling Syscheck (Windows)
On the Windows endpoint, a test folder was created and Syscheck configured to watch it in real time:

```
C:\company data\payroll.txt   →   "this is a secret info plz do not touch"
```

```xml
<directories realtime="yes">C:\company data</directories>
```

This was added alongside Wazuh's default Windows monitoring rules (registry hives, system binaries, startup folders, etc.), which are included out-of-the-box in `ossec.conf`.

---

### Step 3 — Simulating File Changes
To trigger detections, the following actions were performed on the monitored files:
- **Modified** `payroll.txt` / `test.txt` (edited contents)
- **Deleted** `payroll.txt` on the Windows endpoint
- Windows registry values under `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\...` were also altered indirectly by normal system activity, which Syscheck picked up automatically

---

### Step 4 — Detection in Wazuh (Linux Agent — `sahib-linux`)
Viewed via **File Integrity Monitoring → Events** dashboard, filtered by `agent.id: 002`:

| Time | Path | Event | Rule Description | Level | Rule ID |
|---|---|---|---|---|---|
| Aug 23, 17:44:43 | `/opt/company-data/test.txt` | modified | Integrity checksum changed. | 7 | 550 |
| Aug 23, 02:23:17 | `/usr/bin/libsysinternalsEBPFinstaller` | added | File added to the system. | 5 | 554 |
| Aug 23, 02:23:11 | `/usr/bin/sysmon` | added | File added to the system. | 5 | 554 |
| Aug 23, 02:23:10 | `/etc/apt/sources.list.d/microsoft-prod.list` | added | File added to the system. | 5 | 554 |

The custom directory (`/opt/company-data/test.txt`) was correctly flagged as **modified** with a checksum-change alert, confirming real-time FIM was active on the sensitive folder.

---

### Step 5 — Detection in Wazuh (Windows Agent — `sahib-windows`)
Viewed via the same **FIM: Recent events** panel, filtered by `agent.id: 001`:

| Time | Path | Action | Rule Description | Level | Rule ID |
|---|---|---|---|---|---|
| Aug 23, 16:14:55 | `c:\company data\payroll.txt` | deleted | File deleted. | 7 | 553 |
| Aug 23, 16:13:15 | `c:\company data\payroll.txt` | modified | Integrity checksum changed. | 7 | 550 |
| Aug 23, 06:31:47 | `HKEY_LOCAL_MACHINE\System\...\W32Time\...` | modified | Registry Value Integrity Checksum Changed | 5 | 750 |
| Aug 23, 06:31:47 | `HKEY_LOCAL_MACHINE\System\...\W32Time\...` | modified | Registry Key Integrity Checksum Changed | 5 | 594 |

Both **file-level** (create/modify/delete, rules 550/553/554) and **registry-level** (rules 594/750) changes were successfully captured — confirming FIM coverage across the filesystem and the Windows registry.

---

### Outcome
- ✅ Real-time FIM successfully detected file creation, modification, and deletion on both Linux and Windows endpoints.
- ✅ Sensitive custom directories (`/opt/company-data`, `C:\company data`) were monitored in real time (`realtime="yes"`), not just on the default 12-hour scan cycle.
- ✅ Windows registry monitoring extended detection beyond the filesystem, catching configuration/service-related changes.
- ✅ All alerts were correctly categorized by severity (level 5 for additions/registry, level 7 for modifications/deletions).

### Skills Demonstrated
- Wazuh Syscheck (FIM) configuration — real-time vs. scheduled scanning
- Cross-platform monitoring (Linux filesystem + Windows filesystem & registry)
- Simulated insider-threat / unauthorized-access scenario (sensitive file tampering)
- Log correlation and severity-based triage in the Wazuh dashboard

### Screenshots
No dedicated screenshots were captured for this section beyond what's embedded above — log tables and dashboard views are reproduced verbatim from the Wazuh UI in the walkthrough.
