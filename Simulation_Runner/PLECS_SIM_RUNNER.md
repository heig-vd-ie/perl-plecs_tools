# PLECS Python Simulation Runner

A general-purpose Python interface for running **PLECS Standalone** simulations from a Jupyter notebook, with support for parameter sweeps, Monte Carlo analysis, interactive plotting, and CSV export.

Inspired by the Plexim tutorial *"PLECS RPC Interface and Controller Design in Python"*.

---

## Files

| File | Description |
|------|-------------|
| `plecs_sim_library.py` | All functions — connection, simulation, plotting, export |
| `plecs_simulation_runner.ipynb` | Jupyter notebook — the only file you run and edit |

Both files must be in the **same folder**.

---

## Requirements

### Software
- **PLECS Standalone** with the XML-RPC interface enabled (port 1080)
- **Python 3.8+**
- **Jupyter** (classic Notebook or JupyterLab)

### Python packages

```bash
uv pip install -r requirements.txt
```

| Package | Used for |
|---------|----------|
| `xmlrpc.client` | XML-RPC communication with PLECS (built into Python 3) |
| `numpy` | Array processing and downsampling |
| `scipy` | Reading `.mat` simulation data returned by PLECS |
| `plotly` | Interactive charts |
|`jupyterlab` | User interface for Project Jupyter |
| `csv`, `os`, `datetime` | CSV export and file handling (built into Python 3) |

---

## Enabling the PLECS XML-RPC Interface

Before running any simulation:

1. Open PLECS Standalone
2. Go to **Edit → Preferences → General**
3. Check **Enable** next to *XML-RPC interface*
4. Confirm the port is **1080** (or update `PLECS_HOST` in the notebook)
5. **Restart PLECS** for the change to take effect

> PLECS should be open but the model should be close.    
> The XML-RPC connection always uses `localhost`.
> **Environment:** To use the following project, you must set up a Windows environment. To connect Plecs to the Python environment, both systems must be running the same operating system. Instructions for setting up the Windows environment are available in the Environment_Installation.



---

## PLECS Model Setup

Your PLECS model needs one **Outport block** on the **top-level schematic** to expose signals to Python.

1. Add an **Outport** block: Component → Signal Sources → Outport (or search *Out*)
2. Use a **Mux** block to bundle all signals you want to export into a single vector
3. Wire the Mux output into the Outport

The signals arrive in Python as rows of a 2-D array, ordered by Mux input (top to bottom):

| `signal_index` | Signal |
|:-:|---|
| `0` | first signal into the Mux (top) |
| `1` | second signal |
| `2` | third signal |
| … | … |

To discover which index maps to which signal after your first run, call:

```python
inspect_signals(results)
```

This prints a table with the index, min, max, and mean for every signal:

```
Signals available in results (first run):
  Total signals  : 2  -> use signal_index = 0 to 1
  Total samples  : 140000
  Time range     : 0 s  to  0.73 s

  index              min             max            mean
  -------  -------------  -------------  -------------
  0               -0.001          10.01          8.337
  1               -0.023          10.12          8.291
```

---

## Quick Start

### 1. Edit the configuration cell

Open `plecs_simulation_runner.ipynb`. The **Configuration** cell is the only cell you need to edit for each new project:

```python
# Point to your model
MODEL_FOLDER = r"C:\Users\you\PLECS_models"
MODEL_NAME   = "my_model"              # without .plecs

# Define what PLECS receives (see BASE_VARS Modes below)
BASE_VARS = {}

# Choose the simulation type
SIM_MODE = 'sweep'                     # 'sweep' or 'montecarlo'

# Define the sweep
SWEEP_PARAMS = {
    'R' : [0.3, 0.5, 0.8, 1.0],
}

# Name your output signals (in Outport wiring order)
SIGNAL_NAMES = ['Id_ref_A', 'Id_meas_A']

# Configure your plots
PLOT_CONFIG = [
    {
        'signal_index' : [0, 1],
        'signal_name'  : ['Id ref (A)', 'Id meas (A)'],
        'time_window'  : (0.229, 0.234),   # or None for full range
        'title'        : 'D-axis current',
        'ylabel'       : 'Current (A)',
        'show_legend'  : True,
    },
]
```

### 2. Run cells top to bottom

| Step | Cell | What it does |
|------|------|--------------|
| 1 | **Connect** | Opens the XML-RPC connection and loads the model |
| 2 | **Run** | Executes all sweep steps or Monte Carlo runs |
| 3 | **Inspect signals** | Prints the signal index table (run once when setting up) |
| 4 | **Plot** | Renders interactive Plotly charts |
| 5 | **Save CSV** | Writes results to disk (if `SAVE_CSV = True`) |
| 6 | **Close** | Releases the PLECS model |

---

## BASE_VARS Modes

