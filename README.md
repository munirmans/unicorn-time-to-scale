# Unicorn Time to Scale

What predicts how fast a company reaches a one billion dollar valuation, and does the apparent acceleration over time survive scrutiny? This project analyzes 1,073 companies to answer that question, with a specific focus on testing whether the "unicorns are getting faster" trend is real or an artifact of how the data was collected.

The full analysis, with all reasoning and code, is in [`notebook.ipynb`](notebook.ipynb).

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

| Feature | Definition | Notes |
|---|---|---|
| `founding_cohort` | `Year Founded` banded into pre-2000, 2000-2004, 2005-2009, 2010-2014, 2015+ | Built with `pd.cut`, bin edges chosen so the 1990 cleaning cutoff falls inside the first band |
| `investor_count` | Count of comma-separated names in `Select Investors` | 1 missing value (LinkSure Network) left as missing rather than treated as zero, a company cannot become a unicorn with no backers, so the null reflects an unrecorded value, not an absence |
| `has_top_tier_investor` | Binary flag for whether Sequoia, Andreessen Horowitz, Tiger Global, Accel, SoftBank, Founders Fund, or Insight Partners appears in `Select Investors` | Case-insensitive substring match, list defined explicitly in the notebook. Same missing-value handling as `investor_count` |
| `Industry` (cleaned) | Standardized casing before use | Found and fixed a duplicate category: `Artificial intelligence` and `Artificial Intelligence` were being treated as separate industries due to inconsistent capitalization |
| `industry_grouped` | Industries representing less than 3% of the dataset folded into `Other` | Reduces 15 raw categories to 11, avoids one-hot encoded columns and tree splits built on a handful of rows. Folded: Auto & transportation, Edtech, Consumer & retail, Travel |
| `continent_grouped` | Same 3% rule applied to `Continent` | Reduces 6 categories to 4. Folded: South America, Oceania, Africa |
| `valuation_numeric` | Parsed from `Valuation`, see Data Cleaning | Descriptive use, not a model input, see the leakage note below |

`Country/Region` is kept in its raw, ungrouped form and is not used for modeling. Geography for modeling is represented by `continent_grouped`; the raw country column is reserved for the descriptive MENA section.

`Funding` (parsed as `funding_numeric`) is excluded from modeling entirely. It is downstream of the outcome and would leak information about the target. It may appear in a descriptive context only.

## Exploratory Data Analysis

`years_to_unicorn` is right-skewed: most companies cluster in the first several years, with a thinning tail out toward the slowest cases. This is typical of any "time to reach a milestone" variable, there is a hard floor at zero but no equivalent ceiling.

`Valuation` is far more extreme. Since unicorn status is a threshold by definition, almost every company sits close to the $1B floor, with a rapidly thinning tail out to the largest companies (Bytedance at $180B). A plain histogram is unreadable at this range, a log scale is needed to see any structure in the crowded low end. `Valuation` is never used as a model feature, see the leakage note under Modeling.

A boxplot of `years_to_unicorn` by industry shows some industries with a tighter spread than others. This is worth reading carefully: every company in this dataset already reached unicorn status, so a tighter spread does not mean an industry is more likely to succeed, only that the companies which did succeed in that industry did so on a more consistent timeline. The dataset contains no information about companies that tried and did not reach $1B.

## Truncation Bias

### The naive result

Grouping `years_to_unicorn` by `founding_cohort` and averaging shows a sharp downward trend:

| Cohort | Mean years to unicorn |
|---|---|
| pre-2000 | 21.4 |
| 2000-2004 | 16.5 |
| 2005-2009 | 11.5 |
| 2010-2014 | 7.2 |
| 2015+ | 4.0 |

At face value, this says unicorns are getting dramatically faster to build.

### Why it is partly illusory

This dataset only contains companies that have already reached $1B, and only as of when it was collected. The most recent `Date Joined` in the data is April 5, 2022, so the dataset was scraped shortly after that date, meaning no company can appear here for having crossed the $1B line after that point.

This creates **right-truncation**: survivorship bias within an incomplete observation window. A company founded in 1995 has had about 27 years to cross the $1B threshold by 2022, enough time for essentially every unicorn from that cohort, fast or slow, to have already appeared. A company founded in 2020 has had at most 2 years, so only its fastest possible movers could have already qualified. Any 2020-founded company that will take 5, 8, or 10 years to scale is still invisible in this dataset right now, not because it failed, but because it has not had time yet.

