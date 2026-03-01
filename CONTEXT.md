# Bond Yield Rich/Cheap Model — Project Context

## What This Project Does
A Streamlit web app that uses **rolling multiple linear regression (OLS)** to determine
whether government bond yields (US 10Y and AU 10Y) are **RICH** (too low vs fundamentals),
**CHEAP** (too high), or **NEUTRAL**. The signal is derived from the z-score of
regression residuals (actual yield minus model-predicted yield).

## Deployment
- **Platform:** Streamlit Cloud (share.streamlit.io)
- **Entry point:** `app.py`
- **Status:** Initial build — March 2026

## File Structure
| File | Purpose |
|------|---------|
| `app.py` | Entire application — data loading, feature engineering, regression, charts, UI |
| `requirements.txt` | Python dependencies |
| `README.md` | Deployment instructions and model overview |
| `CONTEXT.md` | This file — project background for AI-assisted development |

> Note: This is a **single-file app**. All logic lives in `app.py`.

---

## How the Model Works

### Input Data (Two-Tab Excel File)
User uploads a single `.xlsx` file with **two tabs**:
- **`US Model field and data`** — US 10Y yield + US-relevant independent variables
- **`AU Model field and data`** — AU 10Y yield + AU-relevant independent variables

Each tab has the same layout:
- **Cell A1:** Bloomberg reference date (ignored by model)
- **Row 1 (B1 onwards):** Column headers — must include a Date column and a yield column
- **Row 2:** Feature flags for each independent variable
- **Row 3+:** Monthly data

The user toggles between US and AU models via a sidebar radio button. Each country
has its own independent variable set with its own flags — no flag conflicts.

### Auto-Detection
- **Date column:** Matched by name alias ("date") or by detecting datetime-parseable data
- **US 10Y Yield column:** Matched against aliases: "us 10y yield", "us 10yr yield", etc.
- **AU 10Y Yield column:** Matched against aliases: "au 10y yield", "au 10yr yield", etc.
- **All other columns:** Treated as independent variables, governed by their Row 2 flag

### Feature Flags (Row 2)
| Flag | Features Generated |
|------|--------------------|
| `L`  | Level only |
| `LM` | Level + Month-over-Month change |
| `LMP`| Level + MoM change + Percentage change |
| `MP` | MoM change + Percentage change |
| `P`  | Percentage change only |
| `N`  | Exclude from model entirely |

- Date and yield columns should have **blank** flags (they are auto-detected)
- If a flag is missing or unrecognised, it defaults to `LM`

### Regression Methodology
- **Rolling OLS** via `sklearn.LinearRegression`
- **Lookback window:** User-selectable — 36, 48, or 60 months (default: 60)
- At each month `i`, the model is trained on the prior `lookback` months, then used to predict yield at month `i`
- Residual = Actual Yield − Predicted Yield
- Z-score = residual normalised over a **rolling window** of residuals (same lookback period as the regression — consistent normalisation)
- For variables with competing MoM and % change features (LMP, MP flags), the model auto-selects per rolling window based on highest absolute correlation

### Signal Generation (Symmetric Thresholds)
- **CHEAP** signal: z-score > threshold (default **+1.5σ**) → yields higher than fair value → bonds cheap → BUY
- **RICH** signal: z-score < −threshold (default **−1.5σ**) → yields lower than fair value → bonds rich → SELL/HEDGE
- **NEUTRAL** otherwise
- Single symmetric threshold, adjustable via sidebar slider (0.5σ to 3.0σ)

---

## Suggested Variable Sets

### US 10Y Model
| Variable | Flag | Rationale |
|----------|------|-----------|
| Fed Funds Rate | L | Policy rate anchor; mean-reverting |
| US Breakeven 10Y | LM | Market-implied inflation expectations |
| ISM Manufacturing | L | Activity indicator; mean-reverting |
| ISM Services | L | Services economy activity |
| SPX | MP | Risk appetite; trending so use changes |
| Copper | MP | Global growth proxy; trending |
| German 10Y Yield | LM | Global rates anchor |
| DXY | LM | Dollar strength / capital flow signal |
| US Unemployment Rate | L | Labour market slack; mean-reverting |

### AU 10Y Model
| Variable | Flag | Rationale |
|----------|------|-----------|
| US 10Y Yield | LM | Critical cross-country transmission channel |
| RBA Cash Rate | L | Domestic policy rate anchor |
| AU Breakeven 10Y | LM | Domestic inflation expectations |
| NAB Business Confidence | L | Activity indicator (long history, monthly) |
| ASX 200 | MP | Domestic risk appetite |
| Iron Ore | MP | Key Australian terms-of-trade driver |
| AUD/USD | LM | Foreign demand signal for AU bonds |
| Copper | MP | Global growth proxy |
| AU Unemployment Rate | L | Domestic labour market |

---

## App Sections (UI Layout)

