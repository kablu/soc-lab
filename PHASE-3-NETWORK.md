# Phase 3 — Network Visibility (Suricata + Wazuh)

> Goal: network IDS khada karo jo attacker↔victim traffic dekhe, ET Open rules se alert kare, aur alerts Wazuh mein aayein.
> Test: Kali se nmap scan → Wazuh dashboard mein Suricata alert.

---

## Design choice — sensor kahan?

Host-Only network (192.168.56.0/24) par IDS ke 2 tareeke:

| Option | Kaam | Kab |
|---|---|---|
| **A. Dedicated Ubuntu sensor VM** (recommended) | Promiscuous NIC se **poore segment** ka traffic dekhta hai (attacker→victim bhi). `suricata-update` clean chalta hai. "Real SOC" architecture. | Default — isi ko follow karo. |
| **B. Suricata on victim** | Sirf victim ke apne NIC ka traffic. Promiscuous ki zaroorat nahi, par Windows Suricata (Npcap) fiddly. | Agar extra VM nahi chahiye. |

Neeche **Option A** full steps. Option B ka short note end mein.

**RAM note:** sensor ~2 GB. Is phase mein core(8) + victim(4) + kali(4) + sensor(2) = ~18 GB — 32 mein theek. SOAR/MISP band rakho.

---

## 3.1 Ubuntu sensor VM banao

- VirtualBox → new VM: **Ubuntu Server 22.04/24.04**, 2 GB RAM, 1 vCPU, 10 GB disk.
- Network adapter: **Host-Only** (`vboxnet0`, same 192.168.56.0/24 segment).
- **Promiscuous mode = Allow All** (warna doosri VMs ka traffic nahi dikhega):

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "Ubuntu-Sensor" --nicpromisc1 allow-all
```
> GUI se bhi: Settings → Network → Adapter 1 → Advanced → Promiscuous Mode → **Allow All**.

Sensor ko static IP do (e.g. 192.168.56.10). Victim = .20, Kali = .30 maan lo.

---

## 3.2 Suricata install (sensor VM ke andar)

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update && sudo apt install suricata jq -y
```
Monitoring interface ka naam dekho (Host-Only NIC, aksar `enp0s3`/`enp0s8`):
```bash
ip -br link
```

---

## 3.3 Suricata config

`/etc/suricata/suricata.yaml` edit karo:

HOME_NET apne lab subnet par set:
```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.56.0/24]"
    EXTERNAL_NET: "!$HOME_NET"
```
af-packet interface = aapka monitoring NIC (e.g. `enp0s3`):
```yaml
af-packet:
  - interface: enp0s3
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
```
eve.json (ye default ON hota hai) confirm karo `outputs:` mein:
```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert
        - dns
        - http
        - tls
        - flow
```

---

## 3.4 ET Open rules pull karo

```bash
sudo suricata-update
sudo suricata-update list-sources
```
Config test:
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```
Service start + enable:
```bash
sudo systemctl enable --now suricata
sudo systemctl status suricata
```
Live alerts dekhne ke liye:
```bash
sudo tail -f /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'
```

---

## 3.5 Wazuh agent sensor par + eve.json ingest

Agent install (Phase 2 jaisa, Linux):
```bash
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.2-1_amd64.deb
sudo WAZUH_MANAGER="192.168.56.1" WAZUH_AGENT_NAME="net-sensor" dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload && sudo systemctl enable --now wazuh-agent
```
`/var/ossec/etc/ossec.conf` mein eve.json ko json log ke roop mein add karo:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
Agent restart:
```bash
sudo systemctl restart wazuh-agent
```
> Wazuh mein **built-in Suricata decoders + rules** hote hain — alerts automatically `rule.groups: suricata` ke saath dikhte hain. Kuch extra setup nahi.

---

## 3.6 Verify — nmap test (pehla network alert)

Kali (192.168.56.30) se victim (192.168.56.20) par SYN scan:
```bash
sudo nmap -sS -p- 192.168.56.20
```
Dekho:
1. **Sensor par:** `tail -f eve.json | jq` → ET SCAN alerts (e.g. "ET SCAN Nmap" / portscan).
2. **Wazuh dashboard → Discover / Threat hunting:** filter
```
rule.groups: suricata
```
Suricata alerts, src/dst IP, signature, severity dikhe.

Aur bada test — exploit traffic:
```bash
# Kali: ek known-bad pattern (eicar over http) ya metasploit scan
sudo nmap --script vuln 192.168.56.20
```

---

## ✅ Phase 3 "Done" criteria
- [ ] Ubuntu sensor VM, Host-Only + **promiscuous Allow All**.
- [ ] Suricata installed, HOME_NET=192.168.56.0/24, af-packet interface sahi.
- [ ] `suricata -T` config test pass, service running.
- [ ] ET Open rules loaded (`suricata-update`).
- [ ] `net-sensor` Wazuh agent Active, eve.json ingest.
- [ ] Kali nmap scan → Suricata alert dashboard mein (`rule.groups: suricata`).
- [ ] Endpoint (Sysmon) + network (Suricata) alerts dono ek hi dashboard mein — **correlation samjho**.

---

## Common issues

| Problem | Fix |
|---|---|
| Sensor ko doosri VMs ka traffic nahi dikhta | Promiscuous = **Allow All** (step 3.1); saari VMs same Host-Only segment par. |
| `suricata -T` fail | YAML indentation (spaces, no tabs); interface naam `ip -br link` se match. |
| Alerts eve.json mein par Wazuh mein nahi | ossec.conf json localfile + `restart wazuh-agent`; `net-sensor` Active?; manager logs. |
| `suricata-update` fail | `apt install suricata-update` / network (NAT adapter temporarily on). |
| Bahut noisy alerts | `suricata-update` se specific rulesets enable/disable; thresholds tune. |

---

## Option B — Suricata on Windows victim (alternative)

- OISF Windows MSI install (Npcap bundled).
- `C:\Program Files\Suricata\suricata.yaml`: HOME_NET, pcap interface (`suricata.exe --list-interfaces`? / Npcap device).
- eve.json `C:\Program Files\Suricata\log\eve.json`.
- Wazuh agent (already on victim) mein wahi json localfile block, Windows path ke saath.
- Promiscuous ki zaroorat nahi (sirf victim ka apna traffic), par `suricata-update` Windows par aasaan nahi — rules manually.

---

## Next (Phase 4 preview)
Atomic Red Team (`Invoke-AtomicTest`) se ek-ek ATT&CK technique run karo, phir Phase 5 detection-engineering loop: telemetry dhundo → Sigma/Wazuh rule likho → tune karo.
