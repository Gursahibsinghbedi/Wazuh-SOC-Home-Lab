## 📡 Investigation: Generating & Reading Telemetry (Windows Security Events, Sysmon, SSH Auth)

### Objective
Generate real activity across the Windows and Linux endpoints — account changes, logons, process execution, and suspicious file access — then read and interpret the resulting telemetry in Wazuh to understand what each event actually represents.

---

### Environment
| Component | Details |
|---|---|
| Windows Agent | `sahib-windows` (`192.168.77.132`) |
| Linux Agent | `sahib-linux` (`192.168.77.131`) |
| Telemetry Sources | Windows Security Event Log, Sysmon (Windows + Linux), SSH auth logs |

---

### Step 1 — Generating Windows Account Activity
From an elevated Command Prompt on the Windows endpoint, several account-management actions were performed to generate Security log telemetry:

```cmd
net user guest /active:yes
net user guest sahibPass
net user student1 password /add
net localgroup administrators student1 /add
net user student1 /delete
whoami
ipconfig /all
```

These actions were designed to trigger a range of Windows Security auditing events (account creation, group membership changes, deletion) for later correlation in Wazuh.

---

### Step 2 — Reading Windows Security Event Telemetry
Each resulting event was located and inspected in Wazuh Discover, and cross-referenced against Microsoft's Security Event ID documentation to understand its meaning:

| Event ID | Meaning | Verified In Lab |
|---|---|---|
| **4720** | A user account was created | `student1` account creation — full event showed Subject (`sahib`), New Account (`student1`), SID, and initial (disabled) account attributes |
| **4726** | A user account was deleted | `student1` deletion — event showed Subject and Target Account details, 8 related hits when searched by username |
| **4624** | An account was successfully logged on | 18 hits in a 1-hour window; inspected Logon Type, Process Name (`services.exe`), and Authentication Package (`Negotiate`) fields |

Raw event fields inspected included **Subject** (who performed the action), **New/Target Account** (who was affected), **SID**, **Logon Type**, and **Account Domain** — giving full attribution for each change.

---

### Step 3 — Generating & Reading Sysmon Telemetry
Sysmon was active on both the Windows and Linux endpoints, generating high-volume process and file telemetry:

- **Windows:** Process creation events captured full image paths (e.g. `C:\Windows\system32\wuauclt.exe`), `processGuid`, `processId`, and `targetFilename`. Several events included an automatically mapped **MITRE ATT&CK technique** (`technique_id=T1574.010 — Services File Permissions Weakness`), showing Sysmon's built-in technique correlation.
- **Linux:** Sysmon-for-Linux was also running (`sahib-linux`), generating XML-formatted process/event telemetry (`Linux-Sysmon` provider), accounting for the majority (~91%) of alerts from that agent in a 24-hour window.

---

### Step 4 — Generating & Reading SSH Authentication Telemetry
On the Linux endpoint, both failed and successful SSH authentication attempts were generated to compare their telemetry signatures:

**Failed attempt (invalid user):**
```bash
ssh fake@192.168.77.131
```
Result: `Permission denied (publickey,password)` after 3 failed password prompts — this is the same pattern that later fed the custom brute-force detection rule (100102) used in the Active Response investigation.

**Successful attempt (valid user):**
```bash
ssh sahib@192.168.77.131
```
Result: clean login, dropping into an interactive shell — confirming the corresponding "successful login" telemetry in Wazuh for comparison against the failed-attempt signature.

---

### Step 5 — Simulating Suspicious File Access
To generate telemetry resembling data staging/exfiltration behavior, a sensitive system file was copied to a world-writable staging directory:

```bash
cat /etc/passwd > /tmp/loot.txt
cd /tmp
ls -la
```

This mimics a common attacker pattern — collecting sensitive local files (`/etc/passwd`) into a temp directory ahead of exfiltration. The resulting file (`loot.txt`, owned by `sahib`, 1822 bytes) was confirmed via directory listing, illustrating the kind of file-system activity FIM and command-logging are designed to catch.

---

### Outcome
- ✅ Generated and correctly interpreted core Windows Security auditing events (4720, 4726, 4624) tied to account lifecycle and logon activity.
- ✅ Captured Sysmon telemetry on both platforms, including automatic MITRE ATT&CK technique mapping on Windows.
- ✅ Compared telemetry signatures for failed vs. successful SSH authentication.
- ✅ Simulated a sensitive-file staging action (`/etc/passwd` → `/tmp/loot.txt`) to illustrate suspicious file-access telemetry.

### Skills Demonstrated
- Windows Security Event Log analysis and Event ID interpretation
- Sysmon telemetry review across Windows and Linux
- MITRE ATT&CK technique correlation
- SSH authentication log analysis (failed vs. successful)
- Recognizing attacker-like file-staging behavior at the OS level



