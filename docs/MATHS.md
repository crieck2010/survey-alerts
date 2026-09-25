# The maths: from alert event to digest

This note explains the quantities survey-alerts works with. The detection
maths lives in survey-monitor; this engine is about *faithful, honest
delivery* of what the monitor computed.

## Δ (delta) — how much changed

```
Δ = value − baseline_mean
```

`value` is this pass's watched metric (e.g. mean NDVI inside the site
footprint); `baseline_mean` is the reference the monitor judged it against
(rolling trailing-window mean, or a fixed reference pass). Δ is in index
units — an NDVI Δ of −0.15 is a large vegetation loss; an NDWI Δ of +0.25 is
a lot of new surface water.

## z-score — how unusual it is

```
z = Δ / baseline_std
```

The site's own history sets the scale. |z| > 3 is rare under stable
conditions; the fixture's −12.5σ NDVI drop is an unambiguous event, not
noise. z-scores let one threshold policy work across sites with very
different natural variability.

## Threshold kinds

survey-monitor evaluates two rule families; the event records which fired:

- `absolute_delta` — fires on raw magnitude. "Tell me when NDVI moves more
  than 0.10, however usual that is here."
- `z_score` — fires on unusualness. "Tell me about 5σ moves, however small
  in absolute terms."

The report prints whichever kind (and level) fired, so the reader always
knows *why* an alert exists.

## Severity ladder

| Severity | Meaning | Delivery default |
|---|---|---|
| `info` | Δ reached ≥ 50% of the warning threshold — approaching, not breaching | excluded from digests by default (`--min-severity warning`) |
| `warning` | breached a warning level and survived persistence gating | digested |
| `critical` | breached a critical level | digested, `[CRITICAL]` subject |

`info` is a watch band for drift, not a paging event — the digest default
reflects that.

## Digest selection — "what's new?"

A digest selects events by `alert_id`, which survey-monitor makes
deterministic (`site_id:pass_id:index:severity:seq`). The state file records
delivered ids per site, so selection is exact set-difference:

```
new = {events at/above min_severity, newer than --since}
      − {alert_ids already delivered for this site}
```

Properties:

- **Idempotent** — re-running on unchanged input selects nothing; repeated
  cron runs never re-send.
- **Replay-safe** — if a log is re-read or an event re-appends, its id is
  already in the delivered set.
- **Order-stable** — per site, worst-severity-first, then timestamp; sites
  in first-seen order. Same input → same report → same `report_id`
  (SHA-1 over sorted alert ids).

Quiet hours and cooldowns are scheduling concerns: run `digest` on the
schedule you want (cron), and survey-monitor's persistence gating upstream
already suppresses single-pass flicker before an event ever fires.
