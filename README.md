# 🛡️ Home SOC Lab

A laptop-based Security Operations Center for **blue-team / detection-engineering** practice — built on Wazuh, Sysmon, Suricata, and a full detection + incident-response workflow.

> **Host:** Windows 11 Home, 32 GB RAM · WSL2 + Docker Desktop + VirtualBox
> **Stack:** Wazuh (SIEM/XDR) · Sysmon · Suricata · Atomic Red Team · TheHive/Cortex
> **Focus:** learning to *write and tune detections*, not just collect logs.

---

## Architecture

```
                    +-----------------------------------------------+
                    |  Windows 11 Host (32 GB)                      |
                    |                                               |
   +------------+   |   +---------------- Docker (WSL2) ----------+ |
   | Kali VM    |   |   |  WAZUH ALL-IN-ONE                       | |
   | (attacker) |---+--→|  |- wazuh-manager  (rules, decoders)    | |
   | Atomic/    |   |   |  |- wazuh-indexer  (OpenSearch, logs)   | |
   | Caldera    |   |   |  |- wazuh-dashboard(UI, triage)         | |
   +------------+   |   +--------------▲-------------------------- + |
         |          |                  | agent events (1514/1515)   |
         |          |  +------------+  |                            |
         +----------+→ | Ubuntu     |--+  Suricata (ET Open)        |
         | attacks  |  | sensor VM  |     eve.json → Wazuh          |
         ▼          |  +------------+                               |
   +------------+   |                                               |
   | Windows    |---+------------------+                            |
   | victim VM  |   |  Sysmon + PowerShell logging + Wazuh agent    |
   +------------+   |                                               |
                    |   [Phase 6] TheHive + Cortex + MISP (Docker)  |
                    +-----------------------------------------------+

  Data flow:  telemetry → Wazuh agent → manager (decode + rules + ATT&CK)
              → indexer → dashboard alert → triage → tune detection → repeat
```

Lab network: **Host-Only `192.168.56.0/24`** (isolated from real LAN)
`sensor .10 · victim .20 · attacker .30 · manager = host .1`

---

## Build order

| Phase | Doc | What |
|-------|-----|------|
| 0–1 | [PHASE-0-1-SETUP.md](PHASE-0-1-SETUP.md) | Foundation (VMs, network, WSL tuning) + Wazuh SIEM core |
| 2 | [PHASE-2-ENDPOINTS.md](PHASE-2-ENDPOINTS.md) | Wazuh agent + Sysmon + PowerShell logging → telemetry |
| 3 | [PHASE-3-NETWORK.md](PHASE-3-NETWORK.md) | Suricata network IDS integrated with Wazuh |
| 4–5 | [PHASE-4-5-DETECTION.md](PHASE-4-5-DETECTION.md) | ⭐ Atomic Red Team attacks + detection-engineering loop |
| 6 | [PHASE-6-SOAR-RESPONSE.md](PHASE-6-SOAR-RESPONSE.md) | TheHive/Cortex triage, response, incident postmortems |

Full analysis & roadmap: [PLAN.md](PLAN.md)
Study resources (free + paid, mapped to phases): [LEARNING-PATH.md](LEARNING-PATH.md)

---

## Repo layout

```
soc-lab/
├── README.md                     ← you are here
├── PLAN.md                       ← deep analysis + 6-phase roadmap
├── PHASE-0-1-SETUP.md ... PHASE-6-SOAR-RESPONSE.md
├── detections/
│   ├── coverage.md               ← ATT&CK coverage matrix (track progress)
│   ├── wazuh/local_rules.xml     ← custom Wazuh rules (runtime detections)
│   └── sigma/                    ← Sigma rules (portable source-of-truth)
└── incidents/
    └── TEMPLATE.md               ← postmortem template
```

---

## The detection-engineering loop (the whole point)

```
1. Attack          Invoke-AtomicTest Txxxx
2. Find telemetry  which event / fields identify it?
3. Write detection Wazuh rule  +  Sigma source-of-truth
4. Validate        wazuh-logtest → replay attack → alert fires
5. Tune            kill false positives (parent context, excludes, thresholds)
6. Map & track     ATT&CK id + coverage.md
```

Progress is measured in [detections/coverage.md](detections/coverage.md) — techniques covered vs gaps.

---

## Safety rules

- ⚠️ Attacker/malware VMs **never** touch the real LAN — Host-Only only.
- ⚠️ Run real offensive tooling (mimikatz atomics, etc.) **only** in the isolated victim VM, Defender off, from a clean snapshot.
- ⚠️ Cap resources: WSL `.wslconfig` + indexer heap 4 GB; shut VMs when running the SOAR stack.

---

## Status

- [ ] Phase 0–1 · SIEM core
- [ ] Phase 2 · Endpoints
- [ ] Phase 3 · Network
- [ ] Phase 4–5 · Detections (target: 20+ techniques)
- [ ] Phase 6 · Response + 1 end-to-end incident

_Not production security tooling — a learning lab._
