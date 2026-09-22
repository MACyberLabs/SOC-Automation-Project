# 🛡️ Wazuh Server (Ubuntu) — Install & Configuration

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2022.04-E95420?logo=ubuntu&logoColor=white)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-3AB6E0?logo=wazuh&logoColor=white)
![Status](https://img.shields.io/badge/Status-Lab-informational)

This covers standing up the Wazuh manager on Ubuntu, connecting the Windows 10 agent, configuring log ingestion (Sysmon), building a custom detection rule for Mimikatz, and integrating with Shuffle (SOAR).

> Lab setup: self-hosted Ubuntu VM named **Wazuh Ubuntu 64-bits** in VMware Workstation Pro, running alongside the Windows 10 and theHive Ubuntu 64-bits VMs on the same bridged network.

---

## 1. Prerequisites

- Ubuntu 20.04/22.04 VM, reachable on your LAN (e.g. via bridged networking)
- At least 4GB RAM / 50GB disk (Wazuh's all-in-one installer bundles the manager, indexer, and dashboard)
- If you do add a firewall later (recommended once this leaves an isolated lab network), make sure these stay reachable:
  - `443` – Wazuh dashboard (HTTPS)
  - `1514` – agent event data
  - `1515` – agent registration
  - `55000` – Wazuh API

## 2. Spin up the Ubuntu VM (self-hosted in VMware Workstation Pro)

This lab is self-hosted — no cloud provider. Wazuh runs in its own Ubuntu VM inside VMware Workstation Pro, alongside the Windows 10 client and TheHive VM.

1. Download the **Ubuntu 22.04** Desktop or Server ISO from [ubuntu.com](https://ubuntu.com/download).
2. In VMware Workstation Pro: **Create a New Virtual Machine** → point it at the Ubuntu ISO.
3. Name the VM **Wazuh Ubuntu 64-bits** (matching this project's naming) and pick a storage location.
4. Specs — bump these up if your host can spare it, Wazuh's stack (manager + indexer + dashboard all-in-one) is happier with more:
   - **RAM:** 4GB minimum, more if available
   - **CPU cores:** 2+
   - **Disk:** 50GB
5. **Network adapter: Bridged** — so this VM gets its own LAN IP and the Windows 10 client / TheHive VM can reach it directly.
6. Finish the wizard, power on, and walk through the Ubuntu install (set your username/password).
7. Install **VMware Tools** / open-vm-tools inside the guest for better integration.

> **No firewall was configured for this lab** — since everything stays on the local/bridged network rather than being exposed to the internet, this matches what the original video does with a cloud firewall (restricting access to your own IP) but skips that step since there's no public exposure here. If you ever expose this VM to the internet, lock it down with `ufw` or equivalent first.

### Get the VM's IP and update

Find this VM's IP address (you'll need it repeatedly for configs):

```bash
ip a
```

or

```bash
hostname -I
```

Update the system:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

Accept any prompts to keep default config files and restart services when asked.

## 3. Install Wazuh (all-in-one)

With the base OS updated:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

Download and run the official install script (installs the manager, indexer, and dashboard in one shot). This lab used **Wazuh 4.14.7-1**:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

> The `4.x` branch URL pulls whatever the current 4.x release is at install time — for this project that resolved to **4.14.7-1**. If you need to pin an exact version instead of "whatever's current," check the [official Wazuh installation guide](https://documentation.wazuh.com/current/installation-guide/index.html) for version-specific instructions.

The script generates a credentials archive when it finishes. Extract it:

```bash
sudo tar -xvf wazuh-install-files.tar
```

Move into the extracted folder and read out the passwords:

```bash
cd wazuh-install-files
cat wazuh-passwords.txt
```

The one you care about first is the **admin** password — this logs you into the dashboard. Also note the **Wazuh API user** password; you'll need it later for any automation that calls the Wazuh API.

## 4. Log into the dashboard

Browse to:

```
https://<Wazuh-VM-IP-address>
```

Log in with:
- **User:** `admin`
- **Password:** (from `wazuh-passwords.txt`)

Right after install, you'll see **0 agents**. That's expected until we register the Windows client.

## 5. Add the Windows 10 agent

In the dashboard:

1. Click **Add agent**.
2. Operating system: **Windows**.
3. **Server address:** the Wazuh manager VM's IP address (from `ip a` / `hostname -I` above).
4. **Agent name:** anything identifiable (e.g. `my-dfir` or the hostname).
5. Groups: leave as **default** unless you've created custom groups.
6. Copy the generated PowerShell install command.

On the Windows 10 machine, open **PowerShell as Administrator**, paste the command, and run it. Once it finishes installing:

```powershell
net start wazuh
```

(or open `services.msc` and start the **Wazuh** service manually).

Back in the dashboard, the agent will initially show **Disconnected** — give it a few seconds. Once it's checking in you'll see **Total agents: 1**, **Active: 1**.

Confirm data is flowing under **Security events**.

## 6. Configure log ingestion (Sysmon)

By default, Wazuh's Windows agent forwards `Application`, `Security`, and `System` event logs — not Sysmon. To ingest Sysmon events, edit the agent's `ossec.conf` on the Windows machine:

**Path:** `C:\Program Files (x86)\ossec-agent\ossec.conf`

Before editing, back it up:

```
Copy ossec.conf → ossec-backup.conf
```

Open `ossec.conf` (Notepad **as Administrator**) and find the `<log_analysis>` section. Add a new `<localfile>` block modeled on the existing ones:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

To get the exact channel name, open **Event Viewer** → **Applications and Services Logs** → **Microsoft** → **Windows** → **Sysmon** → **Operational** → right-click → **Properties**, and copy the **Full Name** shown there.

If you only want Sysmon (and not the default Application/Security/System noise), remove those other `<localfile>` blocks, leaving Sysmon and (optionally) `active-response`.

Save the file, then restart the Wazuh agent service on Windows (**Services** → **Wazuh** → Restart) — any `ossec.conf` change requires a service restart to take effect.

## 7. Make Wazuh log everything (not just rule matches)

By default, Wazuh's manager only writes an event to its index when a rule/alert is triggered — it does **not** log every raw event. For testing/detection-building, it's useful to archive everything.

On the **Wazuh manager** (Ubuntu, not the Windows client), back up and edit `ossec.conf`:

```bash
sudo cp /var/ossec/etc/ossec.conf ~/ossec-backup.conf
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<alerts>` section near the bottom and set both to `yes`:

```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</alerts>
```

Save, then restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

This makes Wazuh archive every event to `/var/ossec/logs/archives/` (`archives.log` and `archives.json`).

Enable Filebeat to ship those archive logs into the indexer:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Find the Wazuh archive input and flip it on:

```yaml
archives:
  enabled: true
```

Restart Filebeat:

```bash
sudo systemctl restart filebeat
```

### Create an index pattern for the archives

In the dashboard: hamburger menu (top-left) → **Stack Management** → **Index Patterns** → **Create index pattern**.

- Name it something like `wazuh-archives-*`
- Time field: `timestamp`
- **Create index pattern**

Now go to **Discover**, select the new `wazuh-archives-*` index from the dropdown, and you'll be able to search **all** ingested events, not just ones that triggered a rule.

### Troubleshooting tip

If events aren't showing up in the dashboard, check the raw archive file directly on the manager:

```bash
cat /var/ossec/logs/archives/archives.json | grep -i mimikatz
```

If it's in the archive file but not the dashboard yet, give it more time (or restart the manager to force ingestion — fine in a lab, avoid in production).

## 8. Build a custom detection rule (Mimikatz)

Goal: detect Mimikatz execution via Sysmon **Event ID 1** (process creation), keyed on the **OriginalFileName** field (not the process image name) so a simple rename doesn't bypass the rule.

In the dashboard: **hamburger menu** → drop-down next to it → **Management** → **Rules**.

Click **Manage rule files** → search `sysmon` to find Wazuh's built-in Sysmon EID1 rules (e.g. `0800-sysmon_id1.xml`) for reference. Open one and copy an example `<rule>` block.

Go to **Custom rules**, open the local rule file (pencil icon), and paste your new rule **below** the existing one — match the indentation exactly (Wazuh rule files use spaces, not tabs).

Example custom rule:

```xml
<rule id="100002" level="15">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz</field>
  <description>Mimikatz usage detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

Key points:
- **Rule ID:** custom rules must start at `100000`+. Use the next free ID after any existing custom rules.
- **Level:** severity, 1–15 (15 = highest). Set high for something like credential dumping.
- **Field name:** must match the Sysmon field's exact casing — `originalFileName`, not `filename`. Case-sensitivity matters or the rule silently never fires.
- **Value:** `pcre2` regex, case-insensitive with `(?i)`, matching `mimikatz` anywhere in the original file name.
- **MITRE ATT&CK ID:** `T1003` = OS Credential Dumping, which is what Mimikatz is known for.

Save — the dashboard will prompt you to restart the manager; confirm it.

### Test it

On the Windows client, rename `mimikatz.exe` to something else (e.g. `you-are-awesome.exe`) to prove the rule isn't just matching the filename, then run it from an administrative PowerShell prompt. Back in **Security events**, search `mimikatz` — the alert should fire even though the executable was renamed, because the rule keys on `OriginalFileName`, not the image name.

## 9. Integrate with Shuffle (SOAR)

To forward specific alerts to Shuffle for automation, add an `<integration>` block to the manager's `ossec.conf`:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this after the `<global>` section:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>https://your-shuffle-instance/api/v1/hooks/YOUR_WEBHOOK_ID</hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```

Notes:
- Paste the exact **webhook URL** you copied from your Shuffle trigger.
- Make sure there's whitespace between the URL and the closing tag so the editor doesn't try to merge them into one token.
- By default, Wazuh integrations trigger off a **level** threshold (e.g. `<level>3</level>`, meaning any alert level ≥3 gets forwarded). Swap `<level>` for `<rule_id>` if you only want a specific rule (like your Mimikatz rule) to fire the integration — this is what's shown above.
- Double check the URL scheme is exactly what your Shuffle webhook uses (`http` vs `https`) — a mismatch here is a common reason nothing shows up in Shuffle even though everything else is configured correctly.

Restart the manager to apply:

```bash
sudo systemctl restart wazuh-manager
```

Verify it's running:

```bash
sudo systemctl status wazuh-manager
```

Trigger your Mimikatz rule again on the Windows client — you should now see the execution land as an event in your Shuffle workflow.

## 10. Useful commands recap

```bash
sudo systemctl status wazuh-manager      # check manager status
sudo systemctl restart wazuh-manager     # apply ossec.conf changes
sudo systemctl status filebeat
sudo systemctl restart filebeat          # apply filebeat.yml changes
cat /var/ossec/logs/archives/archives.json | grep -i <term>   # troubleshoot ingestion
```