The consequence: the average computed for recent cohorts only reflects their fastest-scaling members, since the slower ones have not shown up yet. This systematically pulls recent-cohort averages down, which is what produces the sharp naive slope above. Some of that apparent acceleration is real; some of it is an artifact of missing data on companies still in progress.

### Correction

There are two reasonable ways to correct for truncation bias: restrict the data to companies with a full, comparable observation window, or fit a model on the restricted data and compare its coefficients against the naive fit. Cohort restriction is used here rather than fitting and comparing two full models, it is simpler and easier to state plainly: "only companies that all had roughly the same amount of time to qualify are being compared." (The coefficient-comparison approach still gets used later, in Modeling, as a second, independent check on this same finding.)

**Rule:** keep only companies founded on or before 2012. Since the dataset's collection cutoff is April 2022, this gives every remaining company roughly 10 years to have reached $1B.

**Limitation, stated honestly:** `Year Founded` is a whole year, not an exact date, so "founded 2012" could mean January 2012 (about 10 years 3 months by April 2022) or December 2012 (about 9 years 4 months). The rule gives roughly 10 years, not a guaranteed exact 10 for every row. This is a minor, second-order imprecision, a few months, on windows spanning a decade or more, and is not comparable in scale to the original truncation problem it is fixing, which was the difference between a 1-year window and a 27-year window.

### What survives

| Cohort | Naive mean | Restricted mean |
|---|---|---|
| pre-2000 | 21.4 | 21.4 |
| 2000-2004 | 16.5 | 16.5 |
| 2005-2009 | 11.5 | 11.5 |
| 2010-2014 | 7.2 | 8.1 |
| 2015+ | 4.0 | cannot be evaluated, no companies founded after 2012 have had a full window yet |

Two things stand out. First, the `2010-2014` mean rose from 7.2 to 8.1 once the still-partially-observed 2013-2014 companies were removed. This is a direct, concrete confirmation of the truncation mechanism: removing companies that have not had time to finish scaling raises the average, because it is disproportionately the fast movers who show up early, and this was exactly the effect inflating the naive figure downward.

Second, and more importantly, the naive chart's most dramatic data point, `2015+` averaging just 4.0 years, cannot be reproduced under a fair comparison at all. No company founded after 2012 has had enough time yet to be judged on equal footing with the older cohorts, so that number should never have been presented as a real finding.

**What survives:** a real downward trend across `pre-2000` through `2010-2014` (21.4 to 16.5 to 11.5 to 8.1). Companies are still reaching unicorn status meaningfully faster than they did before 2000, even after correcting for the observation window. The acceleration is real, but it is smaller and slower than the naive chart implied, and the naive chart's sharpest-looking claim was substantially an artifact.

**Interview sentence:** the naive result said unicorns are getting faster. Most of the sharpest part of that result was truncation bias, because recent slow-scaling companies are not in the dataset yet. After restricting to cohorts with a full observation window, a real but more moderate acceleration remained, from 21.4 years before 2000 to 8.1 years for 2010-2014.

![Time to unicorn by founding cohort, naive versus truncation-corrected](reports/figures/naive_vs_corrected_cohort_chart.png)

## Modeling

Regression on `years_to_unicorn`, using `founding_cohort`, `industry_grouped`, `continent_grouped`, `investor_count`, and `has_top_tier_investor` as features. `valuation_numeric` and `funding_numeric` are excluded entirely, both are measured at or after the same moment as the outcome itself, so using them would be target leakage.

Three tiers, each run twice, once on the naive full dataset and once on the restricted dataset (founded on or before 2012), with a stratified train and test split so the uneven `founding_cohort` sizes are represented proportionally in both:

1. **Baseline**: predict the training set's median for every row.
2. **Linear regression**: one-hot encoded categoricals, coefficients are the actual point, not accuracy.
3. **Random Forest**: for comparison and feature importance.

| Model | Naive MAE | Naive RMSE | Naive R² | Restricted MAE | Restricted RMSE | Restricted R² |
|---|---|---|---|---|---|---|
| Baseline | 3.40 | 4.59 | -0.03 | 3.44 | 5.18 | -0.04 |
| Linear regression | 1.87 | 2.38 | 0.72 | 2.16 | 2.79 | 0.70 |
| Random Forest | 1.87 | 2.55 | 0.68 | 1.94 | 2.76 | 0.70 |

