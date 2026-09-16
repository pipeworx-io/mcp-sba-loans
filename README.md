# @pipeworx/sba-loans

U.S. Small Business Administration 7(a) and 504 loan data — "did this
business get an SBA loan, from whom, how much, and did it pay it off" — kept
current from SBA's own quarterly FOIA CSV extracts.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

**Phase 1 (this pack, shipped): 7(a) + 504.** PPP is a separate phase 2 —
not built yet, ~5GB, tracked on fleet #1921 as remaining work.

## Tools

- `sba_loan_lookup(borrower_name, state?, program?, limit?)` — a borrower's
  loan(s): amount, lender, approval date, status, charge-off detail.
- `sba_largest_loans(program?, naics_code?, state?, lender_name?, limit?)` —
  largest loans by any combination of filters.
- `sba_lender_league_table(program?, state?, naics_code?, limit?)` — lenders
  ranked by loan count and dollar volume.
- `sba_charge_off_aggregate(group_by, program?, state?, naics_code?, limit?)`
  — charge-off rate/amount grouped by naics | state | lender | program.

## Auth

Keyless. The gateway supplies data-store credentials to this pack
automatically (see the pack's entry in `workers/gateway/src/index.ts`), same
pattern as `zillow` and `usaspending`. No caller-facing key.

## PERSONAL-DATA RULE (mandatory, harris-county precedent)

Many 7(a)/504 (and, when built, PPP) borrowers are **sole proprietors —
individual humans**, not companies. Street addresses (borrower AND
lender-side: `BorrStreet`, `BankStreet`, `CDC_Street`,
`ThirdPartyLender_Street`) are **dropped at ingest** —
`scripts/sba-loans-transform.py` never reads those source columns into the
row it emits, so they cannot reach a stored row or a response no matter what
the source CSV contains. Kept: business/borrower name, city, state, ZIP,
NAICS, lender identity, amounts, dates, status. This is the same rule
applied to Harris County civil dockets
(`supabase/migrations/dockets_005_harris_drop_pii.sql`): the standing
public-data sourcing rule covers bulk public loan records, not bulk
redistribution of individuals' home addresses.

## Data sources

- `https://data.sba.gov/dataset/7a-504-foia` — dataset page. **Portal trap**:
  the CKAN root `/dataset/7-a-504-foia` (with the hyphen) 404s; the real slug
  has no hyphen between "7" and "a".
- Direct CSV downloads (quarterly `asof_YYMMDD` stamp in the filename):
  `https://data.sba.gov/sites/default/files/uploaded_resources/FOIA_7a_FY2020_Present_asof_<stamp>.csv`,
  `..._FY2010_FY2019_...`, and `FOIA_504_FY2010_Present_asof_<stamp>.csv`.
  Two more decades exist (7(a) FY1991-1999, FY2000-2009; 504 FY1991-2009,
  ~520MB more) — not ingested in phase 1 to keep the first load disciplined;
  easy to add later via the same transform script if demand shows up.
- No developer API — this is a bulk-CSV-only source.

## Ingest harness

- `supabase/migrations/195_sba_loans.sql` — unified `sba_loans` table (both
  programs; 504's CDC and 7(a)'s bank are normalized into
  `lender_name`/`lender_city`/`lender_state`, with 504's separate
  `ThirdPartyLender` kept in its own columns rather than forced into the same
  slot) + RPC functions the tools above call.
- `scripts/sba-loans-transform.py` — per-program CSV → unified-schema CSV,
  computing a deterministic `loan_key` (there is **no natural unique loan id**
  in the source — `LocationID` is a lender/office code reused across
  unrelated loans, confirmed live 2026-09-13, not a loan identifier).
  `loan_key` hashes the fields that identify a specific loan (program,
  location_id, borrower, approval date, amount, NAICS) and excludes the
  fields SBA revises over a loan's life (status, PIF/charge-off dates,
  charge-off amount) — so a quarterly re-ingest **updates** a loan's outcome
  in place via `ON CONFLICT (loan_key) DO UPDATE`, instead of duplicating the
  row.
- `scripts/sba-loans-upsert.sh` — download → transform → batched staging-table
  drain into `sba_loans` (same crash-safe, disk-safe shape as
  `scripts/usaspending-upsert.sh`: batches commit independently, a failed run
  drops its staging table rather than leaving it to grow Supabase disk
  forever).
- `.github/workflows/sba-loans-refresh.yml` — quarterly cron (`workflow_dispatch`
  for manual/backfill runs) on a GH-hosted runner, because this needs a
  pooler connection string held as a GitHub Actions secret for the
  `psql \copy`, not a Cloudflare Worker.

## Gotchas

- **7(a) and 504 spell "paid in full" differently in the source**: `"P I F"`
  (7(a), with spaces) vs `"PIF"` (504, no spaces). `sba_charge_off_aggregate`
  only depends on the `CHGOFF` code, which IS consistent across both
  programs (verified live 2026-09-13: 300/2,050/751/3,313 for
  CHGOFF/EXEMPT/CANCLD/"P I F" in a 7(a) sample; 177/1,036/926/3,790 for the
  504 equivalents) — but don't assume other status strings line up between
  programs.
- **504 has no `SBAGuaranteedApproval`, `InitialInterestRate`,
  `FixedorVariableInterestInd`, `RevolverStatus`, or `SoldSecMrktInd`** —
  those columns are NULL for every 504 row by design (7(a)-only fields), not
  a data-quality issue.
- Do not fold this pack into `finra`, `sec-adv`, or any other financial-data
  pack — it is a different agency, different licence-free public-domain
  data, and its own table.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sba-loans": {
      "url": "https://gateway.pipeworx.io/sba-loans/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sba-loans/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "sba-loans": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sba-loans"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-sba-loans
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sba Loans data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
