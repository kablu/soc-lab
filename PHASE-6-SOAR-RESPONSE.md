# Phase 6 — Triage, Response & SOC Maturity

> Goal: alert ko **case** banao, enrich karo, hunt karo, contain decision lo, aur **end-to-end incident** document karo.
> Stack: **TheHive + Cortex** (case mgmt + analyzers) + optional **MISP** (threat intel) + **Velociraptor** (DFIR/hunting).
> Yahan tak aane par aap sirf "alerts dekhne wale" se "incident handle karne wale" ban jaate ho.

---

## 6.0 RAM plan (zaroori)

TheHive stack bhaari hai (Cassandra + Elasticsearch + TheHive + Cortex). Isliye:

- Triage/response kaam ke waqt **victim + attacker VMs band** karo — alerts pehle hi capture ho chuke hain, ab investigation host par.
- Is phase ke liye `.wslconfig` temporarily bump karo:
```ini
[wsl2]
memory=18GB
processors=6
swap=6GB
```
```powershell
wsl --shutdown
```
- Budget: host ~6 + WSL/Docker 18 (Wazuh 8 + TheHive stack ~9) = ~24 GB. VMs off, fits.
- **Lighter alternative:** TheHive ki jagah **DFIR-IRIS** (halka case-mgmt) use kar sakte ho agar RAM tang ho.

---

## 6.1 TheHive + Cortex (Docker)

Ek folder banao `thehive/`, `docker-compose.yml`:
```yaml
version: "3.8"
services:
  cassandra:
    image: cassandra:4.1
    environment:
      - MAX_HEAP_SIZE=1G
      - HEAP_NEWSIZE=1G
      - CASSANDRA_CLUSTER_NAME=thp
    volumes: ["cassandra-data:/var/lib/cassandra"]

  elasticsearch:
    image: elasticsearch:7.17.25
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    volumes: ["es-data:/usr/share/elasticsearch/data"]

  cortex:
    image: thehiveproject/cortex:latest
    depends_on: [elasticsearch]
    ports: ["9001:9001"]

  thehive:
    image: strangebee/thehive:5.2
    depends_on: [cassandra, elasticsearch, cortex]
    ports: ["9000:9000"]
    command:
      - --secret
      - "soc-lab-secret-change-me"
    volumes: ["thehive-data:/opt/thp/thehive/data"]

volumes:
  cassandra-data:
  es-data:
  thehive-data:
```
Up:
```bash
docker compose -f thehive/docker-compose.yml up -d
```
- TheHive: http://localhost:9000 (default admin `admin@thehive.local` / `secret`).
- Cortex: http://localhost:9001 → superadmin banao → organization → enable **analyzers** (free ones: VirusTotal-lite, MISP, Abuse_Finder, etc. API keys jahan chahiye).
- TheHive mein Cortex connect karo (Cortex URL + API key TheHive config/UI mein).

## 6.2 Wazuh → TheHive integration (alert → case auto)

Wazuh manager alert aane par TheHive mein alert bana sakta hai (custom integration script).

`/var/ossec/etc/ossec.conf` (manager) mein:
```xml
<integration>
  <name>custom-w2thive</name>
  <hook_url>http://<host-ip>:9000</hook_url>
  <api_key>THEHIVE_API_KEY</api_key>
  <alert_format>json</alert_format>
  <level>10</level>
</integration>
```
- `custom-w2thive.py` + `custom-w2thive` wrapper `/var/ossec/integrations/` mein (community script; TheHive4py/REST se alert create karta hai).
- `<level>10</level>` = sirf level ≥10 alerts (high-fidelity) TheHive jaayein — noise avoid.
- Restart: `docker compose restart wazuh.manager`.

> Concept yahi hai: **high-severity Wazuh alert → TheHive alert → analyst "Import as Case"**. Manual bhi kar sakte ho shuruaat mein.

## 6.3 MISP — threat intel (optional)

```bash
git clone https://github.com/MISP/misp-docker.git
cd misp-docker
cp template.env .env   # BASE_URL, admin creds edit
docker compose up -d
```
- MISP: https://localhost (default admin `admin@admin.test` / `admin`).
- Feeds enable (CIRCL OSINT, abuse.ch, etc.) → IOCs sync.
- Cortex ka **MISP analyzer** connect → TheHive case ke observables (IP/hash/domain) MISP mein auto-lookup.
- **Value:** aapke Suricata/Sysmon se mile IOCs ko known-bad intel se match karo.

