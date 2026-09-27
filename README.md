# Dwelling type estimation for the Guadalajara EOD

The 2023 Guadalajara Metropolitan Area Origin–Destination Survey (EOD) does not directly identify the physical type of each dwelling. This project estimates that attribute in four categories using the 2020 Population and Housing Census Expanded Questionnaire, then adjusts the EOD probabilities to Census-derived dwelling-type proportions for each AGEB. The final dataset retains both the probabilities and one sampled category per dwelling.

These are estimates, not observed EOD dwelling types. In particular, the AGEB proportions are inferred from Census microdata and published dwelling characteristics; they are not official dwelling-type counts by AGEB. The complete procedure and its assumptions are described in [Methodology.md](Methodology.md).

## Dwelling categories

| Category | Census `CLAVIVP` codes | Meaning |
|---|---|---|
| `casa_unica` | 1 | House unique to its lot |
| `casa_compartida_duplex` | 2, 3 | House sharing a lot or duplex |
| `departamento` | 4 | Apartment in a building |
| `otra_vivienda` | 5–9 | Other specified private dwelling forms |

Code 99 is unspecified and is not treated as a fifth physical category.

## What the repository contains

The four notebooks follow the analysis in order:

1. [Harmonization](notebooks/01_harmonizacion_datos.ipynb) selects the Census and EOD dwelling fields and groups the observed Census dwelling types.
2. [Individual model](notebooks/02_modelo_individual.ipynb) aligns shared predictors, evaluates a probabilistic model on Census data, and applies it to EOD dwellings.
3. [AGEB proportion estimation](notebooks/03_estimacion_proporciones_agebs.ipynb) uses Census dwelling controls and reweighted Expanded Questionnaire donors to estimate the four-category mix within each AGEB.
4. [Spatial calibration](notebooks/04_calibracion_espacial.ipynb) adjusts individual EOD probabilities to those AGEB estimates and draws a reproducible dwelling category from each adjusted vector.

The [`outputs/`](outputs/) directory contains the intermediate tables, fitted individual model, final dwelling-level file [`eod_tipo_vivienda_calibrado.csv`](outputs/eod_tipo_vivienda_calibrado.csv), and comparison figures. The final CSV includes the dwelling identifier, AGEB, EOD expansion weight, original and calibrated probability dictionaries, and both sampled categories. [Methodology.md](Methodology.md) explains the calculations without requiring readers to open the notebooks.

## Reproducing the analysis

The notebooks were developed with Python 3.13. They use [eodgdl](https://github.com/CentroFuturoCiudades/eodgdl) and [mxcensus](https://github.com/CentroFuturoCiudades/mxcensus) to load the survey and Census inputs, along with NumPy, pandas, SciPy, scikit-learn, matplotlib, joblib, IPython, tqdm, tabulate, and JupyterLab. One way to prepare an environment is:

```bash
python3.13 -m venv .venv
source .venv/bin/activate
python -m pip install "git+https://github.com/CentroFuturoCiudades/eodgdl.git" "git+https://github.com/CentroFuturoCiudades/mxcensus.git" numpy pandas scipy scikit-learn matplotlib joblib ipython tqdm tabulate jupyterlab
cd notebooks
jupyter lab
```

Run the notebooks in numerical order from `notebooks/`: their relative paths expect that working directory. Source loaders may download and cache data on first use. The recorded outputs represent a particular run; changes in source data or dependency versions may change a reproduction.
