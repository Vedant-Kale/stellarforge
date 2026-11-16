# StellarForge

An offline, 3D-body health tool for non-doctor astronauts. Built for the **NASA Space Apps Challenge 2026**: *Create Health Monitoring Software for Astronauts on Space Missions*.

It learns each person's own normal, flags drift before it becomes a problem, and, if a crewmate collapses, walks the crew through the emergency one cited step per screen. It builds a ground report that accounts for the Mars communication delay.

> **Decision support, not diagnosis.** Rules and published guidelines decide. Models detect and forecast. The language model only explains. Prototype for a hackathon; the medical content has not been clinically reviewed. Not medical advice.

## My role: Vedant Kale

> Personal copy of the team repo [SCAPE-NASA/stellarforge](https://github.com/SCAPE-NASA/stellarforge) (`main`), with full commit history. StellarForge was built by a team and **submitted to the NASA Space Apps Challenge 2026**. This section describes my part; the rest of the README is the team's.

**Role:** idea owner, team lead for work delegation, and AI / detection lead. About 76 commits, the second-highest on the team.

### What I did

- **Idea and planning.** Shaped the overall idea: an offline, per-person health tool for non-doctor crews, with a cited emergency flow and a Mars-delay ground report. Split the work into five specs and delegated them across the team.
- **Spec 3: detection and forecast core (built).**
  - Personal baselines (14-day Anchor, 28-day adaptive window), Acute and Drift alerts, Quarantine, Status, and the daily briefing (`backend/detection.py`).
  - Emergency auto-trigger rule and likely-cause ranking, with heart-rate cut-offs checked against the AHA 2025 bradycardia and tachyarrhythmia algorithms.
  - Forecast B: tomorrow's readiness with a range and its drivers.
  - Tuned the cut-offs against golden replays and wrote the accuracy page ([docs/accuracy.md](docs/accuracy.md)).
- **Spec 2 upgrade.** Joint-pattern anomaly detector and per-person action-plan learning on top of the crew-data daily loop.
- **Spec 4: medical content.** Full microgravity CPR protocol and chest-pain phase, Safety bounds (glucose, temperature), an ISS-sourced drug blocklist, the Mars transit mission profile, six measurement cards, and the content linter rules that enforce citations and bounds.
- **Spec 5: integration and submission.** Merged all specs, ran the end-to-end golden run on real content (both triggers, offline), recorded the backup demo, and wrote the write-up and video script.
- **Webcam heart rate.** Classic rPPG heart-rate estimate from a webcam (`backend/webcam_hr.py`), measured on the UBFC-rPPG dataset.

### Results (from [docs/accuracy.md](docs/accuracy.md), measured on datasets, not on astronauts)

| What | Result |
|---|---|
| False alerts (drift + acute) | 0.16 per crew-day across 567 crew-days |
| Joint-pattern detector | 0.009 false alerts per crew-day, about 20 times quieter than the per-indicator detector |
| Readiness forecast | 1.12 points mean error on a 0–10 scale; range covers the true value 89.6 % of the time |
| Action-plan learning | Reversed 64 % of drifts vs 52 % for the best fixed plan (simulated response) |
| Webcam heart rate | 4.9 bpm mean error on 5 UBFC-rPPG subjects |

All numbers are labelled "estimate, not validated", and the known limits are listed in [docs/accuracy.md](docs/accuracy.md).

**Skills:** Python, FastAPI, SQLite, React / TypeScript, Three.js, signal processing (rPPG), anomaly detection, forecasting, contextual bandits, local LLMs (Ollama), Playwright e2e tests, team leadership.

## How it works

Everything is **replay-first**: the web app runs from JSON replay files alone, and an optional local service plays the part of the habitat network.

```
 replay/*.json   4 crew days (PMData) · Pulse collapse · ISS cabin and dose · Inspiration4 labs
      │
      ├──────────────► HEALTH APP  (frontend/, index.html)   standalone, no service needed
      │                 3D body glowing by Status · trends and scrubber · forecast · forms · cards · Accuracy and Credits pages
      │                 Emergency button ────────────────┐
      ▼                                                  ▼
 HABITAT SERVICE (backend/service.py)         EMERGENCY SERVICE (backend/app/) + EMERGENCY APP
 Mission clock · replay player · SQLite       (frontend/, emergency.html)
 Detection (baseline, Acute and Drift,        trigger → urgency verdict → 15 s crew alarm →
 auto-detect Trigger, forecast) ─ Alerts ──►  rescuer takes over (record access logged) →
                                              likely contributors → cited steps, one per screen →
                                              "Why this step?" (Ollama) → ground report →
                                              ground console reply 28.5 min later
```

1. **Normal days** come from Earth wearable data (PMData), shaped into four synthetic crew. Each person's first 14 days set their **Anchor**; a 28-day sliding window is their **Adaptive baseline**.
2. **Detection** (`backend/detection.py`) raises **Acute** alerts (off the adaptive baseline) and **Drift** alerts (off the Anchor). Readings taken during an alert are held in **Quarantine**. **Status** (green, amber, red) is the worse of the deviation and the **Safety bounds**.
3. **Emergency**: the Pulse collapse replay trips the auto-detect rule, or anyone presses **Emergency** or **Help a crewmate**. Both open the same flow. Steps come only from the published microgravity CPR guideline, each with its citation. The language model explains why a step exists; it never writes or changes one.
4. **Mars delay**: the ground report shows a 28.5 minute one-way delay. The Mission clock can run at 1x, 60x or skip, so a 57 minute round trip fits a short demo.

## Quick start

Needs **Python 3.11+** and **Node.js 20+**. Ollama is optional (see [The explainer](#the-why-this-step-explainer-ollama)).

```bash
git clone https://github.com/SCAPE-NASA/stellarforge.git && cd stellarforge

# 1. Backend (once)
python3 -m venv backend/.venv
backend/.venv/bin/pip install -r backend/requirements.txt -r contracts/requirements.txt

# 2. Frontend (once)
cd frontend && npm ci && npm run build && cd ..
```

On Windows use `backend\.venv\Scripts\python` and `pip` in place of `backend/.venv/bin/...`.

### Run the emergency flow (service plus both pages)

```bash
backend/.venv/bin/python -m uvicorn app.main:app --app-dir backend --port 8000
```

| Open | What it is |
|---|---|
| http://localhost:8000/ | Health app: 3D body, trends, forecast, forms |
| http://localhost:8000/emergency.html | Tablet picker |
| http://localhost:8000/emergency.html?crew=B | One crew member's tablet (A to D) |
| http://localhost:8000/emergency.html?view=console | Ground console (the flight surgeon's window) |

The service starts with the real CPR protocol, the Mars transit profile and crew A to D. Press **Emergency** on the health app: it opens the flow on that person's tablet. Open a second tablet (`?crew=A`) in another window to be the rescuer. The crew alarm rings on every other tablet after 15 seconds without an answer.

For a **rehearsal** add `SF_SIM=1`. It turns on the demo clock and reset helpers (`/api/sim/*`); the 3 minute script is [docs/rehearsal.md](docs/rehearsal.md).

### Run just the health app (no service)

```bash
cd frontend && npm run dev          # http://localhost:5173, bundled Mars day-3 replay
npm run dev:real                    # a real shared replay (crew A with the planted drift)
```

`npm run dev` also serves `emergency.html` and proxies `/api` to a service on port 8000, so run both for the full hand-off during development.

### Run the habitat service with detection (optional, full chain)

```bash
backend/.venv/bin/python backend/service.py --db /tmp/habitat.sqlite \
  --replay replay/emergency-crew-B.json --crew replay/crew-records.json --port 8765
SF_HABITAT_URL=http://127.0.0.1:8765 backend/.venv/bin/python -m uvicorn app.main:app --app-dir backend --port 8000
```

The habitat service plays the replay on the Mission clock, runs detection, and pushes Alerts and Triggers; the emergency service then follows its clock. `tools/demo/demo_collapse.py` runs the same detection on the crew B collapse and writes the Alert and Trigger the demo recorder posts.

### The "Why this step?" explainer (Ollama)

The explainer needs a local [Ollama](https://ollama.com). The backend starts it for you if it is installed and not running.

```bash
ollama pull llama3.2:3b     # about 2 GB, once
```

Without Ollama the button still works and shows a pre-written, labelled **recorded answer**. Details and settings: [docs/explainer.md](docs/explainer.md).

## Tests and checks

```bash
backend/.venv/bin/python -m pytest backend contracts tools    # 693 tests, 7 skip without raw datasets
backend/.venv/bin/python tools/content_lint.py                # every step and bound sourced; no drugs or doses
backend/.venv/bin/python tools/lint_content.py --deny tools/drug-denylist.txt   # emergency-protocol linter
cd frontend
npx tsc && npm test                                           # typecheck and 247 unit tests
npx playwright install chromium                               # once
npm run test:e2e                                              # health app in a real browser
PYTHON=../backend/.venv/bin/python npm run test:e2e:emergency # emergency flow against the real service
```

The 7 skipped tests need the raw datasets, which are not in the repo (see [Data](#data)).

## Data

**Running the app needs no downloads.** The generated replays are committed in `replay/` and every one has a `.source.json` with its dataset, licence and file hash.

| Data | Used for | Licence |
|---|---|---|
| [PMData](https://datasets.simula.no/pmdata/) | Four crew's normal days; forecast training | CC BY 4.0 |
| Pulse Physiology Engine (Kitware) | The collapse replay (an estimate, not validated) | Apache 2.0 |
| [NASA OSDR EDA](https://visualization.osdr.nasa.gov/eda/), mission RR-1 | Real ISS cabin air and radiation dose | NASA open data, cited |
| NASA OSDR Inspiration4 (OSD-569) | Real astronaut blood counts | Public |
| NASA Apollo AP005 | Return-to-gravity reference values | Public group means |
| Z-Anatomy, Human Reference Atlas | 3D body (`frontend/public/models/`) | CC BY-SA 4.0 and CC BY 4.0 |

The raw downloads (PMData 1.4 GB, the Pulse bundle, EDA, UBFC-rPPG) are only needed to rebuild replays; commands are in [replay/README.md](replay/README.md) and sizes in [docs/datasets.md](docs/datasets.md). Every dataset, model and asset is credited in [docs/credits.md](docs/credits.md).

## Configuration

All optional. Defaults run the real content.

| Variable | Default | Effect |
|---|---|---|
| `SF_SIM` | off | `1` enables `/api/sim/*` (demo clock skip, reset, replay). Never leave on outside a rehearsal. |
| `SF_PROTOCOL` / `SF_PROFILE` / `SF_RECORDS` | `microgravity-cpr` / `mars_transit` / `replay/crew-records.json` | Set to `stub-protocol` / `stub-mars` / `replay/stub-crew-records.json` for the non-clinical test doubles |
| `SF_HABITAT_URL` | none | Follow a habitat service's Mission clock and post Triggers to it |
| `SF_DB` | in memory | SQLite file for the emergency service |
| `SF_CLOCK_SPEED` | `1` | Mission clock speed |
| `SF_OLLAMA_MODEL` / `SF_OLLAMA_URL` | `llama3.2:3b` / `http://127.0.0.1:11434` | Explainer model and server |
| `SF_OLLAMA_AUTOSTART` | on | `0` stops the backend starting a local Ollama |

## Repo layout

```
backend/     detection, forecast, habitat service (service.py), emergency service (app/)
frontend/    React + Three.js: health app (src/) and emergency flow (src/emergency/)
contracts/   Shared JSON Schemas, the only shapes that cross a stream boundary
content/     Protocols, measurement cards, mission profiles, detection cut-offs (every item sourced)
replay/      Generated crew days, collapse, ISS cabin, Inspiration4 labs, recorded replays
tools/       Content linters, replay generators, golden tests, demo recorder (tools/demo/)
docs/        Start at docs/README.md; plan coverage, datasets, accuracy, write-up, AI-use log
```

## Status against the plan

The P0 slice works end to end and the demo records all 13 scenes. Known gaps against the plan, with the full table in [docs/plan-coverage.md](docs/plan-coverage.md):

- Missing: the **daily briefing sentence**, **check-in streaks**, hosted replay on GitHub Pages.
- Partial: webcam heart rate has an algorithm and accuracy test but no morning-check screen.
- Not done: clinical review of the protocol and Safety bounds ([docs/medical-review.md](docs/medical-review.md)).
- **Future scope: the Collapse Copilot.** Its architecture is built and tested (briefing, rescuer choice, hash-chained decision record, ground-report section, L0 airway rule, L3 investigator, HotSummary adapter), but it is **off by default** (`content/copilot/copilot.json`). It needs training data and evaluation (scenario generator, L1/L2 ranking, golden replays, degraded modes) before it may advise a rescuer. With it off, routing and the approved protocols work as without it.

## Working rules

- **Contracts first.** Add a field to a contract only by agreement in the team chat. Never remove one.
- **One test seam.** A replay file goes in; the alerts, status and screens that come out are checked.
- **Cite everything.** Every protocol step and threshold needs a source. No drug names or doses, ever.
- **Honest labels.** Use "team value", "Earth reference, no space data", "estimate, not validated" where they apply.
- **Log AI use.** Add every use of an AI tool to [docs/si-log.md](docs/si-log.md). The event requires a statement.
- **Commits** are `type: description` (feat, fix, docs, test, chore, refactor). Open a pull request; do not push to `main`.

## Known limitations

- No hardware. All live data is simulated or replayed.
- No open astronaut wearable stream exists, so normal days are Earth wearable data.
- The collapse is a Pulse approximation with no ischemia model and a template ECG. Chest pain is crew-reported.
- Published space guidance exists only for CPR. Partial-gravity CPR (Moon, Mars) is not covered.
- Accuracy is measured on datasets, not real astronauts.
- The emergency service trusts who a tablet says it is (`crew=`, `actor=`, `viewer=`). There is no sign-in, so it is for a trusted habitat network only; the access log shows who asked, not who was verified.

## Licence

Code: MIT (see [LICENSE](LICENSE)). Derived 3D assets: CC BY-SA 4.0. Every dataset, model and asset is credited in [docs/credits.md](docs/credits.md).
