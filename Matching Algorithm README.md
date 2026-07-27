# Matching Algorithm — Handover

This notebook matches companies into **interest-rate swaps** and writes the results to a
Google Sheet. Two companies (or one company and a *bundle* of small ones) borrow the loan
type each is comparatively cheaper at, then swap payments so each still ends up with the
exposure it wanted — and the pair pays less interest in total than if each had borrowed its
preferred type directly. That saved interest is the value the business is built on.

The notebook is four cells:

| Cell | Title | Role |
|------|-------|------|
| 0 | Main matcher | Loads data, ranks, matches, rebalances, writes results. **This is the algorithm.** |
| 1 | Fake data generator | Appends random companies to the sheet for stress-testing. |
| 2 | Ranking display | Prints the fixed/floating pool rankings (debug view). |

**To run:** execute cell 0. (Cell 1 first if you want fake test data; cell 2 any time after
cell 0 to inspect the rankings.)

---

## The core financial idea: comparative advantage

Every company can borrow at two rates — a **Lowest Fixed** and a **Lowest Floating**. It also
has a **Preferred Loan Type** (the exposure it actually wants).

A swap saves money when a *fixed-preferring* company is comparatively cheaper at floating, and a
*floating-preferring* company is comparatively cheaper at fixed. Each borrows the type it's
cheaper at, then they swap interest payments. The total rate paid across the pair drops by:

```
benefit (pp) = pref − non_pref
             = (each borrows its PREFERRED type)  −  (each borrows the OTHER type, then swaps)
```

If `benefit > 0`, the swap world (`non_pref`) is cheaper than the no-swap world (`pref`), and
that gap is the rate saving, measured in percentage points. It is independent of loan term —
term only matters when we turn the rate saving into dollars.

Everything downstream is about (a) finding pairs/bundles with a large `benefit`, (b) converting
`benefit` into real dollars, and (c) splitting those dollars fairly.

---

## Cell 0 — section by section

### AUTHENTICATE & LOAD SHEETS
Authenticates to Google, opens the **Matching Algorithm** spreadsheet, and pulls the
**Company info** tab into `data`. **Matches** is the output tab.

### CLEAN & TYPE-CAST DATA
Drops any row missing a required field (`Lowest Fixed`, `Lowest Floating`, `Loan size `,
`Loan term`) and casts the rest to `float` (and the name to `str`). Everything after this point
assumes clean numeric data — a single blank cell would otherwise crash the arithmetic.

> Note: the column key is `'Loan size '` **with a trailing space** — that's how it's spelled in
> the sheet. Keep it.

### LOAN SAVINGS FUNCTION — `total_loan_savings`
The dollar engine. Standard amortizing-loan payment formula:

```
monthly_payment = P · i(1+i)ⁿ / ((1+i)ⁿ − 1)      i = annual_rate/100/12,  n = years·12
```

It computes the monthly payment at the `old_rate` and at the `new_rate`, then returns
`(old_payment − new_payment) · n` — the total nominal interest saved over the life of the loan.
This is what converts a rate improvement (pp) into a dollar figure.

### SWAP SAVINGS SPLIT — `SAVINGS_SPLIT = 0.5`
How the rate `benefit` is shared between the two sides of a swap. `0.5` = the textbook fair
split, each side keeps half the rate improvement.

### PAIR SAVINGS — `pair_savings`
The heart of the dollar math for **one anchor + one counterparty** on a shared notional.

1. Works out `benefit = pref − non_pref` for that pair (direction depends on whether the anchor
   is the fixed-preferring side).
2. Splits `benefit` in two (`SAVINGS_SPLIT`).
3. Amortizes **each half at that party's OWN rate level** and sums the two dollar results.

Why amortize each half separately instead of the whole benefit at one rate? Because each company
actually pays down its own loan at its own rate. Discounting each side's share at its own rate is
the realistic figure, not a fiction priced at some blended rate. Returns `(benefit, dollars)`.

### MEMBER SAVINGS RANK — `member_savings_rate`
A sort key, not a dollar value. When a bundle is bigger than the anchor, the anchor's size caps
the notional and the **last** member only gets partially used (clipped). We want the *lowest-
savings* member to be the one clipped, so we process members **best-savings-first**.

Savings rise as a floating member's spread (`Fixed − Floating`) **falls**, so when the anchor is
fixed we rank by `−spread`; when the anchor is floating it flips to `+spread`. This single key
drives the clip order everywhere savings are summed.

### SHEET WRITE HELPER — `sheet_write`
Wraps every Google Sheets write with retry + exponential backoff on HTTP 429 (rate limit). The
sheet API throttles aggressively; without this, big runs fail mid-write.

