# Phase 4 + 5 — Attack Simulation & Detection Engineering

> Yahi lab ka **dil** hai. Phase 4 = attacks chalao (Atomic Red Team / Caldera).
> Phase 5 = har attack ki telemetry dhundo → detection likho → tune karo → ATT&CK map karo.
> Goal: 15–20 techniques ke liye apni likhi, FP-tuned detections + coverage matrix.

---

# PHASE 4 — Attack Simulation

## 4.0 Safety first (har baar)

- Attacks **sirf isolated victim VM** par. NAT adapter OFF, sirf Host-Only.
- Defender real-time protection victim par OFF (ya exclusions) — warna atomics block ho jaayenge:
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"
```
- Har attack se pehle clean snapshot; attack ke baad restore.
- Credential-dumping atomics (mimikatz) real tools hai — **isolated VM par hi**, kabhi host par nahi.

## 4.1 Atomic Red Team install (victim, Admin PowerShell)

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

## 4.2 Atomic test chalane ka pattern

```powershell
# 1. Technique ke tests dekho
Invoke-AtomicTest T1059.001 -ShowDetails

# 2. Prereqs check + install
Invoke-AtomicTest T1059.001 -CheckPrereqs
Invoke-AtomicTest T1059.001 -GetPrereqs

# 3. Specific test run
Invoke-AtomicTest T1059.001 -TestNumbers 1

# 4. Cleanup (artifacts hatao)
Invoke-AtomicTest T1059.001 -TestNumbers 1 -Cleanup
```

## 4.3 Starter technique set (blue-team ke liye high-value)

Easy → hard, har ek alag telemetry sikhata hai:

| # | Technique | ATT&CK | Primary telemetry |
|---|---|---|---|
| 1 | PowerShell encoded / download cradle | T1059.001 | Sysmon EID 1 + PowerShell 4104 |
| 2 | Scheduled task persistence | T1053.005 | Sysmon EID 1 (schtasks) / Security 4698 |
| 3 | Registry Run-key persistence | T1547.001 | Sysmon EID 13 (registry) |
| 4 | LSASS memory dump | T1003.001 | Sysmon EID 10 (ProcessAccess → lsass) |
| 5 | Process injection | T1055 | Sysmon EID 8 (CreateRemoteThread) |
| 6 | Service creation | T1543.003 | Security 7045 / Sysmon EID 1 |
| 7 | WMI persistence / exec | T1546.003 / T1047 | Sysmon EID 19/20/21 |
| 8 | Credential access via reg save | T1003.002 | Sysmon EID 1 (reg save SAM) |
| 9 | Defender tampering | T1562.001 | Sysmon EID 1 / Security |
| 10 | Remote service (PsExec) lateral | T1021.002 | Security 4624 type3 / Sysmon |
| 11 | Masquerading (renamed binary) | T1036 | Sysmon EID 1 (OriginalFileName mismatch) |
| 12 | Discovery (whoami/net/ipconfig) | T1033/T1016/T1087 | Sysmon EID 1 |
| 13 | Network scan (from Kali) | T1046 | Suricata (Phase 3) |
| 14 | Ingress tool transfer | T1105 | Sysmon EID 1 + Suricata |
| 15 | Clear event logs | T1070.001 | Security 1102 |

## 4.4 Caldera (multi-step adversary emulation — baad mein)

Host par Docker mein:
```bash
git clone https://github.com/mitre/caldera.git --recursive
cd caldera
docker build --build-arg VARIANT=full -t caldera:latest .
docker run -d -p 8888:8888 --name caldera caldera:latest
```
Dashboard http://localhost:8888 → Sandcat agent victim par deploy → ek operation (adversary profile) chala → poora kill-chain telemetry mein dekho.

---

# PHASE 5 — Detection Engineering Loop

## 5.0 Loop (har technique ke liye repeat)

```
1. Attack chalao            (Invoke-AtomicTest)
2. Telemetry footprint dhundo (Discover: kaunsa event, kaunse fields)
3. Detection likho           (Wazuh local rule  +  Sigma source-of-truth)
4. wazuh-logtest se validate; attack dobara → alert fire
5. False positives tune      (normal activity par na bhadke)
6. ATT&CK map + document      (coverage matrix update)
```

## 5.1 Telemetry dhundhna (step 2)

Dashboard → **Discover**. Useful filters:
```
# Sysmon process create
data.win.system.channel: "Microsoft-Windows-Sysmon/Operational" and data.win.system.eventID: "1"

