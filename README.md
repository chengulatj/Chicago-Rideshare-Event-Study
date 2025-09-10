# Chicago Rideshare Event Study
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

## Introduction
This project measures how rideshare demand, pricing, and traveler behavior shift around two 2024 soccer matches at Soldier Field: **Chicago Fire vs Inter Miami (Aug 31, 2024; kickoff 7:30 PM CT)** and **Mexico vs Bolivia (May 31, 2024; kickoff 8:00 PM CT)**. Using the City of Chicago Transportation Network Provider (TNP) dataset (2023–2024), we ingest trip start/end times, pickup/drop-off locations, fares, and durations for trips touching Community Area 33 (Soldier Field). We construct event windows that capture **arrivals** (hours before kickoff, with a small post-kick allowance) and **departures** (around the estimated final whistle through hours after), then build **matched baselines** (±1–4 weeks, same weekday and clock time). With these, we answer three questions: **(1)** How do fares and ride volumes change before, during, and after the matches? **(2)** When do peaks occur, and where are they concentrated? **(3)** How do speeds (a proxy for congestion/wait) respond? Outputs include 15-minute demand curves (event vs. baseline), spatial heatmaps, and indices such as fare-per-mile surge, trip-lift percentages, and speed deltas—providing actionable insight on staging zones, pricing expectations, and communications for future large events.

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

 ## Filter, Load, and Extract the Data
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

