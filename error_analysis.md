# Error Analysis

## Overview

The churn prediction model was evaluated using a Random Forest Classifier on customer transactional features.

The model achieved:

- Accuracy: 65.2%
- Precision: 65%
- Recall: 52.9%
- F1 Score: 58.3%

---

## Observed Errors

### False Positives

Some customers were predicted as churn-risk customers even though they did not churn.

Possible reasons:

- temporary inactivity,
- seasonal purchasing behavior,
- inconsistent ordering patterns.

---

### False Negatives

Some actual churned customers were predicted as non-churned.

Possible reasons:

- missing behavioral signals,
- lack of website engagement features,
- limited support ticket features,
- limited historical transaction depth.

---

## Key Limitations

The model currently uses only transactional order-based features.

Additional features that may improve performance:

- website activity,
- support ticket frequency,
- campaign interaction history,
- customer engagement scores,
- product browsing behavior.

---

## Future Improvements

Future versions of the model can include:

- hyperparameter tuning,
- advanced ensemble models,
- XGBoost or LightGBM,
- feature scaling,
- time-series customer behavior analysis,
- class imbalance optimization.