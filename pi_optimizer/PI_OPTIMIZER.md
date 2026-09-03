# PI Controller Optimiser — PLECS

An automated Python tool for tuning **PI controller gains** (Kp, Ki) using closed-loop
step-response simulations in PLECS Standalone. The optimiser runs simulations from a
Jupyter notebook, extracts step-response metrics, and drives a weighted cost function
to its minimum through a three-stage search: grid search → differential evolution →
Nelder-Mead refinement.

Extends the *PLECS Python Simulation Runner* and reuses its connection and simulation
functions from `plecs_sim_library.py`.

---

## Files

| File | Description |
|------|-------------|
| `pi_optimizer_lib.py` | All functions — metrics, cost function, optimisers, plots |
| `pi_optimizer.ipynb` | Jupyter notebook — the only file you run and edit |
| `plecs_sim_library.py` | PLECS connection and simulation runner |


---

## Requirements

### Software
- **PLECS Standalone** with the XML-RPC interface enabled (port 1080)
- **Python 3.8+**
- **Jupyter** (classic Notebook or JupyterLab)

### Python packages

```bash
pip install plotly scipy numpy
```

| Package | Used for |
|---------|----------|
| `xmlrpc.client` | XML-RPC communication with PLECS (built into Python 3) |
| `numpy` | Array processing and metric computation |
| `scipy` | Reading `.mat` data from PLECS; differential evolution and Nelder-Mead optimisers |
| `plotly` | Interactive cost landscape, convergence and response charts |
|`jupyterlab` | User interface for Project Jupyter |
| `copy`, `dataclasses`, `time` | Internal utilities (built into Python 3) |

---

## Enabling the PLECS XML-RPC Interface

Before running any simulation:

1. Open PLECS Standalone
2. Go to **File → PLECS Preferences → General**
3. Check **Enable** next to *XML-RPC interface*
4. Confirm the port is **1080** (or update `PLECS_HOST` in the notebook)
5. **Restart PLECS** for the change to take effect

> PLECS should be open but the model should be closed before the notebook loads it.  
> The Python script and PLECS must run on the **same machine**.  
> The XML-RPC connection always uses `localhost`.

---

## PLECS Model Setup

Your PLECS model must meet two requirements for the optimiser to work.

### 1. Outport block with the relevant signals

Add an **Outport** block on the **top-level schematic** and use a **Mux** to bundle at
minimum the reference (setpoint) signal and the measured output signal:

| `signal_index` | Recommended signal |
|:-:|---|
| `0` | Reference / setpoint signal |
| `1` | Measured / feedback signal |
| `2`, … | Any additional signals (optional) |

To discover which index maps to which signal after your first run, run the
**Inspect Signals** cell. It prints a table with the index, min, max, and mean for every
signal:

```
Signals available in results (first run):
  Total signals  : 2  -> use signal_index = 0 to 1
  Total samples  : 50000
  Time range     : 0 s  to  0.5 s

  index              min             max            mean
  -------  -------------  -------------  -------------
  0                0.000          10.00          7.812
  1               -0.021          10.43          7.756
```

Then set `SIGNAL_IDX_REF` and `SIGNAL_IDX_MEASURED` in the configuration cell accordingly.

### 2. PI gain variables accessible from Python

The PI gains must be declared as named variables in your model's **initialisation
commands** (accessible via **Simulation → Simulation Parameters → Initialization**):

```matlab
Kp = 1.0;
Ki = 100.0;
```

The variable names (here `Kp` and `Ki`) must match `KP_VAR_NAME` and `KI_VAR_NAME`
in the configuration cell. Python overwrites these values before every simulation run.

---

## Quick Start

### 1. Edit the configuration cell

Open `pi_optimizer.ipynb`. The **Configuration** cell is the only cell you need to edit:

```python
# Point to your model
MODEL_FOLDER = r"C:\Users\you\PLECS_models"
MODEL_NAME   = "my_model"           # without .plecs

# PLECS variable names for the gains
KP_VAR_NAME = "Kp"
KI_VAR_NAME = "Ki"

# Signal indices (from the Inspect Signals cell)
SIGNAL_IDX_MEASURED = 1
SIGNAL_IDX_REF      = 0

# Describe the step
SETPOINT      = 10.0    # target value after the step
INITIAL_VALUE = 0.0     # value before the step
T_STEP        = 0.05    # time at which the step occurs (seconds)

# Search bounds
KP_BOUNDS = (0.01, 100.0)
KI_BOUNDS = (1.0, 10000.0)

# Cost function weights (relative — normalised automatically)
COST_WEIGHTS = {
    "rise_time"     : 1.0,
    "overshoot_pct" : 2.0,
    "settling_time" : 2.0,
    "sse"           : 3.0,
    "ise"           : 0.5,
    "itae"          : 0.5,
}
```

