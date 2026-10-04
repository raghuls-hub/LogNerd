Here is a step-by-step dry run of the **LogNerd** base pipeline using a realistic example log stream from a simulated microservice architecture.

---

## **Input: Raw Unstructured Log Snippet**

Suppose a transaction failure occurs in a deployment pipeline. The system receives these 6 raw log lines:

```text
2026-09-30 10:14:01.102 INFO [auth-service] User user_8810 login successful from 192.168.1.45
2026-09-30 10:14:02.311 WARN [payment-service] Retry 1/3 connecting to db instance db-cluster-primary.internal
2026-09-30 10:14:02.850 WARN [payment-service] Retry 2/3 connecting to db instance db-cluster-primary.internal
2026-09-30 10:14:03.100 ERROR [payment-service] Connection timeout after 5000ms to 10.0.4.12:5432
2026-09-30 10:14:03.105 ERROR [checkout-service] Order tx_99214 failed due to dependency error in payment-service
2026-09-30 10:14:03.110 ERROR [frontend-service] HTTP 500 Internal Server Error returned for session sess_771

```

---

## **Stage 1: Ingestion, Cleaning & Preprocessing**

* **Action:** Parse timestamps, log levels, and service names via regex. Sanitize dynamic variables (IPs, UUIDs, hostnames, numeric retry counts) into standard tokens.
* **Output Data Table:**

| Line | Timestamp | Level | Service | Masked Message Body |
| --- | --- | --- | --- | --- |
| **L1** | 10:14:01.102 | INFO | auth-service | User `<USER>` login successful from `<IP>` |
| **L2** | 10:14:02.311 | WARN | payment-service | Retry `<NUM>`/`<NUM>` connecting to db instance `<HOST>` |
| **L3** | 10:14:02.850 | WARN | payment-service | Retry `<NUM>`/`<NUM>` connecting to db instance `<HOST>` |
| **L4** | 10:14:03.100 | ERROR | payment-service | Connection timeout after `<NUM>`ms to `<IP>`:`<PORT>` |
| **L5** | 10:14:03.105 | ERROR | checkout-service | Order `<TX_ID>` failed due to dependency error in payment-service |
| **L6** | 10:14:03.110 | ERROR | frontend-service | HTTP `<NUM>` Internal Server Error returned for session `<SESS>` |

---

## **Stage 2: Template Parsing & Structural Mining**

* **Action:** Run a online parser (e.g., Drain tree) to group lines with identical static structures into template event IDs ($E_i$).
* **Extracted Template Dictionary:**
* $E_1$: `User <USER> login successful from <IP>`
* $E_2$: `Retry <NUM>/<NUM> connecting to db instance <HOST>`
* $E_3$: `Connection timeout after <NUM>ms to <IP>:<PORT>`
* $E_4$: `Order <TX_ID> failed due to dependency error in payment-service`
* $E_5$: `HTTP <NUM> Internal Server Error returned for session <SESS>`


* **Mapped Log Events:**
`L1 → E1`, `L2 → E2`, `L3 → E2`, `L4 → E3`, `L5 → E4`, `L6 → E5`

---

## **Stage 3: Unsupervised Pattern Mining & Anomaly Scoring**

* **Action:**
1. **Sliding Time Window:** Slice logs into a 3-second window $[10:14:01.000 \text{ to } 10:14:04.000]$.
2. **Baseline Comparison:** Compare $E_2$ (WARN) and $E_3$ (ERROR) against a historical healthy run. In normal runs, $E_2$ rarely appears twice in under 600ms ($\Delta t = 539\text{ms}$).
3. **Isolation Forest & DBSCAN:**
* $E_1$ (login) is clustered into standard user traffic (Anomaly Score: **0.05**).
* $E_2$ & $E_3$ (retry burst + timeout) trigger high spatial-temporal outlier scores (Anomaly Score: **0.88**).
* $E_4$ & $E_5$ are recognized as secondary cascading errors triggered by $E_3$.





---

## **Stage 4: Feature Engineering & Space Construction**

* **Action:** Construct the numerical feature vector $X_i$ for each log line using frequency, temporal, categorical, and Stage 3 anomaly metrics.

