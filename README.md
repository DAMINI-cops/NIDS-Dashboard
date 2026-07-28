# Network Intrusion Detection Dashboard

A working NIDS demo: trains a classifier on NSL-KDD, simulates live network traffic, flags attacks with severity levels, and explains every prediction with SHAP.

---

## How it works

- **Model**: RandomForest classifier, trained on NSL-KDD's `KDDTrain+` set, predicting **attack category** (Normal / DoS / Probe / R2L / U2R) rather than the ~40 raw specific attack labels. Category-level prediction is more useful for a dashboard and comparable across NIDS datasets that use different specific attack names but the same category taxonomy.
- **"Live" traffic**: there's no real packet capture — `app.py` streams rows from NSL-KDD's held-out `KDDTest+` set, shuffled, a few at a time on a timer, simulating a live feed using real (if historical) traffic.
- **Alerts**: any prediction that isn't "Normal" is logged as an alert with a severity — U2R (attacker gets root) = critical, DoS = high, R2L = medium, Probe (reconnaissance only) = low.
- **Explainability**: SHAP's `TreeExplainer` computes, for any single flagged connection, which of the 41 features pushed the model toward that classification and by how much — e.g. "flagged as DoS mainly because of `count` and `serror_rate`," not just a bare label.

**Known limitation:** R2L and U2R recall on the test set is low (visible in the classification report printed during training). This is a well-documented property of NSL-KDD — many R2L/U2R attack *subtypes* in `KDDTest+` never appear in `KDDTrain+` at all, so the model hasn't seen them during training.

---

## Project structure

```
nids-dashboard/
├── data/                    (created by download_data.py — not in git)
├── models/                  (created by train.py — not in git)
├── src/
│   ├── constants.py         column names + attack-category mapping
│   ├── download_data.py     fetches NSL-KDD from a public GitHub mirror
│   ├── train.py             preprocessing + RandomForest training + eval
│   ├── explain.py           SHAP TreeExplainer wrapper
│   └── app.py               Streamlit dashboard (the actual UI)
├── requirements.txt
└── .gitignore
```

---

## Setup

```bash
cd nids-dashboard
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Mac/Linux

pip install -r requirements.txt
```

**Step 1 — Download the data:**
```bash
python src/download_data.py
```
Fetches `KDDTrain+` (125,973 rows) and `KDDTest+` (22,544 rows) from a public GitHub mirror of NSL-KDD, saves to `data/`.

**Step 2 — Train the model:**
```bash
python src/train.py
```
Trains the RandomForest, prints a classification report, and saves model artifacts to `models/`. Takes under a minute on a normal laptop.

**Step 3 — Run the dashboard:**
```bash
streamlit run src/app.py
```
Opens automatically in your browser (usually `http://localhost:8501`). Click **▶ Start** in the sidebar to begin the simulated traffic stream.

---

## Using the dashboard

- **Sidebar**: control simulation speed (connections/tick, tick interval), start/pause/reset.
- **Live connection log** (left): rolling table of the last 25 processed connections, color-coded by severity.
- **Alerts** (right): every non-Normal prediction, most recent first. Select one and click "Explain this prediction (SHAP)" to see which features drove that specific classification.
- **About this model** (bottom expander): a quick model card summarizing the setup.

---

## Tech stack

- Python, scikit-learn (RandomForestClassifier)
- SHAP (TreeExplainer) for per-prediction explainability
- Streamlit for the dashboard UI
- pandas, joblib
