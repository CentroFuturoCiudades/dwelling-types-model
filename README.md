# Dwelling type estimation for the Guadalajara EOD

This repository estimates a four-category dwelling type for dwellings in the 2023 Guadalajara Metropolitan Area Origin–Destination Survey, using the 2020 Census Expanded Questionnaire. It provides dwelling-level probabilities and one reproducible categorical sample. The initial results are **under review**; EOD dwelling types are not directly observed.

The study design, evaluation, results, and limitations are described in [Methodology.md](Methodology.md).

## Run locally

Use Python 3.13 or newer. Create an environment and install the notebook dependencies:

```bash
python3.13 -m venv .venv
source .venv/bin/activate
python -m pip install "git+https://github.com/CentroFuturoCiudades/eodgdl.git" "git+https://github.com/CentroFuturoCiudades/mxcensus.git" numpy pandas scikit-learn matplotlib joblib ipython jupyterlab
cd notebooks
jupyter lab
```

Run `01_harmonizacion_datos.ipynb` before `02_modelo_individual.ipynb`. The first notebook loads source data through [eodgdl](https://github.com/CentroFuturoCiudades/eodgdl) and [mxcensus](https://github.com/CentroFuturoCiudades/mxcensus); these loaders may download and cache data on first use. The second notebook reads the generated CSVs from `outputs/` and writes the model, EOD results, and distribution figure there. Run the notebooks from the `notebooks/` directory because their output paths are relative to it.

The notebooks record the source-package versions used for the reported run. Installing newer package versions may change the results.
