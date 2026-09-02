# Person A — Full Technical Instructions & Workflow
### Honeypot Deployment, SIEM Integration, and Sigma Rule Authoring

This is your hands-on, step-by-step build guide. Follow it top to bottom in order — each phase depends on the one before it. Commands assume **Ubuntu 22.04 LTS** on your cloud VM, and that you're comfortable in a terminal (Kali on Parallels works fine for connecting to the VM).

> **Before you touch anything live:** get your containment/risk-acceptance plan signed off by your supervisor first. Everything below results in a real, internet-facing system.

> **Week mapping:** phase headings below are annotated with the week ranges from `FYP.md` Section 11 (Project Timeline), so this document tracks the official schedule phase-for-phase. If a phase runs long, that's fine — just don't lose track of which week you're actually in relative to the plan.

---

## PHASE 0 — Prerequisites Checklist (Weeks 1–2)

- [ ] Oracle Cloud Always Free account created
- [ ] Azure for Students credit activated (backup)
- [ ] Supervisor sign-off on containment plan obtained
- [ ] Shared GitHub repo created with Person B, folder structure agreed
- [ ] SSH key pair generated on your local machine (Kali): `ssh-keygen -t ed25519 -C "fyp-honeypot"`
- [ ] Weekly sync cadence agreed with Person B at each phase handoff point (per `FYP.md` Section 9 risk register mitigation for team coordination)
- [ ] Containment/risk-acceptance plan cross-checked line by line against `FYP.md` Section 8 (Ethics, Legal, and Safety Requirements) before submitting for supervisor sign-off

---

## PHASE 1 — Provision and Harden the VM (Weeks 3–4)

### 1.1 Create the VM
On Oracle Cloud:
- Shape: VM.Standard.E2.1.Micro (Always Free) or a slightly larger Always Free ARM shape if available for headroom
- Image: Ubuntu 22.04 LTS
- Attach your SSH public key during creation (do NOT use password auth)
- Note the public IP assigned

### 1.1a Put the honeypot in its own isolated network segment
`FYP.md` Feature A1 calls out three containment requirements, not just two: egress restriction, **isolated segment**, and non-default admin access. UFW rules (Section 1.5) and SSH hardening (Section 1.4) cover the last two; this step covers the middle one, which is easy to skip on a single free-tier VM.

