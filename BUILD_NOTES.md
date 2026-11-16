# Build notes — Token Radar entry (make.fun bounty "Build the best token launch radar")

Built 2026-11-16. Node **v24.22.1** (requirement: 22.18+). Standard library only.

## What was built

A static dashboard + scoring program for the bounty as it actually stands:
the brief is about **DefiLlama protocols that record no token**, not Solana
meme launches. The program reads the sealed snapshot, ranks every rankable
pre-token protocol by an implied value derived from the snapshot's own figures,
and writes a board, a dashboard dataset, and a dossier the scorer rebuilds.

- `main.ts` — entry point. Parses `--input` (default `/input`) and `--output`
  (default `$MAKEFUN_OUTPUT_DIR` or `/output`), scores the board, checks it
  against the gate (`boardIssue`), writes `board.json` + `data.js`, copies
  `dashboard/` and `screenshots/`, writes `dossier.json` with the vendored
  `buildDossier`. Deterministic: two runs write identical bytes (verified).
- `lib/model.ts` — the model (v `radar-2`), the farm plans, the snapshot
  parser, the board checks. No protocol name is special-cased anywhere.
- `dashboard/` — `index.html` (629 B), `app.css` (dark terminal theme),
  `app.js` (rendering, search/category/chain/watchlist filters, sorting,
  detail dialog with a recompute check, hash routing). Total 42,207 bytes —
  under the 64 KiB dossier text cap (dossier text is 42,473 bytes, nothing
  truncated). The page renders with **no network**: it reads `data.js` written
  beside `dashboard/`, and recomputes each row from the snapshot figures,
  checking the number `board.json` carries.
- `vendor/makefun-board.ts` — the kit's library, copied verbatim as the kit
  instructs.
- `input/snapshot.json` — the sealed snapshot (758 protocols, as of
  2026-11-14, floor $50M).
- `screenshots/` — the three screenshots (see below).

## How to run

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input input/snapshot.json
cd out && python3 -m http.server 8000
# open http://localhost:8000/dashboard/
```

## The board

- **226 rows**: the 40-protocol watchlist (TVL ≥ $50M, no token on record) +
  186 rankable pre-token protocols below the floor. Rankable = pre-token, not
  an exchange, not a versioned pool.
- The 63 pre-token protocols above the floor that the watchlist leaves out are
  mostly bridges and wrapped/staked products (Binance staked ETH, Coinbase
  Bridge, WBTC, USDT0, …) — the watchlist rule excludes exactly those, so the
  board does not rank them either. (An early count of "289 rankable" did not
  apply the watchlist's exclusion rules to the above-floor set; 226 is the
  correct board under the brief, and it still covers more below-floor rows
  than the starter's 200.)
- Top 5 by implied value: Steakhouse Financial $577.7M, Gauntlet $344.1M,
  Veda $330.8M, USDD $150.3M, Upshift $118.7M. Median implied value $3.5M.
- Model: `implied = round(TVL × category multiple × growth × maturity × chain)`,
  where the category multiple is the median mcap/TVL of live-token protocols
  in that category (≥3 comps, clamped [0.05, 1.50], else the global median
  0.241), growth is the clamped 7d TVL change half-weighted, maturity scales
  0.6–1.0 over three years since first listing, and chains add 2% each up to
  8%. Full derivation in `README.md` and on the page's model panel.
- Live-token protocols (402) are set aside; the top 24 by market cap are on
  the Live token tab with market caps shown as facts.

## Screenshots

All 1440×900 PNG, each ≤512 KiB, captured with Playwright
(`playwright-core` 1.65.1) driving `/opt/meta-chromium/chrome` from the served
`out/` build (served locally; the page itself makes no external requests —
verified: 0 console errors, 0 external requests):

- `screenshots/1-board.png` (140,594 B) — the board, default sort.
- `screenshots/2-detail.png` (195,744 B) — rank 1 detail view (Steakhouse
  Financial) with the model breakdown and farm plan.
- `screenshots/3-filtered.png` (137,442 B) — filtered view (category Lending,
  watchlist only, 2 rows).

Screenshot environment notes (worth knowing if re-capturing elsewhere):
- This Chromium build refuses to start without a WebGL domain policy:
  `--hatch-webgl-allowed-domains-file=<file>` +
  `--hatch-webgl-allowed-domains-sha256=<hex>`. The file's first line must be
  `hatch-webgl-allowed-domains-v1`, then strictly sorted domains, ending with
  a newline. A scratch file `/tmp/webgl-domains.txt` (contents
  `hatch-webgl-allowed-domains-v1` + `example.com`) was used; the page uses
  no WebGL, so the allowlist content does not affect the render.
- The CLI `--screenshot` flag writes nothing in this build; screenshots must
  go through CDP (Playwright).
- Navigating to the local test server (`http://127.0.0.1:8123/`) is blocked by
  the build's Local Network Access checks, so the shots were taken from
  `file://` URLs against `out/` instead. The page's relative `../data.js`
  load works from `file://`, and hash routing (`#p=`, `#category=`,
  `#watch=`) works the same.
