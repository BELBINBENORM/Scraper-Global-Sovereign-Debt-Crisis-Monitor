# Global Sovereign Debt Crisis Monitor (2000-Present)

A yearly country-level panel covering sovereign debt levels, fiscal health, and early-warning
crisis risk signals for ~180 countries from 2000 to the present. Updated automatically every
week via GitHub Actions.

---

## Files

| File | Description |
| ---- | ----------- |
| `sovereign_debt_panel.csv` | Main analysis file - all indicators merged, one row per country per year |
| `ml_features.csv` | ML-ready matrix with lags, rolling windows, YoY changes, and a prediction target |

---

## Data Sources

| Source | What it provides | Key needed |
| ------ | ---------------- | ---------- |
| IMF WEO DataMapper API | Gross debt % GDP, fiscal balance, GDP, population | None |
| World Bank Open Data | External debt, reserves, revenue, interest payments, metadata | None |
| FRED (St. Louis Fed) | US 10-year Treasury yield, US federal debt/GDP | Free |
| Manual reference | S&P sovereign ratings, default events, IMF program flags | N/A |

---

## Column Reference - sovereign_debt_panel.csv

### Identity

| Column | Type | Description |
| ------ | ---- | ----------- |
| year | int | Calendar year (2000-present) |
| country | text | Country name, standardised via World Bank |
| iso3_code | text | ISO 3166-1 alpha-3 code (USA, GBR, IND...) |
| region | text | World Bank region (South Asia, MENA, Sub-Saharan Africa...) |
| income_group | text | High / Upper-middle / Lower-middle / Low |

### Debt Burden

| Column | Type | Description | Source |
| ------ | ---- | ----------- | ------ |
| govt_debt_pct_gdp | float | General govt gross debt as % of GDP | IMF: GGXWDG_NGDP |
| govt_debt_usd_bn | float | Gross govt debt in USD billions | Derived: debt% x GDP |
| external_debt_usd_bn | float | External debt owed to foreign creditors, USD bn | WB: DT.DOD.DECT.CD |
| external_debt_pct_gdp | float | External debt as % of GDP | Derived |
| debt_per_capita_usd | float | Govt debt divided by population, USD | Derived |

### Fiscal Health

| Column | Type | Description | Source |
| ------ | ---- | ----------- | ------ |
| fiscal_balance_pct_gdp | float | Govt revenue minus spending as % of GDP (negative = deficit) | IMF: GGXCNL_NGDP |
| primary_balance_pct_gdp | float | Fiscal balance before interest payments | Derived: fiscal + interest |
| govt_revenue_pct_gdp | float | Total govt revenue excl. grants as % of GDP | WB: GC.REV.XGRT.GD.ZS |
| interest_payments_pct_gdp | float | Annual interest paid on debt as % of GDP | WB: GC.XPN.INTP.GD.ZS |
| interest_pct_revenue | float | Interest payments as % of revenue - debt-trap signal | Derived |
| foreign_reserves_usd_bn | float | Total foreign exchange reserves, USD bn | WB: FI.RES.TOTL.CD |
| reserves_to_external_debt | float | Reserves divided by external debt - solvency buffer | Derived |

### Risk Signals

| Column | Type | Description | Source |
| ------ | ---- | ----------- | ------ |
| sp_credit_rating | text | S&P sovereign rating (AAA to D) | Manual / S&P |
| sp_rating_numeric | int | Rating mapped to integers: AAA=21, AA+=20 ... D=0 | Mapped |
| imf_program_active | 0/1 | 1 = country is under an active IMF loan program | IMF MONA database |
| imf_loan_usd_bn | float | IMF disbursement amount in USD billions | IMF MONA database |
| sovereign_default_event | 0/1 | 1 = a default event occurred this year | Public records |
| debt_crisis_risk_score | float | Composite 0-100 risk score | Derived |

---

## Risk Score Methodology

`debt_crisis_risk_score` is a 0-100 composite weighted across three components:

- **40%** - Debt/GDP ratio, capped at 200% and normalised to 0-100
- **30%** - Interest payments as % of government revenue, capped at 50%
- **30%** - Inverse of the reserves-to-external-debt ratio (low reserves = high risk)

Missing values fall back to a neutral 50 so the score stays defined for all rows.
A score above 65 broadly corresponds to countries that have experienced recent debt distress.
Use it as one signal among many, not a verdict.

---

## ML Features (ml_features.csv)

Every numeric column in the panel gets seven derived features:

| Suffix | What it is |
| ------ | ---------- |
| `_lag1`, `_lag2` | Previous 1 and 2 year values |
| `_roll3_mean`, `_roll5_mean` | 3 and 5 year rolling average (computed on lagged values to avoid leakage) |
| `_roll3_std` | 3 year rolling standard deviation - volatility signal |
| `_yoy_change` | Absolute year-over-year change |
| `_yoy_pct` | Percentage year-over-year change, capped at +/-200% |

**Target column**: `target_risk_rise_next_yr`
- `1` = debt_crisis_risk_score increases by more than 5 points the following year
- `0` = it does not
- `NaN` = next year not yet available (most recent year for each country)

---

## How It Updates

A GitHub Actions workflow runs every Monday at 06:00 UTC. It:

1. Executes `notebook.ipynb` top to bottom via `jupyter nbconvert`
2. Verifies both CSVs were produced
3. Pushes a new version to Kaggle - or creates the dataset on the very first run

To trigger a manual update, go to the **Actions** tab and click **Run workflow**.

---

## Setup (first time only)

1. Fork or clone this repo
2. Add three GitHub secrets under **Settings > Secrets > Actions**:
   - `KAGGLE_USERNAME` - your Kaggle username
   - `KAGGLE_KEY` - your Kaggle API key (from kaggle.com/settings)
   - `FRED_API_KEY` - optional, free key from fred.stlouisfed.org (US data only)
3. Confirm the `id` field in `dataset-metadata.json` matches `your-username/your-dataset-slug`
4. Trigger the workflow once manually - it will create the dataset automatically

---

## License

CC0 1.0 Universal - public domain. Use it however you want.

IMF and World Bank data are published under their respective open data policies.
