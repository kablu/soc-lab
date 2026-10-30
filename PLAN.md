# Laptop SOC Lab — Deep Analysis & Build Plan

> **Target setup:** Windows 11 laptop, 32 GB RAM, WSL2 + Docker experience.
> **Stack:** Wazuh-centric (SIEM + XDR + HIDS in one platform).
> **Goal:** Blue-team / detection-engineering skills.
> _Created: 2026-10-30_

---

## 1. Resource analysis (32 GB budget)

32 GB mein sab kuch ek saath **nahi** chalega. Isliye "always-on core" aur "on-demand" layers:

| Layer | Component | RAM | Kab chalega |
|---|---|---|---|
| **Host reserve** | Windows 11 + Docker Desktop + browser | ~7 GB | Always |
| **SOC core (always-on)** | Wazuh all-in-one (indexer + manager + dashboard) | ~8 GB | Always |
| **Network IDS** | Suricata (Wazuh-integrated) | ~1.5 GB | Always |
| **Victim — Windows** | Win10/11 VM + Sysmon + Wazuh agent | ~4 GB | On-demand |
| **Attacker** | Kali / Parrot VM | ~3–4 GB | On-demand |
| **SOAR + case mgmt** (later) | TheHive + Cortex | ~4 GB | On-demand |
| **Threat intel** (later) | MISP | ~3 GB | On-demand |

**Golden rule:** Wazuh ka **indexer (OpenSearch)** sabse bada RAM-eater hai (JVM heap). Isko ~4 GB heap par cap karo. Ek samay par 1 attacker + 1 victim + core = ~20 GB. SOAR/MISP tab chalao jab victim VMs band hon.

**Kyun Wazuh:**
- Ek platform mein SIEM + HIDS + FIM + vulnerability detection + MITRE ATT&CK mapping + SCA.
- RAM-per-feature sabse accha (ELK se halka, Security Onion se bahut halka).
- Built-in rules + ATT&CK technique IDs har alert par — detection seekhne ke liye ideal.
- Docker Compose se ~10 min mein khada.

---

## 2. Architecture & topology

Windows host par: **Docker Desktop (WSL2 backend)** server stack ke liye, **Hyper-V** victim/attacker VMs ke liye (dono same hypervisor par coexist).

```
                    +-----------------------------------------------+
                    |  Windows 11 Host (32 GB)                      |
                    |                                               |
   +------------+   |   +---------------- Docker (WSL2) ----------+ |
   | Kali VM    |   |   |  WAZUH ALL-IN-ONE                       | |
   | (attacker) |---+--→|  |- wazuh-manager  (rules, decoders)    | |
   | Atomic/    |   |   |  |- wazuh-indexer  (OpenSearch, logs)   | |
   | Caldera    |   |   |  |- wazuh-dashboard(Kibana-like UI)     | |
   +------------+   |   |  Suricata (network IDS) ----------------| |
         | attacks  |   +--------------▲-------------------------- + |
         ▼          |                  | agent events (1514/1515)   |
   +------------+   |                  |                            |
   | Windows    |---+------------------+                            |
   | victim VM  |   |  Sysmon + PowerShell logging + Wazuh agent    |
   +------------+   |                                               |
                    +-----------------------------------------------+
                      Analyst = aap, Wazuh dashboard mein triage
```

**Data flow (SOC ka dil):**
`Endpoint telemetry (Sysmon/EventLog) → Wazuh agent → Manager (decode + rules + ATT&CK tag) → Indexer → Dashboard alert → triage → detection tune → repeat.`

---

## 3. Telemetry sources (detection ka raw material)

Blue-team skills ka 80% yahin. **Acchi telemetry = acchi detection.**

1. **Sysmon** — Olaf Hartong `sysmon-modular` config (ATT&CK-tagged). Process creation, network conn, image load, registry, WMI, named pipes.
2. **Windows Event Logs** — Security (4624/4625/4688/4648), System, Application.
3. **PowerShell logging** — Script Block Logging (4104) + Module Logging + Transcription. Fileless attacks ke liye zaroori.
4. **Wazuh agent native** — FIM, SCA (CIS benchmarks), rootcheck, vulnerability detector.
5. **Suricata** — network IDS, ET Open ruleset. C2 beacons, exploit traffic, DNS anomalies.
6. *(Optional later)* **Zeek** — rich network metadata; **Velociraptor** — endpoint hunting/DFIR.

---

## 4. Phased roadmap

Har phase ka clear **learning outcome** aur **"done" criteria**. Jaldbaazi mat karo.

