# Splunk Free Home Lab — Setup Guide

A practical, end-to-end guide to standing up a free Splunk lab at home and
feeding it real data (Windows + Sysmon). Pairs with the companion reference,
*Splunk Search for Beginners*.

---

## What you're building

```
┌─────────────────────────┐         port 9997          ┌──────────────────────┐
│  Windows endpoint        │  ───────────────────────▶  │  Splunk Enterprise    │
│  • Sysmon (logging)      │   Sysmon events via UF      │  (Free license)       │
│  • Universal Forwarder   │                             │  • indexer            │
└─────────────────────────┘                             │  • search head        │
                                                          │  • Splunk Web :8000   │
                                                          └──────────────────────┘
```

A single Splunk Enterprise instance does everything (indexing, searching, the
web UI). A lightweight **Universal Forwarder (UF)** on a Windows machine ships
Sysmon's event log to it. The two roles can live on **one machine** (install
Splunk and run Sysmon locally) or **two** (a dedicated Splunk box + a separate
Windows endpoint). Two is more realistic; one is simpler to start.

---

## The Free license — read this first

You'll install **Splunk Enterprise**, which starts as a 60-day Enterprise Trial,
then you convert it to the perpetual **Free** license. Know what Free gives and
takes away before you build on it:

**You get:**
- A license that **does not expire**
- **500 MB/day** of indexing (~7 GB of storage per month at that rate)
- Almost all core search, dashboards, and data-input features
- The ability to **receive** data from Universal Forwarders (key for this lab)
- Ability to **bulk-load** a larger one-off dataset up to 2 times per 30 days
  (handy for forensic-style review)

