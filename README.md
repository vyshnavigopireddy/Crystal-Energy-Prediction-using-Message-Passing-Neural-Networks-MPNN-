# Crystal Energy Prediction using Message Passing Neural Networks (MPNN)

A message passing neural network (MPNN) that predicts the energy of crystal structures. Each structure is converted into a graph (atoms as nodes, interatomic distances as edges) and the model predicts the **per-atom energy**. The final model is an **ensemble of 3 MPNNs per fold**, evaluated with **5-fold cross-validation** on 200 structures and compared against Random Forest, Gradient Boosting and Ridge baselines.

Developed during a research internship at **IIT Guwahati** (May - Jul 2025).

## Results

5-fold cross-validation, per-atom energy, evaluated on unseen validation folds only (mean ± std across folds):

| Model | MAE (eV/atom) | RMSE (eV/atom) | R² |
|-------|---------------|----------------|----|
| **MPNN ensemble (3 models per fold)** | **1.400 ± 0.308** | **3.673 ± 0.978** | **0.923 ± 0.022** |
| MPNN single model | 1.498 ± 0.376 | 3.767 ± 1.025 | 0.918 ± 0.023 |

Per-fold ensemble MAE: 0.957, 1.919, 1.438, 1.326, 1.359 eV/atom. Ensemble MAE on total energy: 8.40 ± 1.85 eV.

### Comparison with baselines

The baselines use element-fraction features and were evaluated on a single random 80/20 split (see Notes).

| Model | MAE (eV/atom) | RMSE (eV/atom) | R² |
|-------|---------------|----------------|----|
| Mean prediction (naive) | 10.632 | - | - |
| Ridge regression | 50.975 | 51.159 | -14.594 |
| Gradient Boosting | 9.108 | 11.926 | 0.153 |
| Random Forest | 6.801 | 9.062 | 0.511 |
| **MPNN ensemble (5-fold CV)** | **1.400** | **3.673** | **0.923** |

### Ablation study (5-fold CV)

Full results: [`mpnn_ablation_results.csv`](mpnn_ablation_results.csv). Variants use a single model (no ensemble) except the baseline row.

| Configuration | MAE (eV/atom) | RMSE (eV/atom) | R² |
|---------------|---------------|----------------|----|
| Baseline (RBF, cutoff 5.0 Å, hidden 128, 3 layers, ensemble of 3) | 1.400 ± 0.308 | 3.673 ± 0.978 | 0.923 ± 0.022 |
| No RBF (raw distance as edge feature) | 1.456 ± 0.335 | 3.783 ± 1.106 | 0.918 ± 0.026 |
| No ensemble (single model) | 1.498 ± 0.376 | 3.767 ± 1.025 | 0.918 ± 0.023 |
| Cutoff 3.0 Å | 1.329 ± 0.232 | 3.608 ± 0.875 | 0.925 ± 0.019 |
| Cutoff 4.0 Å | 1.922 ± 0.441 | 3.875 ± 0.882 | 0.913 ± 0.019 |
| Cutoff 6.0 Å / 7.0 Å | 1.498 ± 0.376 | 3.767 ± 1.025 | 0.918 ± 0.023 |
| Hidden size 32 | 1.536 ± 0.332 | 3.772 ± 1.135 | 0.919 ± 0.027 |
| Hidden size 64 | 1.501 ± 0.340 | 3.766 ± 1.003 | 0.919 ± 0.022 |
| Hidden size 256 | 1.465 ± 0.358 | 3.762 ± 0.986 | 0.919 ± 0.023 |
| 1 message-passing layer | 1.351 ± 0.226 | 3.624 ± 0.915 | 0.925 ± 0.020 |
| 2 message-passing layers | 1.489 ± 0.302 | 3.753 ± 0.970 | 0.919 ± 0.023 |
| 4 message-passing layers | 1.469 ± 0.364 | 3.740 ± 0.979 | 0.920 ± 0.021 |

At cutoffs of 5.0 Å and above, the graphs stop changing (average node degree stays at 5.0), so the 5.0, 6.0 and 7.0 Å single-model runs give identical results.

### Learning curve

Validation MAE falls from 7.98 eV/atom (20 training structures) to 1.29 eV/atom (160 training structures).

## Method

