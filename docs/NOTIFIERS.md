# Notifier plug-in guide

survey-monitor *detects* change; delivery is a plug-in. After a run fires
alerts, survey-monitor calls every notifier named in the site config through
its `register_notifier` registry. A notifier is any callable:

```python
def my_notifier(events) -> None: ...
```

`events` is the list of alert events fired by that run — survey-monitor
`AlertEvent` objects when called in-process.

## The two built-in notifiers

`alerts.plugin.install_notifiers()` registers both (survey-monitor is
imported lazily; without it the install reports `{"installed": False}` and
nothing raises):

| Name | Behaviour |
|---|---|
| `survey-alerts:email` | builds the report model and sends it via SMTP. Config from `SURVEY_ALERTS_SMTP_*` env vars (or pass `config=` / `to=`). Raises a clear error — never silently drops — when SMTP is unconfigured. |
| `survey-alerts:pdf` | renders the report to `SURVEY_ALERTS_OUT_DIR` (default `alert-reports/`), as `.pdf` when reportlab is installed, otherwise `.html`. Returns the file path. |

Wire them in a site config (survey-monitor side):

```yaml
notifiers: ["survey-alerts:email", "survey-alerts:pdf"]
```

and register once in the process that runs the monitor:

```python
from alerts.plugin import install_notifiers
install_notifiers()  # or install_notifiers(to=["ops@example.com"])
```

One failing notifier never blocks the others — survey-monitor's `notify_all`
isolates delivery errors per notifier.

## Writing your own notifier

Normalize first, then build on the report model:

```python
from alerts.plugin import _normalize  # or alerts.reader.AlertRecord.from_dict
from alerts.model import build_report
from alerts.html import render_html

def sms_digest(events) -> None:
    records = _normalize(events)          # AlertEvent | dict -> AlertRecord
    report = build_report(records)
    send_sms(f"[{report.worst}] {report.headline}")  # your transport here

from monitor import register_notifier
register_notifier("my-sms", sms_digest)
```

Notes:

- `_normalize` accepts survey-monitor `AlertEvent` objects (via `to_dict()`)
  and plain dicts — tests and the CLI pass dicts, so your notifier is
  testable without survey-monitor installed.
- `build_report` is deterministic: same events → same `report_id`, so your
  transport can deduplicate.
- Keep notifiers synchronous and fast; survey-monitor calls them inline
  after a run. Queue (Celery, cron-spooled outbox) outside if delivery is
  slow.
- Never log credentials: follow `SmtpConfig.redacted()` — log the route,
  not the secret.
