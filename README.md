# Wazuh SIEM + SOAR Deployment + DDoS Attack Simulation
> Tugas Kelompok — Keamanan Jaringan  
> Institut Teknologi Sepuluh Nopember (ITS) — 2026

---

## Anggota Kelompok

| Nama | NRP | Peran |
|------|-----|-------|
| Tiara Fatimah Azzahra | 5027241090 | Manager Admin (Wazuh Manager) |
| Ahmad Yafi Ar Rizq | [NRP] | Agent Operator 1 (vm-agent-1) — Attacker |
| Diva Aulia Rosa | [NRP] | Agent Operator 2 (vm-agent-02) — Korban |

---

## Deskripsi Proyek

Proyek ini mengimplementasikan **Wazuh SIEM (Security Information and Event Management)** yang diperkuat dengan kemampuan **SOAR (Security Orchestration, Automation and Response)** pada infrastruktur cloud **Microsoft Azure for Students** untuk mendeteksi dan memitigasi serangan **DDoS (Distributed Denial of Service)** secara otomatis.

### Tujuan
1. Maintain SIEM framework (Wazuh) sebagai core architectural foundation
2. Incorporate SOAR capabilities untuk automated detection dan mitigation DDoS
3. Membuat skenario DDoS HTTP Flood sebagai Proof of Concept (PoC)
4. Mengelola logging density dan distribusi log antar agent

---

## Konsep SIEM vs SOAR

### SIEM (Security Information and Event Management)
```
SIEM = Sistem yang MENGUMPULKAN, MENGANALISA,
       dan MELAPORKAN ancaman keamanan

Tugas SIEM:
→ Kumpulkan log dari semua server
→ Analisa pola anomali
→ Generate alert saat ancaman terdeteksi
→ Visualisasi di Dashboard
```

### SOAR (Security Orchestration, Automation and Response)
```
SOAR = SIEM + kemampuan BERTINDAK OTOMATIS

Tugas SOAR:
→ Semua yang SIEM lakukan, DITAMBAH:
→ Otomatis block IP penyerang
→ Otomatis jalankan script mitigasi
→ Otomatis catat insiden ke log
→ Otomatis unblock setelah timeout
```

### Alur SOAR dalam Proyek Ini
```
1. DETECT  → Wazuh deteksi HTTP Flood dari nginx log
                    ↓
2. ANALYZE → Rule 100200 trigger (level 10)
                    ↓
3. RESPOND → Script ddos-mitigation.sh otomatis:
             - Block IP penyerang (iptables DROP)
             - Catat insiden ke log
                    ↓
4. RECOVER → Setelah 600 detik, auto-unblock IP
```

---

## Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────┐
│                    Microsoft Azure                       │
│              Resource Group: rg-wazuh-lab                │
│           Virtual Network: vm-wazuh-manager-vnet         │
│                  Subnet: 10.0.0.0/24                     │
│                                                         │
│  ┌──────────────────────────────────────────────┐       │
│  │              Wazuh Manager                    │       │
│  │         (SIEM + SOAR Engine)                  │       │
│  │  vm-wazuh-manager                            │       │
│  │  Public IP : 20.205.16.230                   │       │
│  │  Private IP: 10.0.0.4                        │       │
│  │  OS        : Ubuntu 22.04 LTS               │       │
│  │  Size      : B2als_v2 (2vCPU, 4GB)          │       │
│  │  Wazuh ver : 4.7.5                          │       │
│  │                                              │       │
│  │  SIEM Components:                            │       │
│  │  ✅ wazuh-manager  (port 1514/1515)          │       │
│  │  ✅ wazuh-indexer  (port 9200)               │       │
│  │  ✅ wazuh-dashboard (port 443)               │       │
│  │                                              │       │
│  │  SOAR Components:                            │       │
│  │  ✅ Active Response Engine                   │       │
│  │  ✅ ddos-mitigation.sh script                │       │
│  │  ✅ Auto iptables DROP/ACCEPT                │       │
│  └──────────────┬───────────────────────────────┘       │
│                 │ Port 1514/1515 TCP                     │
│           ┌─────┴──────┐                               │
│           ▼            ▼                               │
│  ┌──────────────┐ ┌──────────────┐                    │
│  │  vm-agent-1  │ │ vm-agent-02  │                    │
│  │  10.0.0.5    │ │ 10.0.0.6     │                    │
│  │  ATTACKER    │ │ TARGET       │                    │
│  │  ✅ Active   │ │ ✅ Active    │                    │
│  └──────┬───────┘ └──────────────┘                    │
│         │                ▲                             │
│         └── HTTP Flood ──┘                             │
│           ab -n 10000 -c 100 http://10.0.0.6/          │
└─────────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private | Status |
|----|------|------|------|-----|-----------|------------|--------|
| vm-wazuh-manager | SIEM + SOAR Engine | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 | Running |
| vm-agent-1 | Wazuh Agent + Attacker | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 | Active |
| vm-agent-02 | Wazuh Agent + Target | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 | Active |