### 2. Run cells top to bottom

| Step | Cell | What it does |
|------|------|--------------|
| 1 | **Imports** | Loads all libraries |
| 2 | **Connect** | Opens the XML-RPC connection and loads the model |
| 3 | **Inspect signals** | Prints the signal index table (run once when setting up) |
| 4 | **Cost function** | Builds the weighted cost function and evaluates the initial gains |
| 5 | **Grid search** | Maps the cost landscape over the full (Kp, Ki) space |
| 6 | **Differential Evolution** | Global optimisation — finds the minimum without a starting guess |
| 7 | **Nelder-Mead** | Local refinement — polishes the DE result to high precision |
| 8 | **Convergence plot** | Plots cost vs. evaluation number across all stages |
| 9 | **Best response** | Re-runs PLECS with the optimised gains and plots the annotated step response |
| 10 | **Summary** | Prints the final Kp, Ki, and all step-response metrics |
| 11 | **Verification sweep** | Optional Kp sensitivity sweep around the optimum |
| 12 | **Close** | Releases the PLECS model |

---

## BASE_VARS Modes

PLECS only overrides the variables you explicitly send. Anything you don't send keeps
the value defined in the `.plecs` model's initialisation commands. This applies to
`BASE_VARS` exactly as in the simulation runner.

### Mode A — Python controls all variables

Send every variable your model needs. The model's built-in values are ignored for those keys.

```python
BASE_VARS = {
    'R'    : 0.5,
    'L'    : 3e-3,
    't_end': 0.5,
}
```

### Mode B — PLECS controls all variables, Python changes only the gains

```python
BASE_VARS = {}   # empty
```

Only `Kp` and `Ki` are sent before each run. Everything else uses the values already
defined in the `.plecs` file. Use this when your model has good working defaults.

---

## Optimisation Stages

The optimiser runs three stages in sequence. Each stage feeds its best result into the
next. Any stage can be skipped by setting its flag to `False` in the configuration cell.

### Stage 1 — Grid Search

Evaluates the cost function at every point on a regular (Kp, Ki) grid. Produces a
heatmap of the full cost landscape — useful for understanding the shape of the problem
before committing to an expensive global search.

```python
RUN_GRID_SEARCH = True
N_KP_GRID       = 6    # number of Kp grid points
N_KI_GRID       = 6    # number of Ki grid points (total = N_KP × N_KI simulations)
LOG_SCALE       = True  # use log-spaced grid (recommended for gains)
```

> Skip the grid search on subsequent runs once you are confident the search bounds are
> well chosen. The DE and NM stages alone are then sufficient.

### Stage 2 — Differential Evolution

A gradient-free global optimiser. Maintains a population of candidate solutions and
evolves them over generations. Does not need a starting point and handles multimodal
cost landscapes reliably.

```python
RUN_DE      = True
DE_MAX_ITER = 25    # maximum number of generations
DE_POPSIZE  = 8     # population multiplier (total population = 8 × 2 = 16 candidates)
DE_TOL      = 1e-3  # convergence tolerance on the cost function
DE_SEED     = 42    # fixed seed for reproducibility; change to explore different paths
```

| Parameter | Effect |
|-----------|--------|
| `DE_MAX_ITER` | Increase for a more thorough search; decrease to limit simulation count |
| `DE_POPSIZE` | Larger population = better global coverage, more simulations per generation |
| `DE_SEED` | Fixed seed gives identical results across runs; `None` = random each time |

### Stage 3 — Nelder-Mead Refinement

A gradient-free local simplex method. Starts from the DE result and converges precisely
onto the nearest minimum. Typically requires fewer than 30 additional simulations.

```python
RUN_NM      = True
NM_MAX_ITER = 80
NM_XATOL    = 1e-4   # stop when the change in gains falls below this
NM_FATOL    = 1e-4   # stop when the change in cost falls below this
```

> If the NM result differs significantly from the DE result, the cost landscape has
> multiple local minima. Re-run DE with a larger `DE_POPSIZE` or wider bounds.

---

## Cost Function

The cost function converts a full set of step-response metrics into a single scalar
that the optimiser minimises.

### Metrics

| Metric | Symbol | Definition |
|--------|--------|------------|
| `rise_time` | t_r | Time for the output to travel from 10 % to 90 % of the step amplitude |
| `overshoot_pct` | OS | Peak exceedance above the setpoint, as % of step amplitude |
| `settling_time` | t_s | Last time the output exits the settling band (±`SETTLING_BAND` × amplitude) |
| `sse` | e_ss | Absolute mean error over the last `ss_window` fraction of the post-step window |
| `ise` | — | Integral of squared error: ∫ e² dt |
| `itae` | — | Integral of time-weighted absolute error: ∫ t · \|e\| dt |

