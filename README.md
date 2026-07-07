# Incident Hypertension: A Longitudinal Cohort Analysis

## Overview

This project estimates the incidence of hypertension in a longitudinally followed cohort, and checks whether age, sex, or BMI predict who develops it. The data is set up like a real cohort study: a baseline enrollment record per participant, a series of follow-up visits with repeated blood pressure readings and medication use, and an exit record for death or study withdrawal. It's simulated data, but it's messy in the way real EHR/claims data is messy (inconsistent coding, implausible values, dates that need reconciling across tables), so most of the work here is in the cleaning and defining the outcome, not just fitting the model at the end.

**[Rendered report (HTML)](./Vidit_WashU_StatDataAnalyst_LDA_Ex.html)** | **[R Markdown source](./Vidit_WashU_StatDataAnalyst_LDA_Ex.Rmd)**

## What's in the analysis

- **Cleaning across three linked tables.** Sex is recorded inconsistently (`1`/`M`/`male`/`Male`), so that gets standardized. Dates are parsed and cross-checked between the baseline and visit tables. Blood pressure readings below a plausible threshold are set to missing.
- **Building the medication/outcome variable.** Antihypertensive use is flagged from a set of indicator columns plus a free-text field, with explicit rules for which medications count (statins, antidiabetics, and anticoagulants are excluded on purpose), plus checks for false positives/negatives in that flagging logic.
- **Two incidence definitions, compared.** Part A counts a single elevated reading or medication start as the event (clinic visits only). Part B requires two separate occurrences (all visit types included), which is a stricter, more sustained definition. Person-years are calculated per participant from entry to the qualifying event or censoring (last visit/study exit).
- **Poisson regression for rate ratios.** Age, sex, and BMI category (derived from baseline height/weight) are modeled univariably and together, with a log person-time offset and a check for overdispersion.

## Key findings

- Incidence was higher under the single-occurrence definition (Part A: 208.5 per 1,000 person-years) than the stricter two-occurrence definition (Part B: 103.8 per 1,000 person-years). Makes sense, since Part B requires sustained rather than transient elevation.
- Sex differences were small in both parts (about 2.6 per 1,000 PY higher in females in Part A, under 1 per 1,000 PY higher in males in Part B). The youngest age band (18-29) didn't behave consistently across definitions: highest incidence in Part A, lowest in Part B, while the other age groups were fairly similar.
- None of age, sex, or BMI category came out as statistically significant predictors of incident hypertension, in either the univariable or multivariable Poisson models.

## Methods at a glance

| Step | Approach |
|---|---|
| Data cleaning | Standardize categorical coding, parse and reconcile dates, flag implausible BP readings |
| Medication classification | Rule-based flagging of antihypertensive use from structured + free-text fields, with QC checks |
| Cohort definition | Restrict to participants free of hypertension/medication at their first visit |
| Outcome definition | Elevated BP (SBP ≥130 or DBP ≥80) or antihypertensive medication use, single vs. repeated occurrence |
| Person-time calculation | Time from entry to first/second qualifying event, or censoring at last visit/exit |
| Modeling | Poisson regression with log(person-years) offset; incidence rate ratios with 95% CIs |

## Modeling notes

Mild overdispersion (~1.3) showed up in all four Poisson models but wasn't corrected for, and that was a deliberate stopping point, not an oversight. Quasi-Poisson only rescales the standard errors (by roughly √1.3, about 1.14x), it doesn't change the point estimates, so it can only widen the confidence intervals around the same estimates, never narrow them. Since every interval already crossed the null under the tighter, more liberal Poisson standard errors, correcting for the overdispersion could only make the null findings more robust, not less. So refitting with quasi-Poisson wasn't necessary to know the conclusion would hold, and I stopped there rather than doing it just to have done it.

## Data quality notes

A couple of things I noticed but didn't fully resolve. Worth calling out rather than glossing over:

- **Outlier handling only goes one direction, and that's deliberate.** Very low readings (SBP ≤40, DBP ≤30) are set to missing, since a living person can't sustain a reading that low, so it's a safe exclusion. Extremely high readings (into the 200s-300s+ mmHg) are kept as recorded. Unlike the low end, these can't be ruled out the same way: severe hypertensive crises in that range are clinically documented, so a high reading could be real or could be a data/measurement error, and there's no way to tell which just from the number. Still a real limitation either way. A sensitivity analysis excluding or capping these values would be a reasonable next step.
- **A few records share the same participant, visit label, and exact date, but have conflicting values** (different BP readings, different medications flagged). This could be genuine duplicate entries, or it could be two real encounters that happened to land on the same day, and there's no way to tell which from the data alone. They aren't deduplicated before analysis. Worth reconciling or dropping as a robustness check either way.

In a live project, both of these are exactly the kind of thing I'd raise with whoever manages or generated the data before making a unilateral call on how to handle them, rather than resolve them alone and hope I guessed right.

## Repository structure

```
├── files/                                          # Input and derived datasets
│   ├── baseline_df.csv                             # Participant demographics at enrollment
│   ├── visits_df.csv                                # Repeated visit-level BP and medication data
│   ├── outcomes_df.csv                              # Exit/death records
│   └── full_data.csv                                # Cleaned, merged analytic dataset
├── Vidit_WashU_StatDataAnalyst_LDA_Ex.Rmd           # Full analysis source (R Markdown)
├── Vidit_WashU_StatDataAnalyst_LDA_Ex.html          # Rendered report with code, tables, and interpretation
└── Incident_hypertension.Rproj                      # RStudio project file
```

## Tools

R, tidyverse, janitor, kableExtra, R Markdown.

## Reproducing this analysis

Open `Incident_hypertension.Rproj` in RStudio and knit `Vidit_WashU_StatDataAnalyst_LDA_Ex.Rmd`. All file paths are relative (via the `here` package). 