# PowerShell script block
data.win.system.channel: "Microsoft-Windows-PowerShell/Operational" and data.win.system.eventID: "4104"

# Process access to LSASS
data.win.system.eventID: "10" and data.win.eventdata.targetImage: *lsass.exe*
```
Note karo: kaunsa `win.eventdata.*` field attack ko uniquely identify karta hai (image, commandLine, parentImage, targetImage, etc.).

## 5.2 Wazuh custom rule likhna (step 3a)

Manager mein `/var/ossec/etc/rules/local_rules.xml`. Edit:
```bash
docker compose exec wazuh.manager vi /var/ossec/etc/rules/local_rules.xml
```
(ya host par volume-mount karke apne editor se.)

**Example 1 — T1059.001 suspicious PowerShell:**
```xml
<group name="sysmon,attack,custom_soc,">
  <rule id="100101" level="12">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)\\powershell\.exe$|\\pwsh\.exe$</field>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)-enc\s|-encodedcommand|downloadstring|iex\s|frombase64string|-nop\s.*-w\shidden</field>
    <description>T1059.001: Suspicious PowerShell (encoded/download cradle/hidden)</description>
    <mitre><id>T1059.001</id></mitre>
  </rule>
</group>
```

**Example 2 — T1003.001 LSASS access (credential dumping):**
```xml
<group name="sysmon,attack,custom_soc,">
  <rule id="100102" level="14">
    <if_group>sysmon_event_10</if_group>
    <field name="win.eventdata.targetImage" type="pcre2">(?i)\\lsass\.exe$</field>
    <field name="win.eventdata.grantedAccess" type="pcre2">0x1010|0x1410|0x1438|0x143a|0x1fffff</field>
    <description>T1003.001: Suspicious LSASS process access (possible cred dump)</description>
    <mitre><id>T1003.001</id></mitre>
  </rule>
</group>
```

**Example 3 — T1053.005 scheduled task via schtasks:**
```xml
<group name="sysmon,attack,custom_soc,">
  <rule id="100103" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)\\schtasks\.exe$</field>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)/create</field>
    <description>T1053.005: Scheduled task created via schtasks</description>
    <mitre><id>T1053.005</id></mitre>
  </rule>
</group>
```

## 5.3 Rule validate karo (step 4)

`wazuh-logtest` — rule restart ke bina test:
```bash
docker compose restart wazuh.manager
docker compose exec wazuh.manager /var/ossec/bin/wazuh-logtest
```
Phir ek sample Sysmon JSON event paste karo — dekho kaunsa rule ID match hua, level, decoded fields. Loop ka sabse fast feedback tool yahi hai.

Attack dobara chalao → dashboard mein custom rule ID (100101…) + level + ATT&CK tag dikhe.

## 5.4 Sigma — portable source-of-truth (step 3b)

Sigma = vendor-neutral detection language. **Isko apna master likho**, phir jahan chaaho convert karo.

Install (host/WSL):
```bash
pipx install sigma-cli
sigma plugin install opensearch
```
Sigma rule (`t1059_001_susp_powershell.yml`):
```yaml
title: Suspicious PowerShell Encoded / Download Cradle
id: 7b2c1d40-soc-lab-0001
status: experimental
logsource:
  product: windows
  category: process_creation
detection:
  selection_img:
    Image|endswith:
      - '\powershell.exe'
      - '\pwsh.exe'
  selection_cmd:
    CommandLine|contains:
      - '-enc '
      - 'DownloadString'
      - 'FromBase64String'
      - 'IEX '
  condition: selection_img and selection_cmd
level: high
tags:
  - attack.execution
  - attack.t1059.001