### CONSTANTS
- `SIZE_LEEWAY = 0.1` — a bundle's combined size may be ±10% of the anchor's size. This is what
  lets one company match a *bundle* of differently-sized companies instead of needing an exact
  size twin.
- `MIN_RATE_DROP = 0.25` — **the only deal gate.** A deal is kept only if each side's interest
  rate drops at least `0.25` percentage points. This is the same number written to the
  `Savings (rate drop)` column (see *SAVINGS — per-side rate drop*, below). Raise it for fewer,
  higher-quality deals; lower it for more coverage.

### REVENUE SPLIT
How each deal's gross dollar saving is divided (fractions sum to 1.0):
`OUR_CUT 20% · BANK_CUT 7.5% · BROKER_CUT 12.5% · BUSINESS_CUT 45% · RESERVES_CUT 15%`. The
business keeps the largest share; the rest is the revenue model (`RESERVES_CUT` is money held
back as a buffer). These cuts feed the **summary** figures only — they no longer affect the
`Savings (rate drop)` column or the deal gate.

### SAVINGS — per-side rate drop (display *and* deal gate)
A single rate-drop metric does double duty: it's written to the `Savings (rate drop)` column
**and** it's the value `MIN_RATE_DROP` is checked against, in the matcher and the rebalancer alike.
It's the raw interest-rate drop each side gets from the swap, in percentage points:

```
avg benefit (pp)    = mean(fixed-side spreads) − mean(floating-side spreads)   (spread = Fixed − Floating)
each side's drop    = avg benefit · SAVINGS_SPLIT     (half the benefit at the 50/50 split)
Savings (rate drop) = each side's drop               (raw pp, e.g. 5.00% → 4.75% shows as 0.25)
```

Two small helpers split the work: `party_rate_drop(fixed_members, float_members)` returns *each
side's drop* (pp), and `displayed_savings(drop)` just rounds it for the sheet. A deal qualifies iff
`displayed_savings(drop) >= MIN_RATE_DROP`.

Example: if each side's rate falls from 5.00% to 4.75%, the column shows `0.25` (not a percentage).
For a bundle, every company on a side is treated as contributing equally (a plain mean of their
spreads), so all companies in a deal display the same value. Because each side's drop maxes out
around `0.5` with the default fake-data gap range, a `0.25` gate is moderately strict — widen the
rate gaps (cell 1) or lower `MIN_RATE_DROP` to let more deals through.

### FIXED POOL RANKING / FLOATING POOL RANKING
Splits companies by preference and scores them so the best swap candidates are tried first.

- **Fixed pool:** `score = (Lowest Fixed − Lowest Floating) · size · term`, descending. A bigger
  spread means a stronger comparative advantage; multiplying by size and term weights it by how
  many dollars are actually at stake.
- **Floating pool:** uses `(Lowest Floating − Lowest Fixed)`, which is normally negative, so it's
  shifted by `min_raw_diff` so the smallest becomes `1` (keeps every score positive and ordered)
  before multiplying by `size · term`.

Ranking matters because matching is greedy: the highest-scoring anchors get first pick of
counterparties.

### BUNDLE FORMATION — `try_form_bundle`
Given an anchor size, slides a window over candidate loans and greedily assembles a bundle whose
combined size lands inside `[anchor·(1−leeway), anchor·(1+leeway)]`. Oversized members are skipped;
when the running total overshoots, the oldest member is dropped from the front. Returns the first
bundle that fits, or `None`. This is how a large loan on one side gets covered by several smaller
loans on the other.

### BUNDLE SAVINGS ($) — `bundle_savings`
Totals the dollar savings of an anchor against its whole bundle. It walks members
best-savings-first (so the clipped member is the cheapest), caps each member's notional at the
anchor's remaining size, and sums `pair_savings` dollars. Returns the gross dollar figure that
later gets split into cuts.

### ROW NAME PARSER — `match_names`
A match row stores bundle members as a comma-joined string (`"A, B, C"`). This splits one column
back into a clean list of names. Used everywhere the rebalancer reads a row.

### POST-MATCHING REBALANCING — `rebalance_connections`
A local-search clean-up that runs after the greedy first pass, looping until no further
improvement plus one extra sweep. Two moves:

- **PART 1 — cover unmatched sparse companies.** The smaller side ("sparse") may have companies
  left unmatched. For each, it looks for dominant-side companies sitting in *worse* matches,
  bundles them to the unmatched company's size, and — if the new deal clears `MIN_RATE_DROP` —
  pulls them out of their old matches and creates a new one. This raises coverage.