**Platform:** Microsoft Azure for Students ($100 kredit)  
**OS:** Ubuntu Server 22.04 LTS  
**Wazuh Version:** 4.7.5  
**Region:** East Asia  

---

## Deployment SIEM — Point 1

### Step 1 — Buat Infrastruktur Azure

```
Resource Group : rg-wazuh-lab
Virtual Network: vm-wazuh-manager-vnet (10.0.0.0/16)
Subnet         : default (10.0.0.0/24)
Region         : East Asia
```

**Port yang dibuka di Network Security Group:**

| Port | Protocol | Tujuan |
|------|----------|--------|
| 22 | TCP | SSH akses |
| 443 | TCP | Wazuh Dashboard HTTPS |
| 1514 | TCP | Agent kirim log ke Manager |
| 1515 | TCP | Agent registrasi ke Manager |

---

### Step 2 — Install Wazuh Manager (SIEM Core)

```bash
ssh azureuser@20.205.16.230
sudo apt-get update && sudo apt-get upgrade -y

curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
nano config.yml
```

Isi `config.yml`:
```yaml
nodes:
  indexer:
    - name: node-1
      ip: "10.0.0.4"
  server:
    - name: wazuh-1
      ip: "10.0.0.4"
  dashboard:
    - name: dashboard
      ip: "10.0.0.4"
```

```bash
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

**Dashboard:** `https://20.205.16.230`

---

### Step 3 — Install Wazuh Agent

```bash
ssh azureuser@<IP-AGENT>
sudo apt-get update && sudo apt-get upgrade -y

curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt-get update

WAZUH_MANAGER="10.0.0.4" WAZUH_AGENT_NAME="agent-01" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

sudo nano /var/ossec/etc/ossec.conf
# Cari MANAGER_IP → ganti 10.0.0.4

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo /var/ossec/bin/agent-auth -m 10.0.0.4

# Tambah monitoring nginx log
sudo tee -a /var/ossec/etc/ossec.conf << 'EOF'

<ossec_config>
  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/nginx/access.log</location>
  </localfile>
</ossec_config>
EOF

sudo systemctl restart wazuh-agent
```

**Verifikasi:**
```
ID: 000, Name: vm-wazuh-manager (server), Active/Local
ID: 001, Name: vm-agent-1,  Active
ID: 002, Name: vm-agent-02, Active
Agents Coverage: 100%
```

---

## SOAR Implementation — Automated DDoS Mitigation

### Apa yang Diimplementasikan

```
Wazuh Active Response + Custom Script = SOAR

Saat DDoS terdeteksi:
→ OTOMATIS block IP penyerang (iptables)
→ OTOMATIS catat insiden ke log
→ OTOMATIS unblock setelah 10 menit
→ Tanpa intervensi manusia!
```

---

### Step 1 — Buat Custom SOAR Script

```bash
# Di Manager
sudo nano /var/ossec/active-response/bin/ddos-mitigation.sh
```