| Step | Details |
|------|---------|
| Data | 200 crystal structures from `st.zip` (Quantum ESPRESSO input files, `.geom.in`, read with ASE); total energies from `output1.xlsx` (range -555.79 to -125.92 eV) |
| Cleaning | 3-sigma outlier filter on energies (all 200 structures kept) |
| Target | Per-atom energy (total energy / number of atoms), mean-centred |
| Graph | Atoms are nodes (element embedding); atom pairs closer than 5.0 Å are connected by edges |
| Edge features | Interatomic distance expanded with 16 Gaussian radial basis functions (centres 0.8 - 6.0 Å, width 0.4) |
| Model | 3 message-passing layers, hidden size 128, mean aggregation, residual node update, average-pooling readout, MLP output head, dropout 0.1 |
| Training | L1 loss, Adam (lr 5e-4, weight decay 1e-5), ReduceLROnPlateau, early stopping (patience 60), up to 400 epochs, batch size 8, seed 42 |
| Validation | 5-fold CV, 160 training / 40 validation structures per fold, folds assigned by composition label |
| Ensemble | 3 models per fold with different seeds (15 trained models, saved as `.pt` files); prediction is the mean, uncertainty is the standard deviation |

## Predictions on unseen structures

`Sample6.zip` contains 35 unseen structures (6 atoms each, for example BaLa, HfNb, ScV, YZr). They are predicted with the 3-model ensemble from the last fold (`mpnn_fold5_ens1-3.pt`).

`predicted_energies.csv` columns: `filename, natoms, energy_eV, per_atom_eV, ensemble_std_eV_per_atom`.

Predicted per-atom energies range from -67.30 eV/atom (HfTa) to -35.92 eV/atom (KZr).

## Figures

Each figure is provided as `.png` and `.eps`.

| File | Shows |
|------|-------|
| `fig_A_learning_curve` | Training vs validation MAE as the training set grows, with baseline lines |
| `fig_B_parity_plot` | Ensemble predicted vs true per-atom energy on unseen validation data |
| `fig_C_model_comparison` | MAE of naive, Ridge, Random Forest, Gradient Boosting, MPNN single and MPNN ensemble |
| `fig_D_cutoff_sensitivity` | Validation MAE vs edge cutoff radius |
| `fig_E_fold_maes` | Single-model vs ensemble MAE for each fold |
| `fig_F_uncertainty` | Ensemble standard deviation vs absolute error (uncertainty calibration) |

Combined dashboards: `mpnn_analysis_dashboard.png` (main results) and `mpnn_ablation_dashboard.png` (ablation study).

## Repository structure

```
.
├── Untitled.ipynb               # Full pipeline: data loading, graphs, baselines, CV training, predictions, plots, ablation
├── st.zip                       # 200 training structures (.geom.in)
├── output1.xlsx                 # Energy labels (columns: file_id, energy)
├── Sample6.zip                  # 35 unseen structures for prediction
├── predicted_energies.csv       # Predicted energies and ensemble uncertainty
├── mpnn_fold{1-5}_ens{1-3}.pt   # Trained model weights (5 folds x 3 ensemble members)
├── mean_per_atom.pt             # Mean per-atom energy used for centring
├── mpnn_ablation_results.csv    # Ablation study results
├── mpnn_analysis_dashboard.png  # Main results dashboard
├── mpnn_ablation_dashboard.png  # Ablation dashboard
└── fig_A ... fig_F (.png/.eps)  # Individual figures
```

## Setup

Versions used in the notebook (CPU only):

```bash
pip install torch==2.0.1 torchdata==0.6.1 dgl==1.1.2 numpy==1.24.4 pandas ase matplotlib openpyxl
```

Open `Untitled.ipynb` and run the cells in order. The notebook uses these paths, so edit them for your machine:

- `st.zip` is extracted to `/content/structures`
- Energy labels are read from `C:\Users\vyshn\Documents\internship\output1.xlsx`
- `Sample6.zip` is read from `C:\Users\vyshn\Documents\internship\`

## Notes

- Small dataset: 200 structures.
- Every structure has a unique composition label, so the composition-based folds act as a grouped random 5-fold split.
- Ensemble members use the validation fold for early stopping and checkpoint selection.
- Baselines use only element-fraction features and a single 80/20 split, so they are not directly comparable to the cross-validated MPNN numbers. Ridge regression was numerically unstable (R² < 0).
- The plotting cells (dashboard and individual figures) use numbers copied from the training log rather than recomputing them.

## Tech stack

Python, PyTorch, DGL, ASE, NumPy, Pandas, Matplotlib

## Author

**Vyshnavi Gopireddy**
B.Tech CSE, Lakireddy Bali Reddy College of Engineering
[LinkedIn](https://www.linkedin.com/in/vyshnavi-gopireddy-9b587031a) | [GitHub](https://github.com/vyshnavigopireddy)