- **PART 2 — upgrade bundle members.** For a match with a single anchor and a multi-member
  bundle, it tries replacing the **worst-ranked** bundle member with a better-ranked *available*
  company of similar size. If the rebuilt deal still clears the threshold, it swaps them and frees
  the old member back into the pool. This raises quality.

Both moves are guarded by the same `sav >= MIN_RATE_DROP` gate, so rebalancing can never push
a deal below the standard every fresh match must meet.

### MAIN MATCHING LOOP
The greedy first pass. Round-robin over rank `i`: take `fixed_pool[i]` as anchor and try to
bundle floating companies, then `floating_pool[i]` as anchor and try to bundle fixed companies.
For each anchor it builds a bundle from the still-available opposite side (worst-first), computes
`sav` (the per-side rate drop shown in `Savings (rate drop)`), and:

- **passes the gate** → record the match row, mark all involved companies as used;
- **fails** → advance past the lowest-savings member and try a different bundle.

Best anchors match first; everyone matches at most once.

### SORT → REBALANCE → RE-SORT
Sorts matches by `Savings (rate drop)` (descending), runs the rebalancer, then re-sorts. Since that
metric is both the gate value *and* the displayed value, it's already correct in every row — there's
no separate end-of-run conversion step. The sheet ends up ordered best-deal-first by the number the
user actually sees.

### WRITE MATCHES TO 'Matches' SHEET
Clears the tab and writes the header + all match rows to `A1`. Column layout (per row):

| Idx | Column | Meaning |
|-----|--------|---------|
| 0 | C Fixed | fixed-side company name(s) |
| 1 | C Floating | floating-side company name(s) |
| 2 | C Fixed Loan Size | fixed-side total loan size |
| 3 | C Floating Loan Size | floating-side total loan size |
| 4 | Total Size | `min(fixed, floating) · 2` — the matched notional counted on both legs |
| 5 | C1 Loan Term | fixed-side term |
| 6 | C2 Loan Term | floating-side term |
| 7 | Total Term | `min` of the two terms (the swap can't outlive the shorter loan) |
| 8 | Savings (rate drop) | each side's avg interest-rate drop in pp (e.g. `0.25`); this *is* the deal-gate metric |
| 9 | Savings ($) | gross dollar saving for the deal |

### SUMMARY (columns K–M)
A three-column block written beside the table: **K = label**, **L = total $**, **M = per-unit $**.

Four headline stat rows first (M left blank):
- **Total companies swapped** — unique names across all deals.
- **Total deals** — number of rows.
- **Average loan size** — `Σ(fixed size + floating size) / companies swapped` (real loan sizes,
  not `Total Size`).
- **Total savings ($)** — `Σ Savings ($)`.

Then one row per revenue cut. **L** holds that cut's whole-pool dollars
(`total_savings · CUT`); **M** holds the per-unit figure:
- **Our cut / Bank cut / Reserves** — counted once per deal → divided by `n_deals` (swaps).
- **Broker cut / Business savings** — spread across the companies → divided by
  `total_companies_swapped`.

---

## Cell 1 — Fake data generator
Appends `N_COMPANIES` random companies to **Company info** for stress-testing. The `PARAMETERS`
block controls count, fixed/floating split, rate midpoint/gap ranges, loan-size distribution
(triangular, peaked at `SIZE_MODE`), term, name prefix, and an optional `RANDOM_SEED`. It
auto-finds the next free `Fake_NNN` index so names never collide. Run it, then run cell 0.

## Cell 2 — Ranking display
Reprints the fixed and floating pools with their scores so you can eyeball the ordering the
matcher uses. Pure diagnostics; writes nothing.

---

## Tuning knobs (quick reference)

| Constant | Cell | Effect |
|----------|------|--------|
| `MIN_RATE_DROP` | 0 | Min per-side interest-rate drop (pp) to keep a deal — the value in `Savings (rate drop)`. **The deal gate.** Default `0.25`. |
| `SIZE_LEEWAY` | 0 | How loosely a bundle's size may match the anchor (±fraction). |
| `SAVINGS_SPLIT` | 0 | Share of the rate benefit each swap side keeps (0.5 = even). |
| `OUR/BANK/BROKER/BUSINESS/RESERVES_CUT` | 0 | Revenue split (must sum to 1.0). Feeds the summary dollar figures only — no longer affects the `Savings (rate drop)` column or the gate. |
| `N_COMPANIES`, `SIZE_MODE`, rate/gap ranges | 1 | Shape of the fake test data. |
