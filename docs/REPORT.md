# Report anatomy

A `SiteReport` is the single in-memory model every renderer consumes.
Fields:

| Field | Meaning |
|---|---|
| `site_id` / `site_name` | from the first (worst) event; multi-site inputs are grouped per site by the CLI |
| `generated_at` | UTC ISO-8601 timestamp of report construction |
| `events` | alert records, sorted worst-severity-first, then timestamp, then `alert_id` |
| `by_severity` | counts, e.g. `{"info": 1, "warning": 3, "critical": 1}` |
| `by_index` | counts per spectral index, e.g. `{"ndvi": 2, "ndwi": 3}` |
| `worst` | highest severity present (`info`/`warning`/`critical`) or `None` when empty |
| `report_id` | first 12 hex chars of SHA-1 over the sorted alert ids — deterministic; identical input always yields the identical id |
| `headline` | one-line worst-severity summary |
| `summary_lines` | one plain-English sentence per event, same order as `events` |

## Renderers

| Renderer | Module | Output |
|---|---|---|
| `render_html(report)` | `alerts.html` | self-contained HTML, print-friendly `@page` CSS |
| `render_text(report)` | `alerts.html` | plain text: header, headline, severity counts, summary lines, aligned detail table |
| `render_pdf(report, path)` | `alerts.pdf` | one-page PDF via reportlab (optional dep); raises `AlertsError` with install hint when reportlab is absent |
| `compose_email(report, to, from_addr)` | `alerts.emailer` | multipart `EmailMessage`: plain-text part + HTML alternative; subject carries `[SEVERITY]` and the alert count |

## Plain-English summary grammar

Each summary line follows one pattern:

```
<Index label> <dropped|rose|changed> <|Δ|> [(<z>σ)] [since <pass date>]
[— exceeds <severity> threshold (<threshold>)].
```

Examples:

- `Vegetation vigor (NDVI) dropped 0.150 (−7.5σ) since 2026-09-01 — exceeds warning threshold (0.100).`
- `Surface water (NDWI) rose 0.250 (+8.3σ) since 2026-09-06 — exceeds warning threshold (0.150).`
- `Vegetation vigor (NDVI) changed (mean).` (when upstream fields are absent — the reader degrades gracefully rather than crashing)

Index labels live in `alerts.model.INDEX_LABELS`; unknown indices fall back
to their upper-cased code.

## Determinism

`render_html` and `render_text` are byte-deterministic for identical input:
same records in → same bytes out (the test suite asserts this). `report_id`
lets downstream systems deduplicate deliveries without parsing content.
