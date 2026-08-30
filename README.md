# NSE Stock Fetcher

A command-line downloader for historical end-of-day (EOD) market data. It can
download selected symbols or discover the full NSE equity universe from the
official NSE securities list, then retrieve daily data through
[yfinance](https://github.com/ranaroussi/yfinance).

Each symbol is stored in its own resume-friendly CSV or Parquet file. Subsequent
runs can skip existing files, append only newer trading days, or retry symbols
that previously failed.

## Features

- Downloads daily Open, High, Low, Close, Adjusted Close, and Volume data
- Fetches approximately 20 years of history by default
- Accepts individual symbols, comma-separated symbols, or the full NSE list
- Filters the NSE universe by security series (`EQ` and `BE` by default)
- Writes one CSV or Parquet file per symbol
- Supports incremental updates without duplicating dates
- Retries failed batches with exponential backoff
- Records logs, run summaries, the downloaded symbol universe, and failures
- Optionally combines all symbol files into one long-format dataset
- Supports non-NSE Yahoo Finance tickers and alternate exchange suffixes

## Requirements

- Python 3.10 or newer
- An internet connection to NSE and Yahoo Finance

Install the dependencies:

```powershell
python -m pip install pandas requests yfinance curl_cffi pyarrow
```

`curl_cffi` improves resistance to Yahoo Finance rate limiting. `pyarrow` is
required only when using Parquet output.

## Quick Start

Download a few NSE equities:

```powershell
python nse_eod_downloader.py RELIANCE TCS INFY HDFCBANK
```

Symbols are case-insensitive. The `.NS` suffix is optional, and commas or
semicolons are accepted:

```powershell
python nse_eod_downloader.py reliance.ns "TCS,INFY"
```

With no positional symbols, the program downloads the full NSE universe. This
can take several hours:

```powershell
python nse_eod_downloader.py --outdir ./nse_data --years 20
```

Run a small full-list test first:

```powershell
python nse_eod_downloader.py --outdir ./nse_data --limit 25
```

## Common Workflows

### Update existing data

Append bars newer than the latest stored date for each symbol:

```powershell
python nse_eod_downloader.py --outdir ./nse_data --update
```

To update only selected symbols:

```powershell
python nse_eod_downloader.py RELIANCE TCS --outdir ./nse_data --update
```

Without `--update` or `--force`, symbols that already have an output file are
skipped.

### Force a fresh download

Ignore an existing symbol file and download it again:

```powershell
python nse_eod_downloader.py RELIANCE --outdir ./nse_data --force
```

### Retry failures

Retry symbols recorded in the previous run's `failures.csv`:

```powershell
python nse_eod_downloader.py --outdir ./nse_data --retry-failed
```

Use the same symbol source and series settings as the original run so the failed
symbols remain in the selected universe.

### Write Parquet files

```powershell
python nse_eod_downloader.py RELIANCE TCS --format parquet
```

### Build one combined dataset

Add `--combine` to create `nse_all_eod.csv` or `nse_all_eod.parquet` after the
per-symbol files are processed:

```powershell
python nse_eod_downloader.py --limit 25 --combine
```

The combined file uses long format, with one row per symbol and trading date.

### Use a local NSE symbol list

If the NSE archive cannot be reached, download `EQUITY_L.csv` manually and pass
it to the program:

```powershell
python nse_eod_downloader.py --symbols-file ./EQUITY_L.csv
```

The file must contain a `SYMBOL` column. A `SERIES` column is optional and, when
present, is filtered according to `--series`.

### Use other Yahoo Finance tickers

Tickers containing a dot or beginning with a caret pass through unchanged:

```powershell
python nse_eod_downloader.py INFY.BO ^NSEI
```

Change the suffix appended to bare symbols with `--suffix`:

```powershell
python nse_eod_downloader.py RELIANCE INFY --suffix .BO
```

## Command-Line Options

```text
positional arguments:
  symbols               Symbols to download; omit for all listed NSE equities

options:
  --outdir PATH          Output directory (default: ./nse_data)
  --years N              Number of years of history (default: 20)
  --end YYYY-MM-DD       Exclusive end date (default: tomorrow)
  --series LIST          NSE series filter (default: EQ,BE; use "all" for all)
  --symbols-file PATH    Read symbols from a local CSV instead of NSE
  --limit N              Process only the first N symbols
  --suffix SUFFIX        Suffix for bare symbols (default: .NS)
  --force                Re-download files that already exist
  --format FORMAT        csv or parquet (default: csv)
  --batch-size N         Tickers per yfinance request (default: 25)
  --sleep SECONDS        Base delay between batches (default: 1.5)
  --retries N            Attempts per batch (default: 3)
  --threads N            yfinance worker threads; 0 or 1 disables threading
  --update               Append only dates newer than stored data
  --retry-failed         Retry symbols from failures.csv
  --combine              Also produce one combined long-format file
  --verbose              Enable debug logging
```

Run `python nse_eod_downloader.py --help` for the authoritative option list.

## Output Layout

The default output directory is `./nse_data`:

```text
nse_data/
|-- data/
|   |-- RELIANCE.csv
|   |-- TCS.csv
|   `-- ...
|-- download.log
|-- failures.csv          # created when symbols return no data
|-- manifest.csv          # one summary row per completed download run
|-- symbols.csv           # cached universe for non-positional runs
`-- nse_all_eod.csv       # created only with --combine
```

Each symbol file is indexed by `Date` and contains these columns:

```text
Open, High, Low, Close, Adj Close, Volume
```

## Rate Limits and Troubleshooting

Yahoo Finance may throttle large downloads. The downloader batches requests,
pauses between batches, and retries with backoff. If throttling persists:

- Increase the delay, for example `--sleep 5`
- Reduce the request size, for example `--batch-size 10`
- Reduce concurrency, for example `--threads 1`
- Run the downloader again with `--retry-failed`
- Use `--verbose` and inspect `nse_data/download.log`

An empty response for an existing file during `--update` usually means there are
no new trading bars and is not treated as a failure.

## Data Source and Disclaimer

Symbol metadata comes from NSE, while price data comes from Yahoo Finance via
`yfinance`. This project is not affiliated with or endorsed by NSE or Yahoo.
Availability, adjustments, and historical accuracy depend on those upstream
services. Verify the data independently before using it for trading, investment,
or production decisions.

## License

See [LICENSE](LICENSE).
