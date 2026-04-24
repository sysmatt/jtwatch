# jtwatch

A terminal monitor for WSJT-X CQ calls. Listens on the WSJT-X UDP broadcast port and prints every CQ decode to stdout, enriched with DXCC entity, CQ zone, ITU zone, continent, and grid square from the AD1C `cty.dat` prefix database.

With an ADIF logbook loaded, jtwatch flags contacts whose entity, CQ zone, or country you have not yet worked as **NEEDED** and rings the terminal bell. Optional regex watchlists let you flag specific callsigns or message patterns. External alerts can be sent via a custom script or [ntfy.sh](https://ntfy.sh) push notifications.

## Features

- Zero dependencies — Python standard library only
- Auto-downloads and caches `cty.dat` on first run
- ADIF log comparison with per-category worked/needed status
- Regex watchlists for callsigns and full decoded messages
- External alerting via script (`--alert-command`) or ntfy.sh push (`--alert-ntfy`)
- Deduplicates alerts within a run — no repeated notifications for the same event
- Interactive call prompt — answer yes and jtwatch sends a WSJT-X Reply (Type 4) UDP message to start TX automatically
- Optional ANSI color output (`--color`) for at-a-glance status scanning
- Fixed-width columnar output for easy terminal scanning
- Optional JSON-lines output for downstream processing
- Portable suffix handling (`W1ABC/P`, `HC2/DH1TW`, `VK9/W1ABC`, etc.)

## Requirements

- Python 3.10 or later
- WSJT-X configured to send UDP broadcasts (see below)

## Installation

```bash
# Clone or download
git clone https://github.com/youruser/jtwatch.git
cd jtwatch

# Make executable (Linux/macOS)
chmod +x jtwatch

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

Each line is printed in fixed-width columns:

```
HHMMSSZ  SNR +XXX dB  dt +XX.Xs  XXXXX Hz  [MODE]  message                   callsign     grid    | entity                              CQz  ITUz  cont  [status]
```

Column descriptions:

| Column | Example | Description |
|--------|---------|-------------|
| `HHMMSSZ` | `120145z` | UTC period start time from WSJT-X |
| `SNR` | `SNR  +12 dB` | Signal-to-noise ratio |
| `dt` | `dt   +0.2s` | Time offset from period start |
| `Hz` | ` 1234 Hz` | Audio frequency |
| `[MODE]` | `[FT8 ]` | Digital mode |
| `message` | `CQ DX W1ABC FN42` | Full decoded message (24 chars) |
| `callsign` | `W1ABC` | Extracted callsign (12 chars) |
| `grid` | `FN42` | Maidenhead grid square (6 chars) |
| `entity` | `United States` | DXCC entity from cty.dat |
| `CQz` | `CQ5` | CQ zone |
| `ITUz` | `ITU8` | ITU zone |
| `cont` | `NA` | Continent |
| `[status]` | `[worked dxcc cqz country]` | Worked/NEEDED/MATCH status |

Sample output:

```
120145z  SNR  +12 dB  dt   +0.2s   1234 Hz  [FT8 ]  CQ DX W1ABC FN42          W1ABC         FN42    | United States                   CQ5   ITU8   NA  [worked dxcc cqz country]
120200z  SNR   -5 dB  dt   -1.1s    234 Hz  [FT4 ]  CQ W2XYZ                  W2XYZ                 | Germany                         CQ14  ITU28  EU  *** NEEDED: NEW-DXCC(DL) ***
120215z  SNR  +30 dB  dt   +1.0s   2899 Hz  [FT8 ]  CQ POTA VK2ABC QF56       VK2ABC        QF56    | Australia                       CQ29  ITU59  OC  *** MATCH: CALL:VK2ABC ***
```

### Color output (`--color`)

Pass `--color` to enable ANSI color coding for at-a-glance scanning:

| Element | Color | Rationale |
|---------|-------|-----------|
| Time, SNR, dt, Hz, mode | dim | Reference data — de-emphasized |
| Callsign | bold | Primary identifier |
| Entity block | cyan | Enrichment data |
| `[worked ...]` | green | Already in log |
| `*** NEEDED: ... ***` | bold red | Unworked entity/zone/country |
| `*** MATCH: ... ***` | bold yellow | Watchlist hit |

```bash
jtwatch --adif ~/wsjtx_log.adi --color
jtwatch --adif ~/wsjtx_log.adi --color --alert-ntfy --ntfy-topic my-ham-alerts
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

Same file format, but patterns are tested against the **full decoded message string** (e.g. `CQ POTA W1ABC FN42`). Matches are flagged `*** MATCH: MSG:<pattern> ***` (note the `MSG:` prefix, vs `CALL:` for callsign patterns). Useful for catching activity modifiers:

```bash
jtwatch --match-message activations.txt
```

Example `activations.txt`:

```
# SOTA and POTA activations
\bSOTA\b
\bPOTA\b
\bIOTA\b
```

A POTA activation would appear as:

```
120215z  SNR  +30 dB  dt   +1.0s   2899 Hz  [FT8 ]  CQ POTA VK2ABC QF56       VK2ABC        QF56    | Australia                       CQ29  ITU59  OC  *** MATCH: MSG:\bPOTA\b ***
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

### `--alert-ntfy` + `--ntfy-topic TOPIC`

POSTs the alert message to `https://ntfy.sh/TOPIC`. Install the [ntfy app](https://ntfy.sh) on your phone and subscribe to the same topic to receive push notifications.

```bash
jtwatch --adif ~/wsjtx_log.adi --alert-ntfy --ntfy-topic my-ham-alerts
```

`--ntfy-topic` is required when `--alert-ntfy` is set; jtwatch will exit with an error if it is omitted.

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
jtwatch --adif ~/wsjtx_log.adi --call --alert-ntfy --ntfy-topic my-ham-alerts
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
| `--alert-command SCRIPT` | — | Script to call on NEEDED/MATCH events |
| `--alert-ntfy` | off | Send push alerts via ntfy.sh |
| `--ntfy-topic TOPIC` | — | ntfy.sh topic (required with `--alert-ntfy`) |
| `--call` | off | Prompt to call on NEEDED/MATCH events |
| `--call-timeout SECONDS` | `15` | Seconds before call prompt auto-answers no |
| `--color` | off | Colorize output with ANSI escape codes |
