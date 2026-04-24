# KG-fin-crime

Knowledge graph for financial crime — the PAI Hackathon 2026 project. Ports the Prevalent SDS medallion pipeline from cybersecurity exposure management to financial crime to demonstrate the platform is domain-agnostic.

## Status

**Hero moment end-to-end — all 7 steps land.**

### Backwards build progress

| # | Step | State |
|---|------|-------|
| 7 | CSV + ring exist | ✅ Real AMLSim; hero ring verified in `scripts/find_cycles.py` |
| 6 | Bronze loaded | ✅ 5.08M txns / 515k KYC / 515k links via `src/bronze/load.py` |
| 5 | Silver entities + edges | ✅ 9 tables populated via `src/silver/transform.py` (~15 min run) |
| 4 | Silver assessment (circular-flow detector) | ✅ 109 rings (32 HIGH / 77 MEDIUM) via `src/assessments/circular_flow.py` (~4 min). Hero ring surfaces as finding_id 81, HIGH. |
| 3 | Gold publisher | ✅ `gold.finding` (109) / `gold.finding_entity` (973) / `gold.finding_edge` (327) via `src/gold/publish.py` (<1s). Flattened summary_stats + control_mapping; party / bank / country denormalized onto entity rows; `owner_party_id` on Account rows drives the party→account graph edges. |
| 2 | FastAPI | ✅ `src/api/` — `GET /findings`, `/findings/{id}`, `/findings/{id}/graph`, `/healthz`. Serves `gold.*` only. |
| 1 | UI renders finding | ✅ `ui/` — Vite + React + TS + Tailwind v4 + Cytoscape.js. Dark glass theme; pipeline header; filterable findings list; per-finding detail with stat strip, parties/controls/traversal cards, and the animated 3-hop ring graph (transfers_to + hasAccount + isHeldAt). |

### Resume commands

```bash
cd repo/
docker compose up -d               # Postgres 16 (auto-applies schema/*.sql on first boot only)
source .venv/bin/activate          # or rebuild: python3 -m venv .venv && pip install -e .
```

Verify bronze + silver + findings + gold are populated:

```bash
docker compose exec postgres psql -U kgfc -d kgfincrime -c "
  SELECT 'bronze.transactions_raw'      AS t, count(*) FROM bronze.transactions_raw
  UNION ALL SELECT 'silver.transfers_to',           count(*) FROM silver.transfers_to
  UNION ALL SELECT 'silver.party',                  count(*) FROM silver.party
  UNION ALL SELECT 'silver.account',                count(*) FROM silver.account
  UNION ALL SELECT 'silver.finding',                count(*) FROM silver.finding
  UNION ALL SELECT 'gold.finding',                  count(*) FROM gold.finding
  UNION ALL SELECT 'gold.finding_entity',           count(*) FROM gold.finding_entity
  UNION ALL SELECT 'gold.finding_edge',             count(*) FROM gold.finding_edge;"
# Expect: 5,078,345 / 5,078,345 / 515,088 / 515,088 / 109 / 109 / 973 / 327
```

Confirm the hero ring is in silver:

```bash
docker compose exec postgres psql -U kgfc -d kgfincrime -c "
  SELECT event_timestamp, from_account_key || ' -> ' || to_account_key AS hop, amount, currency, payment_format
  FROM silver.transfers_to
  WHERE (from_account_key, to_account_key) IN (
    ('0223:8119F8CC0','0222:811D80C30'),
    ('0222:811D80C30','0121:8000E1590'),
    ('0121:8000E1590','0223:8119F8CC0'))
  ORDER BY event_timestamp;"
```

### Cold start (nothing running yet)

```bash
cd repo/
cp .env.example .env
docker compose up -d               # First boot applies schema/*.sql automatically
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
python -m bronze.load              # ~25s
python -m silver.transform         # ~15 min (FK checks are per-row; one-time cost)
python -m assessments.circular_flow  # ~4 min (3-way self-join on 5M edges)
python -m gold.publish             # <1s (109 rows — flatten + denormalize)
uvicorn api.main:app --reload      # FastAPI on http://127.0.0.1:8000 (docs at /docs)

cd ui && npm install && npm run dev  # UI on http://127.0.0.1:5173 (proxies /api → :8000)
```

To rebuild from a clean slate: `docker compose down -v` wipes the volume and all data, then re-run the cold-start sequence.

### Detector validation

Hero ring surfaces as `silver.finding` row with severity `HIGH`:

| Hop | From | To | Amount | Date | fin_txn_id |
|---|---|---|---|---|---|
| 1 | `0223:8119F8CC0` (Ava Miller / NL / MEDIUM) | `0222:811D80C30` (Mia Patel / GB / LOW) | 56,544.74 SAR ACH | 2022-09-05 11:21 | 2,521,343 |
| 2 | `0222:811D80C30` | `0121:8000E1590` (Lucas de Vries / AE / LOW) | 51,178.80 SAR ACH | 2022-09-08 09:41 | 3,934,906 |
| 3 | `0121:8000E1590` | `0223:8119F8CC0` | 51,203.66 SAR ACH | 2022-09-08 16:19 | 4,064,891 |

