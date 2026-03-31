# AOC-IDS: Improved Autonomous Online Intrusion Detection System

**Course:** COMP5311 Internet Infrastructure and Protocols

**Group-4**

| Name | Student ID |
|------|-----------|
| LIU Ying | 25042547G |
| CHEN Shengyan | 25110438G |
| HAN Yijun | 25051289G |
| CHEN Zhengxuan | 25060917G |

---

## Overview

This project builds upon the original **AOC-IDS** framework published at IEEE INFOCOM 2024:

> ["AOC-IDS: Autonomous Online Framework with Contrastive Learning for Intrusion Detection"](https://ieeexplore.ieee.org/document/10621346/)
> Xinchen Zhang, Running Zhao, Zhihan Jiang, Zhicong Sun, Yulong Ding, Edith C.H. Ngai, Shuang-hua Yang.

AOC-IDS is an autonomous online intrusion detection system that uses contrastive learning (CRC Loss) inside an Autoencoder (AE) architecture. It operates in two stages:

- **Stage 1 – Offline Training:** The model is trained on a small labeled subset using a supervised contrastive loss applied to both encoder and decoder representations.
- **Stage 2 – Online Training:** The model continuously adapts to a stream of unlabeled traffic by generating pseudo-labels via Gaussian-mixture-based classification and retraining incrementally.

Our group extended the original codebase with the following improvements to enhance accuracy, robustness, and support for an additional benchmark dataset.

---

## Improvements Over the Original

### 1. CIC-IDS-2017 Dataset Support
The original code only supported NSL-KDD and UNSW-NB15. We added full support for the **CIC-IDS-2017** dataset (`--dataset cic`), including data loading, label parsing, feature normalization, and automatic input dimension detection.

### 2. Confidence Gating for Pseudo-Labels (`--confidence_threshold`)
During online learning, the model generates pseudo-labels for unlabeled incoming traffic. Low-confidence predictions increase label noise and degrade performance. We introduce a **confidence gating** mechanism: only pseudo-labels whose confidence score (the absolute difference between the two Gaussian PDFs) exceeds the threshold are accepted into the training buffer. This significantly reduces noisy label accumulation.

### 3. Sliding Window for Online Training Buffer (`--window_steps`)
The original algorithm accumulates all pseudo-labeled samples from every online step, causing the training buffer to grow unboundedly. We introduce a **sliding window** that retains only the most recent `K` online steps in the buffer, preventing concept drift and reducing memory usage during long-horizon streaming.

### 4. Optimized Evaluation (Faster Inference)
The original `evaluate()` function performed redundant forward passes. We refactored it to use a **single forward pass** over all training data and a **single forward pass** over all test data, reducing the number of forward passes from 8 to 2. This significantly speeds up both online step evaluation and final evaluation.

### 5. Structured Result Saving & Visualization
Every run now automatically saves a full record under `result/{dataset}_seed{seed}_{timestamp}/`:

| File | Contents |
|------|---------|
| `metrics.json` | Config, per-epoch Stage 1 loss, per-step Stage 2 loss, per-step online metrics, final evaluation (encoder / decoder / combined) |
| `model.pth` | Model weights and optimizer state for reproducibility |
| `predictions.npz` | Ground-truth labels and final predicted labels for post-hoc analysis |
| `*_summary.png` | Visualization report: loss curves, per-step metric curves, confusion matrix |

---

## Project Structure

```
AOC_IDS_NEW-main/
├── online_training.py      # Main training script (Stage 1 + Stage 2)
├── utils.py                # Data loading, AE model, CRC Loss, evaluate()
├── visualization.py        # Training summary plots
├── requirements.txt        # Python dependencies
├── aoc-ids-new-faster.ipynb # Notebook for Kaggle / Colab execution
├── NSL_pre_data/           # Preprocessed NSL-KDD dataset
├── UNSW_pre_data/          # Preprocessed UNSW-NB15 dataset
├── CIC_pre_data/           # Preprocessed CIC-IDS-2017 dataset
└── result/                 # Output directory (auto-created)
```

---

## Dependencies

The project has been tested with the following configuration:

- Python 3.8+
- PyTorch 1.13.1 (CPU or CUDA)
- scikit-learn 1.2.1
- numpy 1.23.5
- pandas 1.5.3
- scipy 1.10.0
- matplotlib (for visualization)

### Installation

```bash
pip install -r requirements.txt
pip install matplotlib
```

---

## Usage

### Basic Command

```bash
python online_training.py --dataset <DATASET> --epochs <N> --epoch_1 <M> \
    --flip_percent <F> --sample_interval <S>
```

### All Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `--dataset` | str | `nsl` | Dataset to use: `nsl` / `unsw` / `cic` |
| `--epochs` | int | `4` | Number of Stage 1 (offline) training epochs |
| `--epoch_1` | int | `1` | Training epochs per online step (Stage 2) |
| `--percent` | float | `0.8` | Fraction of training data reserved for online streaming |
| `--flip_percent` | float | `0.2` | Label flip ratio for pseudo-label noise simulation |
| `--sample_interval` | int | `2000` | Number of samples processed per online step |
| `--cuda` | str | `0` | GPU index; CPU is used automatically when unavailable |
| `--window_steps` | int | `0` | Sliding window size (number of recent steps to keep); `0` = disabled (original behavior) |
| `--confidence_threshold` | float | `0.0` | Confidence gating threshold for pseudo-labels; `0` = disabled (original behavior) |

---

## Example Commands

### NSL-KDD (Baseline)
```bash
python online_training.py --dataset nsl --epochs 800 --epoch_1 1 \
    --flip_percent 0.05 --sample_interval 2000
```

### UNSW-NB15 (Baseline)
```bash
python online_training.py --dataset unsw --epochs 800 --epoch_1 1 \
    --flip_percent 0.05 --sample_interval 2784
```

### CIC-IDS-2017 (New Dataset)
```bash
python online_training.py --dataset cic --epochs 800 --epoch_1 1 \
    --flip_percent 0.05 --sample_interval 3000
```

### With Sliding Window + Confidence Gating (Improved)
```bash
python online_training.py --dataset nsl --epochs 800 --epoch_1 1 \
    --flip_percent 0.05 --sample_interval 2000 \
    --window_steps 50 --confidence_threshold 0.3
```

### Specify GPU
```bash
python online_training.py --dataset nsl --epochs 800 --epoch_1 1 \
    --flip_percent 0.05 --sample_interval 2000 --cuda 1
```

### Quick Debug Run
```bash
python online_training.py --dataset nsl --epochs 4 --epoch_1 1 \
    --flip_percent 0.2 --sample_interval 2000
```

---

## Output

After training completes, all results are saved under `result/`:

```
result/{dataset}_seed{seed}_{timestamp}/
├── metrics.json       # Full config + losses + online metrics + final results
├── model.pth          # Saved model weights and optimizer state
├── predictions.npz    # y_true and y_pred arrays for the test set
└── *_summary.png      # Visualization: loss curves, metric curves, confusion matrix
```

Example `metrics.json` structure:

```json
{
  "config": { "dataset": "nsl", "seed": 5009, "epochs": 800, "window_steps": 50, ... },
  "stage1_losses": [0.0123, 0.0098, ...],
  "stage2_losses": [0.0087, 0.0076, ...],
  "online_metrics": {
    "1": { "accuracy": 0.92, "precision": 0.89, "recall": 0.95, "f1": 0.92 },
    "2": { "accuracy": 0.93, "precision": 0.90, "recall": 0.96, "f1": 0.93 },
    ...
  },
  "final_results": {
    "encoder":  { "accuracy": 0.94, "precision": 0.91, "recall": 0.96, "f1": 0.93 },
    "decoder":  { "accuracy": 0.93, "precision": 0.90, "recall": 0.95, "f1": 0.92 },
    "combined": { "accuracy": 0.95, "precision": 0.92, "recall": 0.97, "f1": 0.94 }
  }
}
```

---

## Cloud Execution (Kaggle / Colab)

The notebook `aoc-ids-new-faster.ipynb` supports running on Kaggle or Google Colab with GPU acceleration. Set your GitHub repository URL and dataset in the configuration cell, then run all cells sequentially. Results are automatically packaged into a downloadable zip archive.

---

## Citation

If you use the original AOC-IDS framework, please cite:

```bibtex
@INPROCEEDINGS{zhang2024aoc,
  author={Zhang, Xinchen and Zhao, Running and Jiang, Zhihan and Sun, Zhicong and Ding, Yulong and Ngai, Edith C.H. and Yang, Shuang-Hua},
  booktitle={IEEE INFOCOM 2024 - IEEE Conference on Computer Communications},
  title={AOC-IDS: Autonomous Online Framework with Contrastive Learning for Intrusion Detection},
  year={2024},
  pages={581-590},
  doi={10.1109/INFOCOM52122.2024.10621346}
}
```