- Create a **dedicated VCN (or at minimum a dedicated subnet within your existing VCN)** for the honeypot in the Oracle Cloud console — don't drop it into a subnet shared with any other project or personal resources you control.
- Give that subnet its own **Security List** (cloud-level firewall) containing only the rules the honeypot actually needs (inbound 22 and your admin port, outbound DNS/HTTP/HTTPS during setup, later narrowed to just the Wazuh VM's IP on 1514/1515). Treat this as the cloud-side mirror of your UFW rules in 1.5 — both layers should independently enforce the same restriction.
- If you follow the recommended two-VM layout in Phase 3.1 (Wazuh on a separate VM), keep that VM in a **different subnet** from the honeypot, and only allow the specific agent-communication ports between the two subnets — not full routing. This way, even a fully "compromised" honeypot instance has no network path to anything except the narrow Wazuh ingestion port.
- Document this subnet/Security List layout (with a screenshot or exported config) in your containment write-up — it's direct evidence for the "isolated segment" requirement your supervisor is signing off on.

### 1.2 First login and update
```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<VM_PUBLIC_IP>
sudo apt update && sudo apt upgrade -y
```

### 1.3 Create a dedicated non-root admin user (if not already using one)
```bash
sudo adduser youradmin
sudo usermod -aG sudo youradmin
sudo mkdir -p /home/youradmin/.ssh
sudo cp ~/.ssh/authorized_keys /home/youradmin/.ssh/
sudo chown -R youradmin:youradmin /home/youradmin/.ssh
sudo chmod 700 /home/youradmin/.ssh
sudo chmod 600 /home/youradmin/.ssh/authorized_keys
```

### 1.4 Move real SSH to a non-default port and disable password auth
Edit the SSH config:
```bash
sudo nano /etc/ssh/sshd_config
```
Set/change these lines:
```
Port 2200
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
(Pick any unused high port instead of 2200 — just don't use 22 or 2222, since 2222 will be Cowrie's internal port.)

Restart SSH:
```bash
sudo systemctl restart sshd
```
**Test in a NEW terminal window before closing your current session:**
```bash
ssh -i ~/.ssh/id_ed25519 -p 2200 youradmin@<VM_PUBLIC_IP>
```
Only close your original session once the new one works.

### 1.5 Configure the firewall (UFW + cloud security list)
On the VM:
```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default deny outgoing   # we will selectively allow what's needed
sudo ufw allow 2200/tcp          # your real admin SSH port
sudo ufw allow 22/tcp            # public-facing "bait" port -> will route to Cowrie
sudo ufw allow out 53            # DNS
sudo ufw allow out 80/tcp        # HTTP (for updates/downloads during setup)
sudo ufw allow out 443/tcp       # HTTPS (for updates/downloads during setup)
sudo ufw enable
sudo ufw status verbose
```
Also open the matching **inbound** ports (22, 2200) in Oracle Cloud's Security List / Network Security Group in the web console — cloud-level firewall rules sit in front of UFW and will block traffic even if UFW allows it.

> **Egress lockdown note:** Once Cowrie is installed and tested (Phase 2), tighten `ufw default deny outgoing` further — Cowrie itself doesn't need outbound internet access to do its job (it fakes command output locally), so after setup you can remove the `allow out 80/443` rules if you don't need ongoing package updates, or restrict them to specific package-mirror IPs. Document this decision either way in your containment write-up.

---

## PHASE 2 — Install and Configure Cowrie (Weeks 3–4)

### 2.1 Install dependencies
```bash
sudo apt install git python3-venv python3-dev libssl-dev libffi-dev build-essential -y
```

### 2.2 Create a dedicated, unprivileged user for Cowrie (never run it as root)
```bash
sudo adduser --disabled-password cowrie
sudo su - cowrie
```

### 2.3 Clone and set up Cowrie (as the `cowrie` user)
```bash
git clone https://github.com/cowrie/cowrie.git
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### 2.4 Configure Cowrie
```bash
cp etc/cowrie.cfg.dist etc/cowrie.cfg
nano etc/cowrie.cfg
```
Key settings to review/set:
- `hostname` — set something generic/believable (e.g., `srv-prod-01`), not something that reveals it's a honeypot
- `listen_endpoints = tcp:2222:interface=0.0.0.0` — Cowrie's internal SSH listener (default 2222 is fine, since it's never exposed directly to the internet — port 22 will be redirected to it)
- Under `[output_jsonlog]`, confirm it's enabled — this is what feeds Wazuh later:
  ```
  [output_jsonlog]
  enabled = true
  logfile = ${honeypot:log_path}/cowrie.json
  ```

### 2.5 Set up fake filesystem and credentials
```bash
# Fake filesystem contents (already comes with a default seed):
ls share/cowrie/fs.pickle

# Configure which usernames/passwords the honeypot will "accept"
nano etc/userdb.txt
```
Add a small set of believable weak credentials (e.g., `root:x:123456`, `admin:x:admin123`) — this is what invites successful "break-ins" so you capture post-login behavior, not just login attempts.

**On service emulation believability** (the other half of Feature A1, beyond containment): the default `fs.pickle` seed and default `cowrie.cfg` values are recognizable to anyone who's looked at Cowrie before. Before going live, at minimum: set `hostname` to something plausible for your fake server's stated role, edit the fake filesystem contents (`bin/createfs` or manually editing entries) so `/etc/hostname`, `/etc/issue`, and a couple of home-directory files look lived-in rather than empty/default, and check the SSH banner Cowrie presents doesn't leak an obvious "this is Cowrie" version string. Note in your log/write-up which of these you changed — it's evidence for the believability claim in Feature A1, not just a cosmetic step.

### 2.6 Start Cowrie
```bash
bin/cowrie start
bin/cowrie status
```
Logs will appear at `var/log/cowrie/cowrie.log` and `var/log/cowrie/cowrie.json`.