PLECS only overrides the variables you explicitly send. Anything you don't send keeps the value defined in the `.plecs` model's initialization commands.

### Mode A — Python controls all variables

Send every variable your model needs from Python. The model's built-in values are ignored for those keys.

```python
BASE_VARS = {
    'R'          : 0.5,
    'L'          : 3e-3,
    'fc'         : 20e3,
    'Istep'      : 10,
    't_stepRef'  : 0.23,
    't_end'      : 0.73,
}
```

Use this when you want a self-contained script that is fully independent of the `.plecs` file's default values.

### Mode B — PLECS controls all variables, Python changes only swept ones

```python
BASE_VARS = {}   # empty
```

Only the swept or Monte Carlo parameters are sent. Everything else uses the values already defined in the `.plecs` model. Use this when your model has good working defaults and you only want to vary one or two parameters.

---

## Simulation Modes

### Parameter Sweep

Steps one or more parameters through a fixed list of values. Multiple parameters are stepped **together** (zip-style), so all lists must have the same length.

```python
SIM_MODE = 'sweep'

SWEEP_PARAMS = {
    'R' : [0.3, 0.5, 0.8, 1.0],          # single sweep
}

# Co-sweep two parameters together:
SWEEP_PARAMS = {
    'R' : [0.3, 0.5, 0.8, 1.0],
    'L' : [2e-3, 3e-3, 4e-3, 5e-3],      # same length as 'R'
}

SWEEP_LABEL_PARAM = 'R'    # which parameter appears in the plot legend
```

### Monte Carlo

Randomly varies one or more parameters across `N_MC_RUNS` independent simulations. Each parameter gets its own distribution.

```python
SIM_MODE = 'montecarlo'

MC_PARAMS = {
    'R' : (0.5,  0.10, 'uniform'),   # nominal=0.5, ±10%, flat distribution
    'L' : (3e-3, 0.05, 'normal'),    # nominal=3 mH, σ=5%, Gaussian
}

N_MC_RUNS = 50
MC_SEED   = 42    # fixed seed for reproducibility; None = random each time
```

| Distribution | Formula | Use case |
|---|---|---|
| `'uniform'` | Flat in `[nominal×(1−tol), nominal×(1+tol)]` | Component tolerance bands |
| `'normal'` | Gaussian, σ = `nominal × tol` | Manufacturing variation |

---

## Plot Configuration

Each entry in `PLOT_CONFIG` produces one independent figure.

```python
PLOT_CONFIG = [
    {
        # --- which signals to plot ---
        'signal_index' : [0, 1],              # int or list[int]
        'signal_name'  : ['Id ref', 'Id meas'],  # must match signal_index length

        # --- time axis ---
        'time_window'  : (0.229, 0.234),      # (t_start, t_end) in seconds
        # 'time_window': None,                # None = full simulation range

        # --- labels ---
        'title'  : 'D-axis current',
        'xlabel' : 'Time (s)',                # default: 'Time (s)'
        'ylabel' : 'Current (A)',             # default: 'Value'

        # --- legend ---
        'show_legend' : True,                 # False = hide (many runs)
    },
]
```

Charts are fully interactive: scroll to zoom, drag to pan, hover for exact values, click legend entries to toggle individual runs.

### Plot Performance — `MAX_POINTS`

PLECS at 20 kHz switching can produce 500 k–1 M samples per signal per run. Sending all raw points to the browser would take minutes. The `MAX_POINTS` budget is shared across all traces in a figure and handled by vectorised min-max downsampling (fully numpy, < 1 ms per trace even on 1 M-point signals).

```python
MAX_POINTS = 50_000   # default — good for sweeps with few traces
MAX_POINTS = 20_000   # for 20–100 MC runs
MAX_POINTS = 10_000   # for 100–500 MC runs
MAX_POINTS = None     # no downsampling (only for small datasets)
```

> **CSV export always uses the full raw data**, regardless of `MAX_POINTS`.  
> Downsampling is display-only.

After each plot, the library prints a summary:

```
[Plot] 'D-axis current'
       100 trace(s) | 14,000,000 raw pts -> 50,000 rendered (280x reduction) | build 0.31s | render 0.44s
```

---

## Monte Carlo Histogram

When `SIM_MODE = 'montecarlo'`, an additional histogram cell samples each run at a specific time instant and plots the distribution.

```python
MC_HISTOGRAM = {
    'signal_index' : 1,              # which signal to analyse
    't_eval'       : 0.232,          # sample each run at this time (s)
    'signal_name'  : 'Id meas (A)',
    'title'        : 'MC distribution of Id at t = 0.232 s',
    'xlabel'       : 'Current (A)',
    'n_bins'       : 15,
    'show_legend'  : True,
}
```

The histogram shows mean (red dashed) and median (orange dotted) lines, and prints statistics:

```
[Histogram] Id meas (A)  at  t = 0.232 s
  n      = 50
  Mean   = 9.983
  Std    = 0.3214
  Min    = 9.201
  Max    = 10.67
```

---

## CSV Export

Controlled by two settings in the configuration cell:

```python
SAVE_CSV   = True                      # True = export, False = skip
CSV_FOLDER = r"C:\Users\you\sim_data"  # created automatically if missing
```

### Full time-series CSV (`save_results_csv`)

Written for every simulation mode. One row per time sample per run.

| Column | Content |
|--------|---------|
| `run_index` | Integer run number (0-based) |
| `param_R`, `param_L`, … | Swept/varied parameter values for this run |
| `time_s` | Simulation time in seconds |
| `Id_ref_A`, `Id_meas_A`, … | Signal values (names from `SIGNAL_NAMES`) |

Filename: `sim_results_YYYYMMDD_HHMMSS.csv`

### Monte Carlo stats CSV (`save_montecarlo_stats_csv`)

Written only for Monte Carlo runs. One row per run — compact table with values sampled at specific time instants. Useful for distribution analysis without loading the full time-series.

```python
MC_STATS_TIMES = [0.230, 0.232, 0.234]   # time instants to sample
```

Columns: `run_index`, parameter columns, then `SignalName@t=Xs` for each (signal, time) pair.

Filename: `mc_stats_YYYYMMDD_HHMMSS.csv`

---

## API Reference

### `connect_plecs(host)`
Connect to the PLECS XML-RPC server. Returns a `server` object used in all subsequent calls.

```python
server = connect_plecs("http://localhost:1080/RPC2")
```

### `load_model(server, model_folder, model_name)`
Load a `.plecs` model from disk. `model_folder` is the absolute path; `model_name` is the filename without `.plecs`.

### `close_model(server, model_name)`
Close a loaded model and free it in PLECS.

### `inspect_signals(results)`
Print a min/max/mean table for every signal in the first run. Use this to find the correct `signal_index` values.

### `run_sweep(server, model_name, base_vars, sweep_params)`
Run a parameter sweep. Returns a list of result dicts, one per step.

### `run_montecarlo(server, model_name, base_vars, mc_params, n_runs, seed=None)`
Run Monte Carlo simulations. Returns a list of result dicts, one per run.

### `plot_signals(results, plot_config, sim_mode, label_param, max_points)`
Render interactive Plotly charts. One figure per entry in `plot_config`.

### `plot_montecarlo_histogram(results, signal_index, t_eval, ...)`
Render an interactive Plotly histogram for a single signal sampled at a fixed time.

### `save_results_csv(results, signal_names, save_folder, filename=None)`
Write the full time-series to CSV. Returns the file path.

### `save_montecarlo_stats_csv(results, signal_names, t_eval_list, save_folder, filename=None)`
Write a compact Monte Carlo stats table to CSV. Returns the file path.

---

## Results Data Structure

Both `run_sweep()` and `run_montecarlo()` return the same format:

```python
results = [
    {
        'params' : {'R': 0.3},              # parameter values for this run
        'time'   : np.ndarray,              # shape (n_samples,)
        'values' : np.ndarray,              # shape (n_signals, n_samples)
    },
    ...
]
```

Access raw data directly for custom post-processing:

```python
res    = results[0]             # first run
time   = res['time']            # time vector
id_ref = res['values'][0]       # signal_index 0
id_meas= res['values'][1]       # signal_index 1
params = res['params']          # {'R': 0.3, ...}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ConnectionRefusedError` | PLECS XML-RPC not enabled or wrong port | Enable in Preferences, check port number |
| `ConnectionRefusedError` | PLECS not running | Start PLECS first |
| Model not found | Wrong path or filename | Check `MODEL_FOLDER` and `MODEL_NAME` (no `.plecs`) |
| `ValueError: signal_names has N entries but data has M` | `SIGNAL_NAMES` length mismatch | Run `inspect_signals(results)` to count signals |
| Wrong signal on plot | Incorrect `signal_index` | Run `inspect_signals(results)` to map indices |
| Plot is slow or blank | Too many raw points | Lower `MAX_POINTS` or narrow `time_window` |
| Sweep lists different lengths | Co-sweep lists must be equal length | Make all `SWEEP_PARAMS` lists the same length |

---

## Project Structure

```
your_project/
├── plecs_sim_library.py          # library (do not edit)
├── plecs_simulation_runner.ipynb # notebook (edit configuration cell)
└── sim_data/                     # CSV output folder (created automatically)
    ├── sim_results_20240315_143022.csv
    └── mc_stats_20240315_143055.csv
```

Your `.plecs` model file can be anywhere on disk — just set `MODEL_FOLDER` and `MODEL_NAME` in the configuration cell.
