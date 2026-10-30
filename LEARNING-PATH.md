# 📚 SOC / Blue-Team Learning Path

A curated study path mapped to this lab's 6 phases. Mix of **free** and **paid** resources.
Use the lab as the *practice*; these as the *theory + guided wal-throughs*.

> _Prices/availability change — verify on the provider's site. 🆓 = free (or free tier), 💲 = paid, 🎥 = video, 📖 = book, 🧪 = hands-on._

---

## 0. Orientation (before you start)

| Resource | Type | Why |
|---|---|---|
| **MyDFIR — "SOC Analyst" YouTube series** | 🎥🆓 | Builds a Wazuh/ELK home lab end-to-end — mirrors THIS lab almost exactly. Start here. |
| **Simply Cyber (Gerald Auger)** | 🎥🆓 | Blue-team careers, daily threat brief, roadmap sanity. |
| **MITRE ATT&CK website — Getting Started** | 🆓 | The framework every phase references. Learn tactics vs techniques. |
| **Pyramid of Pain (David Bianco)** | 🆓📖 | Core mental model for detection value. |

---

## Phase 0–1 · SIEM Core (Wazuh)

| Resource | Type | Maps to |
|---|---|---|
| **Wazuh official docs — "Proof of Concept guide"** | 🆓📖 | Installing/understanding the exact stack you built |
| **TryHackMe — "Intro to SIEM", "Wazuh"** rooms | 🧪💲 | SIEM concepts + Wazuh hands-on |
| **Splunk — Search Fundamentals / Intro to Splunk** | 🧪🆓 | SIEM query thinking (transfers to OpenSearch) |
| **Antisyphon / Black Hills — "SOC Core Skills" (John Strand)** | 🎥💲 (pay-what-you-can) | Foundational SOC workflow, logs, tooling |

**Skill goal:** ingest logs, navigate a SIEM, read an alert pipeline.

---

## Phase 2 · Endpoints & Sysmon

| Resource | Type | Maps to |
|---|---|---|
| **13Cubed — "Investigating Windows Endpoints" + Sysmon videos** | 🎥🆓/💲 | Sysmon event IDs, what each tells you — gold standard |
| **TryHackMe — "Sysmon", "Windows Event Logs" rooms** | 🧪💲 | Hands-on endpoint telemetry |
| **Olaf Hartong — sysmon-modular (GitHub + blog)** | 🆓📖 | The ATT&CK-tagged config you deployed |
| **SwiftOnSecurity sysmon-config (annotated)** | 🆓📖 | Learn *why* each Sysmon setting matters |

**Skill goal:** read Sysmon EID 1/3/8/10/13, PowerShell 4104, map to behaviour.

---

## Phase 3 · Network Security Monitoring

| Resource | Type | Maps to |
|---|---|---|
| **"The Practice of Network Security Monitoring" — Richard Bejtlich** | 📖💲 | *The* NSM book — concepts behind your sensor |
| **"Applied Network Security Monitoring" — Chris Sanders** | 📖💲 | Zeek/Suricata, detection mechanics |
| **TryHackMe — "Snort", "Suricata", "Zeek/Brim", "Network Security" rooms** | 🧪💲 | Hands-on IDS |
| **Suricata official docs + Emerging Threats rules** | 🆓📖 | Rule syntax, eve.json, tuning |
| **Malware-Traffic-Analysis.net** | 🧪🆓 | Real pcaps to analyze (free exercises) |

**Skill goal:** read Suricata alerts, correlate network + endpoint.

---

## Phase 4–5 · Attack Simulation & Detection Engineering ⭐

| Resource | Type | Maps to |
|---|---|---|
| **MAD20 / MITRE ATT&CK Defender — "ATT&CK Fundamentals"** | 🧪🆓 | Free official ATT&CK training (+ paid certs) |
| **Atomic Red Team — docs + GitHub** | 🆓🧪 | The attack simulations you run (`Invoke-AtomicTest`) |
| **SigmaHQ — docs + rule repo** | 🆓📖 | Writing portable detections (your source-of-truth) |
| **Red Canary — "Atomic Red Team" + Threat Detection Report** | 🆓📖 | How pros map techniques → detections |
| **The DFIR Report** | 🆓📖 | Real intrusions with the exact telemetry to detect |
| **detection.fyi / Florian Roth blog** | 🆓📖 | Detection-engineering craft, Sigma at scale |
| **TryHackMe — "MITRE", "Threat Emulation", "Pyramid of Pain" rooms** | 🧪💲 | Guided technique→detection practice |
| **MITRE ATT&CK Navigator** | 🆓🧪 | Build your coverage layer (see coverage.md) |

**Skill goal:** run a technique → find telemetry → write + tune a Sigma/Wazuh rule.

---

## Phase 6 · Triage, Response & Incident Handling

| Resource | Type | Maps to |
|---|---|---|
| **LetsDefend — "SOC Analyst" path** | 🧪💲 | Real alert queue, triage practice (closest to a real SOC job) |
| **Blue Team Labs Online (BTLO)** | 🧪💲 | Investigations + incident challenges |
| **CyberDefenders** | 🧪🆓/💲 | Blue-team CTF, DFIR challenges |
| **"Intelligence-Driven Incident Response" — Roberts & Brown** | 📖💲 | IR + threat intel methodology |
| **"Blue Team Handbook: Incident Response Edition"** | 📖💲 | Pocket IR playbook |
| **TheHive / Cortex / Velociraptor official docs** | 🆓📖 | The response tooling you deploy |

**Skill goal:** alert → case → scope → contain → postmortem.

---

## 🎓 Certification ladder (optional, by level)

| Level | Cert | Notes |
|---|---|---|
| Entry | **CompTIA Security+** | Baseline, HR-friendly |
| SOC-focused | **Blue Team Level 1 (BTL1)** — Security Blue Team | Hands-on, very respected for SOC roles |
| Analyst | **CompTIA CySA+** | Detection/analytics focus |
| Advanced | **GIAC GCIA / GMON / GCDA** | Deep NSM / continuous monitoring / detection (pricey) |
| Detection Eng | **MAD20 ATT&CK certifications** | Purple-team / detection focused |

---

## 🗓️ Suggested 8-week schedule (≈6–8 hrs/week)

| Week | Focus | Lab phase |
|---|---|---|
| 1 | SIEM concepts + stand up Wazuh | 0–1 |
| 2 | Endpoints, Sysmon deep-dive | 2 |
| 3 | Network monitoring, Suricata | 3 |
| 4 | ATT&CK fundamentals + first atomics | 4 |
| 5 | Detection engineering: write 5 rules | 5 |
| 6 | Detection engineering: tune + 10 more | 5 |
| 7 | TheHive, triage, 1 full incident | 6 |
| 8 | Capstone: coverage matrix + Navigator layer + writeup | all |

---

## 💡 Learning method (so it sticks)

1. **Watch/read** the resource for a phase.
2. **Do it in the lab** immediately (the phase doc).
3. **Write a detection** for what you learned (coverage.md).
4. **Document** one thing you didn't know before.
5. Repeat. Theory without the lab fades; the lab makes it permanent.

---

## Free-only fast track (zero budget)

MyDFIR YouTube → Wazuh docs → 13Cubed Sysmon → Suricata docs + Malware-Traffic-Analysis → MAD20 ATT&CK Fundamentals (free) → Atomic Red Team + SigmaHQ → The DFIR Report → CyberDefenders (free challenges). Everything else in this lab is open-source.

> _Coursera connector authorize ho jaaye (`/mcp` → Coursera) to bolna — main live catalog se specific courses + enrollment links bhi nikaal dunga._
