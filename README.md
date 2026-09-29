# Direct Indexing Email A/B Test Analysis

An end-to-end analysis of a randomized email experiment designed to drive financial advisors to enroll in Direct Indexing (DI). The project covers data extraction and cleaning (Python + SQLite), hypothesis testing (two-proportion z-tests, Welch's t-tests, chi-square), experiment-design diagnostics, Tableau visuals, and a stakeholder presentation.

> This was completed as a data analyst case study. The data is fictitious and was provided only for evaluating analytical skills.

## Business Question

Two email creatives (Email A and Email B) were sent to eligible advisors between **7/30/2020 and 8/29/2020**. The team's hypothesis was that the creative an advisor received makes no difference. This project tests whether the creative affects an advisor's likelihood to:

1. Access the DI sign-up process on the web (reach the `welcome` page)
2. Enroll in DI

It also reports how many advisors opened and clicked each email, and evaluates the quality of the experiment design itself.

## Key Findings

| Metric | Email A | Email B | Result |
|---|---|---|---|
| Advisors emailed | 15,499 | 21,799 | Not an exact 50/50 split |
| Opened | 1,280 (8.26%) | 1,856 (8.51%) | 3,136 total |
| Clicked | 60 (0.39%) | 89 (0.41%) | 149 total |
| Bounced | 8 | 23 | 31 total |
| Reached sign-up (`welcome`) | 65 (0.42%) | 92 (0.42%) | z = -0.04, p = 0.97, no difference |
| Enrolled in window | 1 (0.007%) | 9 (0.041%) | z = -2.03, p = 0.043, significant |

**Takeaways**

- **Access to sign-up:** no evidence the creative changes whether advisors start enrollment.
- **Enrollment:** Email B shows a statistically significant lift, but it rests on only 10 in-window enrollments, so it is directional rather than conclusive.
- **Randomization check failed on one field:** `call_ct` is almost perfectly separated between arms (Email A mean 0.06 vs. Email B mean 2.09, Welch's t = -599.9). Age, balance, tenure, fund count, logon count, and gender are balanced. The Email B enrollment result should be treated with caution until the call imbalance is explained.
- **Enrollment is a lagging outcome:** 117 of 143 enrollments (82%) occurred outside the one-month campaign window.
- **Tracking gap:** only 34 of the 143 enrolled advisors (24%) show any tracked web activity in the window, which suggests enrollment through untracked channels such as phone.
- **Baseline matters:** 16 of the 26 in-window enrollments came from advisors who received no email.

**Recommendation:** Email B is a better performer on the evidence available but not a confirmed winner. Re-run the comparison with a longer (90+ day) observation window, investigate the `call_ct` assignment imbalance, and look into enrollments with no tracked web visit.

## Approach

The notebook follows an Extract, Transform, Load, Analyze flow.

1. **Extract:** exploratory data analysis on three source files, plus join-integrity checks on `part_id`.
2. **Load:** raw extracts written as-is to a SQLite database (`demo_raw`, `enroll_raw`, `web_raw`).
3. **Transform:** cleaning done in SQL, written to `demo_clean`, `enroll_clean`, and `web_clean`.
4. **Analyze:** a per-advisor web-activity table, A/B tests, engagement funnel, and randomization diagnostics.
5. **Export:** a denormalized advisor-level table (`advisor_demographics_web`) exported to CSV for Tableau.

### Data cleaning highlights

| Dataset | Issue | Handling |
|---|---|---|
| demo | 3 negative `balance` values | Set to `NULL` |
| demo | Engagement dates on advisors who received no email, or whose email bounced | Cleared |
| demo | Click date with no open date | `open_dt` backfilled from `click_dt` |
| demo | Engagement dates outside the send window | Cleared |
| enroll | 21 fully duplicated rows (164 to 143 rows) | Removed with `SELECT DISTINCT` |
| enroll | Enrollments after the send window | Kept, with an `enrolled_in_window` flag added instead of deleting real outcomes |
| web | 90% exact-duplicate hits, 341 null pages, hits outside the window (77,304 to 3,510 rows) | Deduplicated, dropped, and filtered to the window |

### Statistical methods

- **Two-proportion z-test** with Wilson 95% confidence intervals for access and enrollment (Email A vs. Email B, excluding the no-email group since it was not part of the randomized assignment)
- **Welch's t-test** for continuous covariate balance
- **Chi-square test** for gender balance

## Repository Contents

| File | Description |
|---|---|
| `travis_ohm_case_study_analysis.ipynb` | Full analysis notebook (ETL, tests, diagnostics) |
| `travis_ohm_case_study_presentation.pptx` | Stakeholder presentation (10 to 15 minute readout) |
| `travis_ohm_case_study_analysis_visuals.twb` / `case_study_analysis_visuals.twbx` | Tableau workbook (packaged version includes the data extract) |
| `case_study_clean.db` | SQLite database with raw, cleaned, and analysis tables |
| `advisor_demographics_web.csv` | Advisor-level table used as the Tableau data source |
| `demo.xlsm`, `enroll_data.xlsm`, `web_data.xlsx` | Source data files read by the notebook |

**Tableau worksheets:** Email narrative %, Email outcome rates, Enrollment metrics, When advisors actually enroll?, call_ct %

**Database tables:** `demo_raw`, `enroll_raw`, `web_raw`, `demo_clean`, `enroll_clean`, `web_clean`, `advisor_web_summary`, `advisor_demographics_web`

## Data Dictionary

**demo:** `part_id`, `fund_ct`, `logon_ct` (12 months), `call_ct` (12 months), `balance` (AUM, thousands), `tenure`, `age`, `gender` (0 male, 1 female, 2 unknown), `campaign` (0 Email A, 1 Email B, 2 no email), `click_dt`, `bounce_dt`, `open_dt`

**enroll_data:** `part_id`, `curr_enrll_fl`, `unenrll_fl`, `curr_enrll_status`, `enrll_dt`, `unenrll_dt`

**web_data:** `part_id`, `web_session_id`, `hit_dt`, `page`

## Getting Started

```bash
pip install pandas numpy scipy statsmodels openpyxl jupyter
```

1. Place `demo.xlsm`, `enroll_data.xlsm`, and `web_data.xlsx` in the same directory as the notebook.
2. Run `travis_ohm_case_study_analysis.ipynb` top to bottom. It rebuilds `case_study_clean.db` and `advisor_demographics_web.csv`.
3. Open the `.twb` in Tableau and point the data source at `advisor_demographics_web.csv` if the path needs updating.

`sqlite3` is part of the Python standard library.

## Limitations and Next Steps

- Very low base rates (0.007% to 0.041% enrollment) leave the enrollment test with little statistical power.
- The one-month window captures only a small share of enrollments; a 90+ day window is needed.
- The assignment process behind the `call_ct` imbalance should be confirmed with the business.
- Web hits are dated, not timestamped, so session paths and funnel timing cannot be fully reconstructed.
- Phone, app, and call-center enrollments are not tracked, so those cannot be attributed to a creative.
- The content and call to action of each email were not provided.
- Suggested next experiment: run a power analysis up front, stratify or check balance at assignment, keep a true 50/50 split, and add a longer observation window.

## Tools

Python (pandas, NumPy, SciPy, statsmodels), SQL (SQLite), Tableau, PowerPoint
