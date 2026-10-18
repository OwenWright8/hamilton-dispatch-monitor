# Hamilton County Dispatch Monitor

Listens to Hamilton County, Indiana fire/EMS dispatch radio (via
[OpenMHz](https://openmhz.com/system/hamiltonco)), transcribes each transmission, works out the address,
the type of emergency and which departments were sent, pins it on a map, and keeps statistics on where
calls happen, what kind, and when.

The Hamilton County edition of [dispatch-monitor](https://github.com/OwenWright8/dispatch-monitor)
(Tippecanoe County). Both share the same code; everything county-specific is in
[`dispatch/counties/hamilton.py`](dispatch/counties/hamilton.py).

## Channels

OpenMHz system `hamiltonco` (MESA System 3, Hamilton County simulcast):

| Talkgroup | Channel | |
|---|---|---|
| 29010 | 29 FDSP | Countywide fire dispatch: tone-outs with address and call type |
| 29006 | 29 FD CTRL | Countywide fire control: en route / on scene / status traffic |
| 29155 | 29 NFD DSP | Noblesville Fire dispatch |

## Departments

County dispatch calls units, not departments ("Engine 343, Medic 345, 9979 Gwyn Drive, sick person").
Every fire unit is numbered 3xx, and the second digit names the department, often shortened on the
air ("Ladder 72" is Ladder 372). Each incident lists the departments sent:

| Department | Units | Covers |
|---|---|---|
| Carmel Fire | 34x | Carmel / Clay Township |
| Fishers Fire | 39x | Fishers / Fall Creek Township |
| Noblesville Fire | 37x | Noblesville / Noblesville Township |
| Westfield Fire | 38x | Westfield / Washington Township |
| Cicero Fire | 32x | Cicero |
| Jackson Township Fire | 35x | Jackson Township / Arcadia & Atlanta |
| Sheridan Fire | 36x | Sheridan / Adams Township |
| Wayne Township Fire | 31x | Wayne Township |
| White River Township Fire | 33x | White River Township / Strawtown |
| Zionsville, Lawrence, Indianapolis, Lapel, Pendleton, Tipton | by name | Mutual aid from neighboring counties |

Sources: [Hamilton County fire department directory](https://www.hamiltoncounty.in.gov/577/County-Fire-Departments),
[RadioReference](https://wiki.radioreference.com/index.php/Hamilton_County_(IN)). If a department renumbers or a
name keeps being misheard, edit `unit_prefixes` / `aliases` in `hamilton.py`.

## How it works

```
OpenMHz API ──(curl_cffi, gets past Cloudflare)──► transmissions table (work queue)
                                                         │ newest first
                     faster-whisper small.en (speech → text, primed with local names)
                                                         │
                        name fixes from the county profile ("Sisero Fire" → Cicero Fire)
                                                         │
                         Ollama qwen2.5:7b, JSON-schema output (fallback: keyword rules)
                         → is it a dispatch? address, place, call type, units, patient
                         → rule checks: an alarm is not a fire; "Other Medical" only as a last resort
                         → departments: by unit number (34x = Carmel) or by name
                                                         │
          local TIGER/Line street index for the county (offline, fuzzy street names,
          house-number interpolation, intersections) → Census geocoder → Nominatim
                                                         │
          incident grouping: the same address within 20 min on any channel = one incident
                                                         │
                         SQLite ──► FastAPI ──► dashboard (map + statistics)
```

- **Fixed call-type taxonomy** (`dispatch/taxonomy.py`): about 55 types ("Fall", "Stroke",
  "Heart Problem / Blood Pressure", "Lift Assist", "Medical Alarm", "Fire Alarm", "Crash with Injuries"…)
  in 8 categories. An alarm or detector going off with no reported fire is a **Fire Alarm**, never a
  Structure Fire, and "Other Medical" is replaced by a specific type when the complaint names one.
- **Transcripts keep both versions.** The detail view shows the corrected transcript, with what Whisper
  first heard one click away.
- **Location precision** is recorded (`address`, `intersection`, `place`, `street`). The map draws
  street-only matches as hollow markers.
- Live calls are processed first; history backfills behind them. Restarts lose nothing.

## Dashboard

`http://<host>:8087/`

- **Map**: incident markers colored by category (calls from the last 15 minutes pulse and are marked
  NEW) or a heatmap, plus a live feed with the departments sent. Click an incident for its transcripts,
  audio, and earlier incidents at the same address.
- **Statistics**: incidents per day, by category, top call types, a weekday × hour heatmap, hour of
  day by category, repeat locations, by ZIP, and by department.
- Filters (time range, category, department, search, weekday/hour, call type, "only map area") apply to
  everything at once. **Export CSV** downloads the filtered incidents. API docs are at `/api/docs`.

## Install

You need Docker. Everything else, including the AI model server, runs in the stack.

```bash
mkdir hamilton-dispatch-monitor && cd hamilton-dispatch-monitor
curl -fsSLO https://raw.githubusercontent.com/OwenWright8/hamilton-dispatch-monitor/main/docker-compose.yml
docker compose up -d
```

Open **http://<that machine's IP>:8087**. On first start the app loads the last 24 hours of calls in the
background while it listens live; **Load past calls** in the header fetches more.

**Dockge / Portainer:** create a stack, paste in [`docker-compose.yml`](docker-compose.yml), and deploy.

**Ollama already on the host** (to use its GPU): remove the `ollama` service and `depends_on` from the
compose file, and set `OLLAMA_URL=http://host.docker.internal:11434` with
`extra_hosts: ["host.docker.internal:host-gateway"]` on the app.

**Running next to the Tippecanoe edition:** they are separate stacks with separate data; this one
defaults to port 8087 so both fit on one machine and can share one Ollama.

**Update:** `docker compose pull && docker compose up -d`.

### Building from source

```bash
git clone https://github.com/OwenWright8/hamilton-dispatch-monitor.git && cd hamilton-dispatch-monitor
# in docker-compose.yml, swap the app's "image:" line for "build: ."
docker compose up -d --build
```

GitHub Actions publishes `ghcr.io/owenwright8/hamilton-dispatch-monitor` (amd64 + arm64) on pushes to
`main`. The Ollama image comes from the upstream repo.

## Settings

Most settings can be changed from the dashboard: **Settings** in the header. That covers:
- the AI model (on/off, Ollama address, model, timeout)
- the speech-to-text model and beam size
- which channels are monitored, how often to poll, and the minimum clip length
- the incident merge window
- whether audio is kept
- the CARTO map key

Values saved there are stored in the database, override `.env`, and apply within a few seconds without a
restart. **Reset** goes back to the `.env` / built-in value. If the dashboard is reachable from the
internet, set `ADMIN_PASSWORD` in `.env`. Saving settings then requires it, while viewing stays open.

**Data retention** (also in Settings): dispatch audio clips are kept for 24 hours; call details (transcripts,
summaries, patient age/sex, units) for 7 days; and the incident itself (call type, location, departments,
time) for 365 days, which is what the statistics use. `python -m dispatch reprocess` rebuilds only the
incidents that still have transcripts.

## Versions

Releases are tagged `vMAJOR.MINOR.PATCH` and listed on the GitHub *Releases* page with what changed. Each
release publishes the image as `:1.0.0`, `:1.0` and `:1` as well as `:latest`. To stay on a version,
use e.g. `ghcr.io/owenwright8/hamilton-dispatch-monitor:1.0` in `docker-compose.yml`. The running version
is shown in the Settings panel.

## Configuration

All settings are optional and go in `.env` (see `.env.example`). The ones marked ⚙ can also be changed
in the dashboard:

| Variable | Default | |
|---|---|---|
| `ADMIN_PASSWORD` | empty | required to save settings in the dashboard when set |
| `OLLAMA_URL` ⚙ | `http://ollama:11434` | an Ollama on the host: `http://host.docker.internal:11434` |
| `OLLAMA_MODEL` ⚙ | `qwen2.5:7b` | pulled automatically if missing |
| `LLM_ENABLED` ⚙ | `true` | `false` = keyword rules only, no Ollama |
| `WHISPER_MODEL` ⚙ | `small.en` | `base.en` / `small.en` / `medium.en` / `large-v3` |
| `BACKFILL_HOURS` | `24` | history to load on first start; `0` = live only |
| `CHANNELS` | the three above | `talkgroup:label:city-hint,…`; a label naming a department credits it |
| `AUDIO_RETENTION_HOURS` ⚙ | `24` | delete audio clips after this |
| `DETAIL_RETENTION_DAYS` ⚙ | `7` | delete transcripts, summaries, patient info and units after this |
| `INCIDENT_RETENTION_DAYS` ⚙ | `365` | delete incidents (type, location, departments) after this |
| `KEEP_AUDIO` ⚙ | `true` | keep dispatch clips (about 100 KB each) in `/data/audio` |
| `INCIDENT_WINDOW_MIN` ⚙ | `20` | transmissions merge into one incident within this window |
| `WEB_PORT` | `8087` | host port |
| `CARTO_API_KEY` ⚙ | empty | [CARTO basemaps](https://carto.com/basemaps) key for the dark/light map style |

## Maintenance

```bash
# Re-run extraction + geocoding over every stored transcript, e.g. after editing hamilton.py:
docker compose exec app python -m dispatch reprocess

# Back up: everything lives in the dispatch-data volume (/data/dispatch.db + /data/audio)
```

## Caveats

- OpenMHz is a free community service behind Cloudflare. If polling fails the header shows
  **Feed error**; updating `curl_cffi` usually fixes it.
- Radio transcription is imperfect. The detail view shows the transcript and audio so you can judge.
- Smaller departments are only credited when their unit numbers or names are heard; a department that
  answers only on its own talkgroup (not monitored here) won't show up.
