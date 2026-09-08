<h1>OCEΛNO <small><code>sample-market-data</code></small></h1>


<div style="padding-top: 0px;">
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT" /></a>
</div>

<br>
<br>
<br>
<br>

## Introduction

`sample-market-data` is **a fixed set of Binance candles for testing infrastructure, not for research.** BTC, ETH, SOL, and XRP, each as spot and USDT-perpetual (`linear`), at 5-minute bars, from the 2020 to 2021 bull run through July 2026, about 620,000 candles each. Eight parquet files and a CSV index: read the index, pick a series, pull one file, no router and no clone.

It exists for one job: **exercising the code that handles market data, on a machine an exchange will not serve.** Loaders and schema checks, resamplers, storage layers, plotting, notebooks that have to run in CI, the documentation images of a library. Exchanges geoblock and blacklist by IP, and much of the cloud is on that list, Google Colab and many CI runners included, so a live pull through [`exchange-router-service`](https://github.com/atOCEANO/exchange-router-service) or any exchange client fails there. A proxy or a home connection can route around it, but that is not always possible to set up. These files load from GitHub regardless, so the plumbing can be built and tested where the feed cannot reach.

The concrete case it was built for: [`emsl`](https://github.com/atOCEANO/embeddable-market-simulation-library) draws every chart image in its documentation from one file here, so a rebuild produces the same picture on any machine instead of whatever the market did that morning. It is a fixture.

<br>
<br>

## Not a Research Dataset

Nothing here should be used to fit or evaluate a strategy. Three reasons, none of them avoidable by being careful with it.

**It is a sample of survivors.** The four assets were picked in 2026, with the answer already known. Every one of them existed across the whole window and is still traded today. Anything fitted here inherits that selection, and the flattery is invisible from the inside, because a sample of survivors looks exactly like a market that rewards holding.

**It is closer to one series than to eight.** BTC, ETH, SOL and XRP move together, and the spot and perpetual of an asset move together almost exactly. A rule tested across all eight files has not been tested eight times. It has been tested about once, against one sequence of regimes, and the independence that makes a multi-market test worth running is absent.

**It is one frozen window.** The span is fixed and it never updates, so every run is a run against the same history. Results converge on that history rather than on anything that generalises, and no measurement taken inside the sample can tell you it happened. There is no held-out period here. There is only the period.

For research, pull your own data through [`exchange-router-service`](https://github.com/atOCEANO/exchange-router-service): the assets you actually mean to trade, including the ones that failed, over a window you did not choose after seeing it.

<br>
<br>

## Loading One Market

One market is one file. Runs as-is in a Colab cell (pandas and pyarrow only).

```python
import io
import urllib.request
import pandas as pd

BASE = "https://raw.githubusercontent.com/atOCEANO/sample-market-data/main"

manifest = pd.read_csv(f"{BASE}/manifest.csv")
row = manifest[(manifest["symbol"] == "BTCUSDT") & (manifest["market_type"] == "spot")].iloc[0]

raw = urllib.request.urlopen(f"{BASE}/{row['path']}").read()
df = pd.read_parquet(io.BytesIO(raw))
```

Or clone the lot (~170 MB) and read locally: `pd.read_parquet("data/binance_spot_BTCUSDT_5m.parquet")`.

<br>
<br>

## Markets

Four assets, each as spot and `linear` (Binance USDT-perpetual). `manifest.csv` holds one row per file with `start`, `end`, `rows`, a `gaps` count, and the Binance tags.

| Asset | Market | Bars | From |
| :--- | :--- | ---: | :--- |
| BTC | spot | 620,000 | 2020-09-01 |
| BTC | linear | 620,000 | 2020-09-02 |
| ETH | spot | 620,000 | 2020-09-01 |
| ETH | linear | 620,000 | 2020-09-02 |
| SOL | spot | 620,000 | 2020-09-01 |
| SOL | linear | 616,492 | 2020-09-14 |
| XRP | spot | 620,000 | 2020-09-01 |
| XRP | linear | 620,000 | 2020-09-02 |

<br>
<br>

## Inside a File

Seven columns, exactly what the router's SDK returns for candles, in order:

| column     | dtype   | unit                         | meaning |
| :--------- | :------ | :--------------------------- | :------ |
| timestamp  | int64   | Unix epoch milliseconds, UTC | the bar's open time |
| open       | float64 | quote currency               | price at the bar's open |
| high       | float64 | quote currency               | highest price in the bar |
| low        | float64 | quote currency               | lowest price in the bar |
| close      | float64 | quote currency               | price at the bar's close |
| volume     | float64 | base currency                | traded volume over the bar |
| volume_usd | float64 | quote currency (USDT)        | volume as USDT notional (volume x close) |

UTC throughout; `timestamp` is the bar's open time. Rows are sorted and unique, never forward-filled, so real gaps stay visible and are counted per file in `manifest.csv` (each spot file has 287 from exchange halts, the perps none). The columns map straight onto `emsl.to_ohlcv`, which ignores the extra `volume_usd`.

Those 287 gaps are the reason this works as a fixture: they are real exchange halts, they sit at the same bars on every machine, and a loader that mishandles a gap fails here the same way every time.

<br>
<br>

## Provenance and Fidelity

Pulled 2026-07-25 through [`exchange-router-service`](https://github.com/atOCEANO/exchange-router-service). **Each file is exactly what the SDK's `get_candles` returns**, every column, not resampled or adjusted; only the SDK's per-query metadata (its `df.attrs`) is left off, since a parquet row cannot carry it. Load a file and you hold the same DataFrame the router would hand you, so you can build against this and swap in the router with nothing in your loading code to change. It is a snapshot, not a feed: it does not update, which is the point.
