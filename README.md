# SENTINEL-X
### Autonomous SOC Platform for Real-Time Threat Detection, Adaptive Containment and Self-Healing Incident Response

SENTINEL-X is an industry-style mini Security Operations Center (SOC)
platform built as a final-year cybersecurity project. It safely detects
**authorized attack simulations** run against its own decoy service,
correlates related events into incidents, scores risk and confidence,
maps everything to MITRE ATT&CK, and autonomously contains and then
**self-heals** — all visible live on a real-time dashboard.

> ⚠️ **LAB SAFE MODE**: SENTINEL-X never performs real exploitation and
> never touches a real OS, real firewall, real account, or the public
> internet. All "attacks" hit a purpose-built Decoy Service, and all
> "containment" is simulated entirely inside SENTINEL-X's own database.
> See [SAFETY.md](SAFETY.md) for exactly what that means.

---

## 1. What it does

```
Friend Laptop / Test Simulator
        │
        ▼
  Decoy Service   (fake login, search, honeypot file, sudo endpoint,
        │          dummy scannable ports — nothing real behind any of it)
        ▼
  Event Collector      (POST /api/ingest)
        ▼
  Event Normalizer     (raw → canonical event shape)
        ▼
  Rule-Based Detection Engine   (sliding-window thresholds per rule)
        ▼
  Alert Deduplication  (collapse repeats into one alert + counter)
        ▼
  Attack Correlation   (distinct alert types from one source → campaign)
        ▼
  Risk + Confidence Scoring   (base severity, over-threshold, campaign
        │                      bonus, Threat Memory reputation)
        ▼
  MITRE ATT&CK Mapping
        ▼
  Response Engine
        ▼
  Scoped Containment   (mark source contained; Decoy Service starts
        │               refusing that source's requests)
        ▼
  Self-Healing         (CONTAINED → COOLDOWN → AUTO_RECOVERY)
        ▼
  Recovery Verification → RECOVERY_VERIFIED
        ▼
  Real-Time SOC Dashboard  (Flask-SocketIO + Chart.js, updates live)
```

### Attack / event scenarios covered
1. **Failed Login / Brute Force** — repeated bad logins against the decoy login endpoint
2. **Port Probe / Network Reconnaissance** — connections to a set of decoy TCP ports
3. **SQL Probe** — SQL-metacharacter input against a decoy search endpoint
4. **Honeypot Trigger** — any access to a decoy "secret admin backup" resource
5. **Privilege Test** — repeated attempts against a decoy sudo/privilege-escalation endpoint
6. **Multi-stage campaign** — any 3+ distinct scenarios above from the same source within a rolling window get correlated into one incident automatically

### Feature checklist
- Real-time event ingestion (HTTP → SocketIO broadcast)
- Rule-based detection with sliding-window thresholds
- Alert deduplication (occurrence counters, not alert spam)
- Multi-stage attack correlation
- Risk score (0–100) and Confidence score (0–100)
- MITRE ATT&CK technique mapping, with a full reference table
- Threat Memory Bank (per-source reputation that adapts future scoring)
- Adaptive response (reputation feeds back into risk)
- Scoped, temporary containment (per-source, auto-expiring)
- Automatic self-healing state machine
- Recovery verification (a real delayed re-check, not instant)
- Attack-chain replay (full event/alert timeline per incident)
- Explainability panel (plain-English "why was this raised")
- Analyst Confirmed / False-Positive feedback loop
- Full audit trail
- SQLite persistence
- Live dashboard (stat cards, event stream, pipeline visualization,
  incident command, self-healing timeline, threat memory, MITRE
  reference, audit log)
- Emergency Stop (manual override — contain everything, right now)
- LAB SAFE MODE indicator, always visible
- Automated test suite (pytest)

---

## 2. Project structure