**You lose:**
- **Authentication** — no users or roles; there's *no login*, you're admin
  immediately. (Don't expose this box to the internet.)
- **Alerting / scheduled monitoring** — you can't fire scheduled alerts
- **Forwarding out** in TCP/HTTP to non-Splunk systems (receiving is fine)
- Clustering, distributed search, report acceleration, ingest actions

**The 500 MB/day rule that bites:** exceeding 500 MB in a day = one license
**warning**. Rack up **3 warnings in a rolling 30-day window** and Splunk keeps
*indexing* but **disables search** until you fall back below 3. So tuning your
Sysmon config to stay well under the cap isn't optional housekeeping — it keeps
your lab usable.

---

## Part 1 — Install Splunk Enterprise (the server)

### 1.1 Get the installer

1. Create a free account at **splunk.com** (required to download).
2. Go to **Products → Free Trials & Downloads**.
3. Under **Splunk Enterprise**, pick your OS and download the latest version
   (Splunk Enterprise 10.x at time of writing).
4. Accept the Splunk General Terms; the download starts.

> **Pick your server OS.** Splunk Enterprise runs on Windows or Linux. A Linux
> VM (Ubuntu Server, etc.) is the most production-like and resource-light; a
> Windows install is fine too, especially if you're running everything on one
> machine. Steps for both below.

### 1.2a Install on Windows

1. Run the `splunk-...-x64-release.msi` installer as Administrator.
2. Accept the license; the default install path is
   `C:\Program Files\Splunk`.
3. The installer can run Splunk as the **Local System** account — fine for a
   lab. (For ingesting some Windows logs you may later want a dedicated
   account; Local System works for getting started.)
4. Finish; Splunk starts automatically and opens Splunk Web.

### 1.2b Install on Linux (Ubuntu/Debian example)

```bash
# from the folder where you downloaded the .deb (or use the .tgz)
sudo dpkg -i splunk-*-linux-amd64.deb        # adjust filename

# tell Splunk to start at boot and accept the license
sudo /opt/splunk/bin/splunk enable boot-start --accept-license

# start it
sudo /opt/splunk/bin/splunk start
```

On first start you'll be prompted to create the **admin** username and password.
(For the `.tgz` archive, untar into `/opt/splunk` instead of `dpkg -i`, then run
the same `bin/splunk` commands.)

### 1.3 First login

Open a browser to:

```
http://<splunk-server-ip>:8000
```

Log in with the admin account you created (Windows installer also sets this
during setup). You're now in **Splunk Web**.

> **Note:** Once you switch to the Free license (Part 2), the login disappears
> entirely — Free has no authentication. Until then, the Trial behaves like a
> normal authenticated Enterprise instance.

---

## Part 2 — Switch to the Free license

You can do this any time during the 60-day Trial (and you must, before it
expires, to keep using it for free).

1. In Splunk Web, go to **Settings → Licensing**.
2. Click **Change license group**.
3. Select **Free license** and save.
4. Restart Splunk when prompted.

```bash
# (Linux equivalent restart)
sudo /opt/splunk/bin/splunk restart
```

> **One-way door:** once you move off the Trial you can't go back to it — you'd
> have to install a paid Enterprise license or stay on Free. For a learning lab,
> Free is exactly what you want.

After the restart, browsing to `:8000` drops you straight in as admin with no
password prompt. That's expected on Free.

---

## Part 3 — Basic post-install configuration

### 3.1 Create your indexes

Keeping data in purpose-named indexes (rather than dumping everything in `main`)
is good practice and makes searches faster and tidier.

**Settings → Indexes → New Index.** Create at least:

- `sysmon` — for the Sysmon endpoint data
- (optional) `wineventlog` — for general Windows logs
- (optional) `oslogs` — for Linux/syslog if you add it later

Leave size limits at defaults for a lab; the 500 MB/day license cap will be your
real constraint, not disk.

### 3.2 Enable receiving (so forwarders can connect)

Your Splunk instance needs to listen for forwarder traffic on the standard
receiving port, **9997**.

1. **Settings → Forwarding and receiving.**
2. Under **Configure receiving**, click **Add new**.
3. Enter **9997** and save.

CLI equivalent:

```bash
# on the Splunk server
/opt/splunk/bin/splunk enable listen 9997        # Linux
# "C:\Program Files\Splunk\bin\splunk.exe" enable listen 9997   (Windows)
```

### 3.3 Open the firewall

Allow inbound **TCP 9997** on the Splunk server (and **8000** if you want to
reach the web UI from other machines). On Windows, add inbound rules in
Windows Defender Firewall; on Linux, `ufw allow 9997/tcp` (and `8000/tcp`).

---

## Part 4 — Get Sysmon data in

This reuses the Sysmon pipeline: install Sysmon on the Windows endpoint, install
a Universal Forwarder there to ship its event log, and install the parsing
add-on on the Splunk server.

### 4.1 Install the parsing add-on (on the Splunk server)

Sysmon events arrive as raw Windows XML; the add-on turns them into clean,
CIM-compliant fields.

1. Download the **Splunk Add-on for Sysmon** (Splunkbase **app 5709** — the
   newer *Splunk-supported* one, not the older community "Add-on for Microsoft
   Sysmon," app 1914).
2. In Splunk Web: **Apps → Manage Apps → Install app from file**, upload the
   `.tgz`, and restart when prompted.

### 4.2 Install Sysmon (on the Windows endpoint)

1. Download **Sysmon** from Microsoft Sysinternals.
2. Grab a config — start with **SwiftOnSecurity's `sysmon-config`** (simple,
   well-commented) or **Olaf Hartong's `sysmon-modular`** (modular, ATT&CK-mapped).
3. From an **elevated** prompt, in the folder with both files:

```
sysmon -accepteula -i sysmonconfig.xml
```

Verify events in Event Viewer under
**Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**.

> **Stay under 500 MB/day:** a default Sysmon config on a busy machine can
> generate a lot. The SwiftOnSecurity config is already tuned to cut noise. If
> you still trend high, trim chatty event types (DNS query EventCode 22, image
> loads EventCode 7 are common culprits) in the config and reload with
> `sysmon -c sysmonconfig.xml`.

### 4.3 Install + configure the Universal Forwarder (on the Windows endpoint)

1. Download the **Splunk Universal Forwarder** (separate, lightweight installer —
   *not* full Splunk Enterprise) from splunk.com.
2. Install it. During setup (or after), point it at your Splunk server as the
   **receiving indexer**: host = your Splunk server IP, port = **9997**.

CLI to set the forward target (run from the UF's `bin` folder):

```
splunk add forward-server <splunk-server-ip>:9997
splunk restart
```

3. Tell the UF to collect the Sysmon channel. Create/edit
   `inputs.conf` at
   `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
renderXml = true
index = sysmon
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

4. Restart the forwarder.

### 4.4 Fix the permission gotcha (the #1 "no data" cause)

Newer UF installs run as the virtual account **`NT SERVICE\SplunkForwarder`**,
which by default **cannot read the Sysmon Operational channel** — you'll see a
subscribe failure with `errorCode=5` (Access Denied) and no events arrive.

Fix: add `NT SERVICE\SplunkForwarder` to the **Event Log Readers** group on the
endpoint (or grant it Read on the Sysmon Operational channel's Security
properties). Group Policy is the scalable way if you add more endpoints later.

---

## Part 5 — Verify and search

Back in Splunk Web, confirm data is flowing:

```
index=sysmon | stats count by EventCode
```

If you see counts, you're done. A couple of starter searches:

```
# Process creations with their parents — surfaces odd spawn chains
index=sysmon EventCode=1
| stats count by ParentImage, Image
| sort -count
```

```
# Outbound network connections by process
index=sysmon EventCode=3
| stats count by Image, DestinationIp, DestinationPort
| sort -count
```

Sysmon EventCodes worth knowing: **1** process create, **3** network
connection, **7** image loaded, **8** CreateRemoteThread, **11** file created,
**13** registry set, **22** DNS query.

> **Password hygiene:** Sysmon process-creation events can capture secrets that
> appear in command lines. In a closed home lab that's low-risk, but it's a good
> habit to be aware of — in production you'd mask them with a `SEDCMD` in
> `props.conf` on the indexer.

---

## Part 6 — Keeping the lab healthy on Free

- **Watch your daily volume.** Check **Settings → Licensing** for usage against
  the 500 MB cap. The internal index tracks it; you can search:
  ```
  index=_internal source=*license_usage.log type=Usage
  | timechart span=1d sum(b) AS bytes
  | eval MB = round(bytes/1024/1024, 1)
  ```
- **Trim noisy sources first** if you approach the cap (Sysmon DNS/image-load
  events, then verbose Windows channels).
- **Don't internet-expose** a Free instance — remember, no authentication.
- **Snapshots help.** If it's a VM, snapshot after a clean install + license
  switch so you can roll back when you experiment.

---

## Part 7 — Where to take it next

Once the pipeline works, good lab progressions:

- **Add more data sources** — local Windows Security/System logs, Linux syslog
  (`/var/log` via a UF or `monitor://` input), or a router's syslog.
- **Build dashboards** from your Sysmon searches (process trees, network
  egress, new-service creation).
- **Install Splunk Security Essentials** (free app) for ready-made detections
  to study.
- **Generate benign "attacker" activity** (e.g. Atomic Red Team tests) on the
  endpoint and watch how it appears in Sysmon — the single best way to learn
  detection.
- **Practice the free training** ("Intro to Splunk," "Using Fields," the Search
  Tutorial) against your own data instead of the canned sample set.

---

## Quick command reference

| Task | Command |
|------|---------|
| Start / stop / restart Splunk | `splunk start` / `stop` / `restart` |
| Check Splunk status | `splunk status` |
| Enable receiving on 9997 | `splunk enable listen 9997` |
| Add forward target (on UF) | `splunk add forward-server <ip>:9997` |
| Install Sysmon w/ config | `sysmon -accepteula -i config.xml` |
| Reload Sysmon config | `sysmon -c config.xml` |
| List forwarder's inputs | `splunk list inputstatus` |

*(On Linux, prefix with the path, e.g. `/opt/splunk/bin/splunk ...`; on Windows
use `"C:\Program Files\Splunk\bin\splunk.exe" ...` or the UF's own `bin` path.)*

---

*Splunk Free license details (500 MB/day, no auth, no alerting, receive-only
forwarding, 3-strikes search lockout) reflect Splunk's current published terms.
Verify specifics against splunk.com, as licensing and versions change.*
