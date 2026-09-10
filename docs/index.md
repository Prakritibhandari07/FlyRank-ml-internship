# Ranking Content Pages for Refresh Review: What a Rule Catches, What a Model Adds, and What Honest Validation Reveals

## Abstract
FlyRank's content team has more underperforming pages than time to review them by hand, so this work asks a narrow, practical question: **which pages should a human reviewer look at first?** Using a 30,000-row anonymized slice of FlyRank's content-performance data spanning 32 clients, we built a transparent staleness-and-CTR baseline rule and compared it against Logistic Regression and Random Forest models trained to rank pages by decline risk. Under a naive random train/test split, the Random Forest looked like a breakthrough (precision@20 of 0.90) — but a client-held-out split, which keeps every client's pages entirely on one side of the split, showed that most of that lift was the model memorizing individual clients rather than learning a generalizable signal: precision@20 under the honest split drops to 0.50, against a 0.52 base rate. Once validated honestly, the model still edges out the hand-written rule at larger review batches (precision@100 of 0.57 vs. 0.51; precision@500 of 0.58 vs. 0.54) — a modest, real improvement, not a dramatic one. We present this as a decision-support ranking tool for prioritizing manual content review, not a prediction of any single page's future, and we report the validation design alongside the numbers so the result can be trusted rather than taken on faith.

## Introduction / Problem Statement
FlyRank manages search content for dozens of clients, and any given client can have thousands of published pages. Traffic and rankings drift over time — some pages quietly decline while still holding real search visibility, and a content strategist only has time to manually review a handful of pages a week. Today that triage is either ad hoc or done with a simple hand-written rule (flag anything old and underperforming).

The open question: **can a learned ranking beat a transparent rule at picking which pages are actually worth a human's next hour?** The decision this supports: a content reviewer works down a ranked queue of pages and decides whether to refresh, expand, or leave each one — the model doesn't take any action itself, it orders the work. A false positive near the top of the queue wastes a reviewer's time on a page that was never really in trouble; a false negative means a genuinely declining, high-visibility page goes unreviewed for another cycle. Neither is catastrophic on its own, which is exactly why this is framed as decision-support (rank and flag) rather than an automated trigger.

## Data
- **Source:** the FlyRank ML Internship anonymized starter dataset (`data/raw/content_refresh_anonymized.csv`) — a public-safe slice of FlyRank's real content performance warehouse, released for this internship track.
- **Size:** 30,000 content pages across 32 anonymized clients (`client_id`), 44 columns covering search performance (impressions, clicks, CTR, average position), engagement (sessions, scroll rate, engagement rate), content metadata (age, word count, freshness tier), and a trend label.
- **Time window:** each row is a 90-day performance snapshot per page, with a paired last-30-day / prior-30-day comparison used to derive the trend direction.
- **Label:** `is_declining_label` = 1 where `trend_direction == "down"` — an observed historical trend, not a prediction of future decline. Base rate: 54.2% overall, 51.7% in the held-out test split used below.
- **Exclusions:** none — all 30,000 rows are used, disclosed rather than hidden.
- **Public-safe by construction:** no client names, domains, URLs, page titles, or raw search queries appear anywhere in the file or notebook — only anonymized IDs and numeric/categorical signals.

## Methodology
**Baseline rule (transparent, no fitted weights):** a page is flagged `stale_and_ctr_underperforming` if (a) it's stale — `freshness_tier` in `91-180` or `181+` — AND (b) it has a measurable opportunity — `impressions_90d >= 100` and `sessions_90d > 0` — AND (c) its CTR is below 0.7× the median CTR for its `position_tier`. The rule score is the product of these three 0/1 gates times `impressions_90d`, breaking ties by visibility.

