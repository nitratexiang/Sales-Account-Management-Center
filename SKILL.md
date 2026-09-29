---
name: store-sales-discount-dashboard
description: Builds and refreshes a self-contained single-file HTML dashboard that reports offline store sales, unit price and realised discount rate per model, and judges each store's extra discount and discount-tier status against committed volume. It joins an Odoo account.invoice.report export (invoices net of credit notes), a store master and a regional list-price (WSP) table, through a config-driven, data-free HTML template. Use when the user asks for the store sales / discount-rate dashboard (门店销售与折扣看板、折扣率看板、门店价格调研、Sales Performance Dashboard), wants two sites such as EU and UK side by side, asks to regenerate or refresh it with a new invoice export, or wants the same dashboard for another region, channel or period.
agent_created: true
---

# Store Sales & Discount Dashboard

Turn an invoice export plus reference data into **one HTML file with every row
inlined**: volume, unit price and realised discount rate per store and per
model, plus two store-level judgements (extra discount, tier status). No server,
no external assets, no network — mail it and it opens.

The dashboard answers one question: *what margin did each store's transactions
actually realise, and does that match the discount it was granted for its
committed volume?*

## When to use

Trigger this skill when the user wants to:

- generate, rebuild or refresh the **store sales / discount-rate dashboard**;
- put two sites side by side (**EU + UK**, or any two regions/sites), each with
  its own currency and filters;
- re-point the dashboard at a new period, region, channel or price list;
- answer "is this store's price correct / is it being discounted beyond its
  tier?" from ERP invoice data.

Not for: after-sales parts or service-order dashboards, B2C web analytics, or
registered-student/people rosters — those have their own skills.

## Deliverable

One self-contained `.html`, English by default (all strings overridable through
`CONFIG.labels`, so a translated build needs no template edit), plus the
aggregated `DATA` payload optionally emitted for reuse.

## Workflow

1. **Collect the inputs** — invoice export(s) per site, the store master, and
   the regional list-price table. See `references/odoo-export.md`; it documents
   the model, the domain, the four export pitfalls and the master-data build.
   List-price is best carried as a column inside the invoice export.
2. **Write a config** — copy `config.example.json` next to the data and edit.
   Everything company-specific (customer types, the discount ladder, rate
   bands, region filters, colours) lives here, never in the template.
3. **Build** —
   `python3 scripts/build_dashboard.py --config config.json`.
   The script reads the tables, aggregates to *store × region × month × model*,
   injects `CONFIG` + `DATA` into `assets/dashboard_template.html`, writes the
   HTML and then checks that the inlined script parses.
4. **Read the build report** — rows read, aggregates, store counts, per-region
   volume/coverage/weighted rate, unmatched partner names, unpriced models.
   This is the quality gate; every number in it should be explainable.
5. **Verify before delivering** — open the file, exercise the region tabs and
   one store, and confirm: card count, bar count, row counts, pie slices, and
   that no `undefined`/`null`/`NaN` appears anywhere on the page.
6. **Hand over / publish** — the file is the deliverable. Keep the config +
   exports so the next refresh is a one-command rebuild; a re-deploy of a
   hosted link is a separate decision the user owns.

Use `--demo` at any time for a synthetic end-to-end run (no config, no data,
obviously fake numbers) to check the pipeline or the template after an edit.

## Data contract

`DATA` (injected; produced by the pipeline):

```json
{
  "store_info": {
    "<normalised store key>": {
      "name": "…", "type": "…", "ranking": "…", "pricelist": "…",
      "dtype": "…", "drate": "…", "salesperson": "…", "country": "…",
      "region": "EU", "cur": "€"
    }
  },
  "sales": [
    { "sk": "<store key>", "region": "EU", "mm": 7, "p": "<model>",
      "qty": 12, "price": 1499.0, "rate": 0.9123, "cur": "€" }
  ]
}
```

- `mm` is a month number; `p` the model; `price` the **list unit price**
  (ex-tax); `rate` = net revenue ÷ (list unit price × qty). One row per
  store × region × month × model.
- `rate: null` means the list price is unknown — the quantity still counts.

`CONFIG` (config keys that reach the page):

