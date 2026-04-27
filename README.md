# jtwatch

A terminal based (CLI) monitor for WSJT-X CQ calls. Listens on the WSJT-X UDP broadcast port and prints every CQ decode to stdout, enriched with DXCC entity, CQ zone, ITU zone, continent, and grid square from the AD1C `cty.dat` prefix database.

With an ADIF logbook loaded, jtwatch flags contacts whose entity, CQ zone, or country you have not yet worked as **NEEDED** and rings the terminal bell. 

When invoked with the --call option, jtwatch will prompt the pilot for confirmation then tell WSJT-X to begin responding to the CQ (Just like if you double clicked the call in the GUI) -- NOTE, This is NOT an automation tool, you MUST initiate the reply call.  

Optional regex watchlists let you flag specific callsigns or message patterns. External alerts can be sent via a custom script or [ntfy.sh](https://ntfy.sh) push notifications.

### Note: Tested with WSJT-X 3.1.0 Improved

## Features

- Zero external dependencies — Python standard library only (`sqlite3` is bundled with Python)
- Auto-downloads and caches `cty.dat` on first run
- ADIF log comparison with per-category worked/needed status
- Regex watchlists for callsigns and full decoded messages
- Built-in `--pota`, `--sota`, `--iota` flags to match activations without a watchlist file
- **WAS state matching** — `--match-state` resolves callsigns to their FCC-licensed state via [hamdat](https://github.com/sysmatt/hamdat) and flags matching CQs; adds a `ST` column to every output line
- External alerting via script (`--alert-command`) or ntfy.sh push (`--alert-ntfy TOPIC`)
- Deduplicates alerts within a run — no repeated notifications for the same event
- ADIF log auto-reloads when the file changes — no restart needed after logging a QSO
- Interactive call prompt — answer yes and jtwatch sends a WSJT-X Reply (Type 4) UDP message to start TX automatically
- Optional ANSI color output (`--color`) for at-a-glance status scanning
- Fixed-width columnar output for easy terminal scanning
- Optional JSON-lines output for downstream processing
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

# Show all decodes, not just CQ, with local wall-clock time
jtwatch --raw --show-time

# Different port
jtwatch --port 2237
```

## Output Format

The column header is printed on startup and repeats every 15 lines so it stays visible as output scrolls:

```
HHMMSSZ     dB    dt(s)      Hz  mode  message                   callsign      grid    | entity                              CQz   ITUz   ct
----------------------------------------------------------------------------------------------------------------------------------------
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
| `ST` | `NJ` | FCC-licensed state — only shown with `--match-state` |
| `entity` | `United States` | DXCC entity from cty.dat |
| `CQz` | `CQ5` | CQ zone |
| `ITUz` | `ITU8` | ITU zone |
| `ct` | `NA` | Continent (2-char code) |
| `[status]` | `[worked dxcc cqz country]` | Worked/NEEDED/MATCH status |

Sample output:

```
HHMMSSZ     dB    dt(s)    Hz  mode  message                   callsign      grid    | entity                              CQz   ITUz   ct
----------------------------------------------------------------------------------------------------------------------------------------
120145z   +12    +0.2s   1234  FT8   CQ DX W1ABC FN42          W1ABC         FN42    | United States                   CQ5   ITU8   NA  [worked dxcc cqz country]
120200z    -5    -1.1s    234  FT4   CQ W2XYZ                  W2XYZ                 | Germany                         CQ14  ITU28  EU  *** NEEDED: NEW-DXCC(DL) ***
120215z   +30    +1.0s   2899  FT8   CQ POTA VK2ABC QF56       VK2ABC        QF56    | Australia                       CQ29  ITU59  OC  *** MATCH: CALL:VK2ABC ***
```

### Color output (`--color`)

Pass `--color` to enable ANSI color coding for at-a-glance scanning:

| Element | Color | Rationale |
|---------|-------|-----------|
| Time, dB, dt, Hz, mode | dim | Reference data — de-emphasized |
| Callsign | bold | Primary identifier |
| Entity block | cyan | Enrichment data |
| `[worked ...]` | green | Already in log |
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

jtwatch builds three worked sets from the log:

- **DXCC entity** — primary DXCC prefix (e.g. `W`, `DL`, `VK`)
- **CQ zone** — zone number (1–40)
- **Country** — country name string

Any CQ from a callsign whose entity, zone, or country is **not** in the log is flagged with `*** NEEDED: ... ***` and the terminal bell rings (`\a`).

Already-worked contacts show `[worked dxcc]`, `[worked dxcc cqz country]`, etc. — listing only the categories that are both enabled and matched.

The ADIF file is checked for changes on every received packet. If WSJT-X logs a new QSO while jtwatch is running, the log reloads automatically — no restart required.

### Disabling individual NEEDED checks

```bash
# Only alert on new DXCC entities, ignore zone and country
jtwatch --adif ~/wsjtx_log.adi --no-need-cqz --no-need-country

# Only alert on new CQ zones
jtwatch --adif ~/wsjtx_log.adi --no-need-entity --no-need-country
```

## Pattern Matching

Watchlists let you flag specific callsigns or message patterns regardless of your log.

### `--match-calls FILE [FILE ...]`

Each file contains one Python regular expression per line. Patterns are tested against the **resolved callsign**. Matching CQs are flagged `*** MATCH: CALL:<pattern> ***` and ring the bell.

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

Same file format, but patterns are tested against the **full decoded message string** (e.g. `CQ POTA W1ABC FN42`). Matches are flagged `*** MATCH: MSG:<pattern> ***` (note the `MSG:` prefix, vs `CALL:` for callsign patterns). Useful for catching activity modifiers or any other text in the decoded message.

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

## External Alerts

When a NEEDED or MATCH event fires, jtwatch can notify you via a script or a push notification. Both methods deduplicate: the same callsign with the same flags will only trigger one alert per run.

The alert message is a compact one-liner:

```
W1ABC | United States CQ5 | NEEDED: NEW-DXCC(W), NEW-CQZ(5)
VK2ABC | Australia CQ29 | MATCH: CALL:VK2ABC
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

`--call` turns jtwatch into a basic call/no-call decision tool. On each new NEEDED or MATCH event, after printing the decode line, jtwatch writes a prompt to stderr and waits for your answer:

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

Each callsign is prompted at most once per run. If the same station calls CQ repeatedly, you won't be asked again. The prompt is suppressed entirely when stdin is not a terminal (piped or scripted runs).

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

## Options Reference

| Option | Default | Description |
|--------|---------|-------------|
| `--host HOST` | `0.0.0.0` | UDP bind address |
| `--port PORT` | `2237` | UDP port (match WSJT-X Reporting settings) |
| `--show-time` | off | Prepend local wall-clock `HH:MM:SS` to each line |
| `--raw` | off | Print all decoded messages, not just CQ |
| `--save FILE` | — | Append CQ records as JSON lines to FILE |
| `--cty PATH` | `~/.jtwatch_cty.dat` | Path to cty.dat prefix database |
| `--adif FILE` | — | ADIF logbook for NEEDED detection |
| `--no-need-entity` | — | Disable DXCC entity NEEDED check |
| `--no-need-cqz` | — | Disable CQ zone NEEDED check |
| `--no-need-country` | — | Disable country name NEEDED check |
| `--match-calls FILE [...]` | — | Callsign regex watchlist file(s) |
| `--match-message FILE [...]` | — | Full-message regex watchlist file(s) |
| `--pota` | off | Flag CQ POTA calls as MATCH (no file required) |
| `--sota` | off | Flag CQ SOTA calls as MATCH (no file required) |
| `--iota` | off | Flag CQ IOTA calls as MATCH (no file required) |
| `--match-state STATES` | — | Comma-separated state abbreviations or file; flags matching FCC-licensed states as MATCH and adds a `ST` column |
| `--hamdat DB` | `~/.hamdat/hamdat.db` | hamdat SQLite database path (used with `--match-state`) |
| `--alert-command SCRIPT` | — | Script to call on NEEDED/MATCH events |
| `--alert-ntfy TOPIC` | — | POST push alerts to ntfy.sh topic TOPIC |
| `--call` | off | Prompt to call on NEEDED/MATCH events |
| `--call-timeout SECONDS` | `15` | Seconds before call prompt auto-answers no |
| `--color` | off | Colorize output with ANSI escape codes |