## 6.4 Velociraptor — DFIR & hunting (optional, halka)

Single binary, bahut light (~200 MB RAM).
```bash
# Server (host ya ek VM)
./velociraptor --config server.config.yaml frontend -v
```
- GUI https://localhost:8889.
- Victim par client deploy (MSI) → beacon.
- **Hunts** chalao: "koi process lsass access kar raha?", "ye hash kahan-kahan hai?", "persistence autoruns", "recently created scheduled tasks".
- **Value:** alert ke baad fleet-wide "ye attack aur kahan?" — real IR skill.

---

## 6.5 End-to-end incident (capstone exercise) ⭐

Ek attack lo (e.g. Caldera operation ya T1003.001 LSASS dump) aur **poora SOC workflow** chalao:

```
1. DETECT    Wazuh alert fire (aapka custom rule, Phase 5)
2. TRIAGE    Alert → TheHive case. Severity set. What/when/where?
3. SCOPE     Observables nikaalo: process, hashes, IPs, user, host
4. ENRICH    Cortex analyzers + MISP lookup — IOC known-bad?
5. HUNT      Velociraptor: yahi IOC/behaviour aur kahan? (lateral?)
6. TIMELINE  Sysmon + Suricata + Security logs se attack timeline banao
7. CONTAIN   Decision: host isolate? account disable? (lab mein document)
8. ERADICATE Persistence/artifacts identify + remove plan
9. RECOVER   Snapshot restore = "reimage"
10. POSTMORTEM  Blameless writeup
```

### Triage playbook (har alert par 5 sawaal)
1. **Kya hua?** — rule, technique, raw event.
2. **True ya false positive?** — context (parent process, user, time).
3. **Scope?** — ek host ya zyada? kaunsa user/account?
4. **Severity/impact?** — recon vs cred-access vs lateral.
5. **Next action?** — close / investigate / escalate / contain.

### Postmortem template (`incidents/<date>-<name>.md`)
```markdown
# Incident: <title>
- Date/Time (detection → resolution):
- Severity:
- ATT&CK techniques: Txxxx, Txxxx

## Summary
(2-3 lines: kya hua, kaise pakda)

## Timeline
| Time | Event | Source |
|------|-------|--------|

## Detection
- Kaunse rule/alert ne pakda? (Wazuh ID / Suricata sig)
- Agar der se pakda — gap kahan tha?

## Scope & Impact
- Affected hosts/accounts:
- Data/credential exposure:

## Response actions
- Contain / eradicate / recover steps:

## Root cause
- Initial access vector:

## Lessons & improvements
- Naya detection likha? (rule ID)
- Telemetry gap fill kiya?
- Coverage matrix update:
```

---

## ✅ Phase 6 "Done" criteria
- [ ] TheHive + Cortex up, Cortex analyzers enabled.
- [ ] Wazuh high-sev alert → TheHive alert/case (auto ya manual).
- [ ] (Optional) MISP feeds + Cortex MISP analyzer se IOC enrichment.
- [ ] (Optional) Velociraptor hunt chalaya.
- [ ] Kam se kam **1 end-to-end incident** poora workflow + documented postmortem.
- [ ] Triage playbook use kiya; ek detection gap identify + naya rule likha.

---

## Common issues

| Problem | Fix |
|---|---|
| TheHive start nahi / ES error | ES heap 1g; `vm.max_map_count` (Phase 1); Cassandra ko ~1 min do. |
| Host RAM choke | VMs band; `.wslconfig` 18GB; MISP/Velociraptor alag session mein. |
| Wazuh→TheHive alert nahi bana | API key/URL; `custom-w2thive` script executable; `<level>` threshold; manager logs. |
| Cortex analyzer fail | API keys (free-tier); analyzer enabled + org assigned. |
| MISP slow/heavy | Alag samay chalao; feeds selectively enable. |

---

## 🎓 Lab complete — ab kya?

- **Coverage matrix** ko 30+ techniques tak le jao.
- **MITRE ATT&CK Navigator** layer publish karo (portfolio).
- Detections ko **git repo** mein versioned rakho (Sigma + Wazuh XML + coverage).
- Ek **blog/LinkedIn writeup**: "Built a home SOC — detection engineering lessons".
- Agla level: purple-team (attack+detect same exercise), threat-hunting hypotheses, detection-as-code CI.