1. **Sidebar** — File upload, country toggle (US/AU), model settings (lookback, threshold), detected variable summary, coefficients
2. **Current Signal Box** — Large colour-coded CHEAP/RICH/NEUTRAL banner with date and z-score
3. **Key Metrics Row** — Actual yield, predicted yield, residual, R², z-score, signal, OOS R², Adj. R²
4. **Interactive Charts (4 tabs)**
   - Actual vs Predicted yield
   - Residuals over time (with time-varying threshold bands)
   - Z-Score history with threshold bands
   - Factor coefficients (bar chart)
5. **Commentary & Attribution** — Automated narrative showing:
   - Net model fair-value yield move (bps) — fully reconciled to actual predicted change
   - Top 2–3 ranked drivers with signed bps contributions
   - Model Recalibration line (coefficient drift + intercept change)
   - Current signal status, Z-Score, and R²
   - Expandable detail table filtered to factors with ≥ 0.5 bps absolute impact
6. **Regression Statistics** (expandable) — Coefficients, std errors, t-stats, p-values
7. **Historical Signals Table** (expandable) — Full history, colour-coded by signal
8. **Excel Download** — Multi-sheet report

---

## Architecture Notes

### Key Design Decisions
| Decision | Rationale |
|----------|-----------|
| Two-tab Excel input | Each country has independent variable sets with own flags — no conflicts |
| Single `app.py` file | Simplicity for Streamlit Cloud deployment |
| sklearn OLS (not statsmodels) | Lighter dependency; t-stats/p-values calculated manually via QR decomposition |
| Symmetric z-score thresholds | Unlike HY spreads, no carry asymmetry in government bonds |
| Rolling z-score normalisation | Consistent with rolling regression window — avoids stale history inflating std |
| Time-varying threshold bands on residuals chart | Reflects the rolling normalisation visually |
| Fully reconciling attribution | Uses prior-period coefficients for feature effects; Model Recalibration line absorbs coefficient drift + intercept change |
| Per-window feature auto-selection | For competing MoM vs % change pairs, selects highest correlation per window |
| Cell A1 Bloomberg date ignored | load_data skips non-matching columns including A1 date |

### Data Flow
1. `load_data()` → reads correct tab based on country toggle, auto-detects Date + Yield, parses flags
2. `engineer_features()` → generates derived columns dynamically based on flags
3. `run_regression()` → rolling OLS with per-window feature selection
4. `calculate_signals()` → rolling z-score normalisation matching the regression lookback
5. `monthly_attribution()` → fully reconciling decomposition of predicted yield change
6. All downstream (charts, Excel export) receive feature_cols/labels as parameters

### Signal Colour Convention (Bonds)
- **CHEAP (red background):** Yields too high → bonds undervalued → sell-off environment → buying opportunity
- **RICH (green background):** Yields too low → bonds overvalued → rally environment → selling opportunity
- This is inverted from credit spreads where cheap = green (buy) and rich = red (sell)

---

## Relationship to HY Spread Model
This app shares the same core architecture as the US HY Spread Rich/Cheap Model
(separate repo) but with key differences:
- Two-tab input structure (vs single sheet)
- Country toggle (vs single market)
- Symmetric thresholds (vs asymmetric for carry reasons)
- Inverted signal colour scheme (bonds vs credit)
- Rolling z-score normalisation (retrofitted to HY model separately)
- No HYG-specific scaling logic

---

## Dependencies
```
streamlit>=1.30.0
pandas>=2.0.0
numpy>=1.24.0
scikit-learn>=1.3.0
scipy>=1.11.0
plotly>=5.18.0
openpyxl>=3.1.0
```

---

## Known Limitations / Open Issues
- No backtesting / signal performance analytics
- No scenario analysis ("what if yields spike to X?")
- No CSV support (Excel only)
- AU Composite PMI dropped — insufficient history (only from 2023). NAB Business Confidence used instead
- Percentage change can produce inf values if a variable has zero values (handled by dropping)

---

## Planned Enhancements (Wishlist)
- [ ] Add backtesting tab showing historical signal performance
- [ ] Add scenario analysis
- [ ] Add email/alert functionality when signal changes
- [ ] Support CSV uploads
- [ ] Add yield curve model (2s10s as dependent variable)

---

## Handover Prompt Template
Use this when starting a new Claude session to work on this project:

> *"I have a working Bond Yield Rich/Cheap model deployed on Streamlit. The attached
> files are the current production code (`app.py`, `requirements.txt`). Please also
> read `CONTEXT.md` for full project background. The app uses rolling OLS regression
> with dynamic variable intake to generate RICH/CHEAP/NEUTRAL signals on US and AU
> 10Y government bond yields. The Excel input has two tabs — one per country — each
> with its own variable set and flags. It is currently working and deployed. I want to
> enhance it by [DESCRIBE YOUR ENHANCEMENT]. Please review the code and CONTEXT.md
> first, confirm your understanding of the architecture, and then we can proceed."*

---

*Last updated: March 2026 — initial build with rolling z-score, reconciling attribution, two-tab input*
