# Chicago-Rideshare-Event-Study
## Introduction
This project measures how rideshare demand, pricing, and traveler behavior shift around two 2024 soccer matches at Soldier Field: **Chicago Fire vs Inter Miami (Aug 31, 2024; kickoff 7:30 PM CT)** and **Mexico vs Bolivia (May 31, 2024; kickoff 8:00 PM CT)**. Using the City of Chicago Transportation Network Provider (TNP) dataset (2023–2024), we ingest trip start/end times, pickup/drop-off locations, fares, and durations for trips touching Community Area 33 (Soldier Field). We construct event windows that capture **arrivals** (hours before kickoff, with a small post-kick allowance) and **departures** (around the estimated final whistle through hours after), then build **matched baselines** (±1–4 weeks, same weekday and clock time). With these, we answer three questions: **(1)** How do fares and ride volumes change before, during, and after the matches? **(2)** When do peaks occur, and where are they concentrated? **(3)** How do speeds (a proxy for congestion/wait) respond? Outputs include 15-minute demand curves (event vs. baseline), spatial heatmaps, and indices such as fare-per-mile surge, trip-lift percentages, and speed deltas—providing actionable insight on staging zones, pricing expectations, and communications for future large events.

## Contents
- [Introduction](#introduction)
- [Overview](#overview)
- [Data Source](#data-source)
- [Quick Start (Colab)](#quick-start-colab)
  - [Install dependencies](#install-dependencies)
  - [Set a Socrata App Token (optional)](#set-a-socrata-app-token-optional)
  - [Sanity-check sample](#sanity-check-sample)
  - [Fetch one event window (hour-chunked, robust)](#fetch-one-event-window-hour-chunked-robust)
- [Outputs](#outputs)
- [License](#license)
  

## Overview
This repo quantifies how large events affect rideshare activity near Soldier Field:
- Demand curves (15-minute pickups)
- Price pressure (median fare-per-mile)
- Congestion proxy (median mph)
- Event vs matched baselines (same weekday & clock time)

Built to be robust against API timeouts via hour-chunked fetches. Easily adapted to other venues/events.

---

## Data Source
- **Dataset:** Transportation Network Providers — Trips (2023–2024)  
- **Portal:** https://data.cityofchicago.org/Transportation/Transportation-Network-Providers-Trips-2023-2024-/n26f-ihde/about_data  
- **API ID:** `n26f-ihde`  
- **JSON endpoint:** `https://data.cityofchicago.org/resource/n26f-ihde.json`

Notes:
- Timestamps are *floating* (local; no timezone) and rounded to **15 minutes**.
- Fares rounded to nearest **$2.50**; tips to **$1.00**.
- Some geographies outside the city are suppressed; community-area fields are Chicago-only.

---

## Event Windows & Zone
- **Kickoffs (CT):**
  - Chicago Fire vs Inter Miami — **2024-08-31 19:30**
  - Mexico vs Bolivia — **2024-05-31 20:00**
- **Window:** `PRE_H` hours before kickoff and `POST_H` hours after, plus optional 15-min pad.  
  Example used: `PRE_H = 2`, `POST_H = 3` → **17:30 → 22:30** for a 19:30 kickoff.
- **Zone filter:** Community Area **33** (Near South Side), which contains Soldier Field.  
  Filter trips where **pickup OR dropoff community area = 33**.

---

## Quick Start (Colab)

1. Open via the badge above (or start a new Colab notebook).
2. Install dependencies:
   ```python
   !pip -q install pandas numpy sodapy python-dateutil pyarrow matplotlib
   ```
3. (Optional) Set a Socrata App Token for higher rate limits
   ```python
   %env CHICAGO_APP_TOKEN=your_socrata_app_token_here
   ```
5. Imports
- `pandas`, `numpy` — data wrangling and math
- `matplotlib`, `matplotlib.dates` — plotting (including time axes)
- `sodapy.Socrata` — Python client for the City of Chicago (Socrata) API
- `requests.exceptions.ReadTimeout`, `requests.exceptions.ConnectionError` — handle network hiccups gracefully
- `os`, `time` — environment variables (API token) and small delays/retries
   ```python
   import os, time
   import pandas as pd
   import numpy as np
   import matplotlib.pyplot as plt
   import matplotlib.dates as mdates
   from sodapy import Socrata
   from requests.exceptions import ReadTimeout, ConnectionError
   ```
6. Connect to the Chicago Open Data API
   ```python
   DOMAIN  = "data.cityofchicago.org"
   DATASET = "n26f-ihde"   # Transportation Network Providers - Trips (2023-2024)
   APP_TOKEN = os.getenv("CHICAGO_APP_TOKEN", None)
   client = Socrata(DOMAIN, APP_TOKEN, timeout=300)
   ```

- `APP_TOKEN`: Uses your Socrata App Token if set in Colab (e.g., %env CHICAGO_APP_TOKEN=...).
- If not set, the client still works for public datasets, but with lower rate limits.
- `timeout=300`: Allows up to 300 seconds per request to accommodate large queries or slow responses.

7. Choose the slice of data relevant to the analysis
   ```python
   CA = 33          # Soldier Field’s Community Area (Near South Side)
   YEAR = 2024
   MONTHS = [5, 6, 8, 9]  # May, June, August, September
   MONTHS_CSV = ",".join(str(m) for m in MONTHS)
   ```
- You’ll analyze trips that touch Community Area 33 and occur in specific 2024 months (extra months to account for baseline analysis)

8. Build the server-side filter (SoQL WHERE clause)
   ```python
   WHERE_CA33 = f"(pickup_community_area = {CA} OR dropoff_community_area = {CA})"
   ```
- Keep trips where either pickup or dropdoff is **CA 33**:

  ```python
  WHERE_TIME = (
    "("
    f"(date_extract_y(trip_start_timestamp) = {YEAR} AND "
    f"  date_extract_m(trip_start_timestamp) IN ({MONTHS_CSV}))"
    " OR "
    f"(date_extract_y(trip_end_timestamp) = {YEAR} AND "
    f"  date_extract_m(trip_end_timestamp) IN ({MONTHS_CSV}))"
    ")"
  )
  ```
- Limits trips to those whose start time is in your selected year + months.
- Creates the focus on arrivals and departures in the select months.
- In practice, you will still do precise event-window slicing locally (before/after kickoff). This month filter just reduces API volume.

 ```python
 WHERE_FINAL = f"{WHERE_CA33} AND ({WHERE_TIME})"
 ```
- Combines area and time filters: trips touching CA 33 and starting or ending in the specified months of 2024
- 
9. Pagination Setup
- `PAGE_SIZE = 50_000` — ask Socrata for up to 50k rows per request (their max).
- `rows, offset = [], 0` — an accumulator for pages, and the starting offset.

10. Count for progress (Optional)
    ```python
    total = int(client.get(DATASET, select="count(1)", where=WHERE_FINAL)[0]["count"])
    ```
- Asks the API for a row count that match your WHERE_FINAL filter (CA 33 + month/year).
- If it fails, `total = None` and you’ll still stream pages—just without a progress denominator.
  
11. Paged fetch loop (stable ordering)
    ```python
    batch = client.get(
    DATASET,
    where=WHERE_FINAL,
    limit=PAGE_SIZE,
    offset=offset,
    order=":id"
    )
   ```
- Pulls one page of up to 50k rows that satisfy your filter.
- `order=":id"` enforces a stable order across pages, preventing skips/duplicates while paginating.
- On `ReadTimeout` / `ConnectionError`, it sleeps 1s and retries the same page once.

###Control flow:
- Stop when batch is empty (if not batch: break).
- Accumulate with rows.extend(batch).
- Advance the window: `offset += PAGE_SIZE`.
- Print progress: either `offset/total` (capped with `min(offset, total))` or just offset if total is unknown.

12. Building a dataframe
  ```python
  df = pd.DataFrame.from_records(rows)
  ```
- Turns the list of dicts (each record = one trip) into a tabular `DataFrame`.
- 
 13. Enforce “touches CA 33” client-side (belt & suspenders)
  ```python
  for ca_col in ("pickup_community_area", "dropoff_community_area"):
    df[ca_col] = pd.to_numeric(df[ca_col], errors="coerce")
m_either = df["pickup_community_area"].eq(CA) | df["dropoff_community_area"].eq(CA)
if (~m_either).any():
    df = df.loc[m_either].copy()
```
- Coerces CA columns to numeric so comparisons are reliable (`"33"` → 33, bad values → `NaN`).
- Filters out any stragglers that don’t pick up or drop off in CA 33 (should be rare if the server-side filter worked perfectly).
14. Parse timestamps as local wall-clock (naive)
```python
for col in ("trip_start_timestamp", "trip_end_timestamp"):
    s = pd.to_datetime(df[col], errors="coerce")  # no utc=True
    if getattr(s.dt, "tz", None) is not None:
        s = s.dt.tz_localize(None)
    df[col] = s
```

- The dataset uses “Floating Timestamp” (local Chicago time without timezone).
- Parsing as naive preserves the exact wall clock (e.g., 19:30 kickoff stays 19:30).
- Strips any stray timezone to avoid mixing tz-aware/naive datetimes (which breaks comparisons and binning).
15. Coerce numerics & compute derived fields
  ```python
  num_cols = [...]
  for c in num_cols:
    df[c] = pd.to_numeric(df[c], errors="coerce")

df["mph"] = 3600 * df["trip_miles"] / df["trip_seconds"]         # average speed
df["fare_per_mile"] = df["trip_total"] / df["trip_miles"]        # price intensity
 ```
- Ensures numeric columns are actually numeric for math/aggregations.
- `mph` = miles ÷ (seconds/3600).
- `fare_per_mile` = total paid per mile.
- Note: if `trip_seconds` or `trip_miles` are 0/NaN, results may be inf/NaN. (You can add guards later if needed.)

16. Final shape summary
 ```python
    print(f"Rows fetched (CA {CA}, {YEAR}-{{{', '.join(map(str, MONTHS))}}}): {len(df):,} | Columns: {len(df.columns)}")
 ```
- Prints the final row/column counts for your filtered pull (CA 33, selected 2024 months).
