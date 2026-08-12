# Unicorn Time to Scale

What predicts how fast a company reaches a one billion dollar valuation, and does the apparent acceleration over time survive scrutiny? This project analyzes 1,073 companies to answer that question, with a specific focus on testing whether the "unicorns are getting faster" trend is real or an artifact of how the data was collected.

This is a living document. Sections below are filled in as the project progresses.

## Data

Source: `data/raw/unicorn_companies.csv`, 1,073 companies at the raw stage.

Target variable: `years_to_unicorn`, computed as the year of `Date Joined` minus `Year Founded`.

## Data Cleaning

Every cleaning decision is documented with its rule and the row count it affected. The raw file is kept untouched in `data/raw/`, cleaned output goes to `data/processed/`.

| Decision | Rule | Rows affected |
|---|---|---|
| Negative durations | Dropped rows where `years_to_unicorn` was negative and unresolvable from available data | 1 (Yidian Zixun: `Year Founded` of 2021 postdates `Date Joined` of 2017, an impossible timeline with no independent source available to correct it) |
| Old company outliers | Excluded companies with `Year Founded` before 1990. A stated rule, not a one off exclusion. The cutoff sits in a genuine gap in the founding year distribution, no company was founded between 1985 and 1989 | 3 (Otto Bock HealthCare, Promasidor Holdings, Five Star Business Finance) |
| Currency parsing | Parsed `Valuation` and `Funding` from strings like `$11B` or `$500M` into numeric dollar values. `Unknown` values in `Funding` were mapped to missing rather than guessed | 12 `Funding` values set to missing |
| Date parsing | Parsed `Date Joined` using an explicit `%m/%d/%y` format and verified every date resolved to the correct century | 0 |
| Country consolidation | Checked all 45 unique values in `Country/Region` for variant spellings before any grouping | 0, none found |

Final row count after cleaning: 1,070, down from 1,073.

## Feature Engineering

To be documented.

## Exploratory Data Analysis

To be documented.

## Truncation Bias

To be documented.

## Modeling

To be documented.

## MENA Region

To be documented.