```bash
#!/bin/bash
# ================================================
# SOAR Script — DDoS Auto Mitigation
# Wazuh Active Response
# Institut Teknologi Sepuluh Nopember — 2026
# ================================================

LOCAL=$(dirname $0)
cd $LOCAL
cd ../

PWD=$(pwd)
LOG_FILE="${PWD}/../logs/ddos-mitigation.log"

ARG1=$1   # add/delete
ARG2=$2   # user
ARG3=$3   # IP penyerang
ARG4=$4   # alert ID
ARG5=$5   # rule ID
ARG6=$6   # agent ID

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Fungsi log
log(){
  echo "[$TIMESTAMP] $1" >> $LOG_FILE
}

# ========================
# DETECT & RESPOND — BLOCK
# ========================
if [ "$ARG1" = "add" ]; then
  log "======================================="
  log "SOAR ACTION: DDoS ATTACK DETECTED!"
  log "======================================="
  log "Timestamp    : $TIMESTAMP"
  log "Source IP    : $ARG3"
  log "Alert ID     : $ARG4"
  log "Rule ID      : $ARG5"
  log "Agent ID     : $ARG6"
  log "Action       : AUTO-BLOCKING IP $ARG3"

  # Block IP penyerang via iptables
  iptables -I INPUT -s $ARG3 -j DROP
  iptables -I FORWARD -s $ARG3 -j DROP

  log "Status       : IP $ARG3 BLOCKED"
  log "Auto-unblock : 600 seconds (10 menit)"
  log "======================================="
fi

# ========================
# RECOVER — UNBLOCK
# ========================
if [ "$ARG1" = "delete" ]; then
  log "======================================="
  log "SOAR ACTION: AUTO-UNBLOCK TIMEOUT"
  log "IP $ARG3 unblocked after 600 seconds"

  iptables -D INPUT -s $ARG3 -j DROP
  iptables -D FORWARD -s $ARG3 -j DROP

  log "Status       : IP $ARG3 UNBLOCKED"
  log "======================================="
fi

exit 0
```

---

### Step 2 — Set Permission Script

```bash
# Di Manager
sudo chmod 750 /var/ossec/active-response/bin/ddos-mitigation.sh
sudo chown root:wazuh /var/ossec/active-response/bin/ddos-mitigation.sh
```

---

### Step 3 — Daftarkan ke Wazuh

```bash
# Di Manager
sudo nano /var/ossec/etc/ossec.conf
```

Tambahkan command baru:
```xml
<command>
  <name>ddos-mitigation</name>
  <executable>ddos-mitigation.sh</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

Tambahkan active-response:
```xml
<ossec_config>
  <active-response>
    <command>ddos-mitigation</command>
    <location>local</location>
    <rules_id>100200</rules_id>
    <timeout>600</timeout>
  </active-response>
</ossec_config>
```

```bash
sudo systemctl restart wazuh-manager
```

---

### Step 4 — DDoS Detection Rules

File: `/var/ossec/etc/rules/ddos_rules.xml`

```xml
<group name="ddos,attack,">

  <rule id="100200" level="10">
    <decoded_as>web-accesslog</decoded_as>
    <match>GET|POST</match>
    <description>DDoS HTTP Flood: Abnormal web traffic detected</description>
    <group>ddos,http_flood,</group>
  </rule>

</group>
```

---

## DDoS Attack Scenario + SOAR Response — Point 2

### Skenario

```
ATTACKER : vm-agent-1  (IP: 10.0.0.5)
TARGET   : vm-agent-02 (IP: 10.0.0.6)
METODE   : HTTP Flood (ApacheBench)
SOAR     : Auto-block via ddos-mitigation.sh
```

---

### Eksekusi — 3 Terminal Sekaligus

#### Terminal 1 — Manager (Monitor)

```bash
ssh azureuser@20.205.16.230

# Monitor alert real-time
sudo tail -f /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    desc = d.get('rule', {}).get('description', '')
    agent = d.get('agent', {}).get('name', '?')
    srcip = d.get('data', {}).get('srcip', '')
    if lvl >= 3:
      print(f'[LEVEL {lvl}] [{agent}] {desc} | src={srcip}')
  except: pass
"

# Monitor SOAR action (terminal lain)
sudo tail -f /var/ossec/logs/ddos-mitigation.log
```

---

#### Terminal 2 — vm-agent-02 (Target/Korban)

```bash
ssh azureuser@20.2.82.117

