# 🔗 Shuffle SOAR Workflow — VirusTotal Enrichment, TheHive Alerting & Email Notification

![Shuffle](https://img.shields.io/badge/SOAR-Shuffle-6E56CF)
![VirusTotal](https://img.shields.io/badge/Enrichment-VirusTotal-394EFF?logo=virustotal&logoColor=white)
![TheHive](https://img.shields.io/badge/Case%20Management-TheHive-FF7E29)
![Status](https://img.shields.io/badge/Status-Lab-informational)

This covers building the automation itself: Shuffle receives the Mimikatz alert from Wazuh, parses out the file hash, checks its reputation on VirusTotal, creates a case in TheHive, and emails the analyst — all hands-off.

> This ties together Wazuh (see `Wazuh Ubuntu.md`), TheHive (see `theHive Ubuntu.md`), and this Shuffle/VirusTotal workflow. Wazuh's `<integration>` block and TheHive's service-account API key are prerequisites for this doc.

---

## 1. Create a Shuffle account and workflow

1. Go to [shuffler.io](https://shuffler.io) and sign up.
2. Skip the onboarding tour if prompted and go to **Workflows**.
3. Click **+** to create a new workflow.
   - Name: e.g. `SOC Automation Project`
   - Description: e.g. `My DFIR project`
   - Use case: pick anything relevant (doesn't matter much for this lab)
4. You'll land on a blank canvas with a "Change Me" starter node.

## 2. Add a webhook trigger

1. Click the **Triggers** tab.
2. Drag a **Webhook** node onto the canvas.
3. Select it, and on the right-hand panel:
   - Name it something clear, e.g. `wazuh-alerts`.
   - Leave "Find Associated App" blank (optional).
4. **Copy the webhook URI** — you'll paste this into Wazuh's `ossec.conf` `<integration>` block.
5. Click the starter/"Change Me" node connected to the webhook, set its action to **Repeat back to me**, remove the default "Hello World" body, and instead add the **Execution Argument** value via the **+** button. Save.

## 3. Connect Wazuh to this webhook

On the **Wazuh manager**, add the `<integration>` block pointing to this webhook URL (full steps in `Wazuh Ubuntu.md`, section "Integrate with Shuffle"):

```xml
<integration>
  <name>shuffle</name>
  <hook_url>https://<your-shuffle-webhook-url></hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```

Restart the Wazuh manager, then trigger your Mimikatz rule again on the Windows client. In Shuffle, click the webhook node → **Start**, then check the **executions** tab (the person icon at the bottom) to confirm the alert data is arriving.

**Troubleshooting:** if nothing shows up, double-check:
- The `rule_id` matches your custom Mimikatz rule exactly.
- The hook URL scheme is correct — a leftover `http://` vs `https://` mismatch (or vice versa) is a common cause of silent failures.
- The manager was actually restarted after the config change.

## 4. Parse the file hash with Regex

Wazuh's hash fields come back prefixed with the hash type, e.g. `sha1=abcdef123...`. We only want the raw hash value to send to VirusTotal.

1. Click the node after the webhook trigger, change its action from "Repeat back to me" to **Regex Capture Group**.
2. For input data, use the **+** button → **Execution Argument** → find the hash field (e.g. the SHA256 field under the Sysmon event data).
3. If you're not confident writing regex by hand, this is a great use case for AI assistance — ask an LLM to "write a regex to parse the SHA256 value from `sha256=<hash>`" and paste the result in.
4. Save the workflow, rerun it, and expand the node's output to confirm it returns just the clean hash value.
5. Rename this node to something clear, e.g. `SHA256-Rex`.

## 5. Set up VirusTotal

1. Go to [virustotal.com](https://www.virustotal.com) and sign up.
2. Once logged in, copy your **API key** from your account settings.

### Add the VirusTotal app in Shuffle

1. **Apps** tab → search **VirusTotal** → activate it.
2. Drag the VirusTotal node onto the canvas and connect the Regex node's output into it.
3. Rename it (e.g. `VirusTotal`).
4. Under **Find actions**, look for a **hash report** action (not the IP report action) — if only one action is showing up initially, give it a minute and refresh; more actions populate once the app finishes activating.
5. Authenticate either by pasting your API key directly into the field, or via the **Authenticate** button (VirusTotal API v3).
6. For the hash input parameter, select the **Regex node's list output** (not the raw agent/IP fields) — this is the parsed hash value from step 4.
7. Save and rerun the workflow.

### Fixing a common 404 error

VirusTotal has changed their API over time, and the pre-built Shuffle app may be pointing at an outdated endpoint (e.g. `/api/v3/files/report` instead of the current `/api/v3/files/{id}`).

If you get a `404` on this step:

1. Check [VirusTotal's API documentation](https://docs.virustotal.com/reference/file-info) for the current "get a file report by hash" endpoint format.
2. In Shuffle, go to **Apps**, hover the VirusTotal app, open it in a new window, and click **Fork** to make it editable.
3. Find the hash-report action and correct the URL path to match the current API (e.g. replacing a hardcoded `report` segment with the `{id}` parameter the docs specify).
4. Save, then re-add this forked/corrected VirusTotal app to your workflow, re-authenticate with your API key, and re-select the hash-report action with the Regex output as input.
5. Rerun the workflow — you should now get a full JSON response instead of a 404.

### Reading the result

Expand the VirusTotal node's output → `body` → `data` → `attributes` → `last_analysis_stats`. The **`malicious`** field is the count of AV engines flagging the file as malicious — this is the reputation score you'll want to reference or act on downstream.

## 6. Create an alert in TheHive

1. **Apps** → search **TheHive** → drag it onto the canvas, connected after VirusTotal (so its fields are available to reference).
2. **Authenticate**: paste in the **API key** you generated for the Shuffle service account in TheHive (see `theHive Ubuntu.md`, section 13). For the URL, use the theHive Ubuntu 64-bits VM's IP address and port, e.g. `http://<thehive-vm-ip>:9000`.
3. Under **Find actions**, choose **Create Alert** (not a query action).
4. Fill in the alert fields, pulling values from the connected upstream nodes via the **+** → **Execution Argument** picker:

| Field | Suggested value |
|---|---|
| **Date** | the event's UTC time field |
| **Description** | e.g. "Mimikatz detected on host `<computer field>` from user `<user field>`" |
| **External link** | leave blank |
| **Flag** | `false` |
| **PAP** | `2` (default) |
| **Severity** | `2` (adjust to your taste) |
| **Source** | `Wazuh` |
| **Source reference** | your custom rule ID, e.g. `100002` |
| **Status** | `New` |
| **Summary** | e.g. "Mimikatz activity detected on host `<computer field>`, PID `<process ID field>`, command line `<command line field>`" |
| **Tags** | array — include the MITRE ATT&CK tag, e.g. `T1003` |
| **Title** | "Mimikatz detected" (or tie it to the alert data) |
| **TLP** | `2` (default) |
| **Type** | e.g. `internal` |

5. Save the workflow.

### If Shuffle can't reach TheHive

Since there's no firewall on the theHive Ubuntu 64-bits VM, port `9000` is open to anything on the same bridged/LAN network already. If you're using cloud-hosted Shuffle (shuffler.io) rather than self-hosting it, it needs an actual network path to your LAN (e.g. router port-forwarding) to reach TheHive's IP — see `theHive Ubuntu.md`, section 14, for more on this.

6. Rerun the workflow. Check **TheHive** — you should see a new alert automatically created with your description, summary, and metadata populated.

## 7. Send an email notification

1. **Apps** → drag the **Email** node onto the canvas, connected after VirusTotal (or TheHive — either works as long as the fields you need are upstream).
2. **Recipient**: your own email address (use a real inbox you can check — a **Gmail** address works fine for this lab).
3. **Subject**: e.g. `Mimikatz Detected`.
4. **Body**: pull in relevant fields via **Execution Argument**, e.g. the UTC time, the alert title, and the computer/hostname so the analyst immediately knows what happened and where.
5. Save and rerun the workflow.
6. Check your inbox — you should receive an email with the Mimikatz detection details shortly after the workflow runs.

---

## End-to-end recap

1. Mimikatz runs on the Windows 10 client → Sysmon logs it → Wazuh agent ships it → Wazuh manager's custom rule fires.
2. Wazuh's `<integration>` block posts the alert to Shuffle's webhook.
3. Shuffle parses the SHA256 hash out of the raw alert data with regex.
4. Shuffle sends that hash to VirusTotal and pulls back a reputation score.
5. Shuffle creates a case/alert in TheHive with the enriched details.
6. Shuffle emails the analyst with a summary so they can start investigating.

## Notes / gotchas

- Shuffle's pre-built app actions can lag behind a vendor's actual API — if something 404s or errors unexpectedly, check the vendor's current API docs and be ready to **fork and edit** the Shuffle app.
- Always connect nodes in the actual data-flow order (webhook → regex → VirusTotal → TheHive → email) so each node's execution arguments/output fields are selectable in the next node — Shuffle only shows upstream fields for connected nodes.
- Since there's no firewall on these VMs, anything on your LAN can currently reach Wazuh's dashboard, TheHive, etc. — fine for an isolated lab, but add `ufw` (or similar) before this setup touches any network you don't fully trust.