### Control flow:
- Stop when batch is empty (if not batch: break).
- Accumulate with rows.extend(batch).
- Advance the window: `offset += PAGE_SIZE`.
- Print progress: either `offset/total` (capped with `min(offset, total))` or just offset if total is unknown.

12. Building a dataframe
    ```python
    df = pd.DataFrame.from_records(rows)
    ```
    
- Turns the list of dicts (each record = one trip) into a tabular `DataFrame`.

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

 ## Event Windows & Segment Slicing (Arrivals vs Departures)

**Goal:** From a pre-filtered `df` (CA=33, selected months), extract:
- **Arrivals**: trips **dropping off in CA 33** with **END** times inside a pre-match window.  
- **Departures**:  trips **picking up in CA 33** with **START** times inside a post-match window.  
Also build **±1..4 week** baseline windows (same weekday & clock time) for comparison.

---

###  Define matches (kickoffs as local-naive)

```python
EVENTS = [
    {"name": "2024-08-31 Fire v Inter Miami (Soldier Field)", "kick": pd.Timestamp("2024-08-31 19:30")},
    {"name": "2024-05-31 Mexico v Bolivia (Soldier Field)",   "kick": pd.Timestamp("2024-05-31 20:00")},
]
```
### Segment timing parameters & window builder
- Arrivals window: from `kickoff - PRE_H` - PAD_MIN` to `kickoff + LATE_ARRIVAL_MIN + PAD_MIN` (uses `END` times).
- Departures window: from finalWhistle - EARLY_DEPART_MIN - PAD_MIN to `finalWhistle + POST_H + PAD_MIN` (uses START times).
- `PAD_MIN=15` helps absorb the dataset’s 15-minute rounding.
```python
PRE_H  = 3   # hours before kickoff included in arrivals
POST_H = 1.5   # hours after final whistle included in departures
REG_MIN, HALFTIME_MIN, BUFFER_MIN = 90, 15, 5
GAME_MIN = REG_MIN + HALFTIME_MIN + BUFFER_MIN  # ~110 min
LATE_ARRIVAL_MIN = 20   # allow arrivals up to 20 min after kickoff
EARLY_DEPART_MIN = 20   # allow departures up to 20 min before final whistle
PAD_MIN = 15            # absorb dataset's 15-min rounding
```
The function below calculates the time windows for:
- Pre-match arrivals (before and slightly after kickoff).
- Post-match departures (slightly before the game ends and several hours after).
It makes the windows flexible enough to handle real-world quirks like people arriving late, leaving early, or timestamps being rounded

```python
def flexible_bounds(kick_ts: pd.Timestamp,
                    pre_h=PRE_H, post_h=POST_H,
                    game_min=GAME_MIN,
                    late_arrival_min=LATE_ARRIVAL_MIN,
                    early_depart_min=EARLY_DEPART_MIN,
                    pad_min=PAD_MIN):
    # ensure naive (no tz)
    if getattr(kick_ts, "tzinfo", None) is not None:
        kick_ts = kick_ts.tz_localize(None)

    pre_start  = kick_ts - pd.Timedelta(hours=pre_h) - pd.Timedelta(minutes=pad_min)
    pre_end    = kick_ts + pd.Timedelta(minutes=late_arrival_min + pad_min)

    post_start = (kick_ts + pd.Timedelta(minutes=game_min)
                            - pd.Timedelta(minutes=early_depart_min + pad_min))
    post_end   = (kick_ts + pd.Timedelta(minutes=game_min)
                            + pd.Timedelta(hours=post_h, minutes=pad_min))

    if pre_end >= post_start:
        post_start = pre_end + pd.Timedelta(minutes=1)
    return pre_start, pre_end, post_start, post_end
```
- `build_segment_baselines` creates baseline windows for comparison.
- Instead of only looking at rides on the actual game day, we also want to look at the same time windows on nearby weeks (like 1 week before, 2 weeks after, etc.).
- This gives us a “normal pattern” to compare against.

```python
def build_segment_baselines(kick_ts: pd.Timestamp, segment: str, weeks: int = 4):
    """Return list of (start_ts, end_ts) windows for ±1..weeks weeks (same weekday/clock)."""
    pre_start, pre_end, post_start, post_end = flexible_bounds(kick_ts)
    s0, e0 = (pre_start, pre_end) if segment == "pre" else (post_start, post_end)
    wins = []
    for w in range(1, weeks+1):
        for sign in (-1, +1):
            delta = pd.Timedelta(days=7*w*sign)
            wins.append((s0 + delta, e0 + delta))
    return wins
```
- `kick_ts`: kickoff time of the match.
- `segment`: `"pre"` or `"post"` (are we building arrivals or departures baselines?).
- `weeks`: how many weeks before and after to generate (default = 4).
So if kickoff = Aug 31, 2024, 7:30 PM → it will generate baseline windows for:
- Aug 24 (−1 week)
- Sep 7 (+1 week)
- Aug 17 (−2 weeks)
- Sep 14 (+2 weeks) … up to 4 weeks.

### Segment slicers (event windows)

- Arrivals: `dropoff_community_area == CA` AND `trip_end_timestamp` within arrivals window.
- Departures: `pickup_community_area == CA` AND `trip_start_timestamp` within departures window.

```python
def slice_arrivals_event(df_in: pd.DataFrame, kick_ts: pd.Timestamp) -> pd.DataFrame:
    """Arrivals: drop-offs in CA with END times in pre window (incl. late arrivals)."""
    pre_start, pre_end, _, _ = flexible_bounds(kick_ts)
    drop_ca = pd.to_numeric(df_in.get("dropoff_community_area"), errors="coerce")
    mask = (drop_ca == CA) & df_in["trip_end_timestamp"].between(pre_start, pre_end, inclusive="both")
    out = df_in.loc[mask].copy()
    out["segment"] = "arrivals_pre"
    out["segment_start_local"] = pre_start
    out["segment_end_local"] = pre_end
    return out

def slice_departures_event(df_in: pd.DataFrame, kick_ts: pd.Timestamp) -> pd.DataFrame:
    """Departures: pickups in CA with START times in post window (incl. early departures)."""
    _, _, post_start, post_end = flexible_bounds(kick_ts)
    pick_ca = pd.to_numeric(df_in.get("pickup_community_area"), errors="coerce")
    mask = (pick_ca == CA) & df_in["trip_start_timestamp"].between(post_start, post_end, inclusive="both")
    out = df_in.loc[mask].copy()
    out["segment"] = "departures_post"
    out["segment_start_local"] = post_start
    out["segment_end_local"] = post_end
    return out
```
### Baseline slicers (±1..4 weeks)
```python
def slice_arrivals_baselines(df_in: pd.DataFrame, kick_ts: pd.Timestamp, weeks=4) -> pd.DataFrame:
    frames = []
    drop_ca = pd.to_numeric(df_in.get("dropoff_community_area"), errors="coerce")
    for s, e in build_segment_baselines(kick_ts, "pre", weeks=weeks):
        m = (drop_ca == CA) & df_in["trip_end_timestamp"].between(s, e, inclusive="both")
        part = df_in.loc[m].copy()
        if not part.empty:
            part["segment"] = "arrivals_pre"
            part["segment_start_local"] = s
            part["segment_end_local"] = e
            frames.append(part)
    return pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()

def slice_departures_baselines(df_in: pd.DataFrame, kick_ts: pd.Timestamp, weeks=4) -> pd.DataFrame:
    frames = []
    pick_ca = pd.to_numeric(df_in.get("pickup_community_area"), errors="coerce")
    for s, e in build_segment_baselines(kick_ts, "post", weeks=weeks):
        m = (pick_ca == CA) & df_in["trip_start_timestamp"].between(s, e, inclusive="both")
        part = df_in.loc[m].copy()
        if not part.empty:
            part["segment"] = "departures_post"
            part["segment_start_local"] = s
            part["segment_end_local"] = e
            frames.append(part)
    return pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()
```

### Build segments for each event & combine

This step takes the cleaned rideshare dataset and slices it into **event windows** and **baseline windows** for each match. The segmentation helps us isolate arrivals and departures linked to the game, and compare them against “normal” activity on nearby weeks.

#### Process
- **Iterate over events**: Each event in the `EVENTS` list (e.g., Fire vs Inter Miami, Mexico vs Bolivia) is processed separately.  
- **Arrivals (event)**: Trips that **end** in Soldier Field (CA 33) within the pre-match arrival window.  
- **Arrivals (baseline)**: Trips that end in the same area/time slots on ±1…4 weeks around the event date.  
- **Departures (event)**: Trips that **start** in Soldier Field (CA 33) within the post-match departure window.  
- **Departures (baseline)**: Trips that start in the same area/time slots on ±1…4 weeks around the event date.  

#### Outputs
- **`df_seg_events`** → All event-day arrivals and departures (tagged by event name and window type).  
- **`df_seg_baseline`** → All baseline arrivals and departures across comparison windows.  
- **Row counts** are printed for transparency (e.g., how many arrivals vs. departures captured per event).  

This segmentation sets the stage for calculating demand lifts, fare surges, and congestion metrics by directly comparing event-day trip patterns with matched baseline periods.

```python
seg_event_frames, seg_baseline_frames = [], []

for ev in EVENTS:
    print(f"\n--- Segments for {ev['name']} ---")
    dfe_arr = slice_arrivals_event(df, ev["kick"])
    if not dfe_arr.empty:
        dfe_arr["event_name"] = ev["name"]
        dfe_arr["window_type"] = "event"
        seg_event_frames.append(dfe_arr)
        print(f"  Arrivals (event) rows: {len(dfe_arr):,}")
    dfb_arr = slice_arrivals_baselines(df, ev["kick"], weeks=4)
    if not dfb_arr.empty:
        dfb_arr["event_name"] = ev["name"]
        dfb_arr["window_type"] = "baseline"
        seg_baseline_frames.append(dfb_arr)
        print(f"  Arrivals (baseline) rows: {len(dfb_arr):,}")

    dfe_dep = slice_departures_event(df, ev["kick"])
    if not dfe_dep.empty:
        dfe_dep["event_name"] = ev["name"]
        dfe_dep["window_type"] = "event"
        seg_event_frames.append(dfe_dep)
        print(f"  Departures (event) rows: {len(dfe_dep):,}")
    dfb_dep = slice_departures_baselines(df, ev["kick"], weeks=4)
    if not dfb_dep.empty:
        dfb_dep["event_name"] = ev["name"]
        dfb_dep["window_type"] = "baseline"
        seg_baseline_frames.append(dfb_dep)
        print(f"  Departures (baseline) rows: {len(dfb_dep):,}")

df_seg_events   = pd.concat(seg_event_frames, ignore_index=True) if seg_event_frames else pd.DataFrame()
df_seg_baseline = pd.concat(seg_baseline_frames, ignore_index=True) if seg_baseline_frames else pd.DataFrame()

print(f"\nSegment rows → events: {len(df_seg_events):,} | baselines: {len(df_seg_baseline):,}")
```
### Segment Summaries

This section aggregates **event vs. baseline statistics** for arrivals and departures around each match.

#### What it does
1. **Summarize events and baselines**  
   - Groups trips by `event_name` and `segment` (`arrivals_pre`, `departures_post`).  
   - Computes median fare per mile, 75th percentile fare per mile, median speed (mph), and 25th percentile speed.  
   - Labels whether the data came from an *event window* or a *baseline window*.

2. **Merge event vs. baseline**  
   - Combines event stats with baseline stats for direct comparison.  
   - Ensures each segment (arrivals or departures) is aligned with its baseline.

3. **Baseline window sizes**  
   - Counts how many trips occurred in each baseline window.  
   - Produces the **average** and **median** baseline window size to avoid distortions from very small samples.

4. **Key indices created**
   - `trips_multiplier`: Event trips ÷ Average baseline trips  
   - `trip_lift_pct_vs_avg_window`: % lift in trips vs. baseline  
   - `surge_index_fare_per_mile_med`: Fare-per-mile surge (event ÷ baseline)  
   - `mph_change_med`: Change in median trip speed (event – baseline)  
   - `trips_multiplier_vs_median_window`: Event trips ÷ Median baseline trips  
   - `trip_lift_pct_vs_median_window`: % lift using median baseline instead of average

#### Why it matters
These metrics let us see:
- **Demand lift**: How much more (or less) rideshare demand existed during the event window compared to normal weeks.  
- **Price pressure**: Whether fares per mile surged during events.  
- **Traffic effects**: Whether median speeds dropped, indicating congestion.  

This gives a **quantitative basis** to evaluate how large-scale events (like soccer matches at Soldier Field) affect rideshare activity.

```python
# Segment summaries (keep which_event / which_baseline)
def summarize_seg(df_in: pd.DataFrame, label: str) -> pd.DataFrame:
    if df_in.empty:
        return pd.DataFrame(columns=[
            "event_name","segment","rows","med_fare_per_mile","p75_fare_per_mile","med_mph","p25_mph","which"
        ])
    g = (df_in.groupby(["event_name","segment"], as_index=False)
               .agg(rows=("trip_id","count"),
                    med_fare_per_mile=("fare_per_mile","median"),
                    p75_fare_per_mile=("fare_per_mile", lambda x: x.quantile(0.75)),
                    med_mph=("mph","median"),
                    p25_mph=("mph", lambda x: x.quantile(0.25))))
    g["which"] = label
    return g

ev_seg = summarize_seg(df_seg_events, "event")
bl_seg = summarize_seg(df_seg_baseline, "baseline")

summary_seg = ev_seg.merge(
    bl_seg,
    on=["event_name","segment"],
    suffixes=("_event","_baseline"),
    how="left"
)

# Baseline window sizes per segment (avoid tiny multipliers)
bl_win_seg = (
    df_seg_baseline.groupby(["event_name","segment","segment_start_local"], as_index=False)
                   .agg(rows_baseline_win=("trip_id","count"))
)
bl_agg_seg = (
    bl_win_seg.groupby(["event_name","segment"], as_index=False)
              .agg(rows_baseline_avg_win=("rows_baseline_win","mean"),
                   rows_baseline_median_win=("rows_baseline_win","median"),
                   n_baseline_windows=("rows_baseline_win","size"))
)
summary_seg = summary_seg.merge(bl_agg_seg, on=["event_name","segment"], how="left")

# Indices: multiplier vs AVG baseline window + percent lift
summary_seg["trips_multiplier"] = summary_seg["rows_event"] / summary_seg["rows_baseline_avg_win"]
summary_seg["trip_lift_pct_vs_avg_window"] = (summary_seg["trips_multiplier"] - 1.0) * 100.0

# Price & speed indices
summary_seg["surge_index_fare_per_mile_med"] = (
    summary_seg["med_fare_per_mile_event"] / summary_seg["med_fare_per_mile_baseline"]
)
summary_seg["mph_change_med"] = summary_seg["med_mph_event"] - summary_seg["med_mph_baseline"]

# Optional robust (median-window) versions
summary_seg["trips_multiplier_vs_median_window"] = summary_seg["rows_event"] / summary_seg["rows_baseline_median_win"]
summary_seg["trip_lift_pct_vs_median_window"] = (summary_seg["trips_multiplier_vs_median_window"] - 1.0) * 100.0

print("\n Segment summary (arrivals_pre / departures_post)")
pd.set_option("display.max_columns", None)
display(summary_seg)
```
### Findings (Segment Summary)
---
#### 1. Mexico v Bolivia (May 31, 2024, Soldier Field)
- **Arrivals (before match):**
  - Trips increased by **~42% vs baseline (avg window)**  
  - Median fare per mile was **~12% higher** than usual  
  - Median speed dropped by **~3 mph**, signaling heavier congestion  
- **Departures (after match):**
  - Trips spiked by **~51% vs baseline (avg window)**  
  - Median fare per mile was **~21% higher**  
  - Travel speeds again fell **~3.2 mph**, reinforcing congestion after the game  

*Interpretation:* This game produced strong lifts both before and after the match. Departures were especially intense (**50%+ demand lift**).

---

#### 2. Fire v Inter Miami (Aug 31, 2024, Soldier Field)
- **Arrivals (before match):**
  - Trips rose by **~47% vs baseline (avg window)**  
  - Median fare per mile rose **~9%**, a modest surge compared to Mexico v Bolivia  
  - Speeds dropped by **~2.3 mph**, showing localized congestion  
- **Departures (after match):**
  - Trips increased by **~41% vs baseline (avg window)**  
  - Median fare per mile rose **~24%**, indicating stronger price pressure than for arrivals  
  - Speeds dropped by **~3.1 mph**, again showing congestion as fans left together  

 *Interpretation:* The Inter Miami match produced balanced surges, but **price pressure was sharper during departures**.

---

### Key Takeaways
- **Demand lift:** Both matches saw **40–50%+ more trips than baseline**, especially before and after the games  
- **Price surge:** Price-per-mile increased across the board, with the **highest spikes (20–24%) during post-match departures**  
- **Congestion:** Median speeds dropped consistently (**~2–3 mph**) during fan movement periods  
- **Event dynamics:**  
  - *Mexico v Bolivia* → Stronger **departure demand**  
  - *Inter Miami* → More **balanced demand**, but **higher price pressure** post-match

  ### Visualizations

To better understand demand and price surges, we built a series of plots that show event vs. baseline rideshare activity around Soldier Field.  

#### 1. Timestamp Curves (Trips per 15 Minutes)

We group all trips into **15-minute bins** and compare:
- **Event windows** (arrivals before kickoff, departures after the final whistle)  
- **Baseline windows** (same time periods on non-event days, averaged over 4 weeks)  

Each chart shows:
- **Blue solid line** → Event trips  
- **Red dashed line** → Baseline average  
- **Vertical dashed line** → Kickoff (arrivals) or Final Whistle (departures)  
- **Y-axis** → Number of trips  
- **X-axis** → Local time (America/Chicago), labeled every 15 minutes  

```python
# Example plotting snippet
fig, ax = plt.subplots(figsize=(10,5))
ax.plot(ev_curve["ts"], ev_curve["trips"], label="Event", color=event_color, linewidth=2.5)
ax.plot(bl_curve["ts"], bl_curve["trips_avg"], label="Baseline avg", color=baseline_color, linestyle="--", linewidth=2)
ax.axvline(kickoff.floor("15min"), linestyle="--", color=marker_color, linewidth=1.5, label="Kickoff")
stylize_time_axis(ax, title="Arrivals — Mexico v Bolivia", xlab="Local Time", ylab="Trips per 15 min")
ax.legend(frameon=True, fancybox=True, shadow=True, loc="upper left")
plt.tight_layout()
plt.show()
```
#### Trip Lift Bar Chart

We measure **trip lift** as the percentage increase in rides compared to the average baseline window.  
This visualization highlights how demand before and after matches surged above normal levels.  

#### Code Example

```python
# Trip lift % bar chart (thinner bars, custom labels)
if not summary_seg.empty:
    # Custom labels for (event_name, segment)
    custom_labels = {
        ("2024-08-31 Fire v Inter Miami (Soldier Field)", "arrivals_pre"): "Inter Miami — Arrivals",
        ("2024-08-31 Fire v Inter Miami (Soldier Field)", "departures_post"): "Inter Miami — Departures",
        ("2024-05-31 Mexico v Bolivia (Soldier Field)", "arrivals_pre"): "Mexico v Bolivia — Arrivals",
        ("2024-05-31 Mexico v Bolivia (Soldier Field)", "departures_post"): "Mexico v Bolivia — Departures",
    }

    # Build plot dataframe
    plot_df = summary_seg[["event_name","segment","trip_lift_pct_vs_avg_window"]].copy()
    plot_df["label"] = plot_df.apply(
        lambda row: custom_labels.get((row["event_name"], row["segment"])), axis=1
    )

    # Plot
    x = np.arange(len(plot_df))
    y = plot_df["trip_lift_pct_vs_avg_window"].values

    fig, ax = plt.subplots(figsize=(8,7))
    bars = ax.bar(x, y, width=0.3, color=event_color, alpha=0.85, edgecolor="black")

    ax.set_xticks(x)
    ax.set_xticklabels(plot_df["label"], rotation=45, ha="right")
    ax.set_ylabel("Trip Lift (%)")
    ax.set_title("Event Demand Lift at Soldier Field", fontsize=14, weight="bold")
    ax.axhline(0, color="black", linewidth=0.8)

    plt.tight_layout()
    plt.show()
```

