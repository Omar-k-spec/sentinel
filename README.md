# Sentinel

**A personal security system for Linux, built from scratch (alpha stage).**

*Author: Omar_Ibrahim*

---

Sentinel watches your Linux machine around the clock. It learns what "normal" looks like on your specific system, then alerts you the moment something deviates from that. No cloud. No subscriptions. No data leaving your machine.

Most security tools are either massive enterprise systems that require a team to run, or basic scripts that only catch obvious threats. Sentinel sits in the middle — serious enough to catch real attacks, simple enough to run on your personal laptop.

---

## What it detects

**Real-time rule-based detection**
- System files being modified without your knowledge (`/etc/passwd`, `sudoers`, SSH config, etc.)
- Suspicious processes running from memory locations they shouldn't be in
- New network ports opening unexpectedly
- New user accounts being created
- Someone adding themselves to the sudo group
- Cron jobs being added or changed
- Executable files being dropped in `/tmp` or `/dev/shm`

**Network and DNS monitoring**
- Unexpected outbound connections to unknown IP addresses
- Suspicious domain names being resolved (DGA domains, known malware hosts)
- Upload spikes that could mean data is being sent out without your knowledge

**Exploit detection**
- Processes injecting code into other processes (ptrace abuse)
- Programs running entirely from memory with no file on disk (fileless malware)
- LD_PRELOAD being used to hijack shared libraries
- Kernel security parameters being changed
- Processes unexpectedly gaining root access

**Machine learning anomaly detection**
- After collecting 7 days of your normal usage patterns, Sentinel trains an Isolation Forest model
- The model scores your system's behavior continuously
- Anything that deviates significantly from your normal baseline triggers an alert
- It explains *why* something is anomalous, not just that it is

**Threat intelligence**
- Cross-references your installed packages against CISA's list of actively exploited vulnerabilities (1,500+ CVEs)
- Checks suspicious IPs against AbuseIPDB and SANS DShield
- Verifies suspicious domains against URLhaus malware database
- All intelligence is cached locally — only fetches updates once a day

**Attack chain correlation**
- Connects related events that happen close together in time
- A single suspicious DNS query is noise. A suspicious DNS query followed by a new outbound connection followed by a file dropped in /tmp — that's an attack chain, and Sentinel fires a correlated alert for it

---

## How it works

```
Your system
    │
    ├── sentinel_monitor.py        ← Main loop, runs every 60 seconds
    │       │
    │       ├── sentinel_response.py    ← Kills processes, blocks IPs, quarantines files
    │       ├── sentinel_network.py     ← Outbound connection monitoring
    │       ├── sentinel_dns.py         ← DNS query monitoring
    │       ├── sentinel_fs_bw.py       ← Filesystem events + bandwidth
    │       ├── sentinel_suid.py        ← SUID/SGID privilege escalation detection
    │       ├── sentinel_ml.py          ← ML anomaly scoring
    │       ├── sentinel_exploit.py     ← Exploit technique detection
    │       └── sentinel_correlate.py   ← Links related events into attack chains
    │
    ├── sentinel_collector.py      ← Collects behavioral data for ML training
    ├── sentinel_intel.py          ← Pulls threat intelligence from free sources
    ├── sentinel_brain.py          ← Ask questions about your security in plain English
    ├── sentinel_brain_llm.py      ← Same but powered by a local AI model (Ollama)
    ├── sentinel_dashboard.py      ← Web dashboard at http://localhost:8888
    ├── sentinel_report.py         ← Daily security summary report
    └── sentinel_db.py             ← Unified database for all Sentinel data
```

---

## Requirements

**System tools** (all free, most already on Linux):
```bash
sudo apt install inotify-tools tcpdump rkhunter clamav ufw gpg auditd -y
```

**Python packages:**
```bash
pip install -r requirements.txt
```

**Optional — for AI-powered Brain:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen2.5:7b
```

---

## Setup

### Step 1 — Get the code
```bash
git clone https://github.com/YOUR_USERNAME/sentinel.git
cd sentinel
```

### Step 2 — Run the installer
The installer automatically detects your username and sets everything up:
```bash
bash install.sh
```

This does the following for you:
- Creates `~/.local/.sentinel-core/` (hidden, locked to your user only)
- Creates `~/.sentinel/` (where all logs and data live)
- Copies all scripts with correct permissions
- Installs Sentinel as a systemd service that starts automatically on boot
- Sets up the passwordless sudo rules Sentinel needs

### Step 3 — Capture your baseline
Run this when your system is in a clean, normal state with your usual apps open:
```bash
python3 ~/.local/.sentinel-core/sentinel_baseline.py
```

Lock the baseline so it can't be tampered with:
```bash
chmod 444 ~/.sentinel/baseline.json
```

### Step 4 — Capture secondary baselines
With your normal apps running (Steam, browser, etc.):
```bash
# Learn your normal DNS traffic
python3 ~/.local/.sentinel-core/sentinel_dns.py baseline 300