Both real models comfortably beat the baseline on both datasets, and neither model type is a clear winner over the other. An R² around 0.7 means the models explain a meaningful share of the variation in `years_to_unicorn`, but leave real, honest uncertainty unexplained, which is expected: time to unicorn status depends on many factors this dataset does not capture, and a model that explained nearly all of it would be a sign of leakage, not skill.

**A second, independent confirmation of the truncation finding.** The naive linear model's `founding_cohort` coefficients still show almost the full naive trend even after controlling for industry, continent, and investor features (pre-2000 as the baseline, `2015+` at roughly 18 fewer years). This matters: controlling for more variables in a regression does not fix truncation bias, that is not a modeling mistake, it is structural. Confounding variables can be controlled for because they vary within the data already collected; truncation is about entire rows that do not exist yet, which no predictor column can account for.

Once the model is refit on the restricted dataset, every real `founding_cohort` coefficient shrinks (2000-2004: -5.26 to -4.23; 2005-2009: -10.34 to -9.11; 2010-2014: -14.71 to -12.70), and the Random Forest's feature importances tell the same story a third way: `founding_cohort` accounts for roughly 85% of the naive model's importance, and `founding_cohort_2015+` drops to exactly zero importance once restricted, since that category has no companies left to learn from. Three independent techniques, group means, linear coefficients, and tree-based feature importance, all identify the same dominant signal in the naive data and all show it weakening under a fair comparison.

## Regional Concentration and the Israel Comparison

52.5% of companies are from the United States, 16.1% from China. Together that's over two-thirds of the dataset from just two countries, spread across the other 43. Any single-country number outside these two is standing on a small sample.

Among the smallest-represented countries: `Turkey` and `United Arab Emirates` each have 3 companies, `South Africa` and `Nigeria` each have 1. `Israel` has 20, a genuine outlier given how small a population it's drawn from.

**The three United Arab Emirates companies, named individually** rather than summarized, since an aggregate statistic at n=3 would imply a precision that doesn't exist: **Vista Global** (aviation, founded 2004, 13 years to reach $3B), **Emerging Markets Property Group** (real estate, founded 2015, 5 years to reach $1B), and **Kitopi** (food logistics, founded 2018, 3 years to reach $1B).

**Israel, compared honestly against the two dominant markets:**

| Country | n | Median years to unicorn | Mean years to unicorn |
|---|---|---|---|
| Israel | 20 | 6.0 | 7.40 |
| United States | 562 | 6.0 | 6.80 |
| China | 172 | 5.0 | 5.85 |

Israel's median matches the United States exactly, and is a year slower than China's. Israel is not a hidden speed outlier, its unicorns scale at roughly the same pace as the two dominant markets. What makes Israel genuinely notable is volume relative to population, not velocity, a different and more defensible claim than "Israeli startups scale faster," which the data does not support.

One honest data-quality note surfaced while checking this section: the raw file lists `Promasidor Holdings` (South Africa) with `Continent` recorded as `Asia`, a data entry error. This didn't require a separate fix, that row was already removed during cleaning as one of the three pre-1990 outliers, for an unrelated reason. It's mentioned here as an example of the same habit that ran through the whole cleaning process: verify a label against the data itself rather than trust it.

**What this section deliberately does not do:** fit a regional model, report a combined average across these small-sample countries as though it means something, or make any predictive claim about a specific country or region. Twenty companies, or three, or one, is not enough to model or generalize from.

## Conclusion

The naive result said unicorns are getting dramatically faster: 21.4 years for companies founded before 2000, down to 4.0 years for companies founded in 2015 or later. Most of the sharpest part of that trend is truncation bias, this dataset only contains companies that already reached $1B by April 2022, so recent, slow-scaling companies simply haven't had time to appear in it yet.

After restricting to cohorts with a full, comparable observation window, a real but more moderate acceleration remained: 21.4 years before 2000, down to 8.1 years for 2010-2014. The same pattern held up under two further, independent checks, linear regression coefficients and Random Forest feature importance, both of which showed `founding_cohort`'s effect shrinking once the truncated rows were removed.

The models explain a meaningful share of the variation in time to unicorn status (R² around 0.7) without claiming more certainty than the data supports, and `Valuation` and `Funding` were kept out of every model entirely, since both are measured at or after the outcome itself. The regional data is concentrated enough that anything below the country level, Israel included, is reported descriptively rather than modeled, on the same principle that shaped every other decision in this analysis: state what the data can support, and no more.
