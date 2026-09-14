# Cookie Cats A/B Test Analysis

A full statistical analysis of a real mobile-game A/B test — built with PostgreSQL, pandas, scipy/statsmodels, and Tableau Public.

## Objective

Cookie Cats is a mobile puzzle game with "gates" — walls that pause a player's progress at a certain level until they wait out a timer or pay to skip it. The game's team ran an experiment moving the first gate from **level 30** to **level 40**, to see whether delaying it changes how players behave.

This project answers one question with actual statistical rigor rather than eyeballing percentages: **should the gate be moved to level 40, or stay at level 30?**

## Metrics — and why these specifically

- **Day-1 retention** (`retention_1`): did the player come back the day after installing? This is the standard early-engagement metric in mobile games — it's the first signal of whether a change hurts or helps the core experience.
- **Day-7 retention** (`retention_7`): did the player come back a week later? This matters more than Day-1 for judging *durable* impact — a change can look fine on Day 1 and still quietly drive players away over the following week, so both are needed together.
- **Rounds played** (`sum_gamerounds`): a proxy for engagement/session depth, not just whether they returned but how much they played. Included because retention alone doesn't capture whether players who *did* come back are actually more or less engaged.

These three together cover both "did we keep the player" and "how invested is the player" — a single metric on its own can be misleading (e.g. retention could hold steady while engagement quietly drops).

## Key Findings

| Metric | gate_30 | gate_40 | Test used | p-value | Significant? |
|---|---|---|---|---|---|
| Day-1 retention | 41.51% | 40.95% | Two-proportion z-test | 0.103 | No |
| Day-7 retention | 14.61% | 13.79% | Two-proportion z-test | 0.0006 | **Yes** |
| Avg rounds played | 30.67 | 30.71 | Mann-Whitney U | 0.056 | No (borderline) |

**Final Recommendation: keep the gate at level 30.** Day-1 retention and engagement show no significant difference either way. Day-7 retention — the stronger signal of long-term impact — is significantly better with the gate at 30. No metric favors moving it to 40, so there's no case for the change.

Full reasoning, including why each statistical test was chosen and how the data was cleaned, is in `PROJECT_WRITEUP.md`.

## Pipeline

```
Kaggle CSV  →  pandas (clean: dedupe, outlier removal)  →  PostgreSQL on Neon (real SQL storage/queries)
           →  pandas (metrics + significance testing)   →  Tableau Public (dashboard)
```

Data was loaded into a live PostgreSQL database (hosted on Neon) rather than kept as a local file the whole time — this mirrors how analysis actually works at a company, where raw data lands in a warehouse and gets queried out, rather than living forever in someone's CSV.

## Files in this repo

- **`cookie_cats.csv`** — the raw dataset as downloaded from Kaggle: one row per player (`userid`), their assigned variant (`version`: gate_30/gate_40), rounds played, and retention flags. Unmodified.
- **`variant_summary.csv`** — the cleaned, aggregated output of the analysis: one row per variant with user counts, retention rates, and average/median rounds played. This is what feeds the Tableau dashboard.
- **`cookie_cats_analysis.ipynb`** — the full analysis notebook, run in Google Colab. Contains, in order: loading the CSV, pushing it to PostgreSQL, SQL queries (row counts, duplicate check), data cleaning (dedup + IQR-based outlier removal), the SRM (Sample Ratio Mismatch) check, metric computation by variant, the significance tests (two-proportion z-test for retention, Mann-Whitney U for rounds played), and a retrospective statistical power / minimum-detectable-effect calculation.

## Tools Used

- **PostgreSQL** (hosted on Neon, free tier) — real SQL storage and querying
- **pandas** — data cleaning and aggregation
- **scipy / statsmodels** — significance testing: two-proportion z-test with Wilson confidence intervals, Mann-Whitney U test, retrospective power analysis
- **Tableau Public** — the stakeholder-facing dashboard

## Statistical Methods Used

- **Sample Ratio Mismatch (SRM) check** via chi-square goodness-of-fit — confirms the 50/50 assignment split wasn't meaningfully broken before trusting any other result
- **Two-proportion z-test + Wilson confidence intervals** — for the retention metrics (binary outcomes aggregated to rates)
- **Mann-Whitney U test** — for rounds played, since the distribution is right-skewed (confirmed via skewness check) and a rank-based test doesn't assume normality the way a t-test does
- **Retrospective power / Minimum Detectable Effect (MDE)** — quantifies the smallest true effect the sample size could reliably have detected, so a "not significant" result can be read correctly (true null vs. just underpowered)

## Tableau Dashboard

The dashboard connects live to the Neon PostgreSQL database and includes: a variant comparison view (retention and engagement side by side), the lift on Day-7 retention with its confidence interval, and a text panel stating the final recommendation with the supporting reasoning. *.
