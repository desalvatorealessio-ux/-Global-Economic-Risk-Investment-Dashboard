# 🌍 Global Economic Risk & Investment Dashboard

**Dataset:** [World Bank Open Data](https://data.worldbank.org/) via the `wbgapi` Python package (free, no auth) — GDP growth, inflation, unemployment, and FDI inflows for 10 countries (8 European + USA/China as global benchmarks), 2005–2023, 190 country-year rows, zero missing values

**Tools:** Python (pandas, numpy), SQLite (DB Browser for SQLite), Power BI

**Scope:** Descriptive analysis only — what happened, and how it breaks down across countries and time. No forecasting or predictive modeling; these findings are meant to support judgment, not replace it. True cross-border M&A data has no free API, so FDI inflows (% of GDP) are used as a legitimate proxy for cross-border investment activity.

📌 **[Jump straight to the Recommendations →](#-recommendations)**

## 📊 Dashboard Preview

<img width="900" alt="image" src="https://github.com/user-attachments/assets/d9f3f48d-4f4a-4ac9-94f4-6af82308fcf0" />


4-page interactive Power BI dashboard covering a global overview, the 2008 vs. 2020 crisis comparison, investment climate scoring, and economic stability/risk.

Full dashboard below.

## 🐍 Data Pipeline (Python)

This dataset isn't a static file — it's pulled live from the World Bank API. Two Python scripts handle this before any SQL runs:

- **[01_pull_worldbank_data.py](https://github.com/user-attachments/files/32263421/01_pull_worldbank_data.py)** — pulls all 4 indicators for the 10 countries and 19 years via `wbgapi`, then profiles the raw pull (shape, dtypes, missing values per country, summary stats) before any cleaning decision is made, and saves the untouched result to `worldbank_raw.csv`.
- **[02_clean_worldbank_data.py](https://github.com/user-attachments/files/32263439/02_clean_worldbank_data.py)** — reshapes the raw pull into analysis-ready columns, adds a `region` classification (Europe vs. Global Benchmark, via `np.where`), computes year-over-year change per indicator (`pandas` `.diff()`), and flags statistically unusual years with a `numpy`-based z-score computed **per country** (not per dataset) — since a "normal" unemployment rate for Spain isn't the same as a "normal" rate for Germany. Validates the result against known real-world history (Spain's 2013 unemployment peak near 26%, the universal 2020 GDP crash) before saving `worldbank_clean.csv`, which feeds directly into the SQL phase below.

## 📌 Solution

### A. Which economies grow fastest, and which are most consistent?

```sql
SELECT country_name, region,
       ROUND(AVG(gdp_growth_pct), 2) AS avg_gdp_growth,
       ROUND(SQRT(AVG(gdp_growth_pct*gdp_growth_pct) - AVG(gdp_growth_pct)*AVG(gdp_growth_pct)), 2) AS gdp_growth_volatility
FROM worldbank
GROUP BY country_name, region
ORDER BY avg_gdp_growth DESC;
```

**Answer:**

China leads growth by far (8.08% average) but is also the most volatile of the fast growers (volatility 2.88). Among European countries, Poland grows fastest (3.77%) with below-average volatility (2.32) — a notably better combination than Spain (1.24% growth, 3.93 volatility) or the UK (1.42% growth, 3.62 volatility), which grow slower *and* swing harder. Italy is both the slowest-growing country in the dataset (0.30%) and one of the more volatile (3.54) — the weakest growth/stability combination of the ten.

---

### B. How did each country's GDP growth compare in 2008, 2009, and 2020?

```sql
SELECT country_name,
       ROUND(MAX(CASE WHEN year = 2008 THEN gdp_growth_pct END), 2) AS gdp_growth_2008,
       ROUND(MAX(CASE WHEN year = 2009 THEN gdp_growth_pct END), 2) AS gdp_growth_2009,
       ROUND(MAX(CASE WHEN year = 2020 THEN gdp_growth_pct END), 2) AS gdp_growth_2020
FROM worldbank
GROUP BY country_name
ORDER BY gdp_growth_2020;
```

**Answer:**

<img width="600" alt="image" src="https://github.com/user-attachments/assets/6d1a447b-1cba-4f37-b84f-125333898935" />


2020 was a sharper shock than 2008 for every European country: Spain (-10.94%) and the UK (-10.05%) posted the deepest single-year contractions in the entire dataset — worse than any country's 2008 or 2009 figure. China is the outlier throughout both crises, staying positive in 2008 (+9.67%), 2009 (+9.41%), and 2020 (+2.34%) — the only country never to contract in either crisis year.

---

### C. Where and when did inflation spike hardest?

```sql
SELECT country_name, year, inflation_pct, inflation_pct_notable_year
FROM worldbank
WHERE year IN (2021, 2022)
ORDER BY inflation_pct DESC;
```

**Answer:**

<img width="600" alt="image" src="https://github.com/user-attachments/assets/26e35c47-adfe-4d1f-92b0-168535a89999" />

*(SQL result shown directly — this isn't a dedicated dashboard visual, but feeds into the "notable years" logic used in section F.)*

Every country in the dataset crossed its own statistical "notable year" threshold (\|z-score\| > 1.5) for inflation in 2022, and none did in 2021 — a clean signal that 2022, not 2021, is when the global inflation shock actually landed on this basket of economies. Poland's 2022 spike (14.43%) was nearly 1.5× the next-highest reading (Netherlands, 10.00%), consistent with Central/Eastern Europe's heavier exposure to the 2022 energy-price shock.

---

### D. Which countries attract the most FDI, and how stable is it?

```sql
SELECT country_name, region,
       ROUND(AVG(fdi_inflow_pct_gdp), 2) AS avg_fdi_pct_gdp,
       ROUND(SQRT(AVG(fdi_inflow_pct_gdp*fdi_inflow_pct_gdp) - AVG(fdi_inflow_pct_gdp)*AVG(fdi_inflow_pct_gdp)), 2) AS fdi_volatility
FROM worldbank
GROUP BY country_name, region
ORDER BY avg_fdi_pct_gdp DESC;
```

**Answer:**

<img width="1781" height="435" alt="image" src="https://github.com/user-attachments/assets/4fbf737d-b22b-4a9d-923d-e169c2e67974" />


The Netherlands is a clear statistical outlier: it attracts the most FDI by far (17.85% of GDP on average) but with volatility (29.72) several times higher than any other country's — a pattern consistent with its known role as a financial conduit economy, where large pass-through flows inflate both the average and the swings, rather than reflecting genuine investment attractiveness at that scale. Excluding the Netherlands, the UK, Poland, and Sweden attract the most FDI relative to GDP (3.6–3.8%) with comparatively modest volatility.

---

### E. How do GDP growth and unemployment move together across all 190 country-years?

```sql
SELECT country_name, year, gdp_growth_pct, unemployment_pct
FROM worldbank
ORDER BY country_name, year;
```

**Answer:**

<img width="600" alt="image" src="https://github.com/user-attachments/assets/2c8fde60-bc1b-4ff6-a8c5-fe4be07b023f" />


Most country-years cluster tightly — GDP growth between roughly 0% and 12%, unemployment under 10% — regardless of region. The pattern only breaks down in crisis years, most sharply for Spain in 2020 (-10.94% growth paired with 15.53% unemployment, the single most extreme point in the dataset). The Global Benchmark countries (orange) sit consistently toward the low-unemployment, positive-growth side of the cloud.

---

### F. Which countries have the most statistically "notable" years overall?

```sql
SELECT country_name,
       SUM(gdp_growth_pct_notable_year + inflation_pct_notable_year
           + unemployment_pct_notable_year + fdi_inflow_pct_gdp_notable_year) AS total_notable_years
FROM worldbank
GROUP BY country_name
ORDER BY total_notable_years DESC;
```

**Answer:**

<img width="600" alt="Total notable years by country" src="assets/chart_f_notable_years.png" />

China logs the most statistically unusual years across all four indicators combined (14 notable country-years out of a possible 76), followed by Italy (12). Spain has the fewest (8) despite having the worst Misery Index in the dataset — its swings are large in absolute terms but more evenly spread relative to its *own* historical baseline, which is what the z-score-based "notable year" flag actually measures.

---

### G. How does Europe compare to the Global Benchmark overall?

```sql
SELECT region,
       ROUND(AVG(gdp_growth_pct), 2)     AS avg_gdp_growth,
       ROUND(AVG(inflation_pct), 2)      AS avg_inflation,
       ROUND(AVG(unemployment_pct), 2)   AS avg_unemployment,
       ROUND(AVG(fdi_inflow_pct_gdp), 2) AS avg_fdi_pct_gdp
FROM worldbank
GROUP BY region;
```

**Answer:**

<img width="600" alt="Avg GDP growth by year and region" src="assets/chart_g_gdp_growth_by_region.png" />

The Global Benchmark (US + China) grows more than 3× faster than Europe on average (5.07% vs. 1.57%) and carries meaningfully lower unemployment (5.25% vs. 8.25%), while Europe attracts more than double the FDI relative to GDP (4.66% vs. 2.12%). Inflation is nearly identical between the two groups (2.46% vs. 2.25%) — the gap between these two blocs is about growth and jobs, not price stability.

---

### H. Which country has the worst — and best — Misery Index?

```sql
SELECT country_name, ROUND(AVG(inflation_pct + unemployment_pct), 2) AS avg_misery_index
FROM worldbank
GROUP BY country_name
ORDER BY avg_misery_index;
```

**Answer:**

<img width="600" alt="Misery Index by country" src="assets/chart_h_misery_index.png" />

Spain's average Misery Index (18.73) is 66% higher than the next-worst country, Italy (11.27), and more than 2.5× China's, the best in the dataset (6.96). Seven of the ten worst individual country-year Misery Index readings across the entire dataset belong to Spain, peaking at 27.5 in 2013 during its debt-crisis-era unemployment surge.

---

### I. Which country has the strongest Investment Climate in 2023?

```sql
WITH normalized AS (
    SELECT country_name, region, year,
        100.0 * (gdp_growth_pct - MIN(gdp_growth_pct) OVER (PARTITION BY year)) /
            NULLIF(MAX(gdp_growth_pct) OVER (PARTITION BY year) - MIN(gdp_growth_pct) OVER (PARTITION BY year), 0) AS growth_score,
        100.0 * (fdi_inflow_pct_gdp - MIN(fdi_inflow_pct_gdp) OVER (PARTITION BY year)) /
            NULLIF(MAX(fdi_inflow_pct_gdp) OVER (PARTITION BY year) - MIN(fdi_inflow_pct_gdp) OVER (PARTITION BY year), 0) AS fdi_score,
        100.0 - (100.0 * (inflation_pct - MIN(inflation_pct) OVER (PARTITION BY year)) /
            NULLIF(MAX(inflation_pct) OVER (PARTITION BY year) - MIN(inflation_pct) OVER (PARTITION BY year), 0)) AS inflation_score,
        100.0 - (100.0 * (unemployment_pct - MIN(unemployment_pct) OVER (PARTITION BY year)) /
            NULLIF(MAX(unemployment_pct) OVER (PARTITION BY year) - MIN(unemployment_pct) OVER (PARTITION BY year), 0)) AS unemployment_score
    FROM worldbank
)
SELECT country_name, region, year,
       ROUND((growth_score + fdi_score + inflation_score + unemployment_score) / 4.0, 1) AS investment_climate_score
FROM normalized
WHERE year = 2023
ORDER BY investment_climate_score DESC;
```

**Answer:**

<img width="600" alt="Investment Climate Score, 2023 ranking" src="assets/chart_i_investment_score_2023.png" />

<img width="600" alt="Investment Climate Score heatmap, 2005-2023" src="assets/chart_i_investment_heatmap.png" />

China's 2023 Investment Climate Score (91.9) is the highest of any country in any year in the dataset. The top-scoring European country, Germany (59.8), trails it by more than 30 points. This score is deliberately relative — each of the four indicators is rescaled 0–100 *within its own year* before averaging — so it reflects growth and FDI momentum specifically, not overall macro risk: China's own volatility figures elsewhere in this analysis are middling, not the safest in the set.

---

### J. How long did each country take to recover from the 2008 and 2020 crises?

```sql
WITH indexed AS (
    SELECT country_name, year, gdp_growth_pct,
           100.0 * EXP(SUM(LN(1 + gdp_growth_pct/100.0)) OVER (
               PARTITION BY country_name ORDER BY year
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           )) AS gdp_index
    FROM worldbank
),
crisis_2008 AS (
    SELECT country_name, gdp_index AS pre_crisis_level FROM indexed WHERE year = 2007
),
recovery_2008 AS (
    SELECT i.country_name, MIN(i.year) AS recovery_year
    FROM indexed i JOIN crisis_2008 c ON i.country_name = c.country_name
    WHERE i.year > 2008 AND i.gdp_index >= c.pre_crisis_level
    GROUP BY i.country_name
),
crisis_2020 AS (
    SELECT country_name, gdp_index AS pre_crisis_level FROM indexed WHERE year = 2019
),
recovery_2020 AS (
    SELECT i.country_name, MIN(i.year) AS recovery_year
    FROM indexed i JOIN crisis_2020 c ON i.country_name = c.country_name
    WHERE i.year > 2020 AND i.gdp_index >= c.pre_crisis_level
    GROUP BY i.country_name
)
SELECT
    r08.country_name,
    r08.recovery_year                 AS recovered_from_2008_by,
    (r08.recovery_year - 2008)        AS years_to_recover_2008,
    r20.recovery_year                 AS recovered_from_2020_by,
    (r20.recovery_year - 2020)        AS years_to_recover_2020
FROM recovery_2008 r08
LEFT JOIN recovery_2020 r20 ON r08.country_name = r20.country_name
ORDER BY years_to_recover_2008;
```

**Answer:**

<img width="600" alt="Crisis recovery speed by country" src="assets/chart_j_recovery_speed.png" />

Italy took 15 years to return to its pre-2008 GDP level — recovering only in 2023, the very last year in this dataset — nearly twice as long as the next-slowest country, Spain (8 years). By contrast, every one of the 10 countries recovered from the far deeper 2020 COVID shock within just 1–2 years, including Italy itself. The two crises tested fundamentally different things: being hit hard by 2008 didn't predict being slow to recover from 2020.

---
## 📊 Full Dashboard

**Page 1 — Overview**

<img width="700" alt="Page 1 - Overview" src="assets/page1_overview.png" />

**Page 2 — Crisis Impact: 2008 Financial Crisis vs. 2020 COVID Recession**

<img width="700" alt="Page 2 - Crisis Impact" src="assets/page2_crisis_impact.png" />

**Page 3 — Investment Climate: Who's Attracting Capital?**

<img width="700" alt="Page 3 - Investment Climate" src="assets/page3_investment_climate.png" />

**Page 4 — Stability & Risk: Which Economies Are Most Volatile?**

<img width="700" alt="Page 4 - Stability & Risk" src="assets/page4_stability_risk.png" />

---
## 💡 Recommendations

Four actions are directly supported by the findings above:

**1. Treat Italy as a structural-risk case, not just a slow-growth one.**
Italy is the only country in the dataset that took 15 years to recover its pre-2008 GDP level — effectively never fully recovering within this window before the next shock arrived. Combined with the lowest average growth rate (0.30%) in the sample, this points to a persistent structural drag rather than a one-off cyclical dip. Any exposure to Italian assets or operations should be sized with slow-recovery scenarios in mind, not average-case ones.

**2. Weight capital allocation toward growth momentum, but price in China's specific risk profile.**
China's 2023 Investment Climate Score (91.9) and average growth (8.08%) lead the dataset by a wide margin, and the score construction rewards exactly that momentum. But China is also among the most "notable-year"-prone economies (14 statistically unusual years, the most in the dataset) and posted meaningful GDP growth volatility (2.88). Momentum and stability are not the same thing here — allocation decisions should treat them as two separate inputs, not one score.

**3. Discount the Netherlands' FDI figures before using them as a benchmark.**
The Netherlands' FDI-to-GDP ratio (17.85% average) and volatility (29.72, several times any other country's) reflect its role as a financial conduit economy — large pass-through capital flows, not organic investment demand. Any cross-country FDI benchmarking exercise should either exclude the Netherlands or footnote it explicitly; including it unadjusted will distort European averages.

**4. Plan for uneven shock exposure, not uniform "crisis risk."**
2020 hit Spain and the UK hardest (-10.94% and -10.05%) but was recovered from fastest, universally, within 1–2 years. 2008 hit less dramatically in headline terms but left Italy and Spain with multi-year (8–15 year) scars. Contingency planning should distinguish between "sharp but short" shocks and "moderate but structural" ones — the same recovery playbook does not fit both, and the country hit hardest is not necessarily the country most at risk long-term.

**What this data can't justify:**
- **Causal claims about *why* Italy or Spain underperform** — this dataset shows the pattern (slow recovery, high Misery Index) but contains no policy, demographic, or institutional variables that would explain the mechanism behind it.
- **A single blended "risk score" across countries** — the Investment Climate Score (section I) is a relative growth/FDI momentum measure by construction, not a risk-adjusted score; treating a high score as "safe" would be a misuse of what the metric actually measures.
- **Forecasting the next crisis** — this is a descriptive, historical comparison of two specific past shocks (2008, 2020). It says nothing about the timing, size, or shape of any future shock.

## 📝 Notes on the data

- **FDI inflows are a proxy, not a direct measure of M&A activity.** No free, reliable API exists for cross-border M&A deal data at this scope; FDI inflows (% of GDP) are the standard World Bank proxy for cross-border investment activity and are used as such throughout.
- **The Investment Climate Score is relative-per-year by construction.** Each of the four underlying indicators is min-max normalized *within its own year* (comparing only the 10 countries against each other, that year) before being averaged — it answers "who's doing best this year, relative to peers," not "is this country improving over time" (which is what the separate z-score / notable-year columns are for).
- **A `NULL` 2020 recovery year would mean "not yet recovered as of 2023"** (the last year in this dataset), not a data error — as it happens, every country in this dataset did recover within the window, so no `NULL`s appear in the final result, but the query logic explicitly allows for that case.
- **SQLite has no built-in `STDDEV`, `LN`, or `EXP` in a plain build**, so volatility throughout this project is computed manually via `sqrt(avg(x²) − avg(x)²)`, and the compounded GDP index in section J uses the log/exp identity (`100 × EXP(SUM(LN(1 + growth/100)))`) to turn a running product into a running sum, since SQLite has no running-product window function.
