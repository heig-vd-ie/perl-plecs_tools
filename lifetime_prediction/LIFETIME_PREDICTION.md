# IGBT Lifetime Prediction Workflow

A complete simulation, electro-thermal analysis, and reliability assessment pipeline for power semiconductor modules. This repository combines real-time PLECS circuit simulation with junction temperature thermal stress analysis, rainflow cycle counting, accumulated damage calculation (Miner's Rule), dynamic sweep comparisons, and Monte Carlo parameter uncertainty evaluation.

---

## Features

- **PLECS Integration**: Automated connection and simulation execution via PLECS Standalone XML-RPC interface (`plecs_sim_library.py`).
- **Flexible Execution Modes**: Supports both standard time-domain transient simulations (`simulate`) and steady-state thermal/electrical analysis (`analyze`).
- **Multi-Parameter Sweeping**: Run multi-variable parameter sweeps (e.g., thermal models, load currents) with concurrent cycle counting and automatic comparative visualization.
- **Rainflow Counting**: Advanced 2D rainflow counting algorithm categorizing thermal stress into matrix histograms ($\Delta T_j$ vs. $T_{\text{mean}}$).
- **Multiple Lifetime Models**: Built-in implementations of classic and modern physics-of-failure lifetime estimation models:
  - Coffin-Manson
  - Modified Coffin-Manson (Arrhenius temperature dependence)
  - Norris-Landzberg (frequency dependence)
  - Bayerer (2008) (includes $t_{\text{on}}$, current $I$, voltage $V$, bond-wire diameter $D$, and $T_{j,\text{min}}$)
  - Semikron $\eta\rho$-model (2013) (includes bond-wire aspect ratio $ar$, $t_{\text{on}}$, diode factor)
- **Cumulative Damage & Lifetime**: Linear damage accumulation via **Miner's Rule** ($D = \sum \frac{n_i}{N_{f,i}}$) and total operational lifespan conversion ($T_{\text{sim}} / D$).
- **Monte Carlo Uncertainty Analysis**: Evaluates parameter variations (uniform, Gaussian/normal distributions) on model coefficients to produce statistical reliability distributions.
- **Interactive Visualizations**: Overlay time-domain junction temperature traces and interactive 3D/bar plots powered by Plotly.

---

## Prerequisites & Dependencies

### Python Environment
Ensure Python 3.8+ is installed along with the required packages:
```bash
uv pip install requirements.txt
```

### Required Libraries
The notebook relies on two local Python modules located in `src/`:
- `src/plecs_sim_library.py` (PLECS XML-RPC interface and simulation control)
- `src/igbt_lifetime_library.py` (Rainflow counting, lifetime models, damage accumulation, Monte Carlo analysis)

### Software Setup
- **PLECS Standalone**: Must be running with the **XML-RPC interface enabled** (default port `1080`).
- > **Environment:** To use the following project, you must set up a Windows environment. To connect Plecs to the Python environment, both systems must be running the same operating system. Instructions for setting up the Windows environment are available in the [Environment_Installation](https://github.com/heig-vd-ie/perl-plecs_tools/blob/main/Environment_Installation.md).

---

## Pipeline Overview

```text
PLECS simulation(s) ──► Tj waveform(s) ──► Rainflow counting ──► Cycles (ΔT, Tm, n)
                                                                         │
                                                                Lifetime model (Nf)
                                                                         │
                                                               Miner's rule → Damage
                                                                         │
                                                               Lifetime = T_sim / D
                                                                         │
                                                    (optional) Monte Carlo → distribution
```

When a **parameter sweep** is configured, every simulation step is analyzed independently. Each step receives its own rainflow histogram and lifetime estimate, accompanied by automated summary comparison charts.

---

## Supported Lifetime Models

| Model Key | Name | Key Dependencies / Formula Overview |
|---|---|---|
| `coffin_manson` | Coffin-Manson | $N_f = A \cdot \Delta T_j^{-n}$ |
| `modified_coffin_manson` | Modified Coffin-Manson | $N_f = A \cdot \Delta T_j^{-n} \cdot \exp\left(\frac{E_a}{k_B \cdot T_{j,\text{mean}}}\right)$ |
| `norris_landzberg` | Norris-Landzberg | Extends Modified Coffin-Manson with cycling frequency $f^m$ |
| `bayerer_2008` | Bayerer (2008) | Includes $t_{\text{on}}$, current $I$, voltage $V$, bond-wire diameter $D$, $T_{j,\text{min}}$ |
| `semikron_2013` | Semikron $\eta\rho$-model (2013) | Includes bond-wire aspect ratio $ar$, heating time $t_{\text{on}}$, diode factor |

> 💡 **Constants & Units**:
> - **Boltzmann constant**: $k_B = 1.380649 \times 10^{-23} \text{ J K}^{-1}$
> - **Activation Energy ($E_a$)**: Must be provided in **Joules** ($J$).  
>   *Conversion from eV*: $E_a^{(\text{J})} = E_a^{(\text{eV})} \times 1.602176634 \times 10^{-19}$  
>   *(e.g., $0.63\text{ eV} \rightarrow 1.009 \times 10^{-19}\text{ J}$)*

---

## Configuration & Usage

The notebook is structured so that **Section 1 (Configuration)** contains all target user controls:

```python
# 1. PLECS Connection
PLECS_HOST = "http://localhost:1080/RPC2"

# 2. Model Location & Name
MODEL_FOLDER = r"path/to/plecs/models"
MODEL_NAME   = "20260220_ETPS_DoublePhaseBuck60kHz"  # Without .plecs extension

# 3. Execution Mode & Sweeps
EXEC_MODE     = 'analyze'             # 'simulate' or 'analyze'
ANALYSIS_NAME = 'Steady-State Analysis' # Used when EXEC_MODE = 'analyze'

SWEEP_PARAMS = {
    'Therm_mod': ['file:IMZA120R012M2H-SKG', 'file:IMZA120R017M2H-SKG']
}

# 4. Thermal Trace & Mission Profile
SIGNAL_NAMES    = ['Junction temperature IGBT']
TJ_SIGNAL_INDEX = 0
T_SIM_HOURS     = 0.3 / 60  # Simulation duration in hours

# 5. Lifetime Model Selection & Parameters
MODEL_KEY    = 'modified_coffin_manson'
MODEL_PARAMS = {
    'A'  : 3.3e15,
    'n'  : 4.0,
    'Ea' : 1.009e-19,  # 0.63 eV in Joules
}

# 6. Monte Carlo Uncertainty Evaluation
ENABLE_MC = True
N_MC_RUNS = 300
MC_CONFIG = {
    'A'  : (0.15, 'uniform'),  # ±15% uniform distribution
    'n'  : (0.05, 'normal'),   # 5% Gaussian sigma
    'Ea' : (0.10, 'normal'),   # 10% Gaussian sigma
}
```

---

## Workflow Execution Steps

1. **Step 1 — Connect to PLECS**: Establishes XML-RPC connection to PLECS Standalone.
2. **Step 2 — Load Model**: Loads the `.plecs` circuit schematic.
3. **Step 3 — Run PLECS Simulation(s)**: Executes transient simulations or steady-state analysis across configured parameter sweeps.
4. **Step 4 — Inspect & Plot Signals**: Inspects output traces and renders time-domain thermal profiles ($T_j$).
5. **Step 5 — Rainflow Counting**: Extracts thermal cycles ($\Delta T_j, T_{\text{mean}}$) and constructs rainflow histograms.
6. **Step 6 — Lifetime & Damage Computation**: Evaluates fatigue life $N_f$, applies Miner's rule ($D$), and calculates total operational lifespan.
7. **Step 7 — Comparative & Parameter Sweep Summary**: Generates comparative bar charts and lifetime breakdowns across all sweep variations.
8. **Step 8 — Monte Carlo Analysis**: Runs statistical tolerance iterations to output lifetime probability distributions.

---
