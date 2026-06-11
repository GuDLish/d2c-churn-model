# Error Analysis — Churn Model @ Threshold 0.07

## 1. Confusion Matrix (Test)

|  | Pred 0 | Pred 1 |
|---|---|---|
| **True 0** (stayed) | 39 | **129 (False Positive)** |
| **True 1** (churned) | **1 (False Negative)** | 167 |

- The chosen operating point trades precision for recall: we catch 167/168 real churners (99.4% recall) at the cost of 129 wasted contacts.
- Total positives predicted: 296. Of those, 56% are real churners and 44% are not.

## 2. False Positive Analysis (Pred 1, Actual 0)

**What's happening:** the model heavily relies on `recency_days` and `last_visit_days_ago`. Customers who simply went quiet — but were never high-engagement to begin with — get flagged as churn risks even though they were never going to churn in the next 30 days because they were already in a low, stable cadence.

**Business risk:**
- **Wasted retention budget:** vouchers and CSM time sent to customers who would have stayed anyway.
- **Discount cannibalisation:** we condition stable customers to expect discounts.
- **List fatigue:** repeated outreach to dormant Silver/Missing-tier users reduces deliverability for the rest of the list.
- **Brand cost is low** — these are usually neutral or positive contacts — but the financial leakage compounds weekly.

### 10 specific FP customers

| # | customer_id | proba | recency_days | freq_180d | monetary_180d | sessions_30d | neg_ticket_rate_90d | tier | channel | What I'd do |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | CUST01246 | 0.908 | 262 | 0 | 0 | 1 | 0 | Silver | Influencer | **Drop from outreach.** 262d inactive, zero orders/spend in window. Move to reactivation list, not retention. |
| 2 | CUST00437 | 0.882 | 151 | 1 | 729 | 0 | 0 | Silver | Marketplace | **Drop.** No web sessions in 30d, no support signal — they've quietly left. Try a single low-cost win-back email at day 180, not paid retention. |
| 3 | CUST01039 | 0.871 | 129 | 1 | 545 | 12 | 0 | Missing | Marketplace | **Keep but downgrade action.** 12 sessions in 30d means they ARE active — model is wrong because no recent *purchase*. Trigger a browse-abandonment email, not a CSM call. |
| 4 | CUST01405 | 0.857 | 140 | 1 | 1013 | 2 | 0 | Gold | Referral | **Soft touch only.** Gold tier + referral channel + decent monetary justifies one personalised email, but no discount — likely to come back on next replenishment cycle. |
| 5 | CUST01803 | 0.842 | 104 | 2 | 1101 | 1 | 0 | Silver | Referral | **Drop from paid outreach.** Cadence looks normal for this tier; do not include in voucher campaign. |
| 6 | CUST02171 | 0.842 | 120 | 1 | 1067 | 1 | 0 | Missing | Referral | **Drop.** Thin signal; tier unknown so we can't justify spend. Free newsletter only. |
| 7 | CUST01370 | 0.842 | 161 | 2 | 1246 | 2 | 0 | Missing | Organic | **Drop from this week's list.** Already past typical purchase cycle; survey email to learn why, no discount. |
| 8 | CUST01325 | 0.836 | 186 | 0 | 0 | 1 | 0 | Missing | Google Search | **Reactivation, not retention.** 186d since order with 0 frequency — definitely lost; cheapest possible win-back creative only. |
| 9 | CUST01614 | 0.824 | 103 | 2 | 1352 | 4 | 0 | Missing | Google Search | **Hold.** Active on site (4 sess) and £1.3k spend — worth a personalised content email, but not a voucher. |
| 10 | CUST02022 | 0.819 | 44 | 1 | 330 | 1 | **1.0** | Missing | Instagram | **Treat as service-recovery, not retention.** Negative ticket rate is 100% — even though they didn't churn this window, root-cause the complaint before any marketing contact. |

**Mitigations:**
- Add a rules layer that demotes FP-prone profiles (recency >150d AND frequency_180d <=1 AND monetary_180d < tier-median) into a cheap reactivation track rather than the paid retention list.
- Cap spend per contact at the expected value implied by `monetary_180d / 6`.

## 3. False Negative Analysis (Pred 0, Actual 1)

**What's happening:** only **1** real churner is missed at threshold 0.07. That's the win of an aggressive threshold — but the one we miss is informative.

**Business risk:**
- **Silent churn of high-value customers:** the missed cases tend to be recent, high-spend, high-tier customers whose churn is driven by a single non-RFM event (a bad ticket, a delivery failure, a product return). Losing one of these is far more expensive than 10 wasted vouchers.
- **No second chance:** because the CRM never contacts them, there's no recovery attempt before they leave.
- **Reputational risk:** Platinum / Gold churners often become detractors if their last touchpoint was a bad one.

### Specific FN customer

| # | customer_id | proba | recency_days | freq_180d | monetary_180d | sessions_30d | neg_ticket_rate_90d | tier | channel | What I'd do |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | CUST00184 | 0.006 | 14 | 3 | 2457 | 6 | 0 | **Platinum** | Instagram | **Rules override.** RFM looks perfect, but the customer churned anyway — almost certainly a single-event trigger (bad ticket, return, delivery issue) the model can't see at score time. Action: trigger a CSM call for any Platinum/Gold customer with a return or negative ticket in last 30d, regardless of model score. |

**Mitigations:**
- **Rules-layer overrides** stacked on top of the model:
  - Any Platinum/Gold customer with `neg_ticket_rate_90d > 0` → force-include in retention list.
  - Any customer with a refunded order in last 14d → service-recovery flow.
  - Any customer whose `sessions_30d` drops to 0 after being >5 the prior month → soft-touch email.
- **Lower threshold further** (e.g. 0.05) only marginally helps and floods the FP list — better to fix FNs with rules than by threshold.

## 4. Cost Summary

| Error type | Count | Per-error cost (est.) | Total weekly cost |
|---|---|---|---|
| False Positive | 129 | ~$5 (voucher + send) | ~$645 |
| False Negative | 1 | ~$200–$2,000 lost LTV | $200–$2,000 |

The asymmetric cost is exactly why the threshold sits at 0.07 — see `threshold_ev.png`: expected value peaks in the 0.05–0.10 band and falls off a cliff above 0.4.