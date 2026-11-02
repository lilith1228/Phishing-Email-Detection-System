# Phishing Email Detection System

A machine learning system that classifies emails as **legitimate (Ham)**, **Spam** or **Phishing**. It follows the same pipeline as my NIDS project (network intrusion detection), applied to email text instead of network traffic.

## Overview

| Item | Details |
|---|---|
| Dataset | [Enron-Spam](https://github.com/MWiechmann/enron_spam_data) (33,716 emails; 30,459 after cleaning) |
| Task 1 (binary) | Legitimate vs spam/phishing |
| Task 2 (three-class) | Ham / Spam / Phishing |
| Features | 3,000 TF-IDF terms (unigrams and bigrams) + 13 engineered numeric features |
| Models compared | Logistic Regression, Decision Tree, Random Forest, MLP (neural network), XGBoost |
| Best model | MLP (highest cross-validation F1) |

## Pipeline

1. **Download** the dataset (automatic, from GitHub).
2. **Clean**: fill missing text, join subject and body, remove empty emails and 3,257 duplicates.
3. **Label families**: Ham and Spam come from the dataset; Phishing is a spam email containing credential or account terms (account, password, verify, bank, login...) **and** a link or "click" prompt.
4. **Split**: stratified 80/20 (24,367 train, 6,092 test).
5. **Features**:
   - TF-IDF on cleaned text (links replaced by a `urltoken` word, HTML removed).
   - Numeric features: message length, word count, subject length, URL count, shortened or IP-based links, `!` and `$` counts, uppercase and digit ratios, urgent-word count, HTML presence. Features are scaled with `StandardScaler`.
6. **Train and evaluate** all 5 models with 3-fold cross-validation, then on the test set.
7. **Three-class model** using the best model from step 6.
8. **Predict** new emails with `check_email(subject, body)`, which returns a risk score and email type.
9. **Save** `results.json` and the trained models (`.joblib`).

## Results

### Binary: legitimate vs spam/phishing (test set, 6,092 emails)

| Model | CV F1 | Accuracy | Precision | Recall | F1 | False-positive rate | ROC-AUC |
|---|---|---|---|---|---|---|---|
| Logistic Regression | 0.9813 | 0.9864 | 0.9816 | 0.9900 | 0.9858 | 1.70% | 0.9986 |
| Decision Tree | 0.9261 | 0.9338 | 0.9345 | 0.9265 | 0.9305 | 5.94% | 0.9597 |
| Random Forest | 0.9776 | 0.9790 | 0.9696 | 0.9869 | 0.9782 | 2.83% | 0.9979 |
| **Neural Net (MLP)** | **0.9825** | **0.9872** | **0.9843** | 0.9890 | **0.9866** | **1.45%** | **0.9987** |
| XGBoost | 0.9725 | 0.9770 | 0.9660 | 0.9866 | 0.9762 | 3.17% | 0.9973 |

### Three-class: Ham / Phishing / Spam (MLP)

| Class | Precision | Recall | F1 | Test emails |
|---|---|---|---|---|
| Ham | 0.991 | 0.985 | 0.988 | 3,182 |
| Phishing | 0.776 | 0.559 | 0.650 | 68 |
| Spam | 0.973 | 0.986 | 0.980 | 2,842 |

Overall accuracy is **98.1%** and macro F1 is **0.872**.

## Limitations

- **Phishing labels are heuristic.** Enron-Spam only marks emails as ham or spam, so the Phishing class comes from a keyword rule and is not human-verified. Some labels are noisy.
- **Phishing is rare** (about 1.1% of the data), so recall on that class is much lower (55.9%) than on Ham and Spam.
- **Some numeric features are always 0** on this dataset (URL, shortener, IP, `@`-in-URL, uppercase and HTML features) because Enron text is lowercased and has almost no links. They would matter more on modern phishing emails.
- **Random split.** The test set comes from the same distribution as training. Performance on new spam campaigns or modern phishing may be lower.

## Future work

- Train on a human-labelled phishing dataset (for example the Kaggle Phishing Email Dataset or Nazario).
- Add header features (SPF/DKIM/DMARC, sender and reply-to mismatch, domain age) and look-alike domain checks.
- Test on held-out spam campaigns or a time-based split to measure generalisation.
- Try a transformer model (for example DistilBERT) on the email text.
- Deploy as a Flask web app or REST API.

## Project structure

```
PEDS/
├── PhishingSpamDetection.ipynb   # full pipeline with explanations and outputs
├── data/                         # Enron-Spam CSV (downloaded automatically)
├── results.json                  # all metrics
├── email_binary_model.joblib     # legit vs spam/phishing model
├── email_family_model.joblib     # Ham / Spam / Phishing model
└── README.md
```

## How to run

```bash
pip install pandas numpy scikit-learn xgboost joblib jupyter
jupyter notebook PhishingSpamDetection.ipynb
```

Run all cells from top to bottom (about 5 to 8 minutes, mostly the MLP and XGBoost training). The last section lets you test your own emails:

```python
check_email("URGENT: verify your account",
            "Your bank account is suspended. Click http://bit.ly/x1 now and confirm your password immediately!")
# {'subject': 'URGENT: verify your account', 'risk_score': 0.995, 'verdict': 'SPAM/PHISHING', 'type': 'Phishing'}
```

## Tech stack

Python, pandas, NumPy, scikit-learn, XGBoost, joblib, Jupyter Notebook.
