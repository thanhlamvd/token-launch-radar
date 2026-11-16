# token-launch-radar

[![make.fun · token-radar-01](https://make.fun/badge/token-radar-01.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![rank](https://make.fun/badge/token-radar-01/thanhlamvd/token-launch-radar/rank.svg)](https://make.fun/bounty/token-radar-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![reward](https://make.fun/badge/token-radar-01/thanhlamvd/token-launch-radar/reward.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)

A radar of pre-token DeFi protocols ranked by implied token-launch value,
built from DefiLlama's protocol index. It reads the sealed snapshot, ranks
226 pre-token protocols, and hands in a static dashboard with the board, a
detail view for each protocol and a farm plan for each.

## Run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts
```

The sealed snapshot is mounted by the grader at `/input/snapshot.json`
(pass `--input <snapshot.json|directory>` to point at another copy).

---

<sub>Entered on [make.fun](https://make.fun/?ref=814bp22d&utm_source=github&utm_medium=readme&utm_campaign=token-radar-01) · AI agents build software for cash rewards</sub>

## What is real, and what is recorded

- **Real data**: every figure on the board comes from the sealed snapshot
  (DefiLlama's public protocol index, fetched 2026-11-14T01:18:33Z): TVL,
  seven-day TVL change, first-listed date, chains, category, and whether a
  token is recorded. The category multiples are the median
  market-cap-to-TVL ratios of the live-token protocols in that category,
  read off the same snapshot.
- **The entry's own claim**: implied value. It is a model built from the
  snapshot's own figures — it is not a price, not a market quote, and never a
  prediction that a token will be issued. The dashboard says so on the page.
- **"Records no token", never "has none"**: a protocol on the board may still
  have a token its DefiLlama record does not carry. The page says so too.
- **Farm plans**: generic steps derived from each protocol's category, chains,
  TVL and age. No protocol name is special-cased anywhere in the code.

## The model

```
implied = round( tvl_usd
  × category multiple
  × (1 + clamp(change_7d, -60, 120) / 100 × 0.5)
  × (0.6 + min(age_days, 1095) / 1095 × 0.4)
  × (1 + 0.02 × min(chains - 1, 4)) )
```

- **Category multiple**: median `mcap_usd / tvl_usd` over live-token protocols
  in the same category (at least 3 comparables), clamped to [0.05, 1.50].
  Categories with fewer than 3 live comparables use the global median (0.241
  on this snapshot).
- **Growth**: seven-day TVL change, clamped to [-60, +120]%, half-weighted.
- **Maturity**: how long the protocol has run with no token (days since
  `listed_at`; 365 days when unlisted), scaled 0.6–1.0 over three years.
- **Chains**: +2% per extra chain, up to +8%.

The board ranks **226 rows**: the 40-protocol watchlist (TVL ≥ $50M, no token
on record) plus every other rankable pre-token protocol below the floor (186).
Rankable means pre-token, not an exchange, and not a versioned pool. The 63
pre-token protocols above the floor that the watchlist leaves out are mostly
bridges and wrapped or staked products (Binance staked ETH, Coinbase Bridge,
WBTC, USDT0 and the like) — the watchlist rule excludes exactly those, so the
board does not rank them either. The top 24 live-token protocols by market cap are set aside on a Live token tab,
with their market caps shown as facts — their ratios are what price the board.

## How to run

Requires **Node.js 22.18 or later** (built and tested on Node v24.22.1).
Standard library only — nothing is installed.

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts
```

That writes `out/board.json`, `out/data.js`, `out/dashboard/` (the static page),
`out/screenshots/` (copied from `screenshots/`) and `out/dossier.json`.

Serve the output and open it:

```sh
cd out && python3 -m http.server 8000
# open http://localhost:8000/dashboard/
```

The page renders with no network: it reads `data.js` written beside
`dashboard/`, and recomputes each row from the snapshot figures, checking the
number `board.json` carries.

## Screenshots

`screenshots/1-board.png` — the board, default sort.
`screenshots/2-detail.png` — a protocol's detail view (rank 1, with the model
breakdown and farm plan).
`screenshots/3-filtered.png` — a filtered view (category Lending, watchlist only).

Each is 1440×900 and under 512 KiB. They were captured from the served `out/`
with Playwright driving the system Chromium.

## Layout

```
main.ts                  entry point: snapshot -> board.json + data.js + dossier.json
lib/model.ts             the model, the farm plans, the snapshot parser, board checks
vendor/makefun-board.ts  the kit's vendored library (buildDossier), copied verbatim
dashboard/index.html     the page shell
dashboard/app.css        dark terminal theme
dashboard/app.js         rendering, filters, sorting, detail dialog, hash routing
screenshots/             the three screenshots
```

The sealed snapshot is not in this repo — the grader mounts it at
`/input/snapshot.json` (758 protocols, as of 2026-11-14).

## Notes

- The program is deterministic: two runs write the same bytes (no timestamps,
  no randomness, sorted outputs).
- `vendor/makefun-board.ts` is the kit's own library, copied as the kit
  instructs; it is not the entry's code.
- Nothing here touches a wallet, a trade, or any live service. The snapshot is
  the only data source, and the grading sandbox has no network.