### 2.7 Redirect port 22 → Cowrie's port 2222
Exit back to your admin user, then:
```bash
exit   # back to youradmin
sudo apt install iptables-persistent -y
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
sudo netfilter-persistent save
```

### 2.8 Test end to end
From your **local Kali machine** (not from inside the VM):
```bash
ssh root@<VM_PUBLIC_IP>
# should land in Cowrie's fake shell, log in with one of your userdb.txt credentials
```
Then confirm your real access still works separately:
```bash
ssh -i ~/.ssh/id_ed25519 -p 2200 youradmin@<VM_PUBLIC_IP>
```
Check that your test session appears in the log:
```bash
tail -f /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```

### 2.9 Auto-start on reboot
```bash
sudo su - cowrie
cd cowrie
crontab -e
```
Add:
```
@reboot cd /home/cowrie/cowrie && bin/cowrie start
```

**Milestone check:** Cowrie is live, port 22 redirects to it, your real admin access works independently, and JSON logs are being written.

### 2.10 Contingency: widening the attack surface with T-Pot
`FYP.md`'s risk register flags "low attacker traffic volume during collection window" as a Medium/Medium risk, with the mitigation "deploy as early as possible; consider T-Pot's broader service suite to widen attack surface." Cowrie-only (SSH/Telnet) is the baseline scope and is fine to start with — but keep this in your back pocket: if by the start of Week 6 (end of SIEM Integration phase) your session volume looks thin, evaluate switching to or supplementing with **T-Pot** (Docker-based, 20+ emulated services — `FYP.md` Section 4) before the Week 7 data-collection window opens, since collection time lost after Week 7 is harder to recover. This is a scope decision, not a hard requirement — note whichever way you decide (and why) in your write-up.

---

## PHASE 3 — Install and Configure Wazuh (Weeks 5–6)

### 3.1 Decide placement
If your VM has ≥4GB RAM, you can co-locate Wazuh on the same box. If it's tight (common on Always Free micro shapes), provision a **second** free-tier VM (Oracle or Azure) for Wazuh, and only keep Cowrie on the honeypot box. This is the recommended setup — it also means a Wazuh compromise scenario is irrelevant since Wazuh isn't internet-facing/bait itself.

### 3.2 Install Wazuh (single-node, all-in-one installer)
On the Wazuh server VM:
```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
This installs the Wazuh indexer, manager, and dashboard together. At the end it prints the **admin password** — save it securely (e.g., add it to a private, gitignored notes file — never commit credentials to your GitHub repo).

### 3.3 Access the dashboard
Open `https://<WAZUH_VM_IP>` in a browser, log in with `admin` and the password from the installer output. Lock this down in your cloud firewall to only your own IP if possible, since it's a sensitive management interface.

### 3.4 Install the Wazuh agent on the honeypot VM
On the **honeypot** VM:
```bash
curl -o wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.x-1_amd64.deb
sudo WAZUH_MANAGER='<WAZUH_VM_IP>' dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
Confirm outbound firewall rule from honeypot → Wazuh VM on port 1514/1515 (agent communication) is allowed in UFW — this is a deliberate, narrow exception to your egress lockdown (Wazuh's own IP only, not general internet).

### 3.5 Point the agent at Cowrie's log file
Edit the agent config on the honeypot VM:
```bash
sudo nano /var/ossec/etc/ossec.conf
```
Inside `<ossec_config>`, add:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/home/cowrie/cowrie/var/log/cowrie/cowrie.json</location>
</localfile>
```
Restart the agent:
```bash
sudo systemctl restart wazuh-agent
```

### 3.6 Confirm the agent shows as connected
On the Wazuh manager:
```bash
sudo /var/ossec/bin/agent_control -l
```
It should list your honeypot agent as **Active**.

---

## PHASE 4 — Custom Decoders and Rules for Cowrie (Weeks 5–6)

### 4.1 Create a custom decoder
On the Wazuh **manager**:
```bash
sudo nano /var/ossec/etc/decoders/local_decoder.xml
```
Add a decoder that recognizes Cowrie's JSON structure and pulls out key fields (adjust field names to match Cowrie's actual JSON keys, e.g. `eventid`, `src_ip`, `input`, `username`, `password`):
```xml
<decoder name="cowrie">
  <program_name>cowrie</program_name>
</decoder>

<decoder name="cowrie-json">
  <parent>cowrie</parent>
  <plugin_decoder>JSON_Decoder</plugin_decoder>
</decoder>
```

