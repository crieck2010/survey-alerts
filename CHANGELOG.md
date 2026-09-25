# Changelog

All notable changes to survey-alerts. Semantic versioning.

## [0.1.0] — 2026-09-25

Initial release. The delivery layer of the earthwatch-suite monitoring
product: turns survey-monitor `alerts.jsonl` into human-readable reports.

- Tolerant `alerts.jsonl` reader pinned to survey-monitor's ALERT_SCHEMA
  contract (unknown fields preserved, corrupt lines skipped, standalone —
  no survey-monitor import needed)
- `SiteReport` model: deterministic, one model feeding every renderer,
  SHA-1 `report_id` for idempotent delivery
- Renderers: self-contained print-friendly HTML (browser print-to-PDF is
  the default PDF path), plain text, optional native PDF via reportlab
  (`pip install "survey-alerts[pdf]"`)
- Email: multipart plain+HTML composition, stdlib SMTP delivery, config
  from flags or `SURVEY_ALERTS_SMTP_*` env vars; credentials never logged
- Digest engine: per-site "new since last digest" selection with persisted
  state; severity filtering; idempotent re-runs
- `install_notifiers()`: registers `"survey-alerts:email"` and
  `"survey-alerts:pdf"` with survey-monitor's `register_notifier` hook
  (lazy import, fails soft without survey-monitor)
- CLI: `render`, `send` (with `--dry-run`), `digest`, `notifiers`
- 48 pytest tests, no network; docs: README, `docs/REPORT.md`,
  `docs/NOTIFIERS.md`, `docs/MATHS.md`
