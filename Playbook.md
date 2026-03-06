# 🛡️ SecOps Detection Playbooks

This guide contains pre-configured "AI Playbooks." Copy and paste these prompts into **Cursor** or **Claude** to perform complex security tasks using your Splunk data.

---

## 🛑 Playbook 1: Authentication & Brute Force Triage

**Objective:** Identify and visualize potential password spraying or brute force attacks.

> **The Prompt:**
> *"Search the `wineventlog` or `auth` index for the last 6 hours. Identify any user accounts with more than 15 failed login attempts. For those accounts, pivot and find the source IP addresses. If an IP has targeted multiple accounts, create a table showing: User, Source_IP, Failure_Count, and the first/last attempt time."*

* **MCP Tools Used:** `run_splunk_search`, `Youtube`
* **Follow-up action:** *"Based on those results, check the `threat_intel` lookup to see if any of those Source IPs are known malicious scanners."*

---

## 🕵️ Playbook 2: Endpoint Threat Hunting (Process Lineage)

**Objective:** Detect "Living off the Land" techniques where common binaries perform suspicious actions.

> **The Prompt:**
> *"Look for process execution events in the last 24 hours. Specifically, find instances where `powershell.exe` or `cmd.exe` were launched with encoded commands (look for `-enc` or `-e` in the arguments). Once found, tell me the Parent Process and the user who initiated it."*

* **MCP Tools Used:** `run_oneshot_search`, `get_spl_reference` (to optimize the regex)
* **Follow-up action:** *"Check the Splunk documentation for the `sysmon` sourcetype to see if we are capturing the `ParentCommandLine` field for these events."*

---

## 📤 Playbook 3: Data Exfiltration Discovery

**Objective:** Find anomalous outbound traffic spikes that could indicate data theft.

> **The Prompt:**
> *"Run a search to calculate the average outbound byte count per destination IP for the last 7 days. Now, compare that to the traffic from the last 1 hour. List any destination IP where the current traffic is 5x higher than the 7-day average. Exclude known cloud providers like AWS or Azure."*

* **MCP Tools Used:** `run_splunk_search`, `list_lookup_files`
* **Follow-up action:** *"Save this search as a dashboard panel called 'Anomalous Outbound Traffic' using the `create_dashboard` tool."*

---

## 🧹 Playbook 4: CIM & Data Quality Audit

**Objective:** Ensure your logs are compliant with the Common Information Model (CIM) so that your ES (Enterprise Security) alerts work.

> **The Prompt:**
> *"Use the `list_cim_data_models` tool to pull the requirements for the 'Web' data model. Now, check the `access_combined` sourcetype in our environment. Are we missing any mandatory fields like `http_method`, `status`, or `url`? If fields are missing, suggest a `calculated field` or `alias` to fix it."*

* **MCP Tools Used:** `list_cim_data_models`, `get_configurations`
* **Follow-up action:** *"Show me the `props.conf` configuration for this sourcetype so I can review the current extractions."*

---

## 🩺 Playbook 5: SOC Health Check

**Objective:** Quickly verify if the Splunk infrastructure is healthy and indexes are receiving data.

> **The Prompt:**
> *"Perform a full system health check. Use `get_splunk_health` to check the status of the Indexers and Search Heads. Then, check the 'Internal' index to see if any sourcetypes have stopped sending data in the last 30 minutes. Provide a summary of 'Red' or 'Yellow' alerts."*

* **MCP Tools Used:** `get_splunk_health`, `run_splunk_search`
* **Follow-up action:** *"What is the current license usage? Are we close to our daily limit?"*

---

### How to Add Your Own

To build a new playbook, simply think of the **Security Logic** first, then ask the AI:

1. *"Which MCP tools would I need to [Task]?"*
2. *"Write me a prompt I can use to execute that task."*