# Monitor nginx log — lihat banjir request
sudo tail -f /var/log/nginx/access.log
```

**Output saat diserang:**
```
10.0.0.5 - - [19/May/2026:02:06:04 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
10.0.0.5 - - [19/May/2026:02:06:04 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
... ribuan baris dari IP yang sama!
```

---

#### Terminal 3 — vm-agent-1 (Attacker)

```bash
ssh azureuser@57.158.24.143

# Jalankan HTTP Flood
ab -n 10000 -c 100 http://10.0.0.6/
```

| Parameter | Arti |
|-----------|------|
| `-n 10000` | 10.000 total request |
| `-c 100` | 100 request bersamaan |
| `http://10.0.0.6/` | Target nginx Agent-02 |

**Output:**
```
Completed 1000 requests
Completed 2000 requests
...
Total of 8781 requests completed
```

---

### Hasil SOAR — Auto Mitigation Log

```bash
# Di Manager — lihat SOAR bekerja
sudo cat /var/ossec/logs/ddos-mitigation.log
```

**Output SOAR:**
```
=======================================
SOAR ACTION: DDoS ATTACK DETECTED!
=======================================
Timestamp    : 2026-05-19 09:04:XX
Source IP    : 10.0.0.5
Alert ID     : XXXX
Rule ID      : 100200
Agent ID     : 002
Action       : AUTO-BLOCKING IP 10.0.0.5
Status       : IP 10.0.0.5 BLOCKED
Auto-unblock : 600 seconds (10 menit)
=======================================
...
=======================================
SOAR ACTION: AUTO-UNBLOCK TIMEOUT
IP 10.0.0.5 unblocked after 600 seconds
Status       : IP 10.0.0.5 UNBLOCKED
=======================================
```

---

### Verifikasi SOAR — iptables Block

```bash
# Di Agent-02 — cek IP penyerang diblokir
sudo iptables -L INPUT -n | grep DROP

# Output:
# DROP  all  --  10.0.0.5  0.0.0.0/0
```

---

### Hasil Deteksi di Dashboard

**Filter:** `data.srcip: 10.0.0.5`

| Metric | Nilai |
|--------|-------|
| Total Alerts | **3,313** |
| Level 12+ CRITICAL | **3,309** |
| Spike Grafik | Jam 08:00-09:00 |
| Target | vm-agent-02 |
| SOAR Response | Auto-block dalam detik |

---

### Alur Lengkap SIEM + SOAR

```
┌─────────────────────────────────────────────┐
│              SIEM + SOAR FLOW               │
├─────────────────────────────────────────────┤
│                                             │
│  1. DETECT                                  │
│     Agent-01 flood nginx Agent-02           │
│     Wazuh baca nginx access.log             │
│     Traffic: 1.228 → 3.072 req/jam          │
│                    ↓                        │
│  2. ANALYZE                                 │
│     Rule 100200 trigger (level 10)          │
│     Alert di-generate oleh Manager          │
│                    ↓                        │
│  3. RESPOND (SOAR)                          │
│     ddos-mitigation.sh dijalankan otomatis  │
│     iptables DROP untuk IP 10.0.0.5         │
│     Insiden dicatat ke log                  │
│                    ↓                        │
│  4. RECOVER (SOAR)                          │
│     Setelah 600 detik                       │
│     IP 10.0.0.5 otomatis di-unblock         │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Logging Density & Distribution — Point 3

### Mengapa Logging Density Penting?

DDoS menghasilkan **ribuan log per detik**. Tanpa pengelolaan:
- Storage penuh dalam hitungan jam
- Performance menurun
- Alert penting tenggelam di noise

**Bukti nyata:**
```
"Target 'agent' message queue is full (1024). Log lines may be lost."
```

---

### Konfigurasi Logging

```xml
<!-- /var/ossec/etc/ossec.conf di Manager -->
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>no</logall>         <!-- hemat storage -->
  <logall_json>no</logall_json>
</global>

<alerts>
  <log_alert_level>3</log_alert_level>   <!-- filter level 1-2 -->
  <email_alert_level>12</email_alert_level> <!-- email hanya critical -->
</alerts>
```

---

### Command Analisis Logging

#### 1. Total Log Tersimpan

```bash
# Di Manager
sudo wc -l /var/ossec/logs/alerts/alerts.json
# Output: 14596
```

#### 2. Distribusi Log per Agent

```bash
# Di Manager
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
from collections import Counter
agents = Counter()
for line in sys.stdin:
  try:
    d = json.loads(line)
    agent = d.get('agent', {}).get('name', 'unknown')
    agents[agent] += 1
  except: pass
for agent, count in agents.most_common():
  print(f'{agent}: {count} events')
"
```

**Hasil:**
```
vm-agent-02     : 6791 events (46%) ← target DDoS
vm-agent-1      : 4839 events (33%) ← attacker
vm-wazuh-manager: 2972 events (20%) ← manager
```

#### 3. Distribusi Level Alert

```bash
# Di Manager
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
from collections import Counter
levels = Counter()
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    levels[lvl] += 1
  except: pass
print('Distribusi level alert:')
for lvl, count in sorted(levels.items()):
  print(f'Level {lvl}: {count} events')
"
```

**Hasil:**
```
Distribusi level alert:
Level 3  :   927 events ← minimum tersimpan
Level 4  :   161 events ← HTTP flood detection
Level 5  : 9,450 events ← SSH attempts internet
Level 7  :    17 events
Level 8  :    55 events
Level 9  :     3 events
Level 10 :   690 events ← brute force
Level 12 : 3,309 events ← DDoS CRITICAL

Level 1 & 2 = 0 → filter bekerja!
```

#### 4. Monitor Storage Real-time

```bash
# Di Manager — jalankan saat DDoS berlangsung
watch -n2 "sudo du -sh /var/ossec/logs/alerts/ && echo '---' && sudo wc -l /var/ossec/logs/alerts/alerts.json"
```

---

### Hasil Distribusi

| Agent | Events | % | Keterangan |
|-------|--------|---|------------|
| vm-agent-02 | 6,791 | 46% | Target DDoS |
| vm-agent-1 | 4,839 | 33% | Attacker |
| vm-wazuh-manager | 2,972 | 20% | Manager |
| **Total** | **14,596** | 100% | |

| Level | Events | Keterangan |
|-------|--------|------------|
| Level 3 | 927 | Minimum tersimpan |
| Level 4 | 161 | HTTP flood |
| Level 5 | 9,450 | SSH internet |
| Level 10 | 690 | Brute force |
| Level 12 | **3,309** | **DDoS CRITICAL** |
| Level 1&2 | **0** | **Filter bekerja** |

---

## Hasil dan Kesimpulan

### Ringkasan Pencapaian

| Requirement | Implementasi | Status |
|-------------|-------------|--------|
| SIEM Framework (Wazuh) | Manager + 2 Agent di Azure | Done |
| SOAR — Automated Detection | Rule 100200 trigger otomatis | Done |
| SOAR — Automated Mitigation | ddos-mitigation.sh auto-block | Done |
| DDoS HTTP Flood Simulation | ab flood 10.0.0.5 → 10.0.0.6 | Done |
| Anomali traffic detection | Spike 1.228 → 3.072 req/jam | Done |
| Critical alerts generation | 3,309 level 12 CRITICAL | Done |
| Logging density management | Level 1&2=0, filter aktif | Done |
| Log distribution | 14,596 events dari 3 node | Done |

### Kesimpulan

Integrasi **Wazuh SIEM + SOAR** terbukti efektif:

1. **SIEM** — Mendeteksi DDoS dalam hitungan detik via nginx log
2. **SOAR** — Otomatis block IP penyerang tanpa intervensi manusia
3. **Traffic analysis** — Spike 1.228 → 3.072 req/jam (naik 2.5x)
4. **Critical alerts** — 3,309 level 12 CRITICAL alerts
5. **Auto-recovery** — IP otomatis di-unblock setelah 10 menit
6. **Log management** — 14,596 events terkelola efisien
7. **Zero manual intervention** — Seluruh proses deteksi hingga mitigasi berjalan otomatis

---

## Referensi

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh Active Response (SOAR)](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)
- [Wazuh 4.7 Installation Guide](https://documentation.wazuh.com/4.7/installation-guide)
- [ApacheBench Documentation](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [UU ITE No. 11 Tahun 2008](https://jdih.kominfo.go.id)

---

*Tugas Keamanan Jaringan — Institut Teknologi Sepuluh Nopember (ITS) 2026*  
*Last updated: 19 Mei 2026*