```
Convert to OpenSearch query (dashboard detection ke liye):
```bash
sigma convert -t opensearch -p sysmon t1059_001_susp_powershell.yml
```
> Output query ko Wazuh dashboard (OpenSearch) mein saved search / monitor banao, ya logic ko Wazuh XML rule mein translate karo. Dono maintain karne ki jaroorat nahi — Sigma master rakho, Wazuh XML runtime detection.

## 5.5 False-positive tuning (step 5) — asli skill

- Pehle **broad** rule likho → attack pakdo. Phir **narrow** karo taaki legit activity na bhadke.
- Normal baseline banao: snapshot restore karke clean victim par 10 min normal use (browser, office) → kaunse events "normal" hain note karo.
- Tuning techniques:
  - Parent-child context add karo (`win.eventdata.parentImage`) — e.g. PowerShell `winword.exe` se spawn = suspicious.
  - Known-good paths exclude karo (`<field negate="yes">`).
  - Frequency/threshold rules (`<frequency>`, `<timeframe>`) brute-force/scan ke liye.
  - Level calibrate: recon=5, persistence=10, cred-access/injection=12–14.

**Example — FP tune (PowerShell sirf office-doc se spawn ho tabhi alert):**
```xml
<rule id="100104" level="13">
  <if_sid>100101</if_sid>
  <field name="win.eventdata.parentImage" type="pcre2">(?i)\\(winword|excel|outlook|powerpnt)\.exe$</field>
  <description>T1059.001: PowerShell spawned by Office app (macro execution)</description>
  <mitre><id>T1059.001</id><id>T1566.001</id></mitre>
</rule>
```

## 5.6 Coverage matrix (step 6) — track progress

`detections/coverage.md` maintain karo:

| Technique | Data source | Wazuh rule | Sigma | FP-tuned | Status |
|---|---|---|---|---|---|
| T1059.001 | Sysmon 1 + PS 4104 | 100101/100104 | ✅ | ✅ | Done |
| T1003.001 | Sysmon 10 | 100102 | ✅ | ⬜ | Tuning |
| T1053.005 | Sysmon 1 | 100103 | ⬜ | ⬜ | Draft |
| T1055 | Sysmon 8 | — | ⬜ | ⬜ | TODO |
| T1046 | Suricata | built-in | n/a | ⬜ | Done |

> MITRE ATT&CK Navigator layer bhi bana sakte ho — covered techniques highlight karke visual gap-map.

---

## ✅ Phase 4 + 5 "Done" criteria
- [ ] Atomic Red Team installed, safe-run discipline (snapshot + Defender off + isolated).
- [ ] 15+ techniques simulate kiye (4.3 list).
- [ ] Har technique ki telemetry footprint samjhi (kaunsa event/field).
- [ ] Apni likhi Wazuh custom rules (local_rules.xml, 100xxx).
- [ ] Sigma source-of-truth rules + convert kiya.
- [ ] `wazuh-logtest` se validate, attack-replay par fire.
- [ ] FP tuning kiya (parent-child context, excludes, thresholds).
- [ ] Coverage matrix + ATT&CK Navigator layer.
- [ ] (Optional) Caldera multi-step operation detect kiya.

---

## Common issues

| Problem | Fix |
|---|---|
| Atomic test block ho gaya | Defender off + exclusion (4.0); isolated VM. |
| Attack chala par koi event nahi | Phase 2 channels (Sysmon/PS) + agent Active; Discover time-range check. |
| Custom rule fire nahi hua | `wazuh-logtest` se field names verify; `if_group`/`if_sid` sahi; manager restart. |
| pcre2 match nahi | Wazuh `type="pcre2"`; backslash escape `\\`; case-insensitive `(?i)`. |
| Sab kuch alert kar raha (noisy) | Broad→narrow; parent context + excludes; level calibrate. |
| Sigma convert error | `sigma plugin install opensearch`; logsource/field pipeline `-p sysmon`. |

---

## Next (Phase 6 preview)
TheHive + Cortex se alert→case→triage→response playbook; optional MISP threat intel + Velociraptor hunting; ek end-to-end documented incident.