Pillar-1 label-overlap check (detector does NOT read `bronze.is_laundering`; the label is used only post-hoc):

```bash
docker compose exec postgres psql -U kgfc -d kgfincrime -c "
  WITH edge_label AS (
    SELECT fe.finding_id, fe.hop_order, b.is_laundering
    FROM silver.finding_edge fe
    JOIN silver.transfers_to t        ON t.fin_txn_id = fe.fin_txn_id
    JOIN bronze.transactions_raw b
      ON b.event_timestamp                        = t.event_timestamp
     AND (b.from_bank || ':' || b.from_account)   = t.from_account_key
     AND (b.to_bank   || ':' || b.to_account)     = t.to_account_key
     AND b.amount_paid                            = t.amount
  ),
  per_finding AS (
    SELECT finding_id, SUM(is_laundering) AS hops_labeled FROM edge_label GROUP BY finding_id
  )
  SELECT count(*) FILTER (WHERE hops_labeled=3) AS all_labeled,
         count(*) FILTER (WHERE hops_labeled>=1) AS any_labeled,
         count(*) AS total
  FROM per_finding;"
# Expect: 87 / 91 / 109 (~80% fully label-perfect; ~83% touch at least one laundering-flagged edge)
```

### API surface

```
GET /healthz                        → liveness probe
GET /findings?severity=&limit=      → list view (sorted CRITICAL→LOW, then detected_at DESC)
GET /findings/{id}                  → full detail (control_mapping + SLA + amount_ratio)
GET /findings/{id}/graph            → {entities:[...], edges:[...]} for the drill-down view
```

Hero-ring smoke test: `curl localhost:8000/findings/81/graph` → 9 entities (3 accounts, 3 banks, 3 parties with names/countries/risk tiers) + 3 edges.

### UI layout

```
ui/
├── src/
│   ├── App.tsx                        # shell: header + 2-pane grid
│   ├── main.tsx                       # Vite entry
│   ├── index.css                      # Tailwind v4 + theme tokens + glass/pulse utilities
│   ├── lib/
│   │   ├── api.ts                     # fetch wrapper → /api/*
│   │   ├── types.ts                   # TS mirrors of the pydantic response models
│   │   └── format.ts                  # money / date / hours / country-flag / currency-code
│   └── components/
│       ├── PipelineHeader.tsx         # CSV → Bronze → Silver → Gold → UI framing
│       ├── FindingsList.tsx           # filterable left pane (ALL / HIGH / MEDIUM)
│       ├── FindingCard.tsx            # list item: severity pill, amount, country flags, banks
│       ├── SeverityPill.tsx           # severity-coloured pill w/ pulsing dot
│       ├── FindingDetailPane.tsx      # right pane: hero header + stat strip + cards + graph
│       └── RingGraph.tsx              # Cytoscape (cose-bilkent) with 3 edge types:
│                                      #   transfers_to (cycle, bright arrows w/ amount labels)
│                                      #   hasAccount   (dashed party → account)
│                                      #   isHeldAt     (dashed account → bank)
├── vite.config.ts                     # @ alias + /api → :8000 proxy + Tailwind plugin
└── tsconfig.app.json                  # @ path mapping
```

The UI queries `/api/findings`, `/api/findings/{id}`, `/api/findings/{id}/graph` — all served from `gold.*` via FastAPI. No direct DB access, no silver/bronze reachability from the browser.

## Where to look

This repo is the implementation target. The full workspace lives one level up at `/home/nikhil/Hackathon/` — that is where docs, notes, design, reference material, and demo assets live. **Start there, not here**, when onboarding:

- `../CLAUDE.md` — workspace rules + pickup order.
- `../notes/decisions.md` — current direction (newest at top).
- `../docs/problem-statement.md` — what we're solving.
- `../demo/hero-moment.md` — the 30-second demo we're building backwards from.
- `../docs/architecture.md` — the simplified CSV → Postgres → Python pipeline.
- `../reference-repos/CONTEXT.md` — the Prevalent SDS platform we're porting patterns from.

## Planned layout (once code lands)

```
repo/
├── schema/           DDL for bronze.* / silver.* / gold.*
├── src/
│   ├── bronze/       CSV → Postgres loaders
│   ├── silver/       Normalization → entities + relationships
│   ├── assessments/  Graph traversal → candidate findings (e.g. circular_flow.py)
│   └── gold/         Publishers shaping findings for the UI
├── data/             Symlink or copy of ../data/ (curated CSVs)
└── README.md         This file
```

## Stack

CSV (bronze) → PostgreSQL (silver + gold, one DB, schemas per layer) → Python (pandas + SQL, optional `networkx` for graph traversal). No Spark, no Iceberg, no NiFi — scaled for low demo data volume. See `../notes/decisions.md` (2026-04-23 stack simplification) for rationale.

## Upstream

<https://github.com/NIKHIL-523/KG-fin-crime> — empty on GitHub at clone time; this is the first commit target.
