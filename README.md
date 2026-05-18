# d2c-churn-model

# D2C Customer Churn Capstone — Part 3: Churn Prediction Model

## Project Overview

This project focuses on building a machine learning model to predict customer churn for a Direct-to-Consumer (D2C) business.

The goal is to identify customers who are likely to churn within the next 60 days using transactional customer behavior data.

---

## Objectives

- perform feature engineering,
- prepare machine learning datasets,
- train a churn prediction model,
- evaluate model performance,
- analyze important churn-driving features,
- save trained ML artifacts.

---

## Repository Structure

```bash
Part3_Churn_Model/

│── churn_model.ipynb
│── model.pkl
│── metrics.json
│── error_analysis.md
│── model_card.md
│── README.md
│── requirements.txt
```

---

## Technologies Used

- Python
- Pandas
- NumPy
- Matplotlib
- Seaborn
- Scikit-learn
- Jupyter Notebook

---

## Machine Learning Workflow

### Data Preparation

Datasets used:

- customers.csv
- orders.csv
- churn_labels.csv

---

### Feature Engineering

Features created:

- total_orders
- total_spent
- avg_order_value
- avg_delivery_days
- return_rate
- avg_rating

---

### Model Training

Algorithm used:

- Random Forest Classifier

---

### Model Evaluation

| Metric | Score |
|---|---|
| Accuracy | 65.2% |
| Precision | 65% |
| Recall | 52.9% |
| F1 Score | 58.3% |

---

## Feature Importance Insights

Most influential churn prediction features:

1. total_spent
2. avg_order_value
3. total_orders
4. avg_delivery_days

---

## Outputs Generated

- trained model (`model.pkl`)
- evaluation metrics (`metrics.json`)
- confusion matrix visualization
- feature importance visualization

---

## Installation & Setup

### Clone Repository

```bash
git clone https://github.com/GuDLish/d2c-churn-model.git
```

---

### Install Dependencies

```bash
pip install -r requirements.txt
```

---

### Run Notebook

Open:

```bash
churn_model.ipynb
```

using Jupyter Notebook or VS Code.

---

## Author

Prateek Parmar