- The local HTTP server used for capture (port 8123) was stopped after the
  shots; nothing is still serving.

## Gate validation (all 7 rows pass, verified locally 2026-11-16)

1. `dashboard_files` — 3 files, 42,207 bytes total, no `http://`/`https://`
   in any HTML attribute, style body, script import, CSS file or JS import.
   (The snapshot's source string contains an `https://api.llama.fi` URL, but
   it lives in `data.js` at the output root as a string literal and is rendered
   only as visible footer text — both allowed; the starter writes `data.js`
   the same way.)
2. `board_file` — 226 rows, 369,226 bytes.
3. `sorted_by_value` — ranks exactly 1..226, implied value non-increasing,
   not all equal.
4. `coverage` — all 40 watchlist slugs present.
5. `farm_steps` — 7 steps per row, each 20–600 characters.
6. `screenshots` — 3 PNGs, 1440×900, each ≤512 KiB.
7. `dossier_matches` — `dossier.json` (675,419 bytes) is byte-for-byte what
   `buildDossier` writes from the same `out/` (verified by an independent
   rebuild; note `buildDossier` returns a `Uint8Array` — compare bytes, not
   a decoded string, or a UTF-8 length difference will mislead you).

Also verified: the page's recompute check agrees with `board.json` exactly
("the page recomputes this row as $577.7M, the same number board.json
carries"), and the page's farm plan for rank 1 matches `board.json`'s steps
word for word (a naive `textContent` comparison reports a mismatch only
because the action and its cost are separate elements with no whitespace
between them in the DOM; the cost line is a block element, so it renders on
its own line).

## Data: live vs recorded

- **Recorded (real, from the sealed snapshot)**: TVL, 7d change, first-listed
  date, chains, category, token-recorded status for all 758 protocols.
- **The entry's own claim**: implied value and farm plans — a model built
  from the snapshot's figures, never a price or a promise of a token. The page
  says so in the lede, the detail view and the footer.
- **"Records no token", never "has none"**: a protocol may have a token its
  DefiLlama record does not carry. The page says so too.
- Nothing here is live market data, and nothing touches a wallet, a trade, or
  any live service. The PumpPortal soak data is **not** used: this bounty is
  about DefiLlama protocols, and the grading sandbox has no network.

## Missing vs the bounty requirements

- **Not done (needs the parent / Truong)**: nothing in the build is missing.
  What is not done is the *submission* itself — this task was build-only:
  no GitHub repo created, nothing published, nothing submitted to make.fun,
  no wallet connected. Submission needs: a public GitHub repo at a commit
  (created after 2026-11-14, OSI license — MIT is included), the transcript
  bundle of the sessions that built the entry, and sign-in (the make.fun
  account already exists — see the parent agent; GitHub is not yet linked,
  and the parent notes Truong prefers proving ownership via
  `.makefun/owner.json` with the wallet wherever that is accepted instead of
  linking GitHub).
- The entry was **not** run through make.fun's own `check`/`submit` CLI here;
  the 7 gate rows above were validated locally against the documented gate.
  The scorer's own run is the final word.
