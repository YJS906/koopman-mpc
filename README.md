# EDMD-Koopman Residual MPC for Vehicle Path Tracking

This repository contains a research prototype for lateral path-tracking MPC using
an E-Corner vehicle simulation plant.

The main study compares three controllers on the same closed-loop plant:

1. Linear bicycle model MPC
2. Fixed residual EDMD-Koopman MPC
3. Online residual EDMD-Koopman MPC with matrix RLS updates

## Core Idea

The VehicleBody plant with a Fiala lateral tire option is used as the
ground-truth simulation plant. The linear bicycle model and Koopman model are
not the plant; they are prediction models inside the MPC controller.

The residual Koopman structure is:

```text
x_nom_next = linear_bicycle_predictor(x_k, delta_k)
residual   = VehicleBody_true_next - x_nom_next
x_pred     = x_nom_next + r_koopman
```

The fixed Koopman controller uses the initial EDMD model without online
updates. The online Koopman controller starts from the same EDMD model and
updates the residual predictor during driving using RLS and observed plant
transition data.

## Important Code

- `vehicle_sim/controllers/path_tracking_mpc/`
  - `linear_bicycle.py`: linear bicycle prediction model
  - `edmd.py`: EDMD-Koopman model identification and prediction utilities
  - `features.py`: Koopman observable/feature construction
  - `mpc.py`: linear, fixed Koopman, and online Koopman MPC controllers
  - `rls.py`: matrix RLS update logic
- `vehicle_sim/utils/direct_ackermann_steering.py`
  - Converts one bicycle steering command `delta_cmd` to front FL/FR
    Ackermann steering angles while rear steering is fixed.
- `vehicle_sim/utils/path_tracking_sim.py`
  - Closed-loop VehicleBody plant simulation helpers, path definitions,
    segment metrics, tire-state logging, friction and speed schedules.
- `vehicle_sim/experiments/edmd_koopman_mpc_mvp.py`
  - Main experiment/search script for EDMD-Koopman MPC benchmarks.
- `vehicle_sim/models/e_corner/tire/lateral/lateral_tire.py`
  - Linear lateral tire and optional Fiala lateral tire model.

## Final Candidate Result

The final poster candidate is kept in:

```text
vehicle_sim/experiments/results/award_ready_online_koopman_benchmark_gap_boost/best_candidate/
```

Large sweep and smoke-test output folders are intentionally ignored by Git and
can be regenerated from the experiment CLI.

## Example Run

Use the bundled Python runtime in the Codex workspace:

```powershell
& 'C:\Users\HOME\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' -m vehicle_sim.experiments.edmd_koopman_mpc_mvp --scenario nonstationary_adaptive_technical_course --out-dir vehicle_sim\experiments\results\manual_run
```

The main experiment uses:

- controller period: `control_dt = 0.05 s` (20 Hz)
- plant integration period: `plant_dt = 0.01 s` (100 Hz)
- single steering command: `delta_cmd [rad]`
- closed-loop plant: `VehicleBody + Fiala tire + Direct Ackermann wrapper`

## Notes

This is a research prototype. The Python/CVX implementation is intended for
simulation validation and poster-level experimentation, not hard real-time
deployment.

Additional explanation notes are available in:

- `docs/koopman_mpc_explanation.md`
- `docs/professor_qa.md`
