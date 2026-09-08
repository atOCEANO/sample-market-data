<h1>OCEΛNO <small><code>sample-market-data</code></small></h1>


<div style="padding-top: 0px;">
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT" /></a>
</div>

<br>
<br>
<br>
<br>

## Introduction

`sample-market-data` is **a fixed set of Binance candles for testing infrastructure, not for research.** BTC, ETH, SOL and XRP, each as spot and USDT-perpetual (`linear`), 5-minute bars from September 2020 to July 2026, about 620,000 candles each. Eight parquet files and a CSV index.

It exists so that code handling market data can be exercised where an exchange will not serve you. Exchanges geoblock by IP and much of the cloud is on that list, Colab and many CI runners included, so a live pull fails there; these files load from GitHub regardless. [`emsl`](https://github.com/atOCEANO/embeddable-market-simulation-library) draws every chart image in its documentation from one file here. It is a fixture.

<br>
<br>

## Not a Research Dataset

Nothing here should be used to fit or evaluate a strategy. Three reasons, none of them avoidable by being careful with it:

- **It is a sample of survivors.** The four were picked in 2026, with the answer already known. Anything fitted here inherits that selection, and a sample of survivors looks exactly like a market that rewards holding.
- **It is closer to one series than to eight.** These four move together, and an asset's spot and perpetual move together almost exactly. A rule tested across all eight files has been tested about once.
- **It is one frozen window.** The span never changes, so every run is against the same history, and no measurement taken inside the sample can tell you so. There is no held-out period here.

For research, pull your own through [`exchange-router-service`](https://github.com/atOCEANO/exchange-router-service): the assets you actually mean to trade, including the ones that failed, over a window you did not choose after seeing it.

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

`manifest.csv` holds one row per file with `start`, `end`, `rows`, a `gaps` count, and the Binance tags.

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

UTC throughout. Rows are sorted, unique and never forward-filled, so real gaps stay visible: 287 in every spot file from exchange halts, none in the perps. They sit at the same bars on every machine, which is half of why this works as a fixture, since a loader that mishandles a gap fails here the same way every time. The columns map straight onto `emsl.to_ohlcv`, which ignores `volume_usd`.

<br>
<br>

## Provenance

Pulled 2026-07-25 through [`exchange-router-service`](https://github.com/atOCEANO/exchange-router-service). **Each file is exactly what the SDK's `get_candles` returns**, not resampled or adjusted; only its per-query metadata (`df.attrs`) is left off, since a parquet row cannot carry it. Build against this and swap in the router with nothing in your loading code to change. It is a snapshot, not a feed: it does not update, which is the point.
