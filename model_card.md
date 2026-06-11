# Model Card — Customer Churn Classifier

## 1. Overview
- **Task:** Binary classification — will a customer churn in the next 30 days?
- **Model family:** Random Forest classifier (probability-calibrated, scikit-learn).
- **Baseline compared against:** Logistic Regression on the same feature table.
- **Why Random Forest won:** Higher ROC-AUC and PR-AUC, handles mixed numeric / one-hot categorical features without scaling, and is robust to the missing `loyalty_tier` values present in the dataset.

## 2. Intended Use
- **Primary user:** Retention / CRM team. The model produces a churn probability that the CRM uses to build weekly outreach lists (email, push, CSM call, voucher).
- **Decision the model supports:** "Should we spend retention budget on this customer this week?"
- **Not designed for:** credit decisions, account suspension, pricing, or any individual-level financial decision.

## 3. Data
- **Sources combined:**
  - Orders (recency, frequency, monetary, AOV, avg discount, avg delivery time, avg rating)
  - Web activity (sessions in last 30d, product views in last 30d, last visit days ago)
  - Support tickets (ticket count in 90d, negative-ticket rate in 90d)
  - Campaign engagement + acquisition channel
  - Profile (loyalty tier, days since signup, manual_priority_bucket)
- **Split:** train / validation / test from the provided modeling snapshot (chronological, no leakage between sets).
- **Class balance on test:** 168 churners vs 168 non-churners (1:1 in the snapshot used).

## 4. Features Used
Top drivers (RandomForest impurity importance):
`recency_days`, `last_visit_days_ago`, `manual_priority_bucket_high`, `monetary_180d`, `manual_priority_bucket_low`, `frequency_180d`, `product_views_30d`, `days_since_signup`, `monetary_all`, `sessions_30d`, `avg_order_value`, `avg_discount`, `manual_priority_bucket_medium`, `avg_delivery`, `avg_rating`.

Full list is in `feature_list.json`. The model has access to ~36 features in total (RFM + web + support + campaign + profile + one-hot channel/tier).

## 5. Performance (Test Set)
Operating threshold: **0.07** (chosen on validation by maximising expected business value, see `threshold_ev.png`).

| Metric | Value |
|---|---|
| ROC-AUC | 0.876 |
| PR-AUC (AP) | 0.861 |
| Accuracy | 0.613 |
| Precision (churn class) | 0.564 |
| Recall (churn class) | 0.994 |
| F1 (churn class) | 0.720 |

Confusion matrix @ thr=0.07:

|  | Pred 0 | Pred 1 |
|---|---|---|
| **True 0** | 39 | 129 |
| **True 1** | 1  | 167 |

**Interpretation:** the threshold is intentionally aggressive — we accept many false positives (129) to almost never miss a real churner (only 1 missed). This matches the business cost ratio (losing a customer is worth far more than the cost of one voucher / email).

## 6. How to Read a Score
- `proba >= 0.07` → flagged as likely churner → included in this week's retention list.
- `proba <  0.07` → not contacted by the retention engine this week.
- Scores are calibrated probabilities, so a `0.5` score really does mean "about 50/50".

## 7. Known Limitations
- **Recency dominates.** The top two features (`recency_days`, `last_visit_days_ago`) together explain >30% of importance. Any customer who simply hasn't shopped in 100+ days gets flagged, even if they were never an active buyer. This shows up clearly in the false-positive list (see `error_analysis.md`).
- **Single-signal churners are missed.** A recent, high-value customer who churns because of *one* bad support ticket can slip through (e.g. CUST00184). A rules layer is needed on top.
- **`loyalty_tier` has a "Missing" bucket** — many high-probability customers have unknown tier, so tier-based personalisation downstream should fall back to a default.
- **Snapshot bias.** The test split is balanced 1:1; real production traffic is closer to 5–10% churn, so absolute precision in production will be lower than 0.56. ROC-AUC and PR-AUC are the more transferable numbers.

## 8. Ethical & Business Risks
- **Discount cannibalisation:** at 56% precision, almost half the people we send a voucher to would have stayed anyway. Cap voucher value and use frequency caps.
- **Over-targeting dormant low-value users:** the model will keep nominating long-inactive Silver/Missing-tier customers (see CUST01246, CUST01325). Outreach to them is mostly wasted spend, not harmful, but it inflates campaign costs.
- **Fairness:** `acquisition_channel` and `loyalty_tier` are inputs. We should monitor outreach rates per channel/tier so we don't systematically over- or under-serve a group (e.g. Influencer-acquired customers).
- **No PII used as a feature.** Names, addresses, payment data are not in the feature table.

## 9. When NOT to Use This Model
- Customers with **<30 days since signup** — the recency/frequency features aren't meaningful yet. Use an onboarding rule instead.
- **B2B / wholesale accounts** — the model was trained on retail behaviour.
- **Win-back of customers churned >180 days ago** — out of training distribution; use a separate reactivation campaign.
- Any **individual financial decision** (credit, refund eligibility, account closure).

## 10. Monitoring & Maintenance
- **Weekly:** track ROC-AUC and PR-AUC on the last 7 days of labelled outcomes; alert if AUC drops >0.05 vs baseline (0.876).
- **Weekly:** track precision-at-threshold on actuals; alert if it drops below 0.45.
- **Monthly:** recalibrate probabilities (Platt / isotonic) on the most recent 60 days.
- **Quarterly:** full retrain on the latest 12 months of data; re-pick the threshold on the new validation EV curve.
- **Drift checks:** PSI on top features (`recency_days`, `last_visit_days_ago`, `monetary_180d`) — alert if PSI > 0.2.
- **Stop rule:** if 4 consecutive weeks show <3% incremental retention lift vs holdout, pause the campaign and investigate before retraining.

## 11. Owners
- Model owner: Data Science
- Business owner: Retention / CRM
- On-call for drift alerts: Data Science rota