| Line | Level (Ordinal) | $\Delta t$ from previous (ms) | Template ID ($E_i$) | Burst Count in 1s | Stage 3 Anomaly Score |
| --- | --- | --- | --- | --- | --- |
| **L1** | 0 (INFO) | 0 | $E_1$ | 1 | 0.05 |
| **L2** | 1 (WARN) | 1209 | $E_2$ | 1 | 0.62 |
| **L3** | 1 (WARN) | 539 | $E_2$ | 2 | 0.85 |
| **L4** | 2 (ERROR) | 250 | $E_3$ | 3 | 0.94 |
| **L5** | 2 (ERROR) | 5 | $E_4$ | 4 | 0.71 |
| **L6** | 2 (ERROR) | 5 | $E_5$ | 5 | 0.68 |

---

## **Stage 5: Dimensionality Reduction & Feature Selection**

* **Action:** Apply L1 (Lasso) regularization and Mutual Information filtering.
* **Result:** Discard static metadata features (e.g., raw timestamp strings) that cause noise.
* **Selected Key Features ($k=4$):**
1. `Stage 3 Anomaly Score`
2. `$\Delta t$ (time delta acceleration)`
3. `Template Frequency in Window`
4. `Is_Dependency_Target` (extracted from template syntax)



---

## **Stage 6: Base Supervised Modeling & Evaluation**

* **Action:** Feed $X_{\text{selected}}$ into base classifiers (Logistic Regression, Decision Tree, Naive Bayes) to compute root-cause likelihood scores $P(\text{Root Cause} \mid X_i)$.
* **Base Model Predictions:**
* **Naive Bayes:** Flags **L6** as most likely root cause because it contains "HTTP 500" (fails on last-error bias).
* **Decision Tree:** Flags **L4** ($E_3$) and **L3** ($E_2$) due to low $\Delta t$ and high Stage 3 anomaly score.



---

## **Stage 7: Ensemble Stacking & Meta-Learning**

* **Action:** Aggregate base model outputs using a Stacking Meta-Classifier (XGBoost / Random Forest). The ensemble learns that secondary error bursts ($E_4, E_5$) follow primary network timeouts ($E_3$) within a 10ms window.
* **Final Ensembled Causal Likelihood Scores:**

```text
L1 (INFO)  : 0.01  [Normal]
L2 (WARN)  : 0.42  [Warning Signal]
L3 (WARN)  : 0.78  [Early Root Cause Indicator]
L4 (ERROR) : 0.96  <-- TOP 1 ROOT CAUSE (Primary Database Timeout)
L5 (ERROR) : 0.65  [Secondary Cascade]
L6 (ERROR) : 0.41  [Symptom / Surface Error]

```

---

## **Stage 8: Explainability, Confidence & Fallback**

* **Action:** Compute SHAP values for the top ranked line (**L4**).
* **SHAP Attribution Breakdown for L4:**
* $+0.45$ contributed by `Stage 3 Anomaly Score = 0.94` (Rapid retry failure before timeout).
* $+0.32$ contributed by `$\Delta t = 250\text{ms}$` (Abnormal temporal compression).
* $+0.19$ contributed by `Template E3 = Connection Timeout`.


* **Confidence Metric:** $C = 0.96 \implies$ High Confidence ($\ge 0.85$). *No fallback to "Not Sure" needed.*

---

## **Final Output of Base Pipeline**

```text
================================================================================
LOGNERD BASE PIPELINE DETECTION REPORT
================================================================================
Primary Root Cause Identified: LINE 4
Timestamp : 2026-09-30 10:14:03.100
Service   : payment-service
Template  : Connection timeout after <NUM>ms to <IP>:<PORT>
Confidence: 96% (High)

Top 3 Diagnostic Lines Ranked:
  1. [Score: 0.96] L4: Connection timeout after 5000ms to 10.0.4.12:5432
  2. [Score: 0.78] L3: Retry 2/3 connecting to db instance db-cluster-primary...
  3. [Score: 0.65] L5: Order tx_99214 failed due to dependency error...

SHAP Explanation:
  - Flagged primarily due to abnormal time compression (250ms gap after retry) 
    and a high unsupervised anomaly score (0.94 vs healthy baseline).
================================================================================

```

---