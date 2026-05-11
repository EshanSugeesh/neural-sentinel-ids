# Neural-Sentinel: Multi-Stage Intrusion Detection System for Layer-3 Network Security

Neural-Sentinel is a Machine Learning–driven Intrusion Detection System (IDS) designed to protect the Network (Layer 3) and Transport (Layer 4) layers by learning statistical fingerprints of malicious traffic from the NSL-KDD dataset using a Random Forest classifier.  

---

## 1. Executive Summary

Traditional firewalls rely on static, human-defined rules that struggle against zero-day and rapidly evolving network attacks. Neural-Sentinel addresses this gap by learning **data-driven patterns** of intrusions directly from network flow features, enabling real-time detection of Distributed Denial of Service (DDoS), Port Scanning, IP Spoofing, and other attacks. [web:3][web:6][web:11]

Instead of brittle rule sets, Neural-Sentinel uses an ensemble of decision trees (Random Forest) to capture complex, non-linear relationships in network traffic and generalize to previously unseen attack variants. [web:7][web:11]

---

## 2. Technical Project Description

### 2.1 Problem statement

As network traffic volume and complexity grow, manual signature-based intrusion detection becomes a bottleneck and fails to scale. The goal of Neural-Sentinel is to build an automated IDS that analyzes packet-flow and connection-level features (such as duration, protocol type, and byte-transfer rates) to distinguish legitimate activity from malicious intrusions in near real time. [web:2][web:5]

### 2.2 Dataset: NSL-KDD

This project uses the NSL-KDD dataset, an improved version of the classic KDD’99 benchmark for network intrusion detection. [web:2][web:6]

Key properties:

- ~125k connection records split into train and test sets. [web:2]
- 41 features per record plus a label (normal or attack type). [web:2][web:5]
- Attack types grouped into 4 broad categories:  
  - DoS: Denial of Service (e.g., Neptune, Smurf)  
  - Probe: Surveillance and port scanning (e.g., Satan, Nmap)  
  - R2L: Remote-to-Local (e.g., guess_passwd)  
  - U2R: User-to-Root (e.g., buffer_overflow) [web:3][web:6]

Feature families: [web:2][web:6]

- **Intrinsic features:**  
  Protocol type (TCP/UDP/ICMP), service (HTTP/FTP/SMTP, etc.), connection duration.
- **Content features:**  
  Source and destination bytes (`src_bytes`, `dst_bytes`), connection status flags (SF, S0, REJ, etc.).
- **Traffic (time-based) features:**  
  Statistics over a 2-second window, e.g., number of connections to the same host or same service.

The NSL-KDD test split includes novel attack types not seen during training, making it a good benchmark for evaluating generalization. [web:3][web:6]

---

## 3. Methodology & Architecture

Neural-Sentinel follows a 5-stage pipeline:

### Stage 1: Data preprocessing

- Load NSL-KDD train and test files.
- Handle missing or inconsistent records.
- Encode categorical variables (e.g., protocol_type, service, flag) using Label Encoding or One-Hot Encoding as appropriate.
- Standardize or normalize numerical features (e.g., duration, bytes, connection counts) to stabilize training. [web:2][web:5]

### Stage 2: Exploratory Data Analysis (EDA)

- Analyze class distribution across normal traffic and each attack category (DoS, Probe, R2L, U2R).
- Study correlations between features such as SYN error counts and specific DoS attack labels (e.g., Neptune).
- Visualize feature distributions and identify potential sources of data leakage or imbalance. [web:2][web:5]

### Stage 3: Feature engineering

- Compute feature importances from an initial Random Forest model.
- Select the most discriminative features for intrusion detection (e.g., connection counts, error rates, service-based statistics).
- Optionally reduce dimensionality to speed up inference while retaining performance. [web:4][web:11]

### Stage 4: Classifier – Random Forest ensemble

- Train a Random Forest classifier on the preprocessed NSL-KDD training set.
- Use hyperparameters (e.g., number of trees, max depth, class weights) tuned for high recall on attack classes while controlling false positives.
- The ensemble of trees captures non-linear interactions and is robust against overfitting on high-dimensional, heterogeneous network data. [web:7][web:11]

### Stage 5: Rigorous evaluation

- Evaluate on the official NSL-KDD test split containing both known and novel attacks. [web:3][web:6]
- Metrics:
  - Accuracy
  - Precision, recall, F1-score (per class and macro/micro averaged)
  - Confusion matrix
  - ROC-AUC (binary or one-vs-rest for multi-class)
- Particular emphasis is placed on high recall for attack classes to minimize false negatives. [web:7]

---

## 4. Key Project Objectives

- **Real-time anomaly detection**  
  Design preprocessing and model inference to be lightweight enough to run close to line rate on edge routers or security appliances.

- **High recall on intrusions**  
  Prioritize recall for malicious classes to ensure that attacks (especially DoS and Probe) are rarely missed, even at the cost of some false positives.

- **Scalability and deployability**  
  Keep the model and feature pipeline simple and efficient so it can be deployed:
  - As a microservice behind a REST API.
  - On edge devices for Layer-3/4 security monitoring.

---

## 5. Repository structure

```text
├── data/
│   ├── raw/                  # Original NSL-KDD files (not tracked in Git by default)
│   └── processed/            # Cleaned and preprocessed CSVs
├── notebooks/
│   └── 01_nsl_kdd_random_forest.ipynb
├── src/
│   ├── data_preprocessing.py
│   ├── feature_engineering.py
│   ├── train_model.py
│   └── evaluate_model.py
├── models/
│   └── random_forest_ids.pkl
├── api/
│   └── app.py                # Optional REST API for real-time inference
├── reports/
│   ├── figures/
│   └── metrics_summary.md
├── requirements.txt
├── LICENSE
└── README.md
```

---

## 6. Getting started

### 6.1 Prerequisites

- Python 3.9+
- Recommended: virtualenv or conda

Install dependencies:

```bash
pip install -r requirements.txt
```

### 6.2 Data setup

1. Download the NSL-KDD dataset (e.g., from Kaggle or the official NSL-KDD links). [web:2][web:5]
2. Place raw files into:

```text
data/raw/
├── NSL-KDDTrain+.txt
├── NSL-KDDTest+.txt
```

3. Run the preprocessing script (once you add your code):

```bash
python src/data_preprocessing.py
```

This will generate processed CSVs in `data/processed/`.

---

## 7. Model training and evaluation

Train the Random Forest model:

```bash
python src/train_model.py
```

Evaluate on the test set:

```bash
python src/evaluate_model.py
```

The trained model will be saved to `models/random_forest_ids.pkl`, and evaluation plots/metrics will be stored under `reports/`. [web:8]

---

## 8. (Optional) Real-time inference API

A simple FastAPI or Flask app can be added in `api/app.py` to expose an endpoint like `/predict` that accepts connection-level features (JSON) and returns the predicted class (normal / DoS / Probe / R2L / U2R). [web:8]

Example request body (conceptual):

```json
{
  "duration": 0,
  "protocol_type": "tcp",
  "service": "http",
  "src_bytes": 215,
  "dst_bytes": 45076,
  "flag": "SF",
  "...": "..."
}
```

---

## 9. Roadmap

- Add advanced feature selection and dimensionality reduction.
- Experiment with other models (XGBoost, LightGBM, deep learning).
- Deploy a live demo interface for traffic simulation and visualization.
- Extend to modern datasets (e.g., CIC-IDS) for better real-world relevance.

---

## 10. License

This project is licensed under the MIT License – see the `LICENSE` file for details.
