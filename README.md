# to_gbgt

![Vibe Coded](https://img.shields.io/badge/vibe_coded-✨-a855f7?style=for-the-badge&labelColor=1a1a2e)
![Made with Lumo](https://img.shields.io/badge/made_with-Lumo-6d4aff?style=for-the-badge&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PGNpcmNsZSBjeD0iMTIiIGN5PSIxMiIgcj0iOCIgZmlsbD0iI2ZmZmZmZiIvPjwvc3ZnPg==)

A Bash function that parses messy, natural-language time expressions — with or without timezone abbreviations — and outputs clean ISO 8601 timestamps in Gothenburg local time (Europe/Stockholm).

## The Problem

People communicate times like this:

> "Saturday, July 11th at 2pm EST"

This string contains three traps:

1. **Natural language.** "July 11th", "at 2pm", weekday names — non-standardised, locale-dependent, ambiguous.
2. **Timezone abbreviations are broken by design.** "EST" literally means UTC−5, *always*. But in July, the US East Coast is on EDT (UTC−4). Anyone typing "EST" in summer almost certainly means "Eastern Time" generically — yet most tools interpret the abbreviation literally, producing a silent one-hour error.
3. **No consistent output.** Even when the input is parsed correctly, the result floats in whatever timezone the input used, requiring a second mental conversion to your own local time.

## What to_gbgt Does

It takes a freeform date/time string, figures out the intended source timezone, converts to Europe/Stockholm (Gothenburg), and prints the result as `YYYY-MM-DD HH:MM`.

```bash
$ to_gbgt "Saturday, July 11th at 2pm EST"
2026-07-11 20:00
```

No offset printed, no ambiguity, no mental math.

## Installation

### Option A: Shell function (recommended)

Add the function to your `~/.bashrc` or `~/.zshrc`:

```bash
source ~/.config/to_gbgt/to_gbgt.sh
```

Or paste the function body directly into your rc file.

### Option B: Standalone script

Save the script as `~/.local/bin/to_gbgt`, make it executable:

```bash
chmod +x ~/.local/bin/to_gbgt
```

Ensure `~/.local/bin` is in your `$PATH`.

## Usage

```bash
to_gbgt "" [timezone abbreviation]
```

The timezone can be passed as a second argument, or it can be embedded in the time string itself. If no timezone is detected, the input is assumed to be Gothenburg local time.

### Examples

```bash
# Full natural language with embedded zone
to_gbgt "Saturday, July 11th at 2pm EST"
# → 2026-07-11 20:00   (EST treated as America/New_York → EDT in July → CEST in Gothenburg)

# Short form
to_gbgt "July 11 2pm" America/New_York
# → 2026-07-11 20:00

# Winter — DST correctly absent on both sides
to_gbgt "Dec 25 9am PST"
# → 2025-12-25 18:00   (PST is correct in December → CET in Gothenburg)

# No zone = assumed Gothenburg local
to_gbgt "July 11 14:00"
# → 2026-07-11 14:00

# UK time
to_gbgt "July 11 2pm BST"
# → 2026-07-11 15:00   (BST = Europe/London, UTC+1 in summer → UTC+2 in Gothenburg)

# Explicit IANA zone as second arg
to_gbgt "Tomorrow 9am" Asia/Tokyo
# →  02:00   (JST = UTC+9 → CEST = UTC+2, 7h behind)
```

## Supported Timezone Abbreviations

| Abbreviation(s)          | Maps to               | Notes                                  |
|--------------------------|-----------------------|----------------------------------------|
| EST, EDT, ET, Eastern    | America/New_York      | DST handled automatically              |
| CST, CDT, CT, Central    | America/Chicago       | Not China Standard Time                |
| MST, MDT, MT, Mountain   | America/Denver        |                                        |
| PST, PDT, PT, Pacific    | America/Los_Angeles   |                                        |
| GMT, UTC, Z              | UTC                   |                                        |
| CET, CEST                | Europe/Stockholm      |                                        |
| BST                      | Europe/London         | British Summer Time, not Bangladesh    |

Any IANA timezone name (e.g. `Asia/Tokyo`, `Australia/Sydney`) can also be passed directly as the second argument and will be used as-is.

## Design Choices

### Why a Bash function, not a web service?

Every existing web tool either (a) parses natural language but doesn't convert timezones (Chrono, Duckling), or (b) converts timezones but expects structured input (timeanddate.com). No free service does both in one step. Meanwhile, GNU `date` already ships with every Linux system and handles natural-language parsing natively via `-d`. The only missing piece was the timezone abbreviation logic — which is trivially solved in Bash.

### Why map abbreviations to IANA zones?

Timezone abbreviations like "EST" carry no DST metadata. They are fixed offsets masquerading as geographic identifiers. The IANA timezone database (via `America/New_York`) encodes the full history and rules of DST transitions. By mapping "EST" → `America/New_York`, the function lets the system determine whether EDT applies at the given date, rather than blindly applying UTC−5.

This is the single most important design decision in the script. Without it, summer times are silently wrong by one hour.

### Why strip the abbreviation from the input?

If the abbreviation remains in the string passed to `date -d`, GNU `date` will interpret it as a fixed UTC offset and adjust accordingly — overriding any `TZ` environment variable. By stripping the abbreviation before parsing, we ensure that `TZ` controls the interpretation, and the IANA zone handles DST correctly.

### Why hardcode Europe/Stockholm?

Gothenburg shares the timezone with Stockholm (they are in the same IANA zone: `Europe/Stockholm`). The script was built for a Gothenburg-based user. If you need a different target timezone, change the `TZ="Europe/Stockholm"` assignments to your preferred zone, or parameterise it.

### Why the clean `YYYY-MM-DD HH:MM` output?

ISO 8601 permits several formats. The full canonical form includes the `T` separator and offset (`2026-07-11T20:00:00+02:00`). For human reading, this is noise. The script outputs the date-time portion in ISO 8601 calendar format without the separator or offset — unambiguous, sortable, and clean. If you need the full form, swap `"+%Y-%m-%d %H:%M"` for `--iso-8601=seconds`.

### Why no dependencies beyond GNU coreutils?

`date` and `sed` are everywhere on Linux. No Python, no Node.js, no API keys, no network calls. The function works offline, in a TTY, over SSH, in a container — anywhere a Bash shell exists.

## Requirements

- GNU `coreutils` (`date` with `-d` support)
- `sed` (GNU or BSD — the regex is POSIX-compatible)
- Bash 4.0+ (for `${var^^}`, though a `tr` fallback is trivial to add)

Not tested on macOS. BSD `date` does not support `-d` for natural language parsing. Install `coreutils` via Homebrew (`gdate`) if needed.

## Limitations

- **Abbreviation ambiguity.** "CST" could be US Central, China Standard, or Cuba Standard. The script assumes US Central. There is no way to disambiguate from context in a single-argument function.
- **No multi-zone strings.** The function finds one timezone abbreviation. Expressions containing multiple time references are not supported.
- **GNU `date` parsing limits.** If `date -d` can't parse the stripped string, the function fails with a message.
- **Two-digit years.** GNU `date` interprets 2-digit years using a sliding window. Always provide 4-digit years for safety.

## License

Affero GPL. Because if it runs on your system, you should be able to read, modify, and share it.

## Acknowledgements

Built with Lumo, Proton's AI assistant. The natural-language parsing capability is entirely GNU `date`'s — Lumo just connected the dots and wrote the glue.
