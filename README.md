# Bond Yield Rich/Cheap Model

A Streamlit web application that uses **rolling multiple linear regression (OLS)** to determine whether US and Australian 10-year government bond yields are **RICH** (too low vs fundamentals), **CHEAP** (too high), or **NEUTRAL**.

## Quick Start

1. Upload your Excel file containing two tabs of monthly data
2. Select the country (US 10Y or AU 10Y) from the sidebar
3. The model runs automatically and displays the current signal

## How It Works

The model regresses bond yields against macro and market factors using a rolling window (default 60 months). The residual (actual yield minus predicted "fair value" yield) is normalised into a Z-Score. Yields significantly above fair value → **CHEAP** (bonds undervalued, buy signal). Yields significantly below fair value → **RICH** (bonds overvalued, sell/hedge signal).

## Input File Format

A single `.xlsx` file with **two tabs**:

- **`US Model field and data`** — US 10Y yield and its independent variables
- **`AU Model field and data`** — AU 10Y yield and its independent variables

Each tab has the same layout:

| Row | Content |
|-----|---------|
| Row 1 | Column headers (Cell A1 may contain a Bloomberg reference date — ignored) |
| Row 2 | Feature flags: `L`, `LM`, `LMP`, `MP`, `P`, or `N` (blank for Date and yield columns) |
| Row 3+ | Monthly data |

### Feature Flags

| Flag | Features Generated |
|------|--------------------|
| `L`  | Level only |
| `LM` | Level + Month-over-Month Change |
| `LMP`| Level + MoM Change + % Change |
| `MP` | MoM Change + % Change |
| `P`  | % Change only |
| `N`  | Exclude from model |

### Signal Interpretation (Bonds)

- **CHEAP:** Yields are *higher* than fair value → bonds are undervalued → buying opportunity
- **RICH:** Yields are *lower* than fair value → bonds are overvalued → sell/hedge
- **NEUTRAL:** Yields are near fair value

Uses **symmetric thresholds** (default 1.5σ, adjustable via slider).

## Deployment

Deployed on [Streamlit Cloud](https://share.streamlit.io). Push changes to `main` branch and Streamlit auto-deploys.

## Dependencies

See `requirements.txt`. All standard Python packages — no Bloomberg API dependency (data is uploaded via Excel).
