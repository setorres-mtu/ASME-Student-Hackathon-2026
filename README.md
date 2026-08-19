# BWB Inverse Design — NSGA-III Optimization Framework

## Pipeline

```
Mission profile ─▶ NSGA-III ─▶ Generate population ─▶ 21 design variables
                                     │
                        ┌────────────┴────────────┐
                    Planform (10)              Structure (11)
                        │                            │
                  Aero surrogate            Structural surrogate
                   (L/D)                    (Mass, Vp, Vf, Stress)
                        └────────────┬────────────┘
                             Constraint check
                          Stress ≤ 335 MPa?
                        NO ──────┴────── YES
                    penalize/reject   multi-objective ranking
                                            │
                                      Pareto front
                                            │
                                    Select final design
                                   (per-mission scoring loss)
```

## Repo structure

```
.
├── README.md
├── bwb_inverse_design.py            # main optimization pipeline (this is the entry point)
├── data/
│   └── bwb_structures_dataset.csv   # 13,720-row structural training set
├── models/
│   ├── ld_surrogate/                # aero forward surrogate (provided)
│   │   ├── predict_ld.py
│   │   ├── regressor.py
│   │   ├── flight_conversion.py
│   │   ├── reg_full.json
│   │   ├── reg_feasible.json        # used by default — see Design notes
│   │   └── aero_design_space.json
│   ├── structural_forward_surrogate.joblib   # cached after first run (auto-generated)
│   └── bwb_inverse_design_models.joblib      # optional: enables ML-seeded init
├── notebooks/
│   └── flight_conditions_structure.ipynb     # trains the optional inverse-seeding model
├── output_summary.csv               # generated: one row per mission
├── pareto_fronts.png                # generated: Mass vs L/D per mission
└── technical_description.docx       # 3-page technical summary
```

## Setup
`requirements`:
```
numpy
pandas
scikit-learn
xgboost
pymoo
joblib
matplotlib
```

## Running

```bash
python bwb_inverse_design.py
```

On first run this trains and caches the structural forward surrogate
(`models/structural_forward_surrogate.joblib`) from `data/bwb_structures_dataset.csv`;
subsequent runs load the cache. Delete the cache file to force a retrain (e.g. after
changing the training data or model hyperparameters).

If `models/bwb_inverse_design_models.joblib` is present (produced by
`notebooks/flight_conditions_structure.ipynb`), part of each mission's initial
population is seeded from it; otherwise initialization falls back to Latin
Hypercube Sampling. Both paths are fully supported.

Output: `output_summary.csv` (21 design variables + Mass/L/D/Vp/Vf/Stress/Loss per
mission) and `pareto_fronts.png` (Mass vs L/D Pareto front per mission, selected
design highlighted).

## Results

Three mission profiles, each optimized independently (flight condition and soft
targets are mission-specific; only `Stress ≤ 335 MPa` is shared and hard):

| Case | Mass (kg) | L/D | L/D target | Vp (m³) | Vp target | Vf (m³) | Vf target | Stress (MPa) | Loss |
|---|---|---|---|---|---|---|---|---|---|
| 1 — High Speed Dash | 30.2 | 6.23 | 6.0 | 0.777 | 0.75 | 0.324 | 0.45 | 310.5 | 0.298 |
| 2 — Max Endurance | 35.0 | 10.02 | 10.0 | 0.747 | 0.80 | 0.359 | 0.45 | 183.2 | 0.334 |
| 3 — Max Capacity | 20.8 | 15.76 | 15.0 | 0.235 | 1.00 | 0.123 | 0.65 | 192.8 | 0.481 |

Cases 1–2 land close to every target simultaneously. Case 3 trades off volume
against mass (see `technical_description.docx`, §4, for why: hitting its volume
targets requires roughly a 9× mass increase, which the scoring formula's fixed
`M_ref = 50 kg` penalizes more than it rewards for closing the volume shortfall —
a genuine property of the specified scoring metric, not an optimizer failure. The
Pareto front for case 3 does contain a design that meets both volume targets at
~170 kg; see the technical description for the trade-off numbers).

## Design notes

- **Algorithm**: NSGA-III (`das-dennis` reference directions, `n_partitions=8` → 165
  points, `pop_size` matched to that count) — chosen over NSGA-II for its
  reference-direction-guided diversity, which handles the 4-objective front better.
- **Mixed-integer variables** (`# of Ribs`, `# of Fuselage Spars` — integer;
  `# of Fuselage Ribs` — odd integers only) are handled via a repair operator run
  after every crossover/mutation, keeping the rest of the search in continuous SBX/PM.
- **Structural forward surrogate**: 4 independent XGBoost regressors (design →
  Mass/Vp/Vf/Stress), each trained on `log1p(target)` since all four targets are
  strictly positive and heavily right-skewed in the real data (Mass skew=2.6,
  Stress skew=112). The mass model additionally uses monotonic constraints
  (mass non-decreasing in thickness/count variables) and predictions are clamped
  to the observed training range as a second safeguard — both added after an
  earlier version was found to extrapolate to negative mass at the low-material
  corner NSGA-III searches hardest.
- **Aero surrogate**: loads `reg_feasible.json` (not the default `reg_full.json`),
  since it's trained on the narrower 3,860-row envelope that matches this
  project's search box almost exactly, and measurably outperforms the full-data
  checkpoint there (L/D R²=0.994 vs 0.989). Feature construction was verified
  bit-for-bit against the reference `predict_ld()` implementation before use.
  An aero-envelope violation counter (using the published `RANGE_B` Re_L/M_inf
  bounds) is tracked and reported alongside the structural clamp counters.
- **Initialization**: Latin Hypercube Sampling by default; if the optional inverse
  design model is available, part of the population is seeded from it (queried
  near each mission's targets) for a warm start, with LHS filling the remainder.