### Weights and normalisation

```python
COST_WEIGHTS = {
    "rise_time"     : 1.0,
    "overshoot_pct" : 2.0,   # higher = stricter overshoot constraint
    "settling_time" : 2.0,
    "sse"           : 3.0,   # highest weight = prioritise zero steady-state error
    "ise"           : 0.5,
    "itae"          : 0.5,
}

COST_NORMALISATION = {
    "rise_time"     : 5e-3,    # 5 ms is considered 'average'
    "overshoot_pct" : 10.0,    # 10 % is considered 'average'
    "settling_time" : 20e-3,   # 20 ms is considered 'average'
    "sse"           : 0.1,     # 0.1 signal-units is considered 'average'
    "ise"           : 1e-3,
    "itae"          : 1e-4,
}
```

Only the **relative values** of the weights matter — they are normalised automatically
so they sum to 1. The normalisation constants put all metrics on a similar scale before
weighting; tune them to match the expected magnitudes in your system.

**Common tuning strategies:**

| Goal | Recommended adjustment |
|------|----------------------|
| No overshoot | Set `"overshoot_pct"` weight to `5.0` or higher |
| Fast response, overshoot acceptable | Increase `"rise_time"`, reduce `"overshoot_pct"` |
| Smooth, no-overshoot (ITAE-style) | Set `"itae"` to `2.0`, `"overshoot_pct"` to `3.0`, others to `0.5` |
| Zero steady-state error priority | Set `"sse"` to `5.0` |

---

## Settling Band

The settling criterion determines when the response is considered settled:

```python
SETTLING_BAND = 0.02   # ±2 % of the step amplitude
```

The output must enter and **stay within** `setpoint ± SETTLING_BAND × amplitude` for
the settling time to stop increasing. This matches the standard control engineering
definition. The settling band is shown as a green shaded region on the best-response
plot.

---

## Log-Scale Search

```python
LOG_SCALE = True   # recommended for PI gains that span decades
```

When `True`, the grid search uses `geomspace` (logarithmically spaced points) and both
DE and NM operate in log₁₀(Kp) / log₁₀(Ki) space. This ensures the optimiser
explores small and large gains equally — a linear grid would cluster almost all points
near the upper bound.

Set `LOG_SCALE = False` only when the gains are already well localised within less than
one decade.

---

## Output Plots

### Cost Landscape (grid search)

An interactive heatmap of cost over the (Kp, Ki) grid. DE and NM evaluation points are
overlaid as scatter markers, colour-coded by stage. The global best is marked with a
yellow star.

The colour scale is clipped at the 95th percentile of finite cost values to prevent
a few very bad points from compressing the colour range.

### Convergence History

A line chart of cost vs. evaluation number with a running-minimum envelope. Each
optimiser stage (GRID, DE, NM) is shown in a distinct colour. Hover over any point
to see the exact Kp, Ki, and cost values.

```
[Grid] grey markers     — exhaustive grid evaluations
[DE]   orange markers   — differential evolution population evaluations
[NM]   pink markers     — Nelder-Mead simplex steps
       blue dashed line — running minimum (best cost seen so far)
```

### Best Step Response

An annotated time-domain plot of the optimised closed-loop response, showing:

- Measured output (solid blue) and reference (dashed orange)
- Settling band (green shaded region, ±`SETTLING_BAND`)
- Rise time annotation at the 90 % crossing
- Settling time marker (vertical dotted green line)
- Overshoot annotation at the peak (only shown if OS > 0.1 %)

```python
PLOT_TIME_WINDOW = (0.04, 0.30)   # zoom in seconds; None = full simulation
SIGNAL_NAME_MEAS = "Id measured"  # trace label
SIGNAL_NAME_REF  = "Id reference" # reference label
```

---

## API Reference

### `compute_step_response_metrics(time, measured, setpoint, initial_value, t_step, ...)`
Extract rise time, overshoot, settling time, SSE, ISE, and ITAE from a single
simulation result. Returns a `StepMetrics` dataclass.

```python
metrics = compute_step_response_metrics(
    time          = res['time'],
    measured      = res['values'][1],
    setpoint      = 10.0,
    initial_value = 0.0,
    t_step        = 0.05,
    settling_band = 0.02,   # ±2 %
    ss_window     = 0.05,   # last 5 % of the post-step window for SSE
)
```

### `build_cost_function(weights, normalisation)`
Return a callable `cost_fn(metrics) → float`. Pass the result to all evaluation and
optimisation functions.

```python
cost_fn = build_cost_function(
    weights       = COST_WEIGHTS,
    normalisation = COST_NORMALISATION,
)
```

### `evaluate_pi(server, model_name, base_vars, kp, ki, ...)`
Run one simulation with a given (Kp, Ki) pair and return `(metrics, cost)`. Used
internally by all optimisation stages and available for standalone evaluation.

