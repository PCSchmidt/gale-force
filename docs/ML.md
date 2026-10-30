# Gale Force - Machine Learning Specification

**Status:** Draft v1 - design only
**Standard:** Problem formulation, data, baseline, metrics, ablations, reproducible training, export, model card, CI evaluation. Thresholds are targets to justify.

## 0. Common infrastructure
Headless deterministic simulation with Python bindings and Gymnasium-style multi-agent environments; vectorized rollouts; seeds fixed per scenario; train/val/test split by scenario seed and venue; PyTorch; config-driven, seeded training; experiment tracking; ONNX export with parity test; bootstrap CIs; model cards.

## 1. Tactical sailing policy (flagship)
- **Problem:** choose helm and trim decisions in a fleet race to minimize finishing position subject to the rules engine.
- **Formulation:** partially observable multi-agent Markov game.
  - *Observations:* own boat state (speed, heel, heading), apparent/true wind estimate from instruments, nearby boats' relative positions and tacks, mark geometry, laylines, and a short history to infer shifts. Same information a human sees; no oracle wind.
  - *Actions:* hierarchical: strategic (tack/gybe now, hold, go for a side) at 1 Hz; low-level (helm and trim) by a controller. Option: continuous low-level control learned in a second stage.
  - *Reward:* finishing position (sparse) shaped by progress toward the mark (VMG) and penalties for rule violations.
- **Training:** PPO or comparable, self-play against a population (scripted, past checkpoints) with curriculum (flat water → shifts → fleet → rules).
- **Baselines:** polar-following bot; tack-on-headers heuristic bot; random-tack bot.
- **Metrics:** mean finishing position vs the population; win rate; rule-violation rate; exploitability (best-response training against the fixed policy); robustness to unseen venues and wind scripts.
- **Ablations:** reward shaping terms; observation history length; hierarchy vs flat; population size.
- **Risks:** degenerate strategies exploiting rules logic (circling, camping the line). Mitigation: rules-aware penalties, exploit review, held-out venues.

## 2. Shift and puff estimation
- **Problem:** infer persistent vs oscillating wind shifts and upcoming puffs from instrument history and visible water texture.
- **Models:** (a) classical filtering (Kalman filter on wind direction with oscillation model), (b) learned sequence model (GRU/small Transformer), (c) vision model on rendered water texture for puff approach (stretch).
- **Metrics:** direction prediction error at 30-120 s; puff arrival-time error; downstream effect on bot finishing position.
- **Honesty requirement:** performance is simulator-specific; no claims about real-world sailing.

## 3. Boat-model calibration (system identification)
- **Problem:** fit sail and hull coefficients so the model's polars match published performance data.
- **Approach:** Bayesian optimization / gradient-free search over coefficient parameters against reference polars with regularization; report parameter uncertainty and residuals.
- **Metrics:** polar error (percent) over the TWA×TWS grid; parameter identifiability.
- **Value:** a clean "model calibration against data" case with uncertainty quantification.

## 4. Skill rating and matchmaking
Compare Elo, Glicko-2 and TrueSkill-style ratings for fleet racing (variable participants) on bot tournaments; metrics: predictive log-likelihood of finishing orders, rating convergence, and calibration. Publish the comparison.

## 5. Integrity: anomaly detection
Detect impossible control rates and oracle-assisted play in input logs; synthetic cheater policies (oracle wind, instantaneous trim); recall at fixed false-positive rate.

## 6. Release checklist and evaluation matrix
Per model: config/seed/commit/data hash; one-command reproduction; held-out evaluation with CI; ONNX parity; model card; negative results retained.

| Component | Primary metric | Baseline | Split |
|---|---|---|---|
| Tactical policy | Mean finishing position, exploitability | Polar-following + heuristics | Held-out venues/wind scripts |
| Shift estimator | Direction error, downstream effect | Kalman filter | Held-out seeds |
| Calibration | Polar % error with uncertainty | Hand-tuned coefficients | Reference polars |
| Ratings | Predictive log-likelihood | Elo | Bot tournaments |
| Anomaly | Recall @ fixed FPR | Rule thresholds | Held-out replays |
