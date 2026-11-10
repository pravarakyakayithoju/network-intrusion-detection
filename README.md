# Network Intrusion Detection on UNSW-NB15

Detecting malicious network flows, and classifying them into 9 attack types, on the UNSW-NB15 benchmark. The focus is on what a security team actually cares about: **false-alarm rate, rare-attack recall, and whether the numbers survive scrutiny.**

## Results (official test set, 82,332 flows)

**Binary detection: normal vs. attack**

| Model | ROC-AUC | PR-AUC | Recall at validation-chosen ≤1% FPR threshold | Test FPR at that threshold |
|---|---|---|---|---|
| **Hist Gradient Boosting** | **0.984** | **0.988** | **90.6%** | 4.2% |
| Random Forest | 0.983 | 0.987 | 89.2% | 3.7% |
| Logistic Regression | 0.967 | 0.976 | 79.1% | 2.1% |

**Attack-type classification (10 classes)**

| Model | Accuracy | Macro-F1 |
|---|---|---|
| Random Forest | 76.1% | 0.451 |
| Random Forest + balanced class weights | 71.2% | **0.478** |

## What the checks found

1. **False alarms don't transfer from validation to test.** A threshold set for ≤1% FPR on validation data gives 4.2% FPR on the test set. All 1,551 false alarms at that threshold are normal flows from TTL-254/62 hosts, the network segments that carry the attacks in this lab. That kind of normal traffic rises from 24% of normal flows in training to 52% in testing. In other words, the model has partly learned *where traffic comes from*, not *what it does*.
2. **Dropping the TTL columns doesn't remove the shortcut.** ROC-AUC barely moves (0.984 → 0.981) and test FPR stays the same, so other features carry the same network-segment information.
3. **Train/test duplicates inflate results only slightly.** 10.4% of test flows also appear in training. On the unseen 89.6%, ROC-AUC is still 0.981.
4. **Some attack types can't be separated with these features.** 79% of DoS, 77% of Backdoor and 72% of Analysis training rows have a feature-identical twin with a different label. Class weighting lifts rare-class recall (Worms 2% → 39%, Backdoor 5% → 26%) but costs Normal recall (75% → 67%).

## Methodology

- **Leakage control:** `id` and `attack_cat` are excluded from features. Encoders and scalers are fitted on training data only. Categories seen only in the test set (two connection states) are handled with `handle_unknown`.
- **Threshold selection:** made on a 15% stratified validation split of the official training set. The official test set is used once per model.
- **Interpretability:** permutation importance (PR-AUC drop) on a balanced test sample.
- **Limitations:** the data is lab-generated traffic from 2015, the official train and test files differ in their normal-traffic mix, and results come from a single seed.

## Run it

```bash
pip install -r requirements.txt
```

Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` from the [UNSW-NB15 page](https://research.unsw.edu.au/projects/unsw-nb15-dataset) and put them in `data/`. Some copies online swap the two file names; the notebook identifies them by row count. Then run `network_intrusion_detection.ipynb` top to bottom (about 3 minutes on a laptop CPU). It saves the fitted detector and its alert threshold to `models/ids_binary.joblib`.

## Next

A real-time scoring service around the saved detector: a FastAPI endpoint, a stream replay of test flows, Docker, and a load test.

## Dataset citation

N. Moustafa and J. Slay, "UNSW-NB15: a comprehensive data set for network intrusion detection systems (UNSW-NB15 network data set)," *Military Communications and Information Systems Conference (MilCIS)*, IEEE, 2015. The dataset is free for academic and research use; see the UNSW page for its terms.