### Phase 0 — Foundation (Day 1)
- Docker Desktop + WSL2 verify, Hyper-V enable.
- Host-only / internal virtual switch (attacker↔victim isolated; real LAN par malware kabhi nahi).
- Snapshot discipline: har VM ka clean baseline snapshot.
- **Done:** VMs aapas mein ping karein, internet se isolated (ya NAT-gated).

### Phase 1 — SIEM core (Day 2–3)
- Wazuh all-in-one via official `wazuh-docker` compose. Indexer heap 4 GB.
- Dashboard login, certs, single-node tune.
- **Learning:** index lifecycle, alert pipeline, dashboard navigation.
- **Done:** dashboard par green cluster, 0 agents abhi.

### Phase 2 — Endpoints onboard (Day 4–5)
- Windows victim par Wazuh agent + Sysmon (sysmon-modular) + PowerShell logging.
- Wazuh ko Sysmon channel (`Microsoft-Windows-Sysmon/Operational`) ingest karao.
- Linux agent (Kali par bhi).
- **Learning:** log collection config, decoders, kaunsa event kahan.
- **Done:** victim par `whoami` → dashboard mein process-creation event.

### Phase 3 — Network visibility (Day 6)
- Suricata + Wazuh integrate (ET Open rules). Victim traffic monitor.
- **Learning:** network vs endpoint alerts, correlation.
- **Done:** Kali se victim par nmap scan → Suricata alert Wazuh mein.

### Phase 4 — Attack simulation (Week 2) ⭐
- Victim par **Atomic Red Team** (`Invoke-AtomicTest`) — ek-ek ATT&CK technique:
  - T1059.001 (PowerShell), T1055 (process injection), T1003 (cred dumping), T1053 (scheduled task), T1547 (persistence), T1021 (lateral movement).
- Baad mein **Caldera** se multi-step adversary emulation.
- **Learning:** har technique ki telemetry signature.
- **Done:** har atomic test ke baad Wazuh mein us technique ka trail dhoondho — **bina pehle se rule banaye** (raw hunting).

### Phase 5 — Detection engineering loop (Week 2–3) ⭐ core skill
Har attack ke liye:
```
1. Attack chalao (Atomic Red Team)
2. Telemetry mein footprint dhoondho (Sysmon/EventLog/Suricata)
3. Detection likho:
   - Wazuh custom rule (local_rules.xml), YA
   - Sigma rule likho → Wazuh/OpenSearch ke liye convert
4. Attack dobara chalao → alert fire ho
5. False positives tune karo
6. ATT&CK technique ID map karo, document karo
```
- **Sigma** seekhna must — portable, industry-standard detection language.
- **Learning:** detection logic, FP tuning, alert fidelity, coverage gaps.
- **Done:** 15–20 ATT&CK techniques ke liye apni likhi, tuned detections + coverage matrix.

### Phase 6 — Triage, response & maturity (Week 3–4)
- **TheHive + Cortex** (victim VMs band karke RAM free): alert → case → observables → enrichment.
- Incident triage playbook.
- *(Optional)* **MISP** threat intel, IOC match; **Velociraptor** live hunting.
- **Done:** ek end-to-end case: alert → investigate → timeline → contain decision → postmortem.

---

## 5. Success criteria (lab "complete" kab?)

- [ ] Core SOC always-on, endpoints + network telemetry flowing.
- [ ] 20+ ATT&CK techniques simulate kiye, telemetry samjhi.
- [ ] Apni likhi, FP-tuned detections (Wazuh rules + Sigma).
- [ ] ATT&CK coverage matrix (cover vs gap).
- [ ] Ek documented incident case end-to-end.
- [ ] Sab snapshot/compose se ~15 min mein reproducible.

---

## 6. Pitfalls

- ⚠️ **Isolation:** attacker VM kabhi real home LAN par nahi. Internal/host-only switch.
- ⚠️ **Indexer OOM:** heap 4 GB par cap karo.
- ⚠️ **Sab ek saath chalane ki zid:** SOAR/MISP tab chalao jab VMs off hon.
- ⚠️ **Rules copy-paste = "done" nahi:** goal detection *likhna* seekhna hai.
- ⚠️ **Mimikatz/atomic samples:** victim VM par hi, isolated, Defender exclusions/off, snapshot ke baad.
- ⚠️ **No snapshots:** har attack run clean snapshot se.

---

## 7. Integration with existing work

- `owasp-top10-poc` ke vulnerable Spring apps ko **victim targets** banao (real exploit → real SOC alerts).
- WSL / Docker / Sysmon pehle se available.

---

## 8. Next steps

1. [ ] Phase 0 + 1 concrete setup (Docker Compose, Hyper-V switch, indexer heap config).
2. [ ] Phase 4–5 detection-engineering worksheet (ATT&CK → telemetry → Sigma/Wazuh rule template).
3. [ ] Coverage matrix template.
