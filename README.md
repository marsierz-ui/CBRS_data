# CBRS_data - Cerebras Systems (Nasdaq: CBRS) perp + stock dataset

Continuous collection of pre-IPO and post-IPO market data for Cerebras
Systems, which listed on Nasdaq on **2026-05-14** at $185/share under the
ticker **CBRS**. Companion to [SPCX_data](https://github.com/marsierz-ui/SPCX_data):
poll a public API on a schedule, append into ever-growing master CSVs, commit.

The interesting part of this dataset is the **pre-IPO perp**: `xyz:CBRS` traded
for 13 days before the listing and priced the Nasdaq open to within ~1.3%.

## Which markets

Cerebras listed on Nasdaq on **2026-05-14** at $185/share. Exactly one perp
existed before that date:

| market | venue | first data | note |
| --- | --- | --- | --- |
| `xyz:CBRS` | trade[XYZ] (HIP-3 dex on Hyperliquid) | 2026-05-01 01:00 UTC | launched as a Pre-IPO Perpetual (IPOP), converted in place to a standard externally-priced perp at listing - one continuous series |

Every other CBRS perp started at or after the Nasdaq open (13:30 UTC on
2026-05-14), so none of them carry pre-IPO price discovery:

| venue | symbol | listed (UTC) |
| --- | --- | --- |
| Bitget | CBRSUSDT | 2026-05-14 13:47 |
| Aster | CBRSUSDT | 2026-05-14 16:40 |
| Lighter | CBRS (market 175) | 2026-05-14 17:55 |
| MEXC | CBRSSTOCK_USDT | 2026-05-15 03:22 |
| OKX | CBRS-USDT-SWAP | 2026-05-15 06:00 |
| Gate | CBRS_USDT | 2026-05-15 06:12 |
| KuCoin | CBRSUSDTM | 2026-05-15 07:30 |
| Bybit | CBRSUSDT | 2026-05-19 08:24 |
| Binance | CBRSUSDT | 2026-05-19 09:30 |

Checked and empty: every other Hyperliquid perp dex (`vntl`/Ventuals, `flx`,
`hyna`, `km`, `mkts`, `para`, `cash`, `abcd`, core HL) - delisted assets stay
in the `meta` universe, so a settled Cerebras market elsewhere on Hyperliquid
would still show up. Also checked: Paradex, Backpack, dYdX, Drift, Vest,
Hibachi, edgeX, Orderly, Avantis, Aevo.

## History horizon (why several intervals)

`candleSnapshot` serves roughly the most recent **5000 candles per interval**.
Older data is gone permanently:

| interval | reach | covers the 2026-05-01 inception? |
| --- | --- | --- |
| 1m | ~3.5 d | no (lost since ~2026-05-05) |
| 15m | ~52 d | no (lost since ~2026-06-22) |
| 30m | ~104 d | yes, until ~2026-08-13 |
| 1h | ~208 d | yes, until ~2026-11-25 |
| 1d | ~5000 d | yes |

The archive under `out/` already holds the full pre-IPO window at 30m, 1h and
1d, captured 2026-08-06. Keep the collector running: after the dates above the
API can no longer reproduce it.

## Usage

```bash
python cbrs_collector.py                                  # one-shot, all intervals
python cbrs_collector.py --loop --poll-sec 60             # run forever
python cbrs_collector.py --coins xyz:CBRS --intervals 1m  # narrow it
```

Output: `out/xyz_CBRS_<interval>_master.csv` with columns
`ts,open,high,low,close,volume,trades` (ts = candle open, UTC).

`cbrs_hl_api.py` is the one-shot pull for ad-hoc windows, plus optional
alignment of the perp against a stock CSV (`--stock`), writing
`out/merged_basis.csv` and `out/comparison.png`.

`.github/workflows/collect.yml` runs the perp collector every 15 minutes and
commits the CSVs. GitHub only fires scheduled workflows from the default
branch, and you can trigger a run by hand from the Actions tab.

## Stock side

`CBRS_import.py` collects the Nasdaq listing into `data/`:

| dataset | path | granularity | source |
| --- | --- | --- | --- |
| ticks (trades + quotes) | `data/ticks/CBRS_YYYY-MM-DD.csv` | every tick (`TRDPRC_1`, `TRDVOL_1`, `BID`, `ASK`) | LSEG `lseg-data`, RIC `CBRS.O` |
| 1-minute bars | `data/1min/CBRS_YYYY-MM-DD.csv` | 1 min, incl. pre/post-market | Yahoo via `yfinance` |
| quote snapshots | `data/snapshots/CBRS_quotes.csv` | one sample per run | Yahoo real-time quote |

It picks the source automatically: LSEG when `LSEG_APP_KEY` is set (desktop
session, needs LSEG Workspace running and logged in on the same machine),
free Yahoo 1-minute bars otherwise. Put the key in a local `.env`, which is
gitignored:

```bash
cp .env.example .env    # then edit .env and set LSEG_APP_KEY
python CBRS_import.py
```

Yahoo only retains 1-minute bars for the trailing ~7 days, so that source has
the same use-it-or-lose-it property as the perp candles.

**Licensing warning:** exchange data agreements generally prohibit
redistributing raw tick data. This repository is public, so do not commit LSEG
tick output to it - keep `data/ticks/` local, or move the repo to private
first.
