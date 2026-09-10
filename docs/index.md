# Improving Search Ranking for Rare Queries

## Abstract
This paper investigates how anonymized FlyRank search data can be used to improve ranking for rare queries.  
We define the problem, build features, train an XGBoost model, and compare results against a baseline.  
Our evaluation shows measurable improvements in ranking metrics.  
We highlight limitations such as sparse data and leakage risks.  
Finally, we provide ranked recommendations for content teams to act on.

## Introduction / Problem Statement
Rare queries often underperform in search ranking, leading to poor discoverability.  
This project supports the decision of whether to adopt ML‑based ranking improvements for long‑tail content.

## Data
- Source: FlyRank ML Internship dataset ([FlyRank.ai](https://flyrank.ai))  
- Release: Hugging Face warehouse (2026)  
- Tables: [list tables you used]  
- Date window: [insert date range]  
- Exclusions: [document what you excluded and why]

## Methodology
- Assumptions: [state assumptions clearly]  
- Features: [list features engineered]  
- Label definition: [what you predicted]  
- Baseline: [simple heuristic or model]  
- Validation design: [time‑aware split, leakage checks]

## Results
- Baseline vs Model metrics (Accuracy, F1, NDCG)  
- Charts: [insert plots from notebook]  
- Observations: [summarize improvements]

## Limitations & Honest Framing
- Sparse data for rare queries  
- Limited generalization beyond dataset  
- Directional insights, not causal proof

## Ranked Recommendations
1. Prioritize content refresh for queries with low CTR but high impressions.  
2. Merge or rewrite underperforming archetypes.  
3. Monitor rare query clusters for growth opportunities.

## Reproducibility
- [Capstone Notebook](work/notebooks/capstone.ipynb)  
- [GitHub Repo](https://github.com/Prakritibhandari07/FlyRank-ml-internship)

## Acknowledgments & Data Credit
Built on the FlyRank ML Internship dataset: [https://flyrank.ai](https://flyrank.ai)
