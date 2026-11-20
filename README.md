# eeg-model-report

Patient-level 5-fold cross-validation results for **interictal epileptiform discharge (IED) detection** on scalp EEG from Henry Ford Health, comparing three models:

| Model | Type | Reference |
|---|---|---|
| **EEGNet** | Compact CNN for EEG (depthwise + separable convolutions) | Lawhern et al., 2018 |
| **LaBraM** | Large pretrained EEG foundation model, fine-tuned | Jiang et al., ICLR 2024 |
| **SpikeNet2** | 1D ResNet spike detector, with hard-negative mining | Li et al., 2025 |

> **No patient data is included in this repository.** It contains code, configuration, logs and aggregated metrics only.

---

## Task

Binary classification of 1-second EEG windows:

- **Positive (1):** window centered on an expert-annotated IED.
- **Negative (0):** background window at least `MIN_GAP` seconds from any annotated IED.

## Data

| | |
|---|---|
| Source | Henry Ford Health, Curry `.mat` recordings (Raw variant) |
| Patients with IEDs | 10 |
| IED-free control patients | 3 (used for hard-negative mining and false-positive-per-hour evaluation) |
| Channels | 50 EEG channels (fixed order, `ch_names.json`), dead channels interpolated |
| Sampling rate | Recorded at 678.17 Hz → exported at 200 Hz → model input at 128 Hz (EEGNet, SpikeNet2) / model-specific (LaBraM) |
| Window | 1 s (2 s clips stored, center 1 s used, ±0.1 s jitter during training) |

## Evaluation protocol

- **Patient-level splits:** every patient belongs to exactly one of train / val / test in each fold, so no patient is seen in both training and evaluation.
- **5 folds:** each fold has its own held-out test patients; results are reported per fold and as mean ± std across folds.
- **Model selection:** early stopping and checkpoint selection on the **validation** set; the **test** set is evaluated once per fold.
- **Detection threshold:** chosen on validation (Youden's index), then applied unchanged to test.

### Metrics

| Metric | Meaning |
|---|---|
| AUROC | Area under the ROC curve (threshold-free) |
| AUPRC | Area under the precision–recall curve (threshold-free; more informative under class imbalance) |
| Sensitivity / Specificity | At the validation-chosen threshold |
| F1 | At the validation-chosen threshold |
| FP / hour | False detections per hour on held-out IED-free control EEG (SpikeNet2 hard-mining rounds) |

## Results

### Summary (mean ± std over 5 folds, test set)

| Model | AUROC | AUPRC | Sensitivity | Specificity | F1 |
|---|---|---|---|---|---|
| EEGNet | – | – | – | – | – |
| LaBraM | – | – | – | – | – |
| SpikeNet2 | – | – | – | – | – |
| SpikeNet2 + hard mining | – | – | – | – | – |

### Per fold (test AUROC / AUPRC)

| Fold | Test patients | EEGNet | LaBraM | SpikeNet2 |
|---|---|---|---|---|
| 0 | – | – / – | – / – | – / – |
| 1 | – | – / – | – / – | – / – |
| 2 | – | – / – | – / – | – / – |
| 3 | – | – / – | – / – | – / – |
| 4 | – | – / – | – / – | – / – |

Per-fold ROC / PR curves, loss curves and training logs are in `results/<model>/fold<k>/`.

## Training details

| | EEGNet | LaBraM | SpikeNet2 |
|---|---|---|---|
| Initialization | from scratch | pretrained, fine-tuned | from scratch |
| Loss | – | – | Class-balanced BCE (Cui et al., 2019), β = 0.999 |
| Optimizer / LR | – | – | Adam, 1e-4 |
| Batch size | – | – | 256 |
| Epochs | – | – | 30–100, early stopping (patience 5) on val loss |
| Augmentation | – | – | time jitter (±0.1 s), left–right channel flip (p = 0.5) |

**SpikeNet2 hard-negative mining.** After training on the labeled data (round 0), the model scores every 1 s window (0.25 s stride) of the IED-free control recordings. Windows scoring above a threshold τ are false positives; they are added to training as negatives and the model is retrained. τ is lowered each round (0.8 → 0.5). One control patient is held out from mining to measure FP / hour.

## Repository structure

```
eeg-model-report/
├── README.md
├── configs/                 # per-model hyperparameters
├── splits/                  # patient IDs per fold (train / val / test), no data
├── src/                     # data preparation, training and evaluation code
├── results/
│   ├── summary.csv          # one row per model × fold
│   ├── eegnet/fold0 … fold4/
│   ├── labram/fold0 … fold4/
│   └── spikenet2/fold0 … fold4/
│       ├── log.txt
│       ├── history_r*.csv   # train / val loss per epoch
│       ├── loss_r*.png
│       ├── test_roc.png
│       └── test_metrics.json
└── figures/                 # cross-model comparison plots
```

## Reproducing

1. Export recordings to `RunN_X.npy` + `RunN_meta.json` (50 channels, 200 Hz).
2. Build the event lookup table and 2 s clips (`lut_all.csv`, `Events/*.npy`).
3. Assign folds from `splits/fold<k>.json`.
4. Train and evaluate each model per fold; aggregate with `src/summarize.py` into `results/summary.csv`.

Paths to the raw data are set in `configs/` and are not included.

## References

- Lawhern, V. J. et al. *EEGNet: a compact convolutional neural network for EEG-based brain–computer interfaces.* J. Neural Eng. (2018).
- Jiang, W. et al. *Large Brain Model for Learning Generic Representations with Tremendous EEG Data in BCI.* ICLR (2024).
- Li, J. et al. *SpikeNet2* (IED detection with hard-negative mining).
- Cui, Y. et al. *Class-Balanced Loss Based on Effective Number of Samples.* CVPR (2019).
