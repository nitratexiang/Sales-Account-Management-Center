# Store Sales & Discount Dashboard

One self-contained HTML file that answers a single question about every
offline store:

> **What margin did the transactions actually realise — and does it match the
> discount the store was granted for its committed volume?**

It joins an ERP invoice export (invoices *net of credit notes*) with a store
master and a regional list-price table, then reports volume, unit price and
realised discount rate per store and per model, with two store-level verdicts.

Everything is inlined into one `.html`: no server, no CDN, no chart library, no
build step at the reader's end. Mail it, chat it, open it.

```
Filters (region tabs · type · ranking · discount type · salesperson · country · month)
   ↓
Overview   headline stats + monthly volume bars
   ↓
Store summary   one row per store, sorted by volume
   ↓
Details   per model: qty · discount rate · list unit price · actual selling price
   ↓
Distribution   volume and revenue by customer group
```

## What it does

- **Net volume** — sales invoices plus credit notes, so returns cancel out.
- **Realised rate** — `net revenue ÷ (list unit price × net quantity)`,
  aggregated by value, colour-banded into green / amber / red.
- **Extra discount verdict** — granted rate + realised rate vs the tier the
  store was supposed to be on.
- **Tier verdict** — period volume extrapolated to a full year, compared with
  the volume range of the store's discount tier.
- **Multi-site side by side** — one tab per region, each with its own currency,
  its own filters and its own month coverage. No cross-region totals.
- **Country → store → model drill down** — filter the store list, pick a store,
  read its master data, verdicts and model mix.

## Quickstart

```bash
# 1. see it work with synthetic data (no config, no real data)
python3 scripts/build_dashboard.py --demo --out demo.html

# 2. build with your own exports
cp config.example.json config.json     # then edit the paths and columns
python3 scripts/build_dashboard.py --config config.json
```

Requirements: Python 3.9+ and `openpyxl` for `.xlsx` input (`pip install
openpyxl`). `.csv` input needs nothing but the standard library. Node.js is
optional — if present, the build validates the generated page's script syntax.

## Configure

`config.example.json` is annotated; keys prefixed `__` are comments and are
ignored by the build. The essentials:

| Key | Purpose |
|---|---|
| `title` / `subtitle` / `year` | header copy, month-label prefix |
| `regions.<R>` | `label`, `currency`, `filters` (which filters this region has), `summaryExtra` (extra store-summary columns) |
| `typeOrder` / `dtypeOrder` / `ranks` | filter values and their display order |
| `tierDefaults` | volume band per customer type, used when the tier text carries no range |
| `rateBands` / `extraDiscount` / `annualizeFactor` | the three colour bands, the two verdict cuts, the period → full-year factor |
| `pieGroups` | distribution slices (label, colour, member types) |
| `invoices` / `stores` / `pricing` | input files and column mapping (header name, list of names, Excel letter, or 1-based index) |
| `exclude` | rows to drop from a region, e.g. the other site's country |
| `labels` | every UI string — override to translate or re-word, no template edit |

Nothing company-specific lives in the template: the customer types, the
discount ladder, the thresholds and the wording are all configuration.

## Data contract

```json
{
  "store_info": {
    "<normalised store key>": {
      "name": "", "type": "", "ranking": "", "pricelist": "",
      "dtype": "", "drate": "", "salesperson": "", "country": "",
      "region": "EU", "cur": "€"
    }
  },
  "sales": [
    { "sk": "<store key>", "region": "EU", "mm": 7, "p": "<model>",
      "qty": 12, "price": 1499.0, "rate": 0.9123, "cur": "€" }
  ]
}
```

One `sales` row per **store × region × month × model**. `price` is the list
unit price excluding tax; `rate` is `net revenue ÷ (price × qty)`;
`rate: null` means the list price is unknown (the quantity still counts).

The aggregator is generic: point it at another ERP export with the same shape
and it will produce the same payload.

## Design notes

- **Rate, not discount.** The headline percentage is the share of list price
  actually invoiced. 0.89 means "11% off list".
- **Value-weighted averages.** A view's average is `Σ actual ÷ Σ list`, never a
  mean of per-row ratios.
- **Store-level verdicts ignore the month filter**, so a verdict cannot move
  when you click a month.
- **A region only shows filters that vary inside it** — a single-country site
  gets no country filter.
- **Month options come from the data**, never hardcoded to twelve.
- **Unmatched partners are kept**, shown as *Unknown*, and listed by the build
  report, so no volume silently disappears.
- **Self-contained output is a hard requirement** — bars are divs, pies are
  hand-built SVG arcs, fonts are system fonts.

## Repository layout

```
SKILL.md                          agent-facing workflow and rules
README.md                         this file
config.example.json               annotated config skeleton
assets/dashboard_template.html    data-free dashboard (CONFIG + DATA markers)
scripts/build_dashboard.py        read → aggregate → inject → self-check
references/methodology.md         calibers, formulas, verdicts, caveats
references/odoo-export.md         how to pull the invoice report and master data
references/ui-conventions.md      layout, tokens, filter behaviour, CSS hooks
```

## No data is included

This repository ships **no company data**: the template contains an empty
payload, and `--demo` generates obviously synthetic stores and models
(`Demo Cycles Alpha`, `MODEL C1`). Bring your own exports and a config.

If you fork this for a real deployment, keep the generated `.html`, the
`data/` folder and your `config.json` out of version control — `.gitignore`
already excludes them.

## License

No license file is included; add the one you intend to publish under.
