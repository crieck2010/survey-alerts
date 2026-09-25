# survey-alerts

> **Part of [earthwatch-suite](https://github.com/crieck2010/earthwatch-suite)** —
> the remote-sensing project (satellite imagery, change detection, site
> monitoring). Terrestrial surveying lives in
> [survey-suite](https://github.com/crieck2010/survey-suite); the two stay
> compatible through the
> [cross-suite contracts](https://github.com/crieck2010/earthwatch-suite/blob/main/docs/CONTRACTS.md).

The **alert-report delivery engine** for earthwatch-suite: it consumes the
alert events that [survey-monitor](https://github.com/crieck2010/survey-monitor)
writes (`alerts.jsonl`, per the stable schema in
[survey-monitor's `docs/ALERT_SCHEMA.md`](https://github.com/crieck2010/survey-monitor/blob/main/docs/ALERT_SCHEMA.md))
and turns them into human-readable deliverables — one-page HTML site reports
(print-to-PDF ready), plain-text summaries, and multipart email digests.

This is the delivery layer of the per-site monitoring product: survey-monitor
*detects* change, survey-alerts is what the customer actually *sees*.

Engine-first, like its siblings: everything under `alerts/` except
`alerts/cli.py` is stdlib-only pure logic — no UI imports, no network except
SMTP delivery, no hard third-party dependencies.

## Features

- **Tolerant alert reader** — pins to the ALERT_SCHEMA contract: unknown
  extra fields are preserved, optional fields degrade gracefully, and one
  corrupt JSONL line never hides the rest of the log
- **Report model** — one in-memory `SiteReport` feeds every renderer, so the
  email body and the PDF always say the same thing; deterministic
  `report_id` (SHA-1 over alert ids) makes idempotent delivery checkable
- **HTML reports** — self-contained, print-friendly (`@page` rules): any
  browser's *Print → Save as PDF* produces the one-page site report with
  zero extra dependencies; optional native PDF via reportlab
  (`pip install "survey-alerts[pdf]"`)
- **Email** — multipart plain-text + HTML bodies composed from the same
  report model; SMTP config from flags or `SURVEY_ALERTS_SMTP_*` env vars
  (credentials never hardcoded, never logged)
- **Digesting** — per-site "what's new since the last digest" selection with
  a persisted state file: repeated cron runs never re-send the same alerts;
  severity filtering (`--min-severity warning` keeps warning + critical)
- **survey-monitor plug-ins** — `install_notifiers()` registers
  `"survey-alerts:email"` and `"survey-alerts:pdf"` with survey-monitor's
  `register_notifier` hook (lazy import, fails soft when survey-monitor is
  absent; the engine always works standalone on a bare `alerts.jsonl`)
- **Plain-English summaries** — every alert gets a one-liner like
  *"Vegetation vigor (NDVI) dropped 0.150 (−7.5σ) since 2026-09-01 — exceeds
  warning threshold (0.100)."*

## Installation

Requires Python 3.9+. The engine core is **stdlib-only** — no install-time
dependencies.

```bash
# From source
pip install .

# Native PDF rendering (optional; HTML print-to-PDF works without it)
pip install "survey-alerts[pdf]"
```

## Quickstart

```bash
# 1. Render a report from a survey-monitor alert log (sends nothing)
survey-alerts render --alerts /path/to/state/quarry-north/alerts.jsonl \
    --out report.html

# 2. Preview the email without sending it
survey-alerts send --alerts /path/to/state/quarry-north/alerts.jsonl \
    --to ops@example.com --from alerts@example.com --dry-run

# 3. Send it for real (SMTP from env vars, see below)
export SURVEY_ALERTS_SMTP_HOST=smtp.example.com
export SURVEY_ALERTS_SMTP_USER=alerts
export SURVEY_ALERTS_SMTP_PASSWORD='...'
export SURVEY_ALERTS_SMTP_FROM=alerts@example.com
survey-alerts send --alerts /path/to/state/quarry-north/alerts.jsonl \
    --to ops@example.com

# 4. Digest every site under a monitor store directory (cron-friendly)
survey-alerts digest --sites-dir /path/to/state --state digest-state.json \
    --out-dir alert-reports
# re-running digests nothing new — delivery is idempotent
```

Or drive it from Python:

```python
from alerts import read_alerts, dedupe, build_report
from alerts.html import render_html
from alerts.plugin import install_notifiers

records, skipped = read_alerts("state/quarry-north/alerts.jsonl",
                               min_severity="warning")
report = build_report(dedupe(records))
open("report.html", "w").write(render_html(report))

# plug delivery into survey-monitor's notifier registry (lazy, fails soft)
install_notifiers()  # registers "survey-alerts:email" + "survey-alerts:pdf"
```

## CLI reference

```
survey-alerts render --alerts PATH [--format html|text|pdf] [--out FILE]
                    [--min-severity info|warning|critical] [--since ISO]
    Build a site report. Never sends anything — the safe preview path.

survey-alerts send --alerts PATH --to ADDR,... [--from ADDR]
                  [--smtp-host H] [--smtp-port N] [--smtp-user U]
                  [--smtp-password P] [--no-tls] [--dry-run]
                  [--min-severity ...] [--since ...]
    Render and email the report. --dry-run composes without sending.

survey-alerts digest --sites-dir DIR --state FILE
                    [--min-severity warning] [--since ...]
                    [--out-dir DIR] [--email-to ADDR,...] [--dry-run]
    Walk <DIR>/<site_id>/alerts.jsonl (the survey-monitor store layout),
    select only undelivered alerts, and deliver per-site reports.
    State persists in FILE; re-runs are idempotent.

survey-alerts notifiers
    Register this module's notifiers with survey-monitor (needs it
    installed) and print the result.
```

## Report anatomy

One report per site per digest window:

1. **Header** — site name/id, generation timestamp, deterministic report id
2. **Headline** — worst severity at a glance
   (*"3 alerts — CRITICAL change detected at Quarry north pit."*)
3. **Severity badges** — counts of critical / warning / info
4. **What changed** — one plain-English line per alert
5. **Alert detail table** — severity, index, metric, value, Δ, z-score,
   which threshold fired, pass date
6. **Footer** — provenance (`alerts.jsonl` → survey-alerts) and severity
   semantics

See [`docs/REPORT.md`](docs/REPORT.md) for the full field reference.

## SMTP configuration

Flags win; `SURVEY_ALERTS_SMTP_*` environment variables fill the rest:

| Variable | Meaning | Default |
|---|---|---|
| `SURVEY_ALERTS_SMTP_HOST` | SMTP server | (required) |
| `SURVEY_ALERTS_SMTP_PORT` | port | 587 |
| `SURVEY_ALERTS_SMTP_USER` | login | (none) |
| `SURVEY_ALERTS_SMTP_PASSWORD` | password | (none) |
| `SURVEY_ALERTS_SMTP_FROM` | From: address | (required) |
| `SURVEY_ALERTS_SMTP_TLS` | set `0` to disable STARTTLS | enabled |
| `SURVEY_ALERTS_TO` | default recipients (comma-separated) | (none) |
| `SURVEY_ALERTS_OUT_DIR` | report dir for the pdf notifier | `alert-reports` |

Credentials are passed straight to the server and never logged —
`SmtpConfig.redacted()` is what appears in logs.

## The maths

How an alert becomes a sentence — and how digests stay honest:

- **Δ (delta)** = this pass's metric − baseline mean. The raw magnitude of
  change, in index units (e.g. NDVI points).
- **z-score** = Δ / baseline std. How *unusual* the change is against the
  site's own history: |z| > 3 is rare under normal variation.
- **Threshold kinds**: `absolute_delta` fires on raw magnitude ("NDVI fell
  more than 0.10"); `z_score` fires on unusualness ("a 5σ move, whatever its
  size"). survey-monitor evaluates both; this engine reports whichever fired.
- **Severity ladder**: `info` means *approaching* a warning level (≥ 50% of
  the warning threshold) — watch only, never paged; `warning` breached a
  warning level and survived persistence gating; `critical` breached a
  critical level — "needs eyes today".
- **Digesting** selects by `alert_id` (deterministic in survey-monitor:
  `site_id:pass_id:index:severity:seq`), so "new since last digest" is exact
  even if a log is re-read or partially replayed. Same input → same
  `report_id` → a downstream system can deduplicate safely.

## Suite interop

- Reads survey-monitor's `alerts.jsonl` directly — no import of
  survey-monitor needed, so this engine works even where the monitor isn't
  installed.
- `install_notifiers()` plugs `"survey-alerts:email"` / `"survey-alerts:pdf"`
  into survey-monitor's `register_notifier` registry — the exact interface
  survey-monitor's docs anticipated, no changes needed on either side.
- Future engines (vegetation, flood, burn, …) that emit the same alert-event
  schema get delivery for free.
- See [`docs/NOTIFIERS.md`](docs/NOTIFIERS.md) for writing your own
  notifier plug-in.

## Limitations (honest)

- **No real-time push** — this is a batch delivery engine driven by
  `render`/`send`/`digest` runs (cron), not a streaming service. Alerts are
  only as fresh as the last monitor run.
- **PDF fidelity depends on the renderer path** — the default HTML
  print-to-PDF path inherits the browser's print engine; the reportlab path
  is deterministic in *content* but PDF metadata timestamps make
  byte-identical output across runs unguaranteed.
- **Email deliverability is yours** — SPF/DKIM, bounce handling, and list
  management belong to your mail provider; this engine composes and hands
  off to SMTP.
- **No thumbnails yet** — before/after index-raster thumbnails are a
  planned enhancement; the report currently carries tables and prose.
- **Alert quality is upstream** — garbage thresholds in the site config
  produce faithfully-delivered garbage alerts. Tune in survey-monitor.

## Versioning

Semantic versioning. `0.1.0` pins to survey-monitor's alert-event schema v1
(fields added-only); the reader tolerates future added fields by design.

## License

MIT — see [LICENSE](LICENSE).
