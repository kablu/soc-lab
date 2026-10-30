# Phase 0 + 1 — Concrete Setup

> Host: **Windows 11 Home Single Language**, 32 GB RAM, WSL2 + Docker Desktop.
> ⚠️ **Hyper-V is NOT available on Windows Home.** VMs ke liye **VirtualBox** use karenge.
> Docker Desktop (WSL2 backend) aur VirtualBox saath chal sakte hain — VirtualBox "Hyper-V/WHP" paravirt mode mein chalega (thodi slower, lab ke liye theek).

---

## PHASE 0 — Foundation

### 0.1 Pre-flight checks (PowerShell, as Admin)

Windows edition confirm:
```powershell
Get-ComputerInfo | Select-Object WindowsProductName, OsHardwareAbstractionLayer, CsTotalPhysicalMemory
```

Virtualization BIOS mein enabled hai?
```powershell
Get-CimInstance Win32_Processor | Select-Object Name, VirtualizationFirmwareEnabled
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent
```
`HypervisorPresent = True` aur `VirtualizationFirmwareEnabled = True` hone chahiye. Agar False → BIOS/UEFI mein Intel VT-x / AMD-V enable karo.

WSL2 version:
```powershell
wsl --version
wsl --status
```

### 0.2 WSL2 memory cap (bahut zaroori)

WSL2 by default RAM ka ~50% le leta hai — Docker ke saath ye problem banega. Isko cap karo taaki VMs ke liye RAM bache.

File banao: `C:\Users\Kablu\.wslconfig`
```ini
[wsl2]
memory=12GB
processors=4
swap=4GB
```
Phir WSL restart:
```powershell
wsl --shutdown
```

**Budget:** Host ~6 GB + WSL2/Docker 12 GB (Wazuh ~8) + VMs ~8 GB (victim 4 + attacker 4) = ~26 GB. 32 GB mein comfortable.

### 0.3 Docker Desktop verify

```powershell
docker version
docker info --format '{{.MemTotal}}'
```
Docker Desktop → Settings → Resources → confirm "Use WSL2 based engine" ON.

### 0.4 VirtualBox install + lab network

VirtualBox download: https://www.virtualbox.org/wiki/Downloads (Windows host).

Install ke baad, ek **Host-Only network** banao (lab-isolated, aapke real LAN se alag):
```powershell
# VBoxManage usually: C:\Program Files\Oracle\VirtualBox\VBoxManage.exe
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif create
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" list hostonlyifs
```
Is interface ko ek static subnet do (e.g. 192.168.56.0/24 — VirtualBox default). GUI: File → Host Network Manager → IPv4 192.168.56.1/24, DHCP optional.

**Network design:**
- **Adapter 1 = Host-Only** (`vboxnet0`, 192.168.56.0/24) → attacker ↔ victim ↔ Wazuh (via host) sab isolated baat karein.
- **Adapter 2 = NAT** (sirf jab updates/tools download karne hon; attack ke time OFF kar do).

### 0.5 VMs banao

**Victim — Windows 10/11:**
- Free option: Microsoft "Windows 11 development environment" VirtualBox image (ready-made), ya Windows 10/11 Evaluation ISO (90-day).
- RAM 4 GB, 2 vCPU, 2 adapters (Host-Only + NAT).

**Attacker — Kali Linux:**
- Kali ready-made VirtualBox image: https://www.kali.org/get-kali/#kali-virtual-machines
- RAM 3–4 GB, 2 vCPU, 2 adapters.

### 0.6 Isolation + snapshots (discipline)

- Attack/testing ke waqt dono VMs ka **NAT adapter disable** — sirf Host-Only.
- Real malware kabhi real LAN par nahi.
- Clean install ke baad **baseline snapshot** lo har VM ka:
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" snapshot "Win-Victim" take "clean-baseline" --description "Fresh, Sysmon+agent installed, pre-attack"
```
Restore (har attack ke baad):
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" snapshot "Win-Victim" restore "clean-baseline"
```

### ✅ Phase 0 "Done" criteria
- [ ] Windows edition + virtualization confirmed.
- [ ] `.wslconfig` memory capped, WSL restarted.
- [ ] VirtualBox installed, Host-Only network up.
- [ ] Victim + Kali VMs created, 2 adapters each.
- [ ] Baseline snapshots liye.
- [ ] Victim se Kali ping ho (Host-Only subnet), internet NAT-gated.

---

## PHASE 1 — Wazuh SIEM core (Docker, single-node)

### 1.1 Kernel param (indexer requirement)

OpenSearch indexer ko `vm.max_map_count >= 262144` chahiye. Docker Desktop WSL2 par set karo:
```powershell
wsl -d docker-desktop sh -c "sysctl -w vm.max_map_count=262144"
```
> Ye reboot par reset ho sakta hai. Persistent banane ke liye Docker Desktop restart ke baad dobara chala dena, ya `.wslconfig` ke saath workaround.

### 1.2 Wazuh docker repo clone (WSL/Ubuntu ya Git Bash)

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2
cd wazuh-docker/single-node
```

### 1.3 Indexer heap cap (RAM control)

`single-node/docker-compose.yml` kholo, `wazuh.indexer` service ke `environment:` mein add/edit karo:
```yaml
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
```
> Ye hi sabse important RAM knob hai — 4 GB heap par cap. Xms aur Xmx barabar rakho.

### 1.4 Certs generate + stack up

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```
```bash
docker compose up -d
```
Status:
```bash
docker compose ps
```
Pehli baar indexer ko green hone mein ~1–2 min lagte hain. Logs:
```bash
docker compose logs -f wazuh.indexer
```

### 1.5 Dashboard login

Browser: **https://localhost:443** (self-signed cert warning → accept).

Default creds (single-node): user **admin**, password **SecretPassword**.

### 1.6 Passwords change (pehla security step)

Default passwords turant badlo:
```bash
docker compose exec wazuh.manager /var/ossec/framework/python/bin/python3 /var/ossec/framework/scripts/wazuh-passwords-tool.py -a -A
```
Naye passwords note karo; dashboard/`.env`/compose update karke `docker compose up -d` dobara.

### 1.7 Health check

Dashboard → **Server management → Status** → sab services green.
Indexer cluster:
```bash
curl -sk -u admin:SecretPassword https://localhost:9200/_cluster/health?pretty
```
`"status" : "green"` dikhe (single-node par "yellow" bhi acceptable — replicas nahi hote).

### ✅ Phase 1 "Done" criteria
- [ ] `vm.max_map_count` set.
- [ ] Indexer heap 4 GB par capped.
- [ ] `docker compose ps` → saare containers Up/healthy.
- [ ] Dashboard https://localhost:443 par login.
- [ ] Default passwords changed.
- [ ] Cluster health green/yellow, 0 agents (abhi).

---

## Common issues

| Problem | Fix |
|---|---|
| Indexer container exit / `max_map_count` error | Step 1.1 dobara chalao, phir `docker compose restart wazuh.indexer`. |
| Host RAM choke | `.wslconfig` cap + indexer heap 4 GB; VMs band karo jab zaroorat na ho. |
| VirtualBox VM slow / "VT-x unavailable" | WHP coexistence — VirtualBox 7.x use karo; ya VMware Workstation Player try karo. |
| Dashboard 443 na khule | `docker compose logs wazuh.dashboard`; indexer green hone ka wait. |
| Port 443 conflict (IIS/other) | compose mein dashboard port `1443:5601` par map karo. |

---

## Next (Phase 2 preview)
Victim VM par Wazuh agent + Sysmon (sysmon-modular) + PowerShell logging → pehla telemetry event dashboard mein.
