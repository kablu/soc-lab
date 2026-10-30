# ATT&CK Detection Coverage Matrix

Track which techniques you can simulate **and** detect. This is the scoreboard of the lab.

**Status legend:** `TODO` · `Draft` (rule written, untested) · `Tuning` (fires, FP work pending) · `Done` (fires, FP-tuned, documented)
**Data source:** Sysmon EID / Windows event / Suricata.

> Keep `detections/wazuh/local_rules.xml` and `detections/sigma/` in sync with this table.

---

## Coverage

| # | Tactic | Technique | ATT&CK ID | Data source | Wazuh rule | Sigma | FP-tuned | Status |
|---|--------|-----------|-----------|-------------|-----------|-------|----------|--------|
| 1 | Execution | PowerShell encoded/download cradle | T1059.001 | Sysmon 1 + PS 4104 | 100101 | ✅ | ⬜ | Draft |
| 2 | Execution | Office spawns PowerShell (macro) | T1059.001 / T1566.001 | Sysmon 1 | 100104 | ⬜ | ⬜ | TODO |
| 3 | Persistence | Scheduled task (schtasks) | T1053.005 | Sysmon 1 / Sec 4698 | 100103 | ⬜ | ⬜ | Draft |
| 4 | Persistence | Registry Run key | T1547.001 | Sysmon 13 | — | ⬜ | ⬜ | TODO |
| 5 | Persistence | Service creation | T1543.003 | Sec 7045 / Sysmon 1 | — | ⬜ | ⬜ | TODO |
| 6 | Priv Esc / Cred | LSASS memory access | T1003.001 | Sysmon 10 | 100102 | ⬜ | ⬜ | Draft |
| 7 | Cred Access | SAM/registry hive dump | T1003.002 | Sysmon 1 | — | ⬜ | ⬜ | TODO |
| 8 | Defense Evasion | Disable/tamper Defender | T1562.001 | Sysmon 1 / Sec | — | ⬜ | ⬜ | TODO |
| 9 | Defense Evasion | Clear event logs | T1070.001 | Security 1102 | — | ⬜ | ⬜ | TODO |
| 10 | Defense Evasion | Masquerading (renamed binary) | T1036 | Sysmon 1 | — | ⬜ | ⬜ | TODO |
| 11 | Process | Process injection | T1055 | Sysmon 8 | — | ⬜ | ⬜ | TODO |
| 12 | Exec / Lateral | WMI execution | T1047 | Sysmon 19/20/21 | — | ⬜ | ⬜ | TODO |
| 13 | Lateral | Remote service (PsExec) | T1021.002 | Sec 4624 t3 / Sysmon | — | ⬜ | ⬜ | TODO |
| 14 | Discovery | Account/host/network discovery | T1033 / T1016 / T1087 | Sysmon 1 | — | ⬜ | ⬜ | TODO |
| 15 | Discovery | Network service scanning | T1046 | Suricata | built-in | n/a | ⬜ | TODO |
| 16 | Command&Control | Ingress tool transfer | T1105 | Sysmon 1 + Suricata | — | ⬜ | ⬜ | TODO |

---

## Score

| Metric | Count |
|--------|-------|
| Techniques simulated | 0 / 16 |
| Detections written | 3 |
| FP-tuned & Done | 0 |
| Data sources wired | Sysmon ⬜ · PowerShell ⬜ · Security ⬜ · Suricata ⬜ |

---

## Gaps / next detections to build

- [ ] T1055 process injection (Sysmon EID 8 CreateRemoteThread)
- [ ] T1547.001 Run-key persistence (Sysmon EID 13 registry)
- [ ] T1070.001 log clearing (Security 1102) — high-value, low-FP
- [ ] Export a MITRE ATT&CK Navigator layer of everything `Done`

> Tip: generate a Navigator layer at https://mitre-attack.github.io/attack-navigator/ and commit the JSON here as `navigator-layer.json`.