```
sentinel-x/
├── app/                      # Core SOC engine (imported by run.py)
│   ├── __init__.py           # Flask app factory
│   ├── config.py             # ALL tunables: thresholds, windows, ports
│   ├── db.py                 # SQLite schema + connection helper
│   ├── normalizer.py         # Event Normalizer
│   ├── detection.py          # Rule-Based Detection Engine (sliding window)
│   ├── dedup.py               # Alert Deduplication
│   ├── correlation.py        # Attack Correlation (campaign detection)
│   ├── scoring.py             # Risk + Confidence Scoring, Explainability text
│   ├── mitre.py               # MITRE ATT&CK mapping
│   ├── threat_memory.py      # Threat Memory Bank / reputation
│   ├── response.py            # Response Engine, Emergency Stop, feedback
│   ├── containment.py        # Scoped Containment (LAB SAFE)
│   ├── selfheal.py            # Self-Healing state machine (background loop)
│   ├── pipeline.py            # Orchestrates one event through every stage
│   ├── audit.py               # Audit trail
│   ├── realtime.py            # Shared SocketIO instance
│   └── routes.py              # Flask blueprint: dashboard + REST API
├── decoy/
│   └── decoy_service.py      # The ONLY thing an attacker ever touches
├── simulator/
│   └── attack_simulator.py   # Safe CLI to run the 6 required scenarios
├── static/
│   ├── css/dashboard.css
│   └── js/dashboard.js
├── templates/
│   └── dashboard.html
├── tests/                     # pytest suite (25 tests)
│   ├── conftest.py
│   ├── test_detection.py
│   ├── test_dedup.py
│   ├── test_correlation.py
│   ├── test_scoring.py
│   ├── test_selfheal.py
│   └── test_pipeline_e2e.py
├── data/                       # sentinelx.db lives here (created at first run)
├── run.py                      # Start the core dashboard server
├── init_db.py                  # Standalone DB init/reset script
├── requirements.txt
├── README.md                   # (this file)
├── RUN_GUIDE.md                # exact step-by-step run/demo instructions
└── SAFETY.md                   # what "LAB SAFE MODE" actually guarantees
```

---

## 3. Quick start (single laptop)

```bash
cd sentinel-x
python3 -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt

python init_db.py                   # creates data/sentinelx.db

# Terminal 1
python run.py                       # dashboard: http://localhost:5000

# Terminal 2
python decoy/decoy_service.py       # decoy target: http://localhost:8080

# Terminal 3
python simulator/attack_simulator.py --target 127.0.0.1 --scenario campaign
```

Open `http://localhost:5000` and watch the incident appear, get scored,
get mapped to MITRE ATT&CK, get contained, and self-heal — live.

For the two-laptop LAN version, exact IP configuration, and a
scenario-by-scenario demo script, see **[RUN_GUIDE.md](RUN_GUIDE.md)**.

---

## 4. Running the tests

```bash
pip install -r requirements.txt
pytest tests/ -v
```

25 tests cover the detection sliding windows, deduplication, campaign
correlation, risk/confidence scoring, the self-healing state machine,
and full end-to-end pipeline behavior.

---

## 5. Key design decisions (useful for your viva / defense)

- **Why sqlite3 directly instead of an ORM?** Keeps the schema in one
  readable file (`app/db.py`) and lets you `sqlite3 data/sentinelx.db`
  during a demo to show raw persistence — useful when an examiner asks
  "prove it's actually stored."
- **Why is containment "scoped"?** It only affects the one offending
  source (by IP), only for `COOLDOWN_SECONDS`, and only inside the
  Decoy Service's own request handling — never a real firewall rule.
- **Why does self-healing run as a background thread instead of
  instantly flipping state?** So the dashboard's Self-Healing Timeline
  actually animates through each stage in real time, which is the
  whole point of a "self-healing" demo — instant transitions wouldn't
  demonstrate anything.
- **Why does Threat Memory affect scoring?** It's what makes the
  response "adaptive": a source with a track record of analyst-confirmed
  incidents scores hotter next time; one with a track record of
  false-positive feedback scores cooler. This is the feedback loop an
  examiner will expect from "adaptive response."
- **Why cap containment risk threshold above single-alert severity for
  some rules?** So a genuine multi-stage campaign has time to
  correlate across 2–3 stages before the system clamps down — otherwise
  you'd never see "campaign correlation" in action, only "first alert
  triggers containment."

---

## 6. Tech stack

Python · Flask · Flask-SocketIO · SQLite · HTML/CSS/JavaScript ·
Chart.js · Pytest · MITRE ATT&CK mapping
