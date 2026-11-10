# Structural Steel Material Flow Analysis and Emissions Scenarios, 1932–2100

This repository holds the code for:

> **[August M. Organschi, Jay H. Arehart, Wil V. Srubar III]** ([Year]). *[WORKING: Feasibility Analysis of A U.S. Structural Steel Circular Economy through 2100]*. **[Journal]**. https://doi.org/[DOI]

The code projects demand for structural steel to 2100, models the in-use structural steel stock in the United States with a dynamic material flow analysis (MFA), and estimates the embodied CO₂ emissions and energy use under different scenarios for building lifetime, steel reuse, and green steel adoption.

---

## Workflow overview

The analysis is split across three Jupyter notebooks. Run them in this order:

```
input_data/ ──► 1. model_development.ipynb ──► output_data/steel_forecast.csv
                                                        │
                        ┌───────────────────────────────┴──────────────────────┐
                        ▼                                                      ▼
          2. forecast_validation.ipynb                     3. structural_steel_MFA_VER_10.ipynb
           (compare with published forecasts)               (stock model, scenarios, emissions, energy)
```

`model_development.ipynb` must be run first because the other two notebooks read the forecast it writes. After that, notebooks 2 and 3 can run in either order.

### 1. `model_development.ipynb`: demand forecasting

- Loads historical and SSP-projected population, GDP, and final energy use, plus historical steel production.
- Interpolates the projections from 10-year steps to annual values. Population and GDP use cubic interpolation and energy use is linear.
- Fits four regression models of structural steel production against population, GDP, and energy use (SSP2), trained on 1931–2023:
  - Multivariate linear regression
  - Power law
  - Cobb–Douglas (log-linear least squares)
  - An ensemble that averages the three
- Tests each model on the raw data and on 2-, 5-, and 10-year moving averages. Reports adjusted R², RMSE, and MAE.
- Projects every model to 2100 for SSP1–SSP5.
- **Exports** the **power-law model on the 2-year moving average** (historical values and the 2024–2100 forecast for each SSP) to `output_data/steel_forecast.csv`.

### 2. `forecast_validation.ipynb`: validation

- Compares the exported forecast with two external projections:
  - Arehart et al. (2020): steel inflows to the U.S. building stock, by SSP
  - Net Zero Steel: projected U.S. steel production
- Uses the shares of structural steel in construction steel and in total steel to put both sources on the same basis (structural steel only).
- Produces `figures/forecast_validation.png`, which compares SSP2–SSP4.

### 3. `structural_steel_MFA_VER_10.ipynb`: MFA, scenarios, emissions, and energy

**Demand scenarios.** These are taken from the SSP forecasts:

| Scenario | SSP  |
|----------|------|
| Low      | SSP3 |
| Mid      | SSP2 |
| High     | SSP4 |