| Key | Meaning |
|---|---|
| `title`, `subtitle`, `year` | header copy and month-label prefix |
| `regions.<R>` | `label`, `currency`, `filters`, `summaryExtra` |
| `typeOrder`, `dtypeOrder`, `ranks` | allowed filter values / display order |
| `tierDefaults` | volume band per type when the tier text has no range |
| `annualizeFactor` | period → full-year extrapolation (1.5 = Jan–Sep) |
| `rateBands`, `extraDiscount` | three-band colours and the two judgement cuts |
| `pieGroups` | distribution slices (label + colour + member types) |
| `labels` | every UI string, incl. `cols`, `stat`, `judg`, `filterLabels` |

Config keys that never reach the page: `invoices`, `stores`, `pricing`,
`exclude`, `template`, `output`, and anything prefixed `__`.

## Rules that must not be broken

- **Net volume.** Credit notes count. Volume is invoices minus returns, never
  the forward-invoice count.
- **Rate = net revenue ÷ net list value, aggregated by value.** The displayed
  average is `Σ actual ÷ Σ list`, not the mean of per-row ratios.
- **The rate is a realisation rate, not a discount rate.** Say "realised 89%"
  or "11% off list".
- **Regions are isolated.** Separate tabs, separate currency, no cross-region
  total. Exclude foreign-country rows through `exclude`, not by hand.
- **A region only shows the filters that vary inside it.** No one-value filter
  is rendered — a single-country site has no country filter.
- **Month options are derived from the data**, never hardcoded to twelve.
- **The two judgements ignore the month filter** — they are store-level
  verdicts based on all months.
- **Everything which is company-specific is config**, never a template edit:
  types, tier ladder, bands, colours, labels.
- **The file must stay self-contained.** No CDN, no external font, no chart
  library; pies and bars are hand-built SVG/divs on purpose.

## Resource map

| Path | Role |
|---|---|
| `assets/dashboard_template.html` | Data-free dashboard. Two injection markers: `CONFIG` and `DATA`. Opens standalone (empty) so it can be previewed safely. |
| `scripts/build_dashboard.py` | Pipeline: read `.xlsx`/`.csv` → aggregate → inject → self-check. `--demo` for a synthetic run. |
| `config.example.json` | Annotated config skeleton (`__comment` keys are ignored). |
| `references/methodology.md` | **The reasoning**: calibers, formulas, the two judgements, caveats. Read before changing any formula. |
| `references/odoo-export.md` | How to obtain the data: model, domain, fields, master-data build, and the four export pitfalls. |
| `references/ui-conventions.md` | Layout, design tokens, filter behaviour, number formats, and **the CSS classes the runtime depends on**. Read before any restyle. |

## Pitfalls learned the hard way

1. **Truncated exports look complete.** `web_search_read` caps `count` at
   `count_limit` (10 001). Use `search_count` for the true total and assert the
   final row count, or the "year" silently becomes four months.
2. **Store names do not join.** Diacritics, punctuation, `- old` suffixes and
   `name, city` composites silently split one organisation into several. The
   pipeline normalises and reports the leftovers — treat a long unmatched list
   as a data defect, and remember unmatched stores still render (as *Unknown*)
   so no volume disappears.
3. **Versioned models.** `… Pro V1` usually has its own price row. Match the
   full model string and declare a `modelFallback` where a regional price list
   genuinely lacks it; never guess a price.
4. **A missing list price poisons every average above it** while the quantity
   still counts. Watch the report's unpriced-model list.
5. **`colspan` and column counts must follow the config**, or the empty state
   and the header disagree after a per-region change.
6. **Restyling breaks data-driven markup.** Inventory the classes listed in
   `references/ui-conventions.md` §8 first; a wholesale stylesheet swap that
   misses the bar classes leaves the chart as vertical text.
7. **Data-driven DOM must be rebuilt, not left stale**: after a region switch,
   filter options, month options, tables, charts and pies all have to be
   refreshed together, and the selected store cleared.

## Extending

- **Another region/site**: add a `regions.<R>` block, a `regions.<R>.currency`,
  its filter list and its export file. No code change.
- **Another period**: re-export, update `year` and `annualizeFactor`, rebuild.
- **Another language**: override `CONFIG.labels`.
- **Another channel** (e.g. a second sales-team group): a new region entry plus
  a different export filter is enough; keep the calibers identical so the two
  channels stay comparable.
