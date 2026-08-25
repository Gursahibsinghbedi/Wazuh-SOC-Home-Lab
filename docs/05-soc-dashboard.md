## 📊 Custom SOC Monitoring Dashboard

### Objective
Build a centralized Wazuh dashboard to give at-a-glance visibility into authentication failures and account-management activity across both the Windows and Linux endpoints in the lab.

---

### Dashboard: "sahib - SOC activity overview"

A custom dashboard was created in Wazuh combining three visualizations relevant to authentication and account-change monitoring.

#### 1. Failed Windows Logon (Metric)
A single-number metric widget tracking failed Windows login attempts.
- **Query:** `data.win.system.eventID:4625`
- **Type:** Metric
- **Result:** Live count of failed logon events (Event ID 4625) in the selected time range

#### 2. Windows Account Changes Over Time (Line Chart)
Tracks account-management activity on the Windows endpoint by correlating multiple security event IDs relevant to user/group changes:

| Event ID | Meaning |
|---|---|
| 4720 | User account created |
| 4722 | User account enabled |
| 4723 | Password change attempted |
| 4724 | Password reset attempted |
| 4725 | User account disabled |
| 4726 | User account deleted |
| 4732 | Member added to a security-enabled local group |
| 4733 | Member removed from a security-enabled local group |
| 4738 | User account changed |

**Query:**
```
data.win.system.eventID: ("4720" OR "4722" OR "4723" OR "4724" OR "4725" OR "4726" OR "4732" OR "4733" OR "4738")
```
Displayed as a **Date Histogram** (X-axis: `timestamp`) split by `data.win.system.eventID` (Terms aggregation), giving a time-series view of exactly which type of account change occurred and when.

#### 3. Linux Failed SSH Authentication Activity (Data Table)
A table summarizing failed SSH login attempts on the Linux endpoint, useful for quickly spotting brute-force patterns.

**Columns:**
| Column | Description |
|---|---|
| `agent.name` | Source Linux agent (e.g. `sahib-linux`) |
| `timestamp` | When the failed attempt occurred |
| `data.srcuser` | Username attempted |
| `data.srcip` | Source IP of the attempt |
| `Count` | Number of failures |

---

### Assembling the Dashboard
1. Created a new Dashboard in Wazuh's **Dashboards** section.
2. Built each visualization independently in **Visualize → Create**, choosing the appropriate chart type (Metric, Line, Data Table) and index pattern (`wazuh-archives*`).
3. Added all three visualizations to a single dashboard canvas.
4. Applied filters where needed (e.g. `agent.name: sahib-linux` for the SSH table).
5. Saved the dashboard as **"sahib - SOC activity overview"** for reuse and quick access.

---

### Outcome
- ✅ Centralized, at-a-glance view of authentication failures and account-management activity across both Windows and Linux endpoints.
- ✅ Time-series visualization makes it easy to spot spikes in account changes (potential indicators of compromise or misuse).
- ✅ SSH failure table enables rapid triage of brute-force attempts by source IP and username.
- ✅ Dashboard is reusable and can be extended with additional widgets (e.g. Active Response events, FIM alerts) as the lab grows.

### Skills Demonstrated
- Wazuh/OpenSearch Dashboards visualization building (Metric, Line, Data Table)
- Query construction using DQL/Lucene syntax with multi-value OR conditions
- Security event ID correlation (Windows Security log account-management events)
- SOC dashboard design for authentication and account-activity monitoring

### Screenshots
No dedicated screenshots were captured for this section beyond what's embedded above — log tables and dashboard views are reproduced verbatim from the Wazuh UI in the walkthrough.