**Dynamic stock model.** Built with [ODYM](https://github.com/IndEcol/ODYM):
- Building lifetimes follow a Weibull distribution with shape 3.5 and a baseline median of 70 years.
- An inflow-driven baseline sets the in-use stock, which stands for the service level.
- Stock-driven runs then raise the median lifetime from 70 to 110 years, one year at a time, for cohorts that enter the stock after 2024. Each run gives the new inflow needed to keep the same stock, along with outflows and stock by cohort.

**Reuse and green steel scenarios.**
- A grid of 11 × 11 = 121 combinations of the long-run reuse rate ceiling and the green steel share ceiling, each from 0% to 100% in steps of 10%.
- Both rates follow logistic adoption curves with a fixed growth rate G = 0.2 and an inflection year M = 2060.
- Reused outflow is subtracted from demand to give the steel that must be newly produced. That production is then split into green and conventional steel.

> Variables and DataFrames in this notebook are named `qmc` (for example `run_qmc_scenarios`, `df_qmc_mid`) because an earlier version sampled parameters with quasi-Monte Carlo. The current version uses the regular grid described above.

**Emissions and energy factors.** Factors are applied per tonne of conventional, green, and reused steel. Green steel is assumed to emit 33% of the CO₂ of conventional steel and use 113% of its energy. Reused steel accounts only for processing and transport. Factor values are defined in the "Emissions Analysis" section of the notebook.

**Industry target.** A reference pathway that cuts emissions per tonne linearly by 85% between 2024 and 2100, applied to Mid-scenario demand.

**Outputs.** The notebook produces figures of inflows, outflows, outflow/inflow ratios, in-use stock, annual and relative emissions and energy use, and the sensitivity of cumulative 2024–2100 emissions and energy to lifetime, reuse ceiling, and green steel ceiling.

---

## Repository structure

The notebooks set `BASE_DIR = Path.cwd().parent`. They therefore expect to run from a subfolder (shown here as `notebooks/`), with the data and figure folders one level up:

```
.
├── notebooks/
│   ├── model_development.ipynb
│   ├── forecast_validation.ipynb
│   └── structural_steel_MFA_VER_10.ipynb
├── input_data/
│   ├── input_data_population.csv     # population, columns pop_ssp1 … pop_ssp5
│   ├── input_data_GDP.csv            # GDP, columns gdp_ssp1 … gdp_ssp5
│   ├── input_data_production.csv     # steel production, incl. structural-total, total-production
│   ├── final_energy_use.xlsx         # final energy use, column "energy usage"
│   └── validation_data.xlsx          # sheets: arehart validation, net zero steel validation,
│                                     #         construction steel pct, total steel pct
├── output_data/
│   └── steel_forecast.csv            # written by model_development.ipynb
├── figures/                          # all figures are saved here
├── requirements.txt
├── LICENSE
└── README.md
```

`output_data/` and `figures/` must exist before you run the notebooks.

---

## Requirements

- Python 3.10 or later. The analysis was developed with Python 3.14.
- Packages: `numpy`, `pandas`, `scipy`, `scikit-learn`, `matplotlib`, `openpyxl` (needed to read the `.xlsx` inputs), `jupyter`, and `ODYM`.

> **Matplotlib version:** The sensitivity plots call `matplotlib.cm.get_cmap`, which is removed in Matplotlib 3.11. Use Matplotlib earlier than 3.11, as pinned in `requirements.txt`.

### Installation

```bash
git clone https://github.com/[username]/[repo-name].git
cd [repo-name]

python -m venv .venv
source .venv/bin/activate        # on Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

ODYM is installed directly from its GitHub repository, as listed in `requirements.txt`.

---

## Usage

```bash
mkdir -p output_data figures      # if not already present
cd notebooks
jupyter lab                       # or: jupyter notebook
```

Then run each notebook from top to bottom in this order:

1. `model_development.ipynb`
2. `forecast_validation.ipynb`
3. `structural_steel_MFA_VER_10.ipynb`

The MFA notebook runs 41 lifetime cases × 121 reuse/green combinations for each of the three demand scenarios. It can take a few minutes and needs a moderate amount of memory.

### Key parameters

You can change these in the MFA notebook to explore other assumptions:

| Parameter | Default | Where it is set |
|---|---|---|
| Weibull shape | 3.5 | `build_lifetime_scenarios(base_shape=…)` |
| Baseline median lifetime | 70 years | `build_lifetime_scenarios(base_median=…)` |
| Lifetime scenarios | 70–110 years | `build_lifetime_scenarios(medians=…)` |
| Year the lifetime change starts | 2024 | `change_year` |
| Reuse / green adoption speed and inflection year | 0.2 / 2060 | `G_reuse`, `M_reuse`, `G_green`, `M_green` |
| Grid step for the ceilings | 10% | `generate_grid_ceilings(step=…)` |
| Industry target reduction by 2100 | 85% | `TOTAL_REDUCTION` |

---

## Data sources



| File | Source |
|---|---|
| `input_data_population.csv`, `input_data_GDP.csv` | Historical data: [https://doi.org/10.34894/INZBF2]. Projections: Shared Socioeconomic Pathways (SSP) database, [https://ssp.apps.ece.iiasa.ac.at/]. |
| `final_energy_use.xlsx` | [https://climatedeck.rhg.com/us-emissions-and-energy/] |
| `input_data_production.csv` | [https://www.usgs.gov/centers/national-minerals-information-center/iron-and-steel-statistics-and-information, https://doi.org/10.3133/ds140] |
| `validation_data.xlsx` | Arehart, J. H., et al. (2022). [https://doi.org/10.1016/j.resconrec.2022.106583]. Net Zero Steel, [https://netzeroindustry.org/wp-content/uploads/pdf/net_zero_steel_report_ii.pdf]. |



---

## Citation



## License



## Contact

