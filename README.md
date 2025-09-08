# Chicago-Rideshare-Event-Study
End-to-end pipeline to analyze event-time rideshare behavior in Chicago. Pulls City of Chicago TNP trips, slices Soldier Field windows, builds matched ±week baselines, and outputs fare-per-mile surge indexes.
## Contents
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
5. 
