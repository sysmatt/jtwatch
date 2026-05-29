# jtwatch

A terminal based (CLI) monitor for WSJT-X CQ calls. Listens on the WSJT-X UDP broadcast port and prints every CQ decode to stdout, enriched with DXCC entity, CQ zone, ITU zone, continent, and grid square from the AD1C `cty.dat` prefix database.

With an ADIF logbook loaded, jtwatch flags contacts whose entity, CQ zone, or country you have not yet worked as **NEEDED**.

When invoked with the `--call` option, jtwatch rings the terminal bell and prompts the operator for confirmation, then tells WSJT-X to begin responding to the CQ (the same as double-clicking the call in the WSJT-X GUI). **NOTE:** This is NOT an automation tool — you must initiate each reply.

Optional regex watchlists let you flag specific callsigns or message patterns. External alerts can be sent via a custom script or [ntfy.sh](https://ntfy.sh) push notifications.

> **A note on automation:** jtwatch does not automate contacts. Using `--call` is no different from sitting at the WSJT-X UI and double-clicking a callsign yourself — you are still the operator making every decision. The goal of these tools is to help you *find* contacts that matter to you: the entity you need for DXCC, the state you're missing for WAS, the POTA activation you want to chase. jtwatch surfaces those opportunities; you decide whether to call.

### Note: Tested with WSJT-X 3.1.0 Improved

## Features

- Zero external dependencies — Python standard library only (`sqlite3` is bundled with Python)
- Auto-downloads and caches `cty.dat` on first run
- ADIF log comparison with per-category worked/needed status; tracks unique callsigns, grid squares, and states worked; shows days since last worked per callsign
- Per-callsign QSL status indicators: ⭐ (QSL card received) and 🌎 (LoTW confirmed) columns in every decode row
- **QSL-only mode** — `--qsl-only` narrows NEEDED detection to only LoTW-confirmed QSOs; unconfirmed contacts remain flagged as still needed
- Regex watchlists for callsigns and full decoded messages
- Built-in `--pota`, `--sota`, `--iota` flags to match activations without a watchlist file
- **WAS state matching** — `--match-state` resolves callsigns to their FCC-licensed state via [hamdat](https://github.com/sysmatt/hamdat) and flags matching CQs; adds a `ST` column to every output line (Obviously imperfect as an operator might not be working in their home state)
- **Cycle statistics** — `--stats` prints a `STATS:` summary after each decode cycle with total and CQ decode counts plus running averages split by even/odd slot
- External alerting via script (`--alert-command`) or ntfy.sh push (`--alert-ntfy TOPIC`)
- Deduplicates alerts within a run — no repeated notifications for the same event
- ADIF log auto-reloads when the file changes — no restart needed after logging a QSO; optional `--adif-change-script` runs a custom script on each reload
- Interactive call prompt — answer yes and jtwatch sends a WSJT-X Reply (Type 4) UDP message to start TX automatically
- Optional ANSI color output (`--color`) for at-a-glance status scanning, including per-field green highlighting for previously worked callsigns, grids, and states
- Fixed-width columnar output for easy terminal scanning
- Optional JSON-lines output for downstream processing
- **UDP proxy** — `--proxy PORT[,PORT...]` forwards every received WSJT-X packet to additional localhost ports, letting jtwatch and other tools (e.g. GridTracker) share the same data stream without competing for port 2237
- Portable suffix handling (`W1ABC/P`, `HC2/DH1TW`, `VK9/W1ABC`, etc.)

## Requirements

- Python 3.10 or later
- WSJT-X configured to send UDP broadcasts (see below)
- [hamdat](https://github.com/sysmatt/hamdat) SQLite database — optional, only required for `--match-state`

## Installation

```bash
# Clone or download
git clone https://github.com/sysmatt/jtwatch.git
cd jtwatch

# Make executable (Linux/macOS)
chmod +x jtwatch

# Run it, Requires python3 
./jtwatch

# Optional: put it on your PATH
sudo ln -s "$PWD/jtwatch" /usr/local/bin/jtwatch
```

`cty.dat` is downloaded automatically from [country-files.com](https://www.country-files.com/cty/cty.dat) on first run and cached at `~/.jtwatch_cty.dat`. Delete the cache file to force a refresh.

## WSJT-X Setup

In WSJT-X, go to **File → Settings → Reporting** and confirm:

- **UDP Server** is enabled
- **UDP Server port number** matches `--port` (default: `2237`)
- **Accept UDP requests** is checked (optional, not required for receive-only)

jtwatch only listens — it never sends back to WSJT-X.

## Basic Usage

```bash
# Live CQ monitor, no log comparison
jtwatch

# With ADIF log — bell + NEEDED flag for new entities/zones/countries
jtwatch --adif ~/Documents/wsjtx_log.adi

# Multiple ADIF files merged — e.g. WSJT-X log + LoTW export
jtwatch --adif ~/Documents/wsjtx_log.adi ~/Downloads/lotw_export.adi

# Show all decodes, not just CQ, with local wall-clock time
jtwatch --raw --show-time

# Different port
jtwatch --port 2237
```

## Output Format

The column header is printed on startup and repeats every 15 lines so it stays visible as output scrolls:

```
HHMMSSZ     dB    dt(s)      Hz  mode  message                   callsign      grid     days  | entity                              CQz   ITUz   ct  QRZ  LoW
------------------------------------------------------------------------------------------------------------------------------------------------------------
```

Column descriptions:

| Column | Example | Description |
|--------|---------|-------------|
| `HHMMSSZ` | `120145z` | UTC period start time from WSJT-X |
| `dB` | ` +12` | Signal-to-noise ratio (4 chars) |
| `dt(s)` | `  +0.2s` | Delta Time offset from period start (7 chars) |
| `Hz` | `  1234` | Audio frequency (5 chars) |
| `mode` | `FT8 ` | Digital mode (4 chars) |
| `message` | `CQ DX W1ABC FN42` | Full decoded message (24 chars) |
| `callsign` | `W1ABC` | Extracted callsign (12 chars) |
| `grid` | `FN42` | Maidenhead grid square (6 chars) |
| `days` | `  42d` | Days since callsign was last worked — only shown with `--adif` |
| `ST` | `NJ` | FCC-licensed state — only shown with `--match-state` |
| `entity` | `United States` | DXCC entity from cty.dat |
| `CQz` | `CQ5` | CQ zone |
| `ITUz` | `ITU8` | ITU zone |
| `ct` | `NA` | Continent (2-char code) |
| `QRZ` | `⭐` | ⭐ if QRZ logbook confirmed (`app_qrzlog_status=C`) — only shown with `--adif` |
| `LoW` | `🌎` | 🌎 if LoTW confirmation received (`lotw_qsl_rcvd=Y`) — only shown with `--adif` |
| `[status]` | `[worked dxcc cqz country]` | Worked/NEEDED/MATCH status |

Sample output:

```
HHMMSSZ     dB    dt(s)    Hz  mode  message                   callsign      grid     days  | entity                              CQz   ITUz   ct  QRZ  LoW
----------------------------------------------------------------------------------------------------------------------------------------------------------
120145z   +12    +0.2s   1234  FT8   CQ DX W1ABC FN42          W1ABC         FN42      42d  | United States                   CQ5   ITU8   NA  ⭐  🌎
120200z    -5    -1.1s    234  FT4   CQ W2XYZ                  W2XYZ                        | Germany                         CQ14  ITU28  EU      
120215z   +30    +1.0s   2899  FT8   CQ POTA VK2ABC QF56       VK2ABC        QF56           | Australia                       CQ29  ITU59  OC      
```

### Color output (`--color`)

Pass `--color` to enable ANSI color coding for at-a-glance scanning:

| Element | Color | Rationale |
|---------|-------|-----------|
| Time, dB, dt, Hz, mode | dim | Reference data — de-emphasized |
| Callsign | bold | Primary identifier |
| Callsign | bold green | Callsign appears in your ADIF log (previously worked) |
| Grid | green | Grid square appears in your ADIF log (previously worked) |
| `ST` column | green | State has been worked (callsigns from that state in your ADIF log, resolved via hamdat) |
| Entity block | cyan | Enrichment data |
| `*** NEEDED: ... ***` | bold red | Unworked entity/zone/country |
| `*** MATCH: ... ***` | bold yellow | Watchlist hit |

```bash
jtwatch --adif ~/wsjtx_log.adi --color
jtwatch --adif ~/wsjtx_log.adi --color --alert-ntfy my-ham-alerts
```

## ADIF / NEEDED Mode

Load your ADIF logbook to compare incoming CQs against what you have already worked:

```bash
jtwatch --adif ~/Documents/wsjtx_log.adi
```

jtwatch builds several worked sets from the log:

| Set | Content | Used for |
|-----|---------|----------|
| Callsigns | Every `CALL` field in the log | Bold green callsign in output (`--color`); days-since-worked column |
| Grid squares | Every `GRIDSQUARE` field (first 4 chars) | Green grid in output (`--color`) |
| States | FCC-licensed state of each worked callsign, resolved via hamdat | Green `ST` column (`--color` + `--match-state`) |
| DXCC entities | Primary prefix e.g. `W`, `DL`, `VK` | `*** NEEDED ***` detection |
| CQ zones | Zone number 1–40 | `*** NEEDED ***` detection |
| Countries | Country name string | `*** NEEDED ***` detection |
| QRZ confirmed | Callsigns where `app_qrzlog_status=C` | ⭐ in `QRZ` column |
| LoTW confirmed | Callsigns where `lotw_qsl_rcvd=Y` | 🌎 in `LoW` column |

On startup, jtwatch reports the count of each set to stderr, for example:

```
[adif] Loaded 1842 QSOs from wsjtx_log.adi
[adif]   743 unique callsigns worked
[adif]   291 grid squares worked
[adif]   38 states worked
[adif]   64 DXCC entities worked
[adif]   18 CQ zones worked
[adif]   61 countries worked
```

Any CQ from a callsign whose entity, zone, or country is **not** in the log is flagged with `*** NEEDED: ... ***`.

Already-worked contacts show `[worked dxcc]`, `[worked dxcc cqz country]`, etc. — listing only the categories that are both enabled and matched.

The ADIF file is checked for changes on every received packet. If WSJT-X logs a new QSO while jtwatch is running, the log reloads automatically — no restart required.

### Disabling individual NEEDED checks

```bash
# Only alert on new DXCC entities, ignore zone and country
jtwatch --adif ~/wsjtx_log.adi --no-need-cqz --no-need-country

# Only alert on new CQ zones
jtwatch --adif ~/wsjtx_log.adi --no-need-entity --no-need-country
```

### `--worked-lag-days DAYS` — suppress alerts for recently worked callsigns

By default, if a callsign appears in your ADIF log with a contact from **today**, NEEDED and MATCH alerts for that callsign are suppressed (you've already worked them this session). `--worked-lag-days` controls how far back that suppression reaches — for both NEEDED and MATCH events.

| Value | Behaviour |
|-------|-----------|
| `0` (default) | Suppress if worked today (0 days ago) |
| `7` | Suppress if worked within the last 7 days |
| `-1` | Disable suppression entirely — always alert regardless of prior contacts |

```bash
# Suppress for a week — don't re-alert on calls worked recently
jtwatch --adif ~/wsjtx_log.adi --worked-lag-days 7

# Never suppress — always alert even if worked today
jtwatch --adif ~/wsjtx_log.adi --worked-lag-days -1
```

The suppression applies consistently to all alert types: NEEDED (entity/zone/country), MATCH (callsign patterns, message patterns, POTA/SOTA/IOTA, state matches).

### `--qsl-only` — require LoTW confirmation for NEEDED

When chasing DXCC or other awards that require confirmed contacts, `--qsl-only` tightens the NEEDED check: only QSOs with `lotw_qsl_rcvd=Y` are counted as "worked" for entity, zone, and country purposes. A contact you've made but not yet confirmed on LoTW will still show as **NEEDED**.

```bash
jtwatch --adif ~/wsjtx_log.adi --qsl-only
```

Callsign and grid coloring (green for previously worked) is unaffected — it reflects all contacts in the log regardless of QSL status. The LoW column (🌎) shows which callsigns have LoTW confirmations, making it easy to spot who you've worked but not yet confirmed.

## Pattern Matching

Watchlists let you flag specific callsigns or message patterns regardless of your log.

### `--match-calls FILE [FILE ...]`

Each file contains one Python regular expression per line. Patterns are tested against the **resolved callsign**. Matching CQs are flagged `*** MATCH: CALL:<pattern> [<filename>] ***`, where `<filename>` is the basename of the file the pattern came from.

```bash
jtwatch --match-calls dx_watchlist.txt
jtwatch --match-calls rare_dx.txt friends.txt
```

Example `dx_watchlist.txt`:

```
# Rare DX I'm chasing
^3D2
^VP8
^ZL9
# A specific callsign
^ZD8W$
```

### `--match-message FILE [FILE ...]`

Same file format, but patterns are tested against the **full decoded message string** (e.g. `CQ POTA W1ABC FN42`). Matches are flagged `*** MATCH: MSG:<pattern> [<filename>] ***` (note the `MSG:` prefix, vs `CALL:` for callsign patterns). The `<filename>` shows which watchlist file the pattern came from. Useful for catching activity modifiers or any other text in the decoded message.

```bash
jtwatch --match-message activations.txt
```

### `--match-state STATES` — Worked All States (WAS)

Resolves each CQ callsign to its FCC-licensed US state using a [hamdat](https://github.com/sysmatt/hamdat) SQLite database and flags matching stations as `*** MATCH: STATE:XX ***`. A `ST` column is added to every CQ output line showing the resolved state (blank for non-US or unlicensed callsigns).

States can be supplied as a comma-separated list or a file with one abbreviation per line (`#` = comment):

```bash
# Match specific states inline
jtwatch --match-state TX,CA,NY

# Load from a file (one state per line)
jtwatch --match-state was_needed.txt

# Alternate hamdat database path
jtwatch --match-state TX,FL --hamdat /data/hamdat.db
```

Example `was_needed.txt`:

```
# States still needed for WAS
AK
HI
ND
SD
MT
```

The state is resolved from the FCC ULS record for the callsign's **current active license** — this is the licensee's registered QTH, which is what counts for WAS credit.

**Valid state abbreviations (FCC ULS codes):**

```
AL  AK  AZ  AR  CA  CO  CT  DE  FL  GA  HI  ID  IL  IN  IA  KS  KY  LA
ME  MD  MA  MI  MN  MS  MO  MT  NE  NV  NH  NJ  NM  NY  NC  ND  OH  OK
OR  PA  RI  SC  SD  TN  TX  UT  VT  VA  WA  WV  WI  WY  DC
```

**Prerequisite:** Build the hamdat database first:

```bash
hamdat --pull        # downloads FCC ULS data (~300 MB), takes a few minutes
```

The database is stored at `~/.hamdat/hamdat.db` by default. See the [hamdat repository](https://github.com/sysmatt/hamdat) for details.

### `--pota` / `--sota` / `--iota`

Shorthand flags that inject the corresponding `\bPOTA\b`, `\bSOTA\b`, or `\bIOTA\b` message pattern without needing a file. Each flag behaves identically to having that pattern in a `--match-message` file.

```bash
# Alert on any POTA or SOTA activation
jtwatch --pota --sota

# Combine with ADIF and ntfy alerts
jtwatch --adif ~/wsjtx_log.adi --pota --sota --iota --alert-ntfy my-ham-alerts
```

A POTA activation would appear as:

```
120215z   +30    +1.0s   2899  FT8   CQ POTA VK2ABC QF56       VK2ABC        QF56    | Australia                       CQ29  ITU59  OC  *** MATCH: MSG:\bPOTA\b ***
```

### Pattern file format

- One regular expression per line
- Case-insensitive matching
- Lines starting with `#` are comments
- Blank lines are ignored
- Multiple files are merged into a single pattern list

## Cycle Statistics (`--stats`)

Pass `--stats` to print a `STATS:` summary line after each completed decode cycle. The line reports the decode counts for the just-finished cycle and cumulative running averages, with even- and odd-slot totals tracked separately.

```bash
jtwatch --stats
jtwatch --adif ~/wsjtx_log.adi --stats --color
```

Example output:

```
STATS: 120000z [even]  decodes=47 cq=12  avg: dec=38.2 cq=9.1  even: dec=39.1 cq=9.8  odd: dec=37.4 cq=8.5
```

| Field | Description |
|-------|-------------|
| `120000z [even]` | UTC period time and slot parity of the completed cycle |
| `decodes=47` | Total decoded signals in that cycle (CQ + non-CQ) |
| `cq=12` | CQ calls decoded in that cycle |
| `avg: dec=38.2 cq=9.1` | Running average across all cycles since startup |
| `even: dec=39.1 cq=9.8` | Running average for even slots only |
| `odd: dec=37.4 cq=8.5` | Running average for odd slots only |

Slot parity follows the FT8 convention: `(time_ms // 15000) % 2` — even slots at 0 s and 30 s, odd slots at 15 s and 45 s within each minute. FT4 uses 7.5-second slots with the same even/odd alternation. Only fresh decodes (`new=True`) are counted; replayed decodes from waterfall scrolling are ignored.

## External Alerts

When a NEEDED or MATCH event fires, jtwatch can notify you via a script or a push notification. Both methods deduplicate: the same callsign with the same flags will only trigger one alert per run.

The alert message is a compact one-liner:

```
W1ABC | United States CQ5 | NEEDED: NEW-DXCC(W), NEW-CQZ(5)
VK2ABC | Australia CQ29 | MATCH: CALL:VK2ABC
```

### `--adif-change-script SCRIPT`

Calls `SCRIPT <path> [<path> ...]` (non-blocking) each time the ADIF log file(s) are reloaded due to a detected file change. Each ADIF file path is passed as a separate argument. This runs immediately after the reload completes, making it useful for triggering downstream processing (e.g. re-exporting data, sending a notification, or updating a scoreboard).

```bash
jtwatch --adif ~/Documents/wsjtx_log.adi --adif-change-script ~/bin/on_log_reload.sh
```

Example `on_log_reload.sh`:

```bash
#!/bin/bash
# $@ contains the ADIF file path(s) that were just reloaded
notify-send "jtwatch" "ADIF log reloaded: $*"
```

### `--alert-command SCRIPT`

Calls `SCRIPT "<alert message>"` (non-blocking) on each new event. The script receives the message as `$1`.

```bash
jtwatch --adif ~/wsjtx_log.adi --alert-command ~/bin/ham_alert.sh
```

Example `ham_alert.sh`:

```bash
#!/bin/bash
notify-send "jtwatch" "$1"
```

### `--alert-ntfy TOPIC`

POSTs the alert message to `https://ntfy.sh/TOPIC`. Install the ntfy app on your phone and subscribe to the same topic to receive push notifications.

```bash
jtwatch --adif ~/wsjtx_log.adi --alert-ntfy my-ham-alerts
```

#### ntfy.sh resources

- **Website & docs:** [https://ntfy.sh](https://ntfy.sh)
- **Android app (Google Play):** [https://play.google.com/store/apps/details?id=io.heckel.ntfy](https://play.google.com/store/apps/details?id=io.heckel.ntfy)
- **Android app (F-Droid):** [https://f-droid.org/en/packages/io.heckel.ntfy/](https://f-droid.org/en/packages/io.heckel.ntfy/)
- **iPhone/iPad app (App Store):** [https://apps.apple.com/us/app/ntfy/id1625396347](https://apps.apple.com/us/app/ntfy/id1625396347)

ntfy.sh is a free, open-source pub/sub notification service — no account required for basic use. Pick any topic name (e.g. `my-ham-alerts`), subscribe to it in the app, and any POST to `https://ntfy.sh/<topic>` will push to your device instantly. Choose an obscure topic name to avoid receiving messages from others. Your callsign makes a natural choice — e.g. `--alert-ntfy W1ABC-jtwatch`.

Both `--alert-command` and `--alert-ntfy` can be used simultaneously.

## Interactive Call Prompt

`--call` turns jtwatch into a basic call/no-call decision tool. On each new NEEDED or MATCH event, jtwatch rings the terminal bell (`\a`) and writes a prompt to stderr, then waits for your answer:

```
Do you want to call W1ABC? [Y/n]
```

- **Enter** or **y** → yes; jtwatch prints `[call] Calling W1ABC` to stderr
- **n** → no
- **No input within `--call-timeout` seconds** → automatically answers no

```bash
# Prompt with 15-second default timeout
jtwatch --adif ~/wsjtx_log.adi --call

# Tighter window — auto-skip after 8 seconds
jtwatch --adif ~/wsjtx_log.adi --call --call-timeout 8

# Combine with ntfy so your phone buzzes at the same time
jtwatch --adif ~/wsjtx_log.adi --call --alert-ntfy my-ham-alerts
```

Each callsign is prompted (and belled) at most once per run. If the same station calls CQ repeatedly, you won't be asked again. The prompt is suppressed entirely when stdin is not a terminal (piped or scripted runs).

When you answer **yes**, jtwatch sends a **Reply (Type 4)** UDP message back to WSJT-X. This is the same action as double-clicking a decode in the WSJT-X UI: WSJT-X sets the DX callsign, generates the appropriate response message, and enables TX.

**Prerequisite:** In WSJT-X, go to **Settings → Reporting** and check **"Accept UDP requests"**. Without this, WSJT-X silently ignores the Reply message and will not transmit.

## JSON Output

Append every CQ record as a JSON line to a file for logging or downstream analysis:

```bash
jtwatch --save cq.jsonl
jtwatch --adif ~/wsjtx_log.adi --save cq.jsonl
```

Each line is a complete JSON object:

```json
{
  "wall_utc": "2024-11-15T12:01:45Z",
  "period_utc": "120145z",
  "snr_db": 12,
  "dt_s": 0.2,
  "df_hz": 1234,
  "mode": "FT8",
  "message": "CQ DX W1ABC FN42",
  "callsign": "W1ABC",
  "modifier": "DX",
  "gridsquare": "FN42",
  "primary_prefix": "W",
  "entity": "United States",
  "cq_zone": 5,
  "itu_zone": 8,
  "continent": "NA",
  "needed": []
}
```

`needed` is `null` when no ADIF log is loaded, an empty list when all checks pass, or a list of flag strings (e.g. `["NEW-DXCC(DL)"]`) when the contact is new.

## UDP Proxy

By default, only one process can usefully bind a UDP port at a time. If you run jtwatch alongside GridTracker, WSJT-X Log, or any other tool that listens on port 2237, the two processes will compete for packets — each receiving roughly half the stream and appearing broken.

`--proxy` solves this by making jtwatch the single listener on port 2237. It forwards every raw packet it receives — verbatim, before any processing — to one or more additional localhost ports. Other tools connect to those forwarded ports and see the full unmodified WSJT-X protocol stream, including all message types (Heartbeat, Status, QSO Logged, Decode, etc.).

```bash
# jtwatch on 2237, GridTracker pointed at 2238
jtwatch --adif ~/wsjtx_log.adi --proxy 2238

# Forward to two additional tools
jtwatch --adif ~/wsjtx_log.adi --proxy 2238,2239
```

**Setup:**

1. In WSJT-X (**File → Settings → Reporting**), set the UDP server address to `127.0.0.1` and port to `2237` (jtwatch's port — unchanged).
2. In GridTracker (or the other tool), change its UDP port from `2237` to `2238` (or whichever port you chose).
3. Run jtwatch with `--proxy 2238`.

On startup, jtwatch confirms the forwarding destinations:

```
[proxy] Forwarding all packets to: localhost:2238
```

If a destination isn't listening yet, send errors are silently ignored — jtwatch continues normally and the proxy resumes automatically when the other tool connects.

## Options Reference

| Option | Default | Description |
|--------|---------|-------------|
| `--host HOST` | `0.0.0.0` | UDP bind address |
| `--port PORT` | `2237` | UDP port (match WSJT-X Reporting settings) |
| `--proxy PORT[,PORT...]` | — | Forward all received packets to these additional localhost ports; lets other tools share the WSJT-X stream without competing for port 2237 |
| `--show-time` | off | Prepend local wall-clock `HH:MM:SS` to each line |
| `--raw` | off | Print all decoded messages, not just CQ |
| `--save FILE` | — | Append CQ records as JSON lines to FILE |
| `--cty PATH` | `~/.jtwatch_cty.dat` | Path to cty.dat prefix database |
| `--adif FILE [FILE ...]` | — | ADIF logbook(s) for NEEDED detection; multiple files are merged |
| `--no-need-entity` | — | Disable DXCC entity NEEDED check |
| `--no-need-cqz` | — | Disable CQ zone NEEDED check |
| `--no-need-country` | — | Disable country name NEEDED check |
| `--qsl-only` | off | Require LoTW confirmation (`lotw_qsl_rcvd=Y`) for NEEDED checks; unconfirmed QSOs count as still needed |
| `--worked-lag-days DAYS` | `0` | Suppress NEEDED and MATCH alerts for callsigns worked within this many days; `-1` disables suppression entirely |
| `--match-calls FILE [...]` | — | Callsign regex watchlist file(s) |
| `--match-message FILE [...]` | — | Full-message regex watchlist file(s) |
| `--pota` | off | Flag CQ POTA calls as MATCH (no file required) |
| `--sota` | off | Flag CQ SOTA calls as MATCH (no file required) |
| `--iota` | off | Flag CQ IOTA calls as MATCH (no file required) |
| `--match-state STATES` | — | Comma-separated state abbreviations or file; flags matching FCC-licensed states as MATCH and adds a `ST` column |
| `--hamdat DB` | `~/.hamdat/hamdat.db` | hamdat SQLite database path (used with `--match-state`) |
| `--stats` | off | Print a `STATS:` summary line after each decode cycle with decode counts and running averages by even/odd slot |
| `--adif-change-script SCRIPT` | — | Script to run (non-blocking) after each ADIF reload; file path(s) passed as arguments |
| `--alert-command SCRIPT` | — | Script to call on NEEDED/MATCH events |
| `--alert-ntfy TOPIC` | — | POST push alerts to ntfy.sh topic TOPIC |
| `--call` | off | Ring the terminal bell and prompt to call on NEEDED/MATCH events; each callsign prompted at most once per run |
| `--call-timeout SECONDS` | `15` | Seconds before call prompt auto-answers no |
| `--color` | off | Colorize output with ANSI escape codes |

## Deeper explanation of NEEDED/MATCH logic and implementation

### Two independent alert systems

**NEEDED** and **MATCH** both ring the bell and print `*** ... ***` lines, but they answer completely different questions and are computed separately.

---

### NEEDED — "have I worked this entity before?"

Requires `--adif`. Fires when the incoming station's **DXCC entity**, **CQ zone**, or **country** (any combination, independently) is not yet in your log. These are entity-level checks, not callsign-level.

The ADIF log populates **three pairs of sets** at startup:

| Set | What goes in | Used for |
|---|---|---|
| `worked_dxcc` | QSL-filtered DXCC prefixes | NEEDED check |
| `worked_dxcc_all` | Every worked DXCC prefix | ❓ display |
| `worked_cqz` | QSL-filtered CQ zones | NEEDED check |
| `worked_cqz_all` | Every worked CQ zone | ❓ display |
| `worked_country` | QSL-filtered countries | NEEDED check |
| `worked_country_all` | Every worked country | ❓ display |

Without `--qsl-only`, "filtered" and "all" are identical — every QSO counts.

With `--qsl-only`, only QSOs where **`lotw_qsl_rcvd=Y`** populate the filtered sets. The `_all` sets always get every QSO regardless.

**Note on DXCC population:** The code uses `cty.dat` to resolve callsigns from the log (not just the ADIF `dxcc` field), so the entity match at decode time and the entity recorded in the log use the same lookup engine. This avoids false NEEDED alerts from ADIF/cty.dat country-name mismatches.

---

### The three entity-block emojis (✅ ❓ ❌)

These appear per-category (DXCC / CQZ / country) in the entity block for each decoded line:

- **✅** — in the **filtered** worked set (`worked_dxcc` / `worked_cqz` / `worked_country`)
- **❓** — in `_all` but **not** in the filtered set — meaning worked in some QSO but **no LoTW confirmation yet** (only meaningful when `--qsl-only` is active; without it ✅ and ❓ are the same)
- **❌** — not in either set; genuinely never worked

When `--qsl-only` is off, you will only ever see ✅ or ❌. The ❓ state only exists when the two sets can diverge.

---

### MATCH — "is this station on a watchlist?"

Completely independent of the log. Fires based on patterns or rules you specify:

- `--match-calls FILE` — regex on the callsign itself
- `--match-message FILE` — regex on the full decoded message string
- `--pota` / `--sota` / `--iota` — built-in message pattern shortcuts (compiled into the message pattern list)
- `--match-state STATES` — FCC state lookup via hamdat SQLite; matches callsigns licensed in specified states

MATCH does not care whether you have worked the entity. It fires even on a callsign you have worked 50 times, unless lag suppression kicks in.

---

### QRZ ⭐ and LoTW 🌎 column indicators

These are **per-callsign, display-only** indicators. They tell you whether you have a logged confirmation *for that specific callsign*:

- **⭐** — callsign has `app_qrzlog_status=C` in the ADIF (QRZ logbook confirmation received)
- **🌎** — callsign has `lotw_qsl_rcvd=Y` in the ADIF (LoTW QSL received for a contact with this call)

**Important:** These indicators have no effect on NEEDED logic. They show per-callsign confirmation, whereas NEEDED operates on per-entity sets. You can have 🌎 for a callsign and still see a NEEDED flag if that entity's primary prefix is somehow missing from `worked_dxcc` — for example after clearing your log or if the cty.dat prefix assignment changed.

With `--qsl-only`, the 🌎 indicator is indirectly meaningful because `lotw_qsl_rcvd=Y` on a QSO is what determines whether that QSO's entity/zone/country counted toward the filtered sets. But the indicator itself is just a callsign-level display fact.

---

### Per-callsign tracking (always ignores `--qsl-only`)

These sets and dicts are always populated from every QSO, regardless of QSL status:

| Field | Tracks | Used for |
|---|---|---|
| `worked_calls` | Every worked callsign | Bold-green callsign color |
| `worked_grids` | Every worked grid square | Green grid column |
| `worked_states` | US states of worked callsigns (via hamdat) | Green `ST` column |
| `last_worked[call]` | Most recent QSO date per callsign | Days column; lag suppression |

---

### `--worked-lag-days` suppression

After NEEDED and MATCH flags are computed, a suppression pass runs:

> If the callsign was worked within N days → clear the flag list

This is **callsign-level**, not entity-level. The intent: if you just worked PY1ABC (Brazil) yesterday but the LoTW confirmation has not arrived yet, you do not want Brazil to keep firing NEEDED every 15 seconds. Lag suppression silences it for N days regardless of QSL status. The same suppression applies to MATCH: if you already worked a watchlist station recently, it will not keep alerting.

Default is `0` (worked today → suppress). Set to `-1` to disable entirely and always alert regardless of recency.

---

### At a glance — what respects `--qsl-only`?

| What it tracks | Level | Respects `--qsl-only`? |
|---|---|---|
| Callsign color (bold-green) | Per callsign | No — all QSOs |
| Grid color | Per grid | No — all QSOs |
| State color | Per state | No — all QSOs |
| Days column / lag suppression | Per callsign | No — all QSOs |
| ⭐ QRZ indicator | Per callsign | No — all QSOs |
| 🌎 LoTW indicator | Per callsign | No — all QSOs |
| ✅ entity emoji (NEEDED check) | Per entity/zone/country | **Yes** |
| ❓ entity emoji (worked but unconfirmed) | Per entity/zone/country | N/A — shows the gap between filtered and unfiltered |
| NEEDED alert itself | Per entity/zone/country | **Yes** |
| MATCH alert | Callsign/message/state pattern | No |

## Using multiple WSJT-X UDP integrations simultaneously

WSJT-X sends its UDP stream to a single configured destination. If you run jtwatch alongside GridTracker, WSJT-X Log, or any other tool that expects to receive those packets, only one of them can be the primary listener — the other tools need to receive a forwarded copy.

Either jtwatch or GridTracker can act as the proxy. The simplest approach depends on which tool you prefer to configure.

### Option A — jtwatch as proxy (recommended)

Point WSJT-X at jtwatch (port 2237, unchanged), then have jtwatch forward a copy to GridTracker on a free port of your choice. Port 2238 is a natural pick since it's adjacent and almost always free.

**Step 1** — In WSJT-X (**File → Settings → Reporting**), confirm the UDP server is set to `127.0.0.1:2237`. No change needed if jtwatch was already working.

**Step 2** — In GridTracker (**Settings → WSJT-X**), change the UDP port from `2237` to `2238` and the address to `127.0.0.1`.

**Step 3** — Start jtwatch with `--proxy 2238`:

```bash
jtwatch --adif ~/wsjtx_log.adi --proxy 2238
```

jtwatch will confirm on startup:

```
Listening on 0.0.0.0:2237  [CQ calls only]
[proxy] Forwarding all packets to: localhost:2238
```

Every packet WSJT-X sends — Decode, Heartbeat, Status, QSO Logged, etc. — is forwarded to GridTracker verbatim before jtwatch processes it. GridTracker sees the full unmodified protocol stream.

To forward to a third tool as well, just add another port:

```bash
jtwatch --adif ~/wsjtx_log.adi --proxy 2238,2239
```

### Option B — GridTracker as proxy

If you prefer to keep GridTracker on its default port, reverse the arrangement: WSJT-X sends to GridTracker, and GridTracker relays a copy to jtwatch.

**Step 1** — In GridTracker (**Settings → WSJT-X**), enable the UDP relay/forwarding option and set it to forward to `127.0.0.1:2238`.

**Step 2** — Start jtwatch on port 2238 instead of 2237:

```bash
jtwatch --adif ~/wsjtx_log.adi --port 2238
```

WSJT-X and GridTracker keep their existing configuration; jtwatch simply listens on the forwarded port.

### Choosing a free port

Port 2238 is used in the examples above but any unused port above 1024 works. To confirm a port is free before using it:

```bash
ss -uln | grep 2238      # Linux
netstat -anu | grep 2238 # macOS / older Linux
```

No output means the port is available.