**Features used by the models:** numeric — `days_since_last_update`, `content_age_days`, `impressions_90d`, `clicks_90d`, `pageviews_90d`, `sessions_90d`, `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `search_volume`, `competition`, `cpc`, `word_count`, `char_count`; categorical — `position_tier`, `freshness_tier`, `content_type`, `main_intent`, `competition_level`, `age_tier`.

**Forbidden features (excluded to prevent leakage):** `trend_direction`, `trend_pct`, `is_declining_label`, `impressions_last_30d`, `impressions_prev_30d`, plus ID columns.

**Validation design — the part that matters most:** rows from the same client share publishing batches, templates, and cadence. A random row-level 75/25 split lets near-duplicate pages from the same client land on both sides, letting a model partly memorize client-level quirks instead of a generalizable signal. We therefore use a **client-grouped** split (`GroupShuffleSplit`, 75/25, fixed seed) that keeps each client entirely on one side — 0 client overlap between train and test by construction — alongside a naive random split as a deliberate comparison point.

**Models:** Logistic Regression (scaled features, readable coefficients), then Random Forest (300 trees, max depth 8), compared against the baseline rule on the same test set using **precision@K** at K = 20, 50, 100, 500.

## Results
**The headline finding is about validation, not just the model.** Under the naive random split, roughly all 32 clients appear on both sides of the train/test split, and the Random Forest's precision@20 comes out to **0.90** — a number that looks like a clear win. Under the honest, client-grouped split (0 client overlap), that same architecture's precision@20 drops to **0.50**, next to a 0.52 base rate. That ~40-point gap *is* the finding: most of the "before" score was the model recognizing clients it had already seen, not ranking genuinely unseen content.

Once validated honestly, the model's real edge is real but modest:

| K | Base rate | Baseline rule | Logistic Regression | Random Forest |
|---:|---:|---:|---:|---:|
| 20 | 0.517 | 0.450 | 0.500 | 0.500 |
| 50 | 0.517 | 0.540 | 0.540 | 0.540 |
| 100 | 0.517 | 0.510 | 0.560 | 0.570 |
| 500 | 0.517 | 0.544 | 0.556 | 0.584 |

At small review batches (K=20–50) the model is roughly tied with the hand-written rule. At larger batches (K=100–500), the Random Forest opens a small, consistent lead (0.57–0.58 vs. 0.51–0.54).

**Where it's wrong:** a leakage-and-error audit found that false positives cluster heavily in the `page_3_5` position tier (pages ranked around position 24–47), which both the rule and the model over-flag. That tier has a structural CTR ceiling (organic CTR drops toward zero past roughly position 20 regardless of content quality), so "low CTR" there is a weak signal for genuine decline.

## Limitations & Honest Framing
- **Cross-sectional, not causal.** A single 90-day snapshot per page; nothing here shows that *refreshing* a page causes it to recover, only that certain observed signals co-occur with an observed decline label.
- **Modest effect size.** The honest edge over the baseline rule is real but small, especially at K=20–50 where the model and rule are roughly tied — decision-support, not a step-change.
- **Known structural blind spot.** Both the rule and the model over-flag the `page_3_5` position tier, where CTR is a weak signal due to a structural click ceiling at low rankings.
- **Scale.** Measured on a 30,000-row anonymized starter slice across 32 clients, not the full ~79M-row FlyRank warehouse; the relative pattern may hold at scale, but exact precision numbers should be re-measured there.
- **Decision-support only.** The ranked queue prioritizes a human reviewer's time — it does not trigger automated changes to any page.

## Ranked Recommendations
1. **Use the model's ranked queue for larger review batches (K ≥ 100), not the smallest ones.** That's where its precision edge over the hand-written rule (0.57–0.58 vs. 0.51–0.54) is real and worth the added complexity; at K=20–50 the simpler rule performs about as well.
2. **Route `page_3_5`-tier flags to a ranking review, not a CTR/snippet review.** Both methods over-flag this tier for the wrong reason — CTR is structurally capped at low rank positions, so the real lever there is improving rank, not rewriting a snippet.
3. **Never report the random-split number.** Precision@20 of 0.90 is real code, correctly run, on a genuinely wrong validation design — always pair any reported precision@K with the split methodology and the test-set base rate.

## Reproducibility
- **Live paper:** https://prakritibhandari07.github.io/FlyRank-ml-internship/
- **Capstone Notebook:** [capstone.ipynb](https://github.com/Prakritibhandari07/FlyRank-ml-internship/blob/main/work/notebooks/capstone.ipynb)
- **GitHub Repo:** https://github.com/Prakritibhandari07/FlyRank-ml-internship
- **Data:** `data/raw/content_refresh_anonymized.csv` (bundled with the repo, public-safe).
- **Environment:** Python 3.10+, pandas, scikit-learn, matplotlib (see `requirements.txt`).
- **Seed:** `RANDOM_SEED = 42`, used for both the split and both models, for anyone re-running the notebook to get the same numbers.
- **To rerun:** clone the repo, `pip install -r requirements.txt`, run the capstone notebook top to bottom.

## Acknowledgments & Data Credit
Built on the FlyRank ML Internship dataset: [https://flyrank.ai](https://flyrank.ai)