# Learn your normal outbound connections
python3 ~/.local/.sentinel-core/sentinel_network.py baseline

# Snapshot your current SUID files
python3 ~/.local/.sentinel-core/sentinel_suid.py baseline
```

### Step 5 — Pull threat intelligence
Requires internet. Only about 1MB of data total:
```bash
python3 ~/.local/.sentinel-core/sentinel_intel.py update
```

### Step 6 — Check that everything is running
```bash
sudo systemctl status sentinel sentinel-collector
```

---

## Daily use

**Ask Sentinel questions:**
```bash
python3 ~/.local/.sentinel-core/sentinel_brain.py
```
Then type things like:
- "Am I safe?"
- "What happened last night?"
- "Is 1.2.3.4 suspicious?"
- "Check CVE-2024-1234"
- "Block 5.6.7.8"
- "Show me the report"

**Open the dashboard:**
```bash
python3 ~/.local/.sentinel-core/sentinel_dashboard.py
```
Then open `http://127.0.0.1:8888` in your browser.

**View the daily report:**
```bash
python3 ~/.local/.sentinel-core/sentinel_report.py
```

**Re-baseline after you intentionally change something:**
```bash
chmod 644 ~/.sentinel/baseline.json
python3 ~/.local/.sentinel-core/sentinel_baseline.py
chmod 444 ~/.sentinel/baseline.json
```

---

## Training the ML model

After 7 days of normal usage, Sentinel has enough data to train:
```bash
# Check how much data you have
python3 ~/.local/.sentinel-core/sentinel_collector.py stats

# Train the model
python3 ~/.local/.sentinel-core/sentinel_ml.py train

# Test it on your live system
python3 ~/.local/.sentinel-core/sentinel_ml.py test
```

---

## Privacy

Everything stays on your machine. Sentinel:
- Never sends logs anywhere
- Never calls home to any server
- Only fetches data from the internet when you explicitly run the threat intel update
- Encrypts all logs nightly with AES256 and shreds the plaintext
- Stores the encryption key locally at `~/.local/.sentinel-core/.vault_key`

Back up your vault key. If you lose it, you can't read your encrypted logs.

---

## What the files actually are

| File | What it does |
|---|---|
| `sentinel_monitor.py` | The main loop that runs everything |
| `sentinel_baseline.py` | Takes a snapshot of your clean system state |
| `sentinel_response.py` | Automated actions: kill, quarantine, block, lock |
| `sentinel_network.py` | Watches outbound connections |
| `sentinel_dns.py` | Watches DNS queries via tcpdump |
| `sentinel_fs_bw.py` | Watches file changes and bandwidth |
| `sentinel_suid.py` | Watches for privilege escalation |
| `sentinel_exploit.py` | Watches for active exploitation techniques |
| `sentinel_correlate.py` | Links related events into attack chains |
| `sentinel_collector.py` | Collects behavioral data for ML |
| `sentinel_ml.py` | Trains and runs the anomaly detection model |
| `sentinel_intel.py` | Manages threat intelligence feeds |
| `sentinel_brain.py` | Conversational security interface |
| `sentinel_brain_llm.py` | Same but with a local AI model |
| `sentinel_brain_actions.py` | Gives the Brain the ability to take actions |
| `sentinel_dashboard.py` | Local web dashboard |
| `sentinel_report.py` | Daily security summary |
| `sentinel_db.py` | Unified database layer |
| `sentinel_config.py` | All settings in one place |
| `sentinel_encrypt_logs.sh` | Nightly log encryption |
| `install.sh` | Automated setup script |

---

## What's coming next

- [ ] Threat intel integration when WiFi is available (AbuseIPDB live blocking)
- [ ] Multi-signal correlation improvements
- [ ] LSTM-based time-series anomaly detection (upgrade from Isolation Forest)
- [ ] Mobile alert notifications
- [ ] Better ML explanation — tell you exactly why something scored as anomalous
- [ ] Package vulnerability scanner integrated with KEV

---

## Contributing

Read `CONTRIBUTING.md` before opening a pull request.

---

## License

MIT — use it, learn from it, build on it.

*Built on Linux Mint 22.3. Tested on Ubuntu 24.04.*