### 4.2 Create custom rules
```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```
Example starter rules (expand these as you learn Cowrie's exact event types — `cowrie.login.failed`, `cowrie.login.success`, `cowrie.command.input`, `cowrie.session.file_download`):
```xml
<group name="cowrie,honeypot,">

  <rule id="100100" level="3">
    <decoded_as>cowrie-json</decoded_as>
    <field name="eventid">cowrie.session.connect</field>
    <description>Cowrie: new honeypot session opened</description>
  </rule>

  <rule id="100101" level="5">
    <decoded_as>cowrie-json</decoded_as>
    <field name="eventid">cowrie.login.success</field>
    <description>Cowrie: attacker successfully "logged in"</description>
  </rule>

  <rule id="100102" level="10">
    <decoded_as>cowrie-json</decoded_as>
    <field name="eventid">cowrie.session.file_download</field>
    <description>Cowrie: file dropped/downloaded by attacker — investigate</description>
  </rule>

  <rule id="100103" level="6">
    <decoded_as>cowrie-json</decoded_as>
    <field name="eventid">cowrie.command.input</field>
    <description>Cowrie: attacker executed a command</description>
  </rule>

</group>
```

### 4.3 Restart the manager and validate
```bash
sudo /var/ossec/bin/wazuh-control restart
sudo tail -f /var/ossec/logs/ossec.log
```
Generate a fresh test session against the honeypot from your Kali machine, then confirm matching alerts appear in the Wazuh dashboard under **Security Events**.

---

## PHASE 5 — Build the Four Dashboard Views (Week 6)

In the Wazuh dashboard (built on OpenSearch Dashboards), go to **Dashboards Management → create visualizations**, using your decoded Cowrie fields as the data source:

1. **Session volume over time** — Line/bar chart, X-axis = `@timestamp` (date histogram), Y-axis = count, filtered to `eventid: cowrie.session.connect`.
2. **Attacker geolocation** — If you enable GeoIP enrichment (Wazuh supports a GeoIP module, or you can enrich `src_ip` via a lookup), use a **Coordinate Map** or **Region Map** visualization. The live dashboard can show IP-level detail for your own operational monitoring, but per `FYP.md` Section 8.4 (data minimization/privacy), anything that leaves this dashboard for the thesis or a public report — screenshots, tables, figures — must aggregate to country/region level, never list individual attacker IPs or attempt attacker identification.
3. **Command-frequency analysis** — A **Data Table** or **Terms aggregation** bar chart on the `input` field (the command text), filtered to `eventid: cowrie.command.input`, sorted by count descending.
4. **File-drop events** — A dedicated **Data Table** filtered to `eventid: cowrie.session.file_download`, showing timestamp, source IP, and filename/hash.

Combine all four into one **Dashboard** view for quick daily monitoring.

### 5.1 Set up the file-drop alert
Wazuh → **Rules → Manage rules**, confirm rule `100102` (or your equivalent) is set to a high enough level to trigger a notification. Configure email/Slack alerting under **Wazuh → Settings → Alerts** if you want a push notification rather than checking the dashboard manually (optional but recommended so you don't miss samples for Person B).

---

## PHASE 6 — Daily Operations During Data Collection (Weeks 7–9)

Repeat this routine every 1–2 days during the collection window:

1. Check the dashboard for new sessions and any file-drop alerts.
2. For any file-drop event, retrieve the captured file from the honeypot:
   ```bash
   ls /home/cowrie/cowrie/var/lib/cowrie/downloads/
   ```
   Hand the file (and its SHA-256 hash) to Person B for triage.
3. For any session you judge "notable" (successful login + real command activity, not just a login attempt), log it manually in a shared tracking sheet/doc in your repo:
   - Timestamp, source IP, session ID, short summary of what happened.
4. Watch for **repeating behavior patterns** across multiple attacker sessions — these become your Sigma rule candidates in Phase 7. Note them as you spot them rather than trying to remember later.
5. Spot-check that Cowrie and the Wazuh agent are both still running:
   ```bash
   # on honeypot VM
   sudo su - cowrie -c "cd cowrie && bin/cowrie status"
   sudo systemctl status wazuh-agent
   ```

---

## PHASE 6a — Supporting Malware Analysis Handoff (Weeks 10–11)

`FYP.md` Section 11's timeline gives you a specific job during Person B's Malware Analysis phase, separate from your own Rule Development phase which starts in Week 12: **"support log correlation per flagged sample."** It's easy to miss this because your own headline deliverables (Sigma rules) don't start until Phase 7 — but Person B needs this from you *during* Weeks 10–11, not after.

For each file Person B is actively reverse-engineering:

1. Identify the session ID and source IP tied to that file's `cowrie.session.file_download` event (from your Phase 6 tracking sheet or the Wazuh dashboard's file-drop table).
2. Pull the **full session timeline** around that event from Wazuh (Discover, filtered to that session ID): every command the attacker ran before and after the download, timestamps, and any follow-on events (e.g., execution attempts, further downloads).
3. Hand that timeline to Person B alongside the sample, so their static/dynamic RE findings (what the binary actually does) can be cross-checked against what you actually observed it do live (what commands triggered it, what happened after).
4. Keep informally logging any **repeating behavioral patterns** you notice across sessions during this window — these become your Sigma rule candidates the moment Phase 7 starts, so don't wait until Week 12 to start noticing them.

---

## PHASE 7 — Writing and Testing Sigma Rules (Weeks 12–13)

### 7.1 Install the Sigma toolchain (on your own Kali machine or a working directory in the repo)
```bash
pip install pysigma sigma-cli
```

### 7.2 Sigma rule structure
A Sigma rule is a YAML file. Basic skeleton:
```yaml
title: Cowrie - Download then Execute Pattern
id: <generate-a-uuid>
status: experimental
description: Detects attacker downloading a file and immediately attempting to execute it, observed on honeypot sessions.
author: <your name>
date: 2026/XX/XX
logsource:
  product: cowrie
  service: honeypot
detection:
  download:
    eventid: 'cowrie.session.file_download'
  execute:
    eventid: 'cowrie.command.input'
    input|contains:
      - 'chmod +x'
      - './'
  condition: download and execute
level: high
tags:
  - attack.execution
  - attack.t1059
```
Write one YAML file per rule candidate you identified in Phase 6, aiming for **at least 3 distinct rules** (Objective O5). Store them in your repo under `/sigma-rules/`.

### 7.3 Convert Sigma → Wazuh-native rule format
```bash
sigma-cli convert -t wazuh -p wazuh ./sigma-rules/download-then-execute.yml
```
(If a direct Wazuh backend isn't available in your installed Sigma toolchain version, convert to a close-enough intermediate like a generic Elastic/OpenSearch query format, then hand-adapt the logic into a Wazuh `local_rules.xml` entry using the same field-matching approach as Phase 4.3 — document this translation step clearly in your methodology, since it's a legitimate part of your process.)

### 7.4 Deploy into Wazuh
Add the converted logic into `/var/ossec/etc/rules/local_rules.xml` on the manager, following the same `<rule>` structure as your Phase 4 rules, referencing your Sigma rule's ID/title in the `<description>` for traceability. Restart the manager and re-test against a known matching session.

### 7.5 Tune for false positives
Run each rule against a held-out slice of your corpus (sessions you haven't used to build the rule). Check the Wazuh dashboard for unexpected fires. If a rule is too broad (matching harmless sessions), tighten the `detection` logic (e.g., require a specific command string rather than any `chmod +x`).

---

## PHASE 8 — Evaluation Workflow (Joint with Person B) (Weeks 14–15)

1. Export your full corpus of labeled sessions (ground truth: malicious vs. benign, agreed with Person B) as a CSV or structured export from Wazuh (**Discover → Export**).
2. Run your custom Sigma-derived rules against the corpus; record which sessions fired.
3. Separately, import and run **SigmaHQ's public rules** (filtered to any that are log-source-compatible with SSH/honeypot-style data) against the same corpus as your baseline comparison.
4. Build a results table (TP/FP/FN/TN per rule, per rule set) — a spreadsheet in your repo's `/evaluation/` folder works well.
5. Calculate precision, recall, false-positive rate, F1, coverage delta, and time-to-detection for both sets, and write up the comparison.

---

## PHASE 9 — Thesis Write-up: Your Chapters (Weeks 16–17)

Per `FYP.md` Section 11, you own drafting the **Architecture, Containment, and Ethics chapters** (Person B owns Analysis and Detection-Engineering chapters; results/discussion and defense prep are joint). This phase is writing, not building, so there's no command list — but don't skip it out of this document just because it's not technical:

- **Architecture chapter:** write up from `FYP.md` Section 3 (system architecture, layer definitions, data flow diagram) plus your actual as-built configuration — call out anywhere your real deployment diverged from the original plan and why.
- **Containment chapter:** write up from Section 1.1a (isolated segment), Section 1.4–1.5 (egress/admin hardening), and Section 8 (ethics/legal/safety requirements) — this chapter is effectively the evidence trail for the containment plan your supervisor signed off on in Phase 0, so reference the actual configs/screenshots you produced along the way.
- **Ethics chapter:** work through `FYP.md` Section 8 point by point (authorization, non-participation in harm, safe sample handling, data minimization, responsible disclosure posture, institutional review) and document how each was actually satisfied in practice, not just as a plan.
- Feed your Phase 8 Sigma-vs-default evaluation results into the joint results/discussion chapter alongside Person B's YARA-vs-default results.

## PHASE 10 — Joint Final Review (Week 18)

- Joint revision pass on the full thesis with Person B.
- Rehearse the live demo end-to-end: a fresh attacker session hitting the honeypot → visible in the Wazuh dashboards (Phase 5) → a custom Sigma rule firing on it (Phase 7) → that result traceable into your Phase 8 evaluation numbers. If anything in that chain is flaky (stale dashboard, a rule that doesn't reliably fire), fix it before the defense, not during it.
- Confirm the repo (`/honeypot/`, `/siem/`, `/sigma-rules/`, `/evaluation/`, `/docs/`) is in a state a stranger could clone and follow, per the "reproducible by future students" outcome in `FYP.md` Section 12.

---

## Troubleshooting Quick Reference

| Symptom | Likely cause | Check |
|---|---|---|
| Can't reach port 22 at all | Cloud security list blocking it | Check Oracle Cloud console Security List, not just UFW |
| Port 22 connects but not to Cowrie | iptables redirect not applied/persisted | `sudo iptables -t nat -L -n`, re-run `netfilter-persistent save` |
| Real admin SSH (2200) not working | sshd_config typo or firewall | `sudo sshd -t` to test config syntax; check UFW/cloud rules for 2200 |
| No Cowrie logs appearing | Cowrie not running, or logfile path wrong | `bin/cowrie status`; check `etc/cowrie.cfg` `[output_jsonlog]` path |
| Wazuh agent shows "Never connected" | Firewall blocking 1514/1515, or wrong manager IP | Re-check `WAZUH_MANAGER` value used at install, UFW outbound rule |
| Wazuh dashboard shows no honeypot events | Decoder/rule mismatch with actual JSON field names | `sudo tail -f /var/ossec/logs/ossec.log`, inspect raw `cowrie.json` field names directly |
| Sigma rule never fires in Wazuh | Field name mismatch between Sigma logsource and your decoder's actual field names | Compare Sigma `detection` field names against your `local_decoder.xml`/`local_rules.xml` field names exactly |

---

## Repo Structure Reminder

```
/honeypot/          -> Cowrie config notes, hardening scripts, iptables rules
/siem/              -> Wazuh decoders, rules, dashboard exports
/sigma-rules/        -> Your original .yml Sigma rules + conversion notes
/evaluation/          -> Ground truth data, results tables, metrics scripts
/docs/                -> Containment plan, session logs, thesis drafts
```

Commit configuration files and rules regularly — never commit credentials, private keys, or the Wazuh admin password.
