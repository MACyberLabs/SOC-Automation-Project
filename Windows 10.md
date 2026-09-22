# 🖥️ Windows 10 Client — VM, Sysmon, Wazuh Agent & Mimikatz Testing

![Windows](https://img.shields.io/badge/OS-Windows%2010-0078D6?logo=windows&logoColor=white)
![Sysmon](https://img.shields.io/badge/Telemetry-Sysmon-0078D6?logo=windows&logoColor=white)
![Status](https://img.shields.io/badge/Status-Lab-informational)

This is the "victim" endpoint in the lab. It runs Windows 10 with Sysmon for detailed telemetry and the Wazuh agent to ship that telemetry to the Wazuh manager. We also use this machine to generate Mimikatz activity to validate detections end-to-end.

> If you're on Apple Silicon (M1/M2/M3), VMware Workstation Pro isn't available for Mac and won't run a Windows 10 x86 VM locally anyway — spin up the Windows 10 client in the cloud instead (e.g. Azure/AWS) and skip straight to the Sysmon section. (VMware Fusion is the Mac equivalent, but Apple Silicon still can't run x86 Windows guests at usable speed.)

---

## 1. Install VMware Workstation Pro

1. Download **VMware Workstation Pro** from [Broadcom/VMware's site](https://www.vmware.com/products/workstation-pro.html) (free for personal use as of the current licensing).
2. Run the installer, accept the license agreement, keep the default install location unless you need it elsewhere.
3. Reboot if prompted.

## 2. Get a Windows 10 ISO

1. Use Microsoft's **Media Creation Tool** (linked from Microsoft's official Windows 10 download page).
2. Run it, accept the license terms.
3. Choose **Create installation media for another PC** (not "Upgrade this PC").
4. Keep the recommended language/edition/architecture options (or customize as needed).
5. Choose **ISO file** as the media type, and save it somewhere you'll remember (e.g. `Documents`).
6. Let it download — this can take a while depending on your connection.

## 3. Create the VM in VMware Workstation Pro

1. Open VMware Workstation Pro → **Create a New Virtual Machine**.
2. Choose **Typical** configuration.
3. Point it at your Windows 10 ISO (**Installer disc image file (iso)**).
4. Name the VM **Windows 10** (matching what's used across this project) and pick a storage location.
5. Specs — a reasonable baseline for this lab:
   - **RAM:** 4GB
   - **CPU cores:** 1–2
   - **Disk:** 50GB (store as a single file or split — either works)
6. **Network adapter: Bridged** — this puts the VM on your actual LAN with its own IP, so it can reach the Wazuh and TheHive VMs directly. (If you used NAT or Host-only instead, the Wazuh/TheHive IPs you plug into configs later need to match that network instead.)
7. Finish the wizard and power on the VM.

### Install Windows 10

1. Click through the initial **Next → Install now**.
2. On the product key screen, choose **I don't have a product key**.
3. Select an edition — **Windows 10 Pro**.
4. Accept the license terms.
5. Choose **Custom: Install Windows only** (not upgrade).
6. Let it install and complete the out-of-box setup.
7. Once installed, install **VMware Tools** (VM menu → **Install VMware Tools**) for better performance, resolution scaling, and clipboard/drag-drop support between host and guest.

## 4. Install Sysmon

Sysmon gives you far richer process/network/registry telemetry than default Windows logging — this is what lets Wazuh detect things like Mimikatz reliably.

1. Download **Sysmon** from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) (or via the GitHub mirror if you're not on Windows).
2. Download a Sysmon config — the SwiftOnSecurity config is a solid, widely-used starting point: [sysmon-config on GitHub](https://github.com/SwiftOnSecurity/sysmon-config) → open `sysmonconfig-export.xml` → **Raw** → right-click → **Save As** → save it (e.g. as `sysmonconfig.xml`) in the same folder you'll extract Sysmon into.
3. Extract the Sysmon zip (right-click → **Extract All**).
4. Make sure the Sysmon config file ends up in the **same folder** as the extracted Sysmon executables.
5. Open **PowerShell as Administrator**.
6. `cd` into the folder containing Sysmon and the config, e.g.:
```powershell
   cd "C:\Users\<you>\Downloads\Sysmon"
```
7. Use the **64-bit** binary on a 64-bit machine. Running it with no arguments just shows the help/usage text:
```powershell
   .\Sysmon64.exe
```
8. Install Sysmon with your config:
```powershell
   .\Sysmon64.exe -i sysmonconfig.xml
```
   Accept the Sysinternals license prompt.

### Verify Sysmon is running

- **Services** (`services.msc`) → look for **Sysmon64** / **Sysmon**.
- **Event Viewer** → **Applications and Services Logs** → **Microsoft** → **Windows** → **Sysmon** → **Operational**. If you don't see it right away, close and reopen Event Viewer (or hit refresh).

You should now see Sysmon generating telemetry (process creations, network connections, etc.) under that Operational log.

## 5. Install the Wazuh agent

Do this after your Wazuh manager is up and running (see `Wazuh Ubuntu.md`).

1. In the Wazuh dashboard: **Add agent**.
2. OS: **Windows**.
3. **Server address:** the Wazuh manager VM's IP address (run `ip a` or `hostname -I` on it).
4. **Agent name:** anything identifiable.
5. Groups: leave as **default**.
6. Copy the generated PowerShell command.
7. On the Windows 10 VM, open **PowerShell as Administrator**, paste the command, run it.
8. Start the service:
```powershell
   net start wazuh
```
   or via `services.msc`, find **Wazuh** and start it.

Back on the dashboard, the agent should flip from **Disconnected** to **Active** within a few seconds — confirming Wazuh is receiving telemetry from this host.

## 6. Configure the agent to ship Sysmon logs

By default, the Wazuh Windows agent forwards Application/Security/System event logs — not Sysmon. To fix that:

1. Navigate to: `C:\Program Files (x86)\ossec-agent\ossec.conf`
2. **Back it up first** — copy it and rename the copy something like `ossec-backup.conf`.
3. Open `ossec.conf` with **Notepad running as Administrator** (you'll get a permissions error otherwise).
4. In the `<log_analysis>` section, add a new block modeled after the existing `<localfile>` entries:
```xml
   <localfile>
     <location>Microsoft-Windows-Sysmon/Operational</location>
     <log_format>eventchannel</log_format>
   </localfile>
```
5. To get the exact channel name: **Event Viewer** → **Applications and Services Logs** → **Microsoft** → **Windows** → **Sysmon** → right-click **Operational** → **Properties** → copy the **Full Name** value → paste it into `<location>`.
6. (Optional) Remove the default `Application`, `Security`, and `System` `<localfile>` blocks if you only want Sysmon forwarded — leave `active-response` alone.
7. Save the file.
8. Restart the **Wazuh** service (`services.msc` → Wazuh → Restart) — any `ossec.conf` edit requires a service restart.

Confirm it's working: in the Wazuh dashboard, under **Security Events**, search `sysmon`. It may take a little time for events to start appearing.

## 7. Download and run Mimikatz (test telemetry)

Mimikatz is a well-known credential-dumping tool used here purely to validate detection — it will absolutely be flagged by antivirus, so we need to allow it temporarily in this lab environment.

### Exclude the Downloads folder from Defender

1. Open **Windows Security** → **Virus & threat protection**.
2. **Manage settings** → scroll to **Exclusions** → **Add or remove exclusions**.
3. **Add an exclusion** → **Folder** → select your **Downloads** folder.

### Allow the download in Chrome (if it blocks it)

1. Chrome **Settings** → **Privacy and security** → **Security**.
2. Scroll down and set download protection to **No protection** (temporarily, for this lab only).

### Download and run

1. Download Mimikatz (search "mimikatz github releases" for the official repo) and save it to your Downloads folder.
2. Extract it (right-click → **Extract All**).
3. Open **PowerShell as Administrator**.
4. `cd` into the extracted Mimikatz folder.
5. Run it:
```powershell
   .\mimikatz.exe
```

### Confirm it shows up

- **Event Viewer** → Sysmon **Operational** log → look for **Event ID 1** (process creation) referencing Mimikatz.
- Wazuh dashboard → **Security Events** → search `mimikatz`.

If nothing shows in the dashboard yet but you can see it in the raw Wazuh archive logs on the manager (`/var/ossec/logs/archives/archives.json`), it just needs a little time to propagate — or Wazuh's default "only log rule matches" behavior means you need the custom rule and archive settings described in `Wazuh Ubuntu.md`.

3. Confirm the alert still fires in Wazuh — see `Wazuh Ubuntu.md` for building that custom rule.

## Notes / gotchas

- Any change to `ossec.conf` on the agent **requires a service restart** to take effect.
- Field names in Wazuh rules are **case-sensitive** — this matters again later when building the detection rule on the manager side.
- Re-enable Defender protections and download protection once you're done testing — don't leave a lab machine wide open.
