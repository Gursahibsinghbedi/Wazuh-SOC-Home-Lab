## 🛡️ Investigation: SSH Brute-Force Detection & Automated Blocking (Active Response)

### Objective
Simulate an SSH brute-force attack against a monitored Linux endpoint and validate that Wazuh detects the attack and automatically blocks the offending host using **Active Response**.

---

### Environment
| Component | Details |
|---|---|
| Wazuh Manager | `192.168.77.132` |
| Wazuh Dashboard | `https://192.168.77.129` |
| Monitored Agent | `sahib-linux` (`192.168.77.131`) |
| Attacker Machine | Windows (PowerShell, RDP session `192.168.77.132`) |
| Active Response Used | `firewall-drop` |

---

### Step 1 — Custom Detection Rule
A custom rule was created in `local_rules.xml` to detect repeated SSH authentication failures:

| Rule ID | Description | Level | Groups |
|---|---|---|---|
| `100102` | Multiple SSH failed login attempts: 3 or more failures within 60 seconds | 10 | authentication_failed, ssh, authentication |

---

### Step 2 — Active Response Configuration
The following block was added to `/var/ossec/etc/ossec.conf` on the manager to trigger an automatic firewall block whenever rule `100102` fires:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100102</rules_id>
  <level>10</level>
  <timeout>60</timeout>
</active-response>
```

**Issue faced during configuration:**
An initial typo (`<rule_id>` instead of `<rules_id>`) caused the Wazuh manager service to fail on restart:

```
wazuh-analysisd: ERROR: (1230): Invalid element in the configuration: 'rule_id'.
wazuh-analysisd: CRITICAL: (1202): Configuration error at 'etc/ossec.conf'.
wazuh-manager.service: Failed with result 'exit-code'.
```

**Fix:** Corrected the tag to `<rules_id>`, saved the file, and restarted the service successfully:
```bash
sudo nano /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-manager.service
```
Service came up clean with no further errors.

---

### Step 3 — Attack Simulation
From the attacker machine, repeated SSH login attempts were made against the Linux agent using an invalid password:

```powershell
ssh sahib@192.168.77.131
```
Output showed multiple consecutive authentication failures:
```
Permission denied, please try again.
Permission denied, please try again.
Permission denied (publickey,password).
```

---

### Step 4 — Detection in Wazuh Dashboard
Alerts were observed in **Discover** (index: `wazuh-alerts-*`, filtered by `agent.name: sahib-linux`):

| Time | Rule Description |
|---|---|
| 10:18:04 – 10:18:16 | `sshd: authentication failed.` |
| 10:18:18.197 | `syslog: User missed the password more than one time` |
| 10:18:18.197 | `sshd: connection reset` |
| **10:18:18.769** | **`Host Blocked by firewall-drop Active Response`** |

This confirms the custom rule (`100102`) correctly correlated the repeated failures and triggered the active response.

---

### Step 5 — Verification of Block
After the Active Response fired, connectivity from the attacker machine to the agent was tested:

**Ping test:**
```
Reply from 192.168.77.131: bytes=32 time<1ms TTL=64
...
Request timed out.
Request timed out.
```

**SSH test:** subsequent SSH attempts to `192.168.77.131` stopped responding entirely, and querying the dashboard with `agent.name: sahib-linux` for the block window returned **No Results** — confirming the host's traffic was actively being dropped by the firewall rule.

---

### Outcome
- ✅ Custom correlation rule successfully detected brute-force SSH activity.
- ✅ Active Response (`firewall-drop`) automatically blocked the attacking source without manual intervention.
- ✅ Block was verified independently via both ICMP (ping) and SSH connection tests.

### Skills Demonstrated
- Wazuh custom rule authoring (`local_rules.xml`)
- Active Response configuration & troubleshooting (`ossec.conf`)
- Log analysis via Wazuh Discover / Kibana-style queries
- Attack simulation and blue-team verification workflow

### Screenshots
No dedicated screenshots were captured for this section beyond what's embedded above — log tables and dashboard views are reproduced verbatim from the Wazuh UI in the walkthrough.
