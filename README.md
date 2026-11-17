# Anchor Collapse

**Overfit regressors make generative downscaling overconfident — and cross-fitting fixes it.**

This repository accompanies the paper *Anchor Collapse: Overfit Regressors Make Generative Downscaling Overconfident* (CVPR 2027 submission). It contains the core components of the method: the anchor-gap diagnostic, cross-fitted anchoring, anchor noise, residual EDM and flow-matching training, and the evaluation protocol. The full training pipeline, trained models and the processed test years will be released after review.

---

## TL;DR

Many probabilistic downscalers and restorers are *two-stage*: a regressor predicts an **anchor**, and a diffusion or flow model samples the **residual** around it (CorrDiff-style weather downscaling, residual diffusion for super-resolution).

The generator learns how wrong the anchor can be **from the training set** — exactly where an overfit regressor is almost never wrong. On new data it then collapses onto the anchor and produces sharp, plausible and badly overconfident ensembles. We call this **anchor collapse**.

| | What it is | Cost |
|---|---|---|
| **Diagnose** | Anchor gap `ρ = RMSE_train / RMSE_held-out` of the regressor. The generator's spread–skill ratio tracks `ρ` across checkpoints and architectures. | One evaluation |
| **Fix (CFA)** | *Cross-fitted anchoring*: split the training years into folds and give every year the anchor of a regressor fitted without it. The generator then trains on realistic anchor errors. | `F` extra regressor fits; inference unchanged |
| **Fix (noise)** | Perturb the in-sample anchor with spatially smooth noise of std `α·σ*`, `σ* = sqrt(σ_out² − σ_in²)`. | No refits; one hyperparameter |

On subseasonal 2-m temperature downscaling over China (1.5° → 0.0625°, held-out test years 2016–2017, lead 1), CFA lowers the CRPS of a CorrDiff-style generator by 8% and anchor noise by 11%, while at least doubling its spread–skill ratio.

---

## Repository layout

```
anchor_collapse/
  diagnostics/anchor_gap.py   # ρ and σ* from regressor predictions
  anchors/crossfit.py         # year-wise folds, out-of-fold anchor assembly (with coverage checks)
  anchors/noise.py            # spatially smooth anchor noise, scaled by α·σ*
  models/edm_residual.py      # residual EDM: training step and Heun sampler
  models/flow_residual.py     # residual flow matching: training step and Euler sampler
  metrics/crps.py             # fair (ensemble) CRPS, area-weighted, land mask
  metrics/ssr.py              # spread–skill ratio (simple form used in the paper + Fortin et al. 2014 form)
  metrics/bootstrap.py        # paired bootstrap over initialization dates (members kept together)
configs/                      # E3, E3+CFA, E3+noise
scripts/
  compute_rho.py              # anchor gap of a trained regressor
  make_cf_anchors.py          # build cross-fitted anchors from fold regressors
  evaluate.py                 # CRPS / RMSE / SSR / paired CIs
examples/
  toy_collapse.py             # 2-minute CPU demo: collapse and its fix on synthetic data
tests/
```

## Quick start

```bash
pip install -r requirements.txt

# See anchor collapse on synthetic data (CPU, ~2 min)
python examples/toy_collapse.py

# Anchor gap of a regressor (predictions on training and held-out samples)
python scripts/compute_rho.py --pred-train train_pred.npy --pred-valid valid_pred.npy \
    --target-train y_train.npy --target-valid y_valid.npy --weights area_weights.npy

# Cross-fitted anchors from fold regressors
python scripts/make_cf_anchors.py --folds 2008-2009 2010-2011 2012-2013 2014 \
    --fold-preds fold_*/holdout_pred.npz --out anchors_cf.npz
```

Training a residual generator on cross-fitted anchors only changes which anchor file is passed at training time; sampling uses the regressor fitted on all training years, as usual.

## Data

The experiments use CMA S2S reforecasts (1.5°), CLDAS analyses (0.0625°) and CWRF regional hindcasts (0.125°). These archives cannot be redistributed; they are available on application from the China Meteorological Administration (CMA S2S, CLDAS) and the National Climate Center of China (CWRF). This repository provides the splits (initialization dates by year) and the preprocessing steps, so that the testbed can be rebuilt from the original archives. A processed copy of the test years for evaluation will be released together with the full code.

## Status

- [x] Diagnostics, cross-fitted anchors, anchor noise
- [x] Residual EDM / flow-matching training and sampling
- [x] Evaluation protocol (fair CRPS, SSR, paired bootstrap)
- [ ] Full training pipeline and configs for all baselines
- [ ] Trained models
- [ ] Processed test years for evaluation

## Citation

```bibtex
@inproceedings{anchorcollapse2027,
  title     = {Anchor Collapse: Overfit Regressors Make Generative Downscaling Overconfident},
  author    = {Anonymous},
  booktitle = {Under review},
  year      = {2027}
}
```

## License

MIT for the code. Data remain subject to the terms of their providers.
