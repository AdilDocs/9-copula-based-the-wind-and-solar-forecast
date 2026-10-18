# Copula-based risk-constrained reserve scheduling of multi-service BESSs with SMCS reliability evaluation

Source code for the three-stage framework of the paper:

1. **Stage 1 – Copula-based wind–solar risk modelling** (`src/stage1.py`): month/hour-stratified marginals,
   24-h-ahead marginal forecasts in normal space, pseudo-observations, five dependence models (independence,
   static Gaussian, static Student-t (proposed), dynamic Gaussian, dynamic Student-t with the autoregressive recursion
   driven by realised errors only), joint shortfall probability, hourly joint scenarios and daily synthetic years.
2. **Stage 2 – Risk-constrained reserve scheduling of multi-service BESSs** (`src/risk_sched.py`, `src/env.py`):
   backward required-energy recursion per scenario, (1−δ) quantile, SOC set-point, BESS power with efficiency,
   balancing reserve, and the safety layer (exact radial minimum-shedding / minimum-curtailment dispatch).
3. **Stage 3 – SMCS-based reliability evaluation** (`src/env.py`, `src/common.py`): exponential failure/repair
   histories of supply point, transformers and line sections, EENS, LOLP, LOLE, total cost, and the coefficient of
   variation stopping criterion; all policies are simulated on the same failure histories.

## Data
Download the OPSD time series (60-min, single index) `time_series_60min_singleindex.csv` from
https://data.open-power-system-data.org/time_series/ and place it in `data/`. Only German wind/solar profiles,
generation, capacity, load and day-ahead price columns are used (2015-01-01 to 2020-09-30).

## Installation and run
```bash
pip install -r requirements.txt
bash run_all.sh          # full pipeline; run from this folder
```
All scripts are run from the project root as `python src/<script>.py` and read/write `data/`, `results/`,
`runs/`, `figures/`. The training of the learning-based benchmarks (`jobs_baselines.txt`) is by far the most
time-consuming step; the proposed scheduler itself requires no training.

## Pipeline and outputs
| Step | Script(s) | Output | Paper item |
|---|---|---|---|
| Data | `data_prep.py`, `price_model.py` | `data/de_hourly.pkl` | Test system and data |
| Stage 1 | `run_signals.py`, `run_scenarios.py` | `results/table3_dependence_models.csv`, `signals.pkl` | Table: dependence models |
| Benchmarks (training) | `jobq.py jobs_baselines.txt` → `mappo.py`, `ppo_lag.py`, `maddpg.py` | `runs/*.pt` | Benchmarks |
| Proposed scheduler | `risk_sched.py precompute / tune / evaluate` | `risk_ereq.pkl`, `risk_tuning_2018.csv`, `smcs_risk.pkl` | Stages 2–3 |
| Benchmarks (SMCS) | `evaluate.py`, `eval_extra2.py`, `stress2020_k3000.py` | `smcs_eval.pkl`, `smcs_extra2.pkl`, `stress2020_k3000.pkl` | Table: comparison |
| Alternative Stage-1 models | `alt_stage1.py`, `conformal_sched.py` | `smcs_alt.pkl`, `smcs_conformal.pkl` | Table: Stage-1 models |
| Sensitivity | `sensitivity.py`, `sens_risk.py` | `sensitivity*.pkl` | Sensitivity table, Fig. 4 |
| Figures | `fig2_data.py`, `fig2_risk.py`, `make_report.py`, `fig3_conv.py` | `figures/fig2–fig4` | Figs. 2–4 |
| Tables | `make_tables.py` | `results/paper_table_*.csv` | Result tables |

`results_reference/` contains the tables obtained with the reported runs, for checking a reproduction.

## Notation (paper ↔ code)
| Paper | Code |
|---|---|
| risk level δ | `alpha` (`ALPHAS` in `risk_sched.py`) |
| balancing reserve r, reserve price p^res | `fcr`, `COST["fcr"]` |
| reservation time T_r | `COST["fcr_tau"]` |
| cycle-ageing cost p^cyc, value of lost load p^VOLL | `COST["cyc"]`, `COST["voll"]` |
| horizon H, averaging window L, shortfall quantile q | `SIG["H"]`, `SIG["L"]`, `SIG["q_short"]` |
| copula parameters ω, β, α, ν | `omega`, `beta`, `alpha`, `nu` in `stage1.Copula` |
| required energy E^req | output of `risk_sched.ereq_table` |
| coefficient of variation CV | `cov` / `beta` in the evaluation scripts |
| SOC^res, SOC^min, SOC^max | `TAU_RES`, `SMIN`, `SMAX` |

## Main settings (`src/config.py`)
3-zone RBTS-based radial feeder; BESS 1.5/6, 1.0/4, 1.5/6 MW/MWh, η = 0.95, SOC ∈ [0.1, 1]; two 3.75 MW
transformers; export limit 3 MW; VOLL 10 000 EUR/MWh; reserve price 10 EUR/MW/h with 2 h reservation;
cycle-ageing cost 15 EUR/MWh; H = 24 h, M = 500 scenarios. Train 2015–2017, validation 2018, test 2019,
stress January–September 2020.

## Version notes
- Proposed Stage-1 model: static Student-t copula (`"t"` in the code); the dynamic t copula is kept as a comparison model.
- Failure sampling: every simulated year starts with all components in service and the first time to failure is
  exponential with the nominal rate (an earlier version scaled it by a uniform random number, which inflated the
  failure frequency).
- SMCS sizes: K = 300 for 2019 and K = 3000 for January–September 2020, so that CV ≤ 5% for every reported EENS.
- Shortfall-probability statistics and Fig. 2(a) use M = 8000 draws (`pi_highM.py`); the scheduler uses M = 500.
- `figures_reference/` and `results_reference/` contain the outputs of this version.
