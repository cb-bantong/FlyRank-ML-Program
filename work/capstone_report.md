# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** Intern (FlyRank ML Program)
- **Lane:** CTR / Engagement Opportunity Scoring (Lane 4)
- **Repo:** [https://github.com/cb-bantong/FlyRank-ML-Program](https://github.com/cb-bantong/FlyRank-ML-Program)
- **Deployed Paper:** [https://cb-bantong.github.io/FlyRank-ML-Program/](https://cb-bantong.github.io/FlyRank-ML-Program/)
- **Date:** November 2026

---

## 1. Problem Framing

This project solves an operational resource allocation problem in content search optimization: **Which 20 to 50 URLs out of a website's entire inventory should an editorial team review and optimize first this week to maximize organic click capture?**

- **Unit of Analysis:** One row = one pseudonymized content item (`content_id`) belonging to a pseudonymized client (`client_id`), aggregated over a trailing 90-day observation window ($N=30,000$).
- **Output:** An operational ranked queue of candidate URLs scored from 0 to 100 with human-interpretable reason codes (`page_one_click_leak`, `striking_distance_opportunity`, `high_exposure_underperformer`) and concrete suggested actions (`rewrite_title_and_meta`, `improve_serp_snippet_intent`, `on_page_engagement_review`, `monitor_visibility`).
- **Action Taken by Human Editors:** Rewriting title tags to match user search intent, testing emotional hooks in meta descriptions, and injecting structured FAQ schema for Google featured snippet capture.
- **Cost of a Wrong Call:** 
  - *False Positive:* Wastes scarce human review hours on pages where clicks are suppressed by Google AI Overviews or navigational intent.
  - *False Negative:* Leaves thousands of recoverable organic search visitors on the table while competitors capture market share.
- **Why Data & ML Help:** A flat rule (e.g. `CTR < 0.5%`) flags 18,614 visible URLs (84.6% of inventory), overwhelming human capacity. Machine learning smoothly models the continuous non-linear relationship between search rank, exposure volume, and query intent, weighting each candidate by estimated recoverable click volume.

---

## 2. Data Safety

- **Release & Tables:** FlyRank Anonymized Search Intelligence Starter Dataset (`data/raw/content_refresh_anonymized.csv`), comprising 30,000 rows × 44 columns across 32 clients.
- **Public Safety Assurance:** The dataset contains zero client names, domain names, URLs, page titles, or raw search queries. Identifiers (`content_id`, `client_id`) are cryptographically pseudonymized hashes used exclusively for joining and grouped train/test splitting.
- **Deliberately Excluded Fields:**
  1. `ctr` itself: Excluded from feature inputs because the proxy target is derived from CTR percentiles.
  2. `trend_direction`, `trend_pct`: Excluded to eliminate temporal direction leakage.
  3. `*_last_30d`, `*_prev_30d` sub-windows: Excluded due to forward-window overlap.
  4. `provider_used`, `model_used`: Incomplete LLM metadata (>70% missing).
  5. `health_score`, `needs_ctr_fix`: Internal FlyRank product decision flags excluded upstream to prevent circular learning.

---

## 3. Baseline

We implemented a transparent, hand-coded heuristic baseline combining visibility, click deficit, and ranking leverage:

$$\text{baseline\_score} = 0.40 \times \text{visibility\_score} + 0.35 \times \text{ctr\_deficit\_score} + 0.25 \times \text{position\_leverage\_score}$$

Where:
- $\text{visibility\_score} = \min(100, \frac{\log(1 + \text{impressions\_90d})}{\log(1 + 50000)} \times 100)$
- $\text{ctr\_deficit\_score} = \min(100, \max(0, \frac{\text{expected\_ctr} - \text{ctr}}{\text{expected\_ctr}} \times 100))$
- $\text{position\_leverage\_score} = \max(0, \frac{20 - \text{avg\_position}}{20} \times 100)$ for ranks $\le 20$.

On the unseen client holdout test set ($N=4,511$), the baseline achieved:
- **Precision@50 = 0.380** (38.0% of top 50 picks are severe underperformers)
- **Precision@20 = 0.450**
- **ROC-AUC = 0.615**
- **Average Precision = 0.446**

---

## 4. Model / Analysis

- **Methods Evaluated:** Logistic Regression (linear baseline with StandardScaler), Decision Tree (`max_depth=5`, `min_samples_leaf=50`), and Random Forest (`n_estimators=100`, `max_depth=8`, `min_samples_leaf=25`).
- **Feature Vector (26 columns):**
  - Continuous log counts: `log_impressions`, `log_clicks`, `log_sessions`.
  - Position metrics: `avg_position`, `position_gap_from_p1` ($\max(0, \text{pos} - 10)$).
  - Activity & content: `content_age_days`, `days_since_last_update`, `word_count`, `has_keyword_data`, `days_active_pct`, `engagement_rate`, `scroll_rate`.
  - Encoded categoricals: `content_type`, `main_intent`, `position_tier`.
- **Target Definition:**
  - Continuous Target: $\text{Estimated Lost Clicks} = \max(0, \text{Expected CTR}_{\text{tier}} - \text{Observed CTR}) \times \frac{\text{impressions\_90d}}{100}$.
  - Binary Proxy Label: $\text{is\_underperforming\_ctr} = 1$ if $\text{CTR} \le 25\text{th percentile CTR of position tier}$ for visible pages ($\ge 100$ impressions), else 0 (base rate = 32.4% on test set).

---

## 5. Evaluation

- **Validation Design:** Grouped client-holdout split using `client_id`. 26 clients (17,495 rows) were allocated to training; 6 clients (4,511 rows) were held out completely for test evaluation. This design guarantees the model is tested on domains it has never seen before.
- **Model Comparison Table on Unseen Client Holdout:**

| Model / Baseline | ROC-AUC | Avg Precision | Precision@50 | Precision@20 | Lift over Baseline |
|---|:---:|:---:|:---:|:---:|:---:|
| **Random Forest** | **0.748** | **0.624** | **0.740** | **0.800** | **1.95x** |
| **Decision Tree** | 0.718 | 0.558 | 0.580 | 0.650 | 1.53x |
| **Logistic Regression** | 0.692 | 0.512 | 0.520 | 0.550 | 1.37x |
| **Baseline Rule (W04)** | 0.615 | 0.446 | 0.380 | 0.450 | 1.00x |
| **Base Rate (Random)** | 0.500 | 0.324 | 0.324 | 0.324 | 0.85x |

- **Error Analysis:**
  - *False Positive:* `content_4a6607efcb46` ranks at position 2.2 with 128,068 impressions and 13 clicks (0.01% CTR). Error reason: Zero-click SERP where Google's AI Overview answers the query directly, depressing organic CTR across all results.
  - *Random vs. Grouped Split Gap:* Random split yielded Precision@50 = 0.780 vs. Grouped split 0.740. The 4.0 percentage point difference represents domain-level memorization, proving the necessity of grouped validation.

---

## 6. Interpretation

- **Feature Importance:** Random Forest relies primarily on `avg_position` (28.4%), `log_impressions` (22.1%), `log_clicks` (14.6%), and `days_active_pct` (11.2%).
- **Findings:**
  1. *Position non-linearity is decisive:* Page 1 median CTR is 0.23% vs. 0.06% on Page 3–5.
  2. *Word count exhibits diminishing returns:* Articles exceeding 3,500 words do not capture higher snippet CTR than 2,000–3,500 word articles (median 0.14% vs 0.20%).
  3. *Search intent matters:* Transactional queries capture higher CTR (0.25%) than informational queries (0.22%) when ranking on Page 1.

---

## 7. Recommendation

We provide a production queue of 22,006 visible content items with action playbooks:
- **`rewrite_title_and_meta` (3,037 URLs, 13.8%):** Page 1 URLs with high exposure and severe CTR deficits. Primary editorial target.
- **`improve_serp_snippet_intent` (1,135 URLs, 5.2%):** Striking-distance URLs (ranks 11–20) needing structured FAQ schema.
- **`on_page_engagement_review` (2,775 URLs, 12.6%):** Low scroll/session read rates.
- **`monitor_visibility` (15,059 URLs, 68.4%):** Steady baseline items.

**Editorial Protocol & Limits:**
Editors triage the 651 HIGH-confidence URLs in weekly batches of 25. Before updating titles, editors must check live Google SERPs for zero-click AI Overviews and verify query intent.

---

## 8. Reproducibility

- **Environment:** Python 3.10+, `pandas>=2.2`, `numpy>=1.26`, `scikit-learn>=1.4`.
- **Random Seeds:** `RANDOM_STATE = 42` fixed across all splits and models.
- **Re-run Instructions:**
  ```bash
  git clone https://github.com/cb-bantong/FlyRank-ML-Program.git
  cd FlyRank-ML-Program
  pip install -r requirements.txt
  python scripts/run_all.py
  ```
- **Notebooks:** All assignments ML-02 through ML-11 are located in [`work/notebooks/`](notebooks/) and are runnable top-to-bottom in Colab or Jupyter.

---

## 9. Acknowledgments & Data Credit

Built on the **[FlyRank ML Internship dataset](https://flyrank.ai/)** (Applied Search Intelligence Track: Google Search Ranking & Discoverability). All data is pseudonymized and governed under [`DATA_USE.md`](../DATA_USE.md).
