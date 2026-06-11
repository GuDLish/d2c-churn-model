# Part 3 — Churn Prediction

## Goal
Predict whether a customer will churn in the next 30 days, and produce a weekly list of customers the retention team should contact.

## Data
Built from four raw sources, joined per `customer_id`:
- **Orders** → recency_days, frequency_180d, monetary_180d, monetary_all, avg_order_value, avg_discount, avg_delivery, avg_rating
- **Web activity** → sessions_30d, product_views_30d, last_visit_days_ago
- **Support tickets** → ticket_count_90d, neg_ticket_rate_90d
- **Campaigns + profile** → acquisition_channel (one-hot), loyalty_tier (one-hot incl. "Missing"), days_since_signup, manual_priority_bucket (one-hot)

Final feature table: ~36 columns. Full list in `feature_list.json`.

## Train / Validation / Test
Chronological split from the provided modeling snapshot. No leakage: features for a customer are computed strictly before the label window.

## Models
1. **Baseline:** Logistic Regression on the same scaled feature table.
2. **Final:** **Random Forest** (probability-calibrated) — chosen for higher ROC-AUC + PR-AUC and tolerance of the "Missing" tier bucket.

## Operating Threshold
**0.07**, chosen by maximising expected business value on validation (see `threshold_ev.png`). The EV curve is flat between 0.05 and 0.10 and falls sharply above 0.4.

## Test Performance
| Metric | Value |
|---|---|
| ROC-AUC | 0.876 |
| PR-AUC | 0.861 |
| Accuracy | 0.613 |
| Precision | 0.564 |
| Recall | 0.994 |
| F1 | 0.720 |

Confusion matrix @ thr=0.07: TN=39, FP=129, FN=1, TP=167.

## Files in this folder
| File | What it is |
|---|---|
| `notebook.ipynb` | End-to-end pipeline: features → split → train → evaluate → save artefacts |
| `model.pkl` | Calibrated Random Forest, ready for `joblib.load` |
| `feature_list.json` | Exact feature order expected at inference |
| `metrics.json` | All test-set metrics |
| `threshold.json` | Chosen threshold + EV at that threshold |
| `false_positives_top10.csv` | Top FP cases with feature snapshot — input to error_analysis |
| `false_negatives_top10.csv` | Top FN cases with feature snapshot — input to error_analysis |
| `roc_pr_curves.png` | ROC + Precision-Recall curves on test |
| `feature_importance.png` | RF impurity importance, top 15 |
| `threshold_ev.png` | Expected value vs threshold on validation |
| `confusion_matrix.png` | Confusion matrix at the chosen threshold |
| `model_card.md` | Full model card (intended use, limitations, monitoring) |
| `error_analysis.md` | FP + FN analysis with specific customer IDs |
| `README.md` | This file |

## Reproducing
1. Open `notebook.ipynb`.
2. Run all cells in order. The notebook re-creates the feature table, retrains both models, picks the threshold, and overwrites every CSV/PNG/JSON in this folder.
3. No CSV in this folder is hand-edited — they are all notebook outputs.

## Known limitations
See `model_card.md` §7 and §9. In short: recency dominates, single-event churners are missed, and a rules layer is needed on top for Platinum/Gold customers with bad support signals.