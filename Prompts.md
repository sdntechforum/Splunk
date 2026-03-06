As a Security Operations (SecOps) professional, you aren't just looking for data; you’re looking for **signals in the noise.** The power of having an MCP-connected AI is that you can stop worrying about perfect SPL syntax and start focusing on the "what if" scenarios.

Here is a categorized list of high-value prompts tailored for your daily SecOps workflow, using your 90+ tools.

---

## 🕵️‍♂️ Category 1: Threat Hunting & Triage

*Use these when you’re looking for "breadcrumbs" or abnormal behavior across the environment.*

* **Mass Login Failure:** "Search the `wineventlog` or `auth` index for the last 4 hours. Group failed login attempts by user and source IP. Highlight any IP that has more than 20 failures."
* **Suspicious Process Spawning:** "Using the `run_splunk_search` tool, find instances in the last hour where `cmd.exe` or `powershell.exe` was a child process of `w3wp.exe` (IIS) or `sqlservr.exe`."
* **Outbound Data Spikes:** "Compare outbound traffic volume per destination IP for the last 24 hours against the 7-day average. List any destinations that show a 300% increase."
* **Living off the Land:** "Look for any use of `certutil.exe` with `-urlcache` or `-split` arguments across all windows hosts in the last 12 hours."

---

## 🔍 Category 2: Deep-Dive Investigation

*Use these when an alert has fired and you need to pivot on a specific entity.*

* **User Activity Audit:** "I have a suspicious user `jsmith`. Use `Youtube` and `run_splunk_search` to list every index they’ve touched in the last 48 hours and summarize their most frequent actions."
* **IP Reputation Pivot:** "Check all internal logs for the external IP `1.2.3.4`. If found, use the `list_cim_data_models` tool to see if these events map to the 'Network Traffic' model for easier filtering."
* **KVStore Session Analysis:** "Pull the latest session data from our `web_sessions` KVStore collection for user `jsmith` and see if the `session_id` matches any concurrent logins from a different geographic location."

---

## 🛠 Category 3: CIM Compliance & Data Hygiene

*Use these to ensure your "Security Brain" (Splunk Enterprise Security) is actually seeing the data correctly.*

* **Data Model Validation:** "Check the 'Authentication' and 'Malware' data models using `list_cim_data_models`. Are there any indexes that should be contributing to these models but have 0 events in the last hour?"
* **Field Extraction Check:** "Run a oneshot search for the last 50 events in the `firewall` index. Check if the `src_ip` and `dest_port` fields are being correctly extracted. If not, suggest a regex fix."
* **Lookup File Review:** "List all lookup files using `list_lookup_files`. Check if the `threat_intel_ip_list.csv` has been updated in the last 24 hours."

---

## 🏗 Category 4: Splunk Admin & Audit

*Use these to watch the watchers and ensure the platform is stable.*

* **Unauthorized App Changes:** "Use the `list_apps` tool to find any apps that were installed or modified in the last 7 days. Who was the last user to modify the `search` app?"
* **Saved Search Efficiency:** "List all saved searches that have a high 'run time' but haven't triggered an alert in 30 days. We might need to disable these to save CPU."
* **System Health Check:** "Run the `get_splunk_health` tool. If there are any red or yellow indicators, explain exactly which component (Indexer, Search Head, or KVStore) is struggling."

---

## 📖 Category 5: The "Instant Expert" (Documentation)

*Use these when you’re building a new detection and need a quick syntax refresher.*

* **SPL Syntax Help:** "I need to join two indexes on a `dest_ip` field but only where the `timestamp` is within 5 minutes of each other. Give me the SPL for a `map` or `join` command to do this."
* **CIM Reference:** "What are the mandatory fields for the 'Network Resolution' (DNS) CIM model? Give me a list so I can check my sourcetypes."
* **Command Cheat Sheet:** "Give me a cheat sheet for the `stats` command, specifically showing how to use `count`, `distinct_count`, and `values`."

---

### Pro-Tip for SecOps

When you run these in **Cursor**, you can follow up by saying: *"That SPL looks good. Now, turn that into a saved search called 'Potential Brute Force - [Date]' and set it to run every hour."*