```python
metrics, cost = evaluate_pi(
    server, model_name, BASE_VARS,
    kp=1.5, ki=200.0,
    kp_var_name="Kp", ki_var_name="Ki",
    setpoint=10.0, initial_value=0.0, t_step=0.05,
    signal_idx_meas=1,
    cost_fn=cost_fn,
    verbose=True,
)
```

### `grid_search_pi(server, model_name, base_vars, kp_range, ki_range, n_kp, n_ki, ...)`
Exhaustive 2-D grid search. Returns `(KP, KI, COST, history)` where `KP`, `KI`, and
`COST` are 2-D arrays of shape `(n_ki, n_kp)`.

### `optimize_pi_differential_evolution(server, model_name, base_vars, kp_bounds, ki_bounds, ...)`
Global optimisation with SciPy's differential evolution. Returns `(best_kp, best_ki, history)`.

### `optimize_pi_nelder_mead(server, model_name, base_vars, kp_start, ki_start, ...)`
Local refinement with Nelder-Mead simplex. Returns `(best_kp, best_ki, history)`.

### `plot_cost_landscape(KP, KI, COST, history, ...)`
Render an interactive heatmap of the cost grid with optional history overlay.

### `plot_convergence(history, title, show_best)`
Render an interactive cost-vs-evaluation chart across all optimisation stages.

### `plot_best_response(server, model_name, ..., best_kp, best_ki, ...)`
Re-run PLECS with the optimised gains and render an annotated step-response chart.
Returns the `StepMetrics` of the best run.

### `print_optimization_summary(best_kp, best_ki, metrics, cost)`
Print a formatted table of the final gains and all step-response metrics.

---

## StepMetrics Dataclass

`compute_step_response_metrics()` and `plot_best_response()` return a `StepMetrics`
object. All fields are accessible as plain attributes:

```python
metrics.rise_time       # float — seconds
metrics.overshoot_pct   # float — percent of step amplitude
metrics.settling_time   # float — seconds
metrics.sse             # float — absolute, same unit as the signal
metrics.ise             # float — unit² · s
metrics.itae            # float — unit · s²

# Convert to a plain dict for custom post-processing
d = metrics.as_dict()
```

---

## OptimHistory Object

A running log of every (Kp, Ki, cost) evaluation. Shared across all three stages so
the convergence plot covers the full optimisation run.

```python
history = OptimHistory()   # created once before the grid search

# Inspect at any point
best_kp, best_ki, best_cost = history.best()
print(len(history))                   # total number of evaluations so far

# Access raw lists for custom analysis
history.kp_list    # list[float]
history.ki_list    # list[float]
history.cost_list  # list[float]
history.tag_list   # list[str]  — 'grid' | 'de' | 'nm'
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ConnectionRefusedError` | PLECS XML-RPC not enabled or wrong port | Enable in Preferences, check port number |
| `ConnectionRefusedError` | PLECS not running | Start PLECS first |
| Model not found | Wrong path or filename | Check `MODEL_FOLDER` and `MODEL_NAME` (no `.plecs`) |
| All costs are `1e6` | Simulation crashes for every (Kp, Ki) | Check `BASE_VARS` and that the model runs manually in PLECS |
| Cost never below initial | Bounds too narrow | Widen `KP_BOUNDS` and `KI_BOUNDS`; run the grid search to inspect the landscape |
| Optimiser converges to wrong minimum | Multiple local minima | Increase `DE_POPSIZE` to `15`+, or widen bounds |
| NM result very different from DE | Local minima near the DE result | Increase `DE_MAX_ITER` or restart NM from a different point |
| `ValueError: Step amplitude is zero` | `SETPOINT == INITIAL_VALUE` | Set distinct values for `SETPOINT` and `INITIAL_VALUE` |
| Metrics are all `inf` | Step occurs after simulation ends | Increase simulation duration in PLECS or reduce `T_STEP` |
| Wrong signal evaluated | Incorrect `SIGNAL_IDX_MEASURED` | Run the **Inspect Signals** cell and read the index table |
| Rise time equals full window | Output never crosses 90 % threshold | Controller gains too low; widen `KP_BOUNDS` upward |

---

## Project Structure

```
your_project/
├── pi_optimizer.ipynb        # notebook (edit configuration cell)
├──src 
    ├──plecs_sim_library.py      # existing PLECS runner library (do not edit)
    ├── signal_analysis_lib.py    # existing signal analysis library (do not edit)
    ├── pi_optimizer_lib.py       # PI optimiser library (do not edit)
└── your_model.plecs          # can be anywhere on disk
```

Your `.plecs` model file can be anywhere on disk — just set `MODEL_FOLDER` and
`MODEL_NAME` in the configuration cell.
