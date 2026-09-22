# 🐝 TheHive Server (Ubuntu) — Install & Configuration

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2022.04-E95420?logo=ubuntu&logoColor=white)
![TheHive](https://img.shields.io/badge/Case%20Management-TheHive-FF7E29)
![Cassandra](https://img.shields.io/badge/DB-Cassandra-1287B1?logo=apachecassandra&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Index-Elasticsearch-005571?logo=elasticsearch&logoColor=white)
![Status](https://img.shields.io/badge/Status-Lab-informational)

TheHive is the case management platform in this lab. It's backed by **Cassandra** (database) and **Elasticsearch** (indexing/search), fronted by the **TheHive** application itself. This doc covers standing up the server, installing all four components, configuring them to talk to each other, and setting up organizations/users for the SOAR integration.

---

## 1. Spin up the Ubuntu VM (self-hosted in VMware Workstation Pro)

Same approach as the Wazuh box — see `Wazuh Ubuntu.md` for the general VM walkthrough. Specifically for TheHive:

1. Download the **Ubuntu 22.04** ISO from [ubuntu.com](https://ubuntu.com/download) (if you don't already have it from setting up the Wazuh VM).
2. In VMware Workstation Pro: **Create a New Virtual Machine** → point it at the Ubuntu ISO.
3. Name the VM **theHive Ubuntu 64-bits** (matching this project's naming) and pick a storage location.
4. Specs — bump up RAM if Cassandra/Elasticsearch feel sluggish once running:
   - **RAM:** 4GB minimum, more if available
   - **CPU cores:** 2+
   - **Disk:** 50GB
5. **Network adapter: Bridged** — same network as the Wazuh and Windows 10 VMs, so they can all reach each other.
6. Finish the wizard, power on, and walk through the Ubuntu install.
7. Install **VMware Tools** / open-vm-tools inside the guest.

> **No firewall was configured for this VM either** — everything stays on the local/bridged network. If this box is ever exposed to the internet, lock it down first.

Get this VM's IP address (you'll need it repeatedly below):

```bash
ip a
```

Update the system:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

## 2. Install prerequisites

```bash
sudo apt install -y wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl \
  software-properties-common python3-pip lsb-release
```

## 3. Install Java

TheHive requires Java 11. Using Amazon Corretto:

```bash
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | sudo tee -a /etc/apt/sources.list.d/corretto.sources.list
sudo apt update
sudo apt install -y java-common java-11-amazon-corretto-jdk
echo JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto" | sudo tee -a /etc/environment
source /etc/environment
```

(OpenJDK 11 works too if you prefer: `sudo apt install openjdk-11-jre-headless`.)

## 4. Install Cassandra

```bash
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update
sudo apt install -y cassandra
```

> Match the Cassandra branch (`40x`, `311x`, `41x`) to whatever version TheHive's current docs recommend for the TheHive version you're installing.

## 5. Install Elasticsearch

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install -y apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install -y elasticsearch
```

## 6. Install TheHive

```bash
wget -O- https://archives.strangebee.com/keys/strangebee.gpg | sudo gpg --dearmor -o /usr/share/keyrings/strangebee-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/strangebee-archive-keyring.gpg] https://deb.thehive-project.org release main" | sudo tee -a /etc/apt/sources.list.d/strangebee.sources.list
sudo apt update
sudo apt install -y thehive
```

> Check [TheHive's install docs](https://docs.strangebee.com/thehive/installation/) for the exact current repo line/version — package names and repos have changed across TheHive 4 → 5.

The whole install (Java + Cassandra + Elasticsearch + TheHive) generally takes 10–15 minutes.

---

## 7. Configure Cassandra

Edit Cassandra's config:

```bash
sudo nano /etc/cassandra/cassandra.yaml
```

Change the following:

- **`cluster_name`**: give it something identifiable, e.g. `my-dfir` (default is `Test Cluster`).
- **`listen_address`**: change from `localhost` to this VM's **IP address**.
- **`rpc_address`**: same — change from `localhost` to this VM's IP address.
- **`seed_provider` → `seeds`**: change `127.0.0.1` to this VM's IP address as well.

(Tip: in `nano`, `Ctrl+W` searches the file — handy for jumping straight to `listen_address`, `rpc_address`, and `seed_provider`.)

Save (`Ctrl+X`, `Y`, Enter).

### Clear old data and restart

Because Cassandra was installed via the package and briefly started with default (localhost) settings, clear out its data directory before restarting with the new config:

```bash
sudo systemctl stop cassandra
sudo rm -rf /var/lib/cassandra/*
sudo systemctl start cassandra
sudo systemctl status cassandra
```

Confirm it shows **active (running)**.

## 8. Configure Elasticsearch

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Uncomment and set:

- **`cluster.name`**: e.g. `thehive`
- **`node.name`**: e.g. `node-1`
- **`network.host`**: this VM's **IP address**
- **`http.port`**: `9200` (default, just uncomment if needed)
- Since this is a single-node demo setup, uncomment `cluster.initial_master_nodes` and leave only this node in the list (remove any second node placeholder) rather than configuring `discovery.seed_hosts` for a multi-node cluster.

Save, then start and enable the service:

```bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
sudo systemctl status elasticsearch
```

Double check Cassandra is still up too (it occasionally needs a restart if it stopped):

```bash
sudo systemctl status cassandra
```

## 9. Fix directory ownership for TheHive

TheHive needs write access to its data directory:

```bash
ls -la /opt/thp
```

If it's owned by `root`, fix it:

```bash
sudo chown -R thehive:thehive /opt/thp
```

Verify:

```bash
ls -la /opt/thp
```

It should now show the `thehive` user/group as owner.

## 10. Configure TheHive's application.conf

```bash
sudo nano /etc/thehive/application.conf
```

Update:

- **Cassandra host**: change `127.0.0.1` (or `localhost`) to theHive Ubuntu 64-bits VM's IP address.
- **Cassandra cluster name**: change to match what you set in `cassandra.yaml` (e.g. `my-dfir`).
- **Elasticsearch host**: change `127.0.0.1` to this VM's IP address.
- **Storage path**: leave as-is, but make sure it points to a directory owned by the `thehive` user/group (see step 9 — this is exactly why that ownership change matters).
- **Application base URL**: change from `localhost` to this VM's IP address (e.g. `http://<thehive-vm-ip>:9000`).

By default, both **Cortex** and **MISP** integrations are enabled in this file — you can leave those as-is for this lab (they're not required for the Wazuh/Shuffle integration).

Save the file.

## 11. Start TheHive

```bash
sudo systemctl start thehive
sudo systemctl enable thehive
sudo systemctl status thehive
```

**If login fails** — if you can't authenticate with the default credentials below, check that **all three services** are running:

```bash
sudo systemctl status cassandra
sudo systemctl status elasticsearch
sudo systemctl status thehive
```

A common culprit is Elasticsearch crashing/stopping due to memory limits. Fix by setting a JVM heap cap:

```bash
sudo nano /etc/elasticsearch/jvm.options.d/jvm.options
```

Add (adjust size to your VM's available RAM — e.g. 2GB on an 8GB box):

```
-Xms2g
-Xmx2g
```

Save, then restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

## 12. Log in

Browse to:

```
http://<theHive-VM-IP-address>:9000
```

Default credentials:

- **Username:** `admin@thehive.local`
- **Password:** `secret`

**Change this password immediately** once logged in, especially before opening any ports to the internet.

---

## 13. Create an organization and users (for the SOAR integration)

By default there's only one organization (**admin**). Create a dedicated one for this project:

1. Click the **+** button (top-left) → **New organization**.
2. Name: `my-dfir` (or your project name).
3. Description: e.g. `SOC automation project`.
4. Confirm, then click into the new organization.

### Create an analyst user

1. **Users → Add user**.
2. Type: **Normal**.
3. Login: e.g. `my-dfir@test.com`.
4. Name: `my-dfir` (or your name).
5. Profile: **analyst**.
6. Save.
7. Highlight the user → **Preview** → scroll to **Set a new password** → set one → confirm.

### Create a service account for Shuffle

1. **Users → Add user**.
2. Type: **Service** (not Normal).
3. Login: e.g. `shuffle@test.com`.
4. Name: `soar` (or similar).
5. Profile: **analyst** for this lab. (In production, create a scoped custom profile following least-privilege rather than reusing `analyst`/`org-admin`.)
6. Save.
7. Highlight the service user → **Preview** → generate an **API key** → copy it and store it securely — this is what Shuffle will use to authenticate to TheHive (see `VirusTotal.md` for the Shuffle-side configuration).

Log out of the admin account and log in as your new analyst user to confirm access — you should now see the **Cases** and **Alerts** views for this organization.

## 14. Reachability for automation testing

Since there's no firewall in front of this VM, port `9000` is already reachable from anything on the same bridged network (including the Windows 10 VM). If Shuffle is cloud-hosted (e.g. shuffler.io) rather than self-hosted on this same network, it will need a path to reach this VM's IP on port `9000` — which typically means either port-forwarding on your router/gateway, or self-hosting Shuffle locally too. Check `VirusTotal.md` for how the Shuffle side connects.

> If you ever do put a firewall in front of this VM (recommended before exposing it beyond your LAN), make sure `9000` stays open to whatever needs to reach TheHive's API.

## Useful commands recap

```bash
sudo systemctl status cassandra
sudo systemctl status elasticsearch
sudo systemctl status thehive
sudo systemctl restart <service>     # after any config change
ls -la /opt/thp                      # verify ownership
```
