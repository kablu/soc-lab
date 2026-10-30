# Phase 2 — Endpoints Onboard (telemetry flowing)

> Goal: Windows victim par **Wazuh agent + Sysmon (ATT&CK-tagged) + PowerShell logging** → pehla telemetry event dashboard mein.
> Manager = Docker par host ki Host-Only IP **192.168.56.1** (Phase 0 wala network).

---

## 2.0 Manager reachable hai? (host side)

Docker single-node 1514 (events) + 1515 (enrollment) publish karta hai. Windows host firewall par Host-Only se inbound allow karo:
```powershell
New-NetFirewallRule -DisplayName "Wazuh 1514-1515 (lab)" -Direction Inbound -Protocol TCP -LocalPort 1514,1515 -Action Allow -Profile Any
```
Victim se test (VM ke andar):
```powershell
Test-NetConnection 192.168.56.1 -Port 1514
```
`TcpTestSucceeded : True` aana chahiye.

---

## 2.1 Wazuh agent — Windows victim

**Aasaan raasta (recommended):** Dashboard → **Agents → Deploy new agent** → OS=Windows, Server address `192.168.56.1`, agent name `win-victim` → wizard ek PowerShell command dega. VM mein (Admin PowerShell) paste karo.

**Manual:**
```powershell
# MSI download (version apni stack se match karo, e.g. 4.9.2)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi" -OutFile "$env:TEMP\wazuh-agent.msi"
```
```powershell
msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q WAZUH_MANAGER="192.168.56.1" WAZUH_AGENT_NAME="win-victim" WAZUH_REGISTRATION_SERVER="192.168.56.1"
```
Service start:
```powershell
NET START WazuhSvc
```
Dashboard → Agents → `win-victim` **Active** dikhe.

---

## 2.2 Sysmon install (ATT&CK-tagged config)

Sysmon = telemetry ka backbone. Config: **Olaf Hartong sysmon-modular** (ATT&CK-mapped) ya SwiftOnSecurity.

```powershell
# Sysmon (Sysinternals)
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive "$env:TEMP\Sysmon.zip" -DestinationPath "C:\Sysmon" -Force
```
```powershell
# ATT&CK-tagged config (Olaf Hartong merged)
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/olafhartong/sysmon-modular/master/sysmonconfig.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"
```
```powershell
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\sysmonconfig.xml
```
Verify Sysmon chal raha hai:
```powershell
Get-Service Sysmon64; Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 3
```

---

## 2.3 PowerShell logging enable (fileless attacks ke liye)

Script Block + Module + Transcription logging ON (registry, Admin):
```powershell
# Script Block Logging (Event ID 4104)
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1
```
```powershell
# Module Logging (Event ID 4103)
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" -Name EnableModuleLogging -Value 1
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames" -Name "*" -Value "*"
```

---

## 2.4 Wazuh agent ko Sysmon + PowerShell channels ingest karao

Victim par `C:\Program Files (x86)\ossec-agent\ossec.conf` edit karo. `<ossec_config>` ke andar ye blocks add karo:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```
Agent restart:
```powershell
Restart-Service WazuhSvc
```

---

## 2.5 Kali attacker par agent (optional — apne hi actions dekhne ke liye)

Dashboard → Deploy new agent → Linux → command copy, ya manual:
```bash
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.2-1_amd64.deb
sudo WAZUH_MANAGER="192.168.56.1" WAZUH_AGENT_NAME="kali-attacker" dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload && sudo systemctl enable --now wazuh-agent
```

---

## 2.6 Verify telemetry (pehla "SOC moment")

Victim VM mein chalao:
```powershell
whoami
Start-Process notepad.exe
powershell.exe -NoProfile -Command "Write-Output 'soc-test-4104'"
```
Dashboard → **Discover** (ya Agents → win-victim → Security events). Dhundho:
- **Sysmon Event ID 1** (ProcessCreate) — `notepad.exe` / `powershell.exe`.
- **PowerShell 4104** (ScriptBlock) — `soc-test-4104` string.
- Rule par **MITRE ATT&CK technique** tag dikhe.

Quick filter (Discover search bar):
```
data.win.system.channel: "Microsoft-Windows-Sysmon/Operational" and data.win.eventdata.image: *notepad*
```

---

## ✅ Phase 2 "Done" criteria
- [ ] `win-victim` agent **Active** dashboard mein.
- [ ] Sysmon service running, Operational log bharr raha hai.
- [ ] PowerShell Script Block (4104) + Module logging ON.
- [ ] ossec.conf mein Sysmon + PowerShell + Security channels.
- [ ] Victim par `notepad` chalane par dashboard mein ProcessCreate event + ATT&CK tag.
- [ ] (Optional) `kali-attacker` agent Active.
- [ ] **Clean snapshot** dobara lo — ab "Sysmon+agent installed" wala baseline (Phase 0 wala purana replace karo).

---

## Common issues

| Problem | Fix |
|---|---|
| Agent "Never connected" | Firewall 1514/1515 (step 2.0); manager IP sahi (192.168.56.1); `Test-NetConnection`. |
| Agent Active par events nahi | ossec.conf channels add kiye? `Restart-Service WazuhSvc`; manager par `docker compose logs wazuh.manager`. |
| Sysmon Operational log khaali | `Sysmon64.exe -c` se config confirm; service running? |
| 4104 events nahi | Registry keys set + nayi PowerShell session se test. |
| Bahut zyada events / indexer slow | sysmon-modular selectively tune karo, ya noisy Event IDs filter. |

---

## Next (Phase 3 preview)
Suricata (ET Open rules) + Wazuh integrate → Kali se nmap scan par network alert. Phir Phase 4: Atomic Red Team se attacks + detection engineering.
