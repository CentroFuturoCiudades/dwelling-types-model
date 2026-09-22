# Methodology

*Working draft — initial results are under review. The Census holdout evaluation is promising for probability estimation, but it does not verify dwelling types for individual EOD records.*

## Study objective and data

This project estimates a four-category dwelling type for each dwelling in the 2023 Guadalajara Metropolitan Area Origin–Destination Survey (EOD). The EOD does not record that outcome directly. Supervised information comes from the 2020 Population and Housing Census Expanded Questionnaire for nine Jalisco municipalities: Guadalajara, Ixtlahuacán de los Membrillos, Juanacatlán, El Salto, Tlajomulco, Tlaquepaque, Tonalá, Zapopan, and Zapotlanejo. The analysis unit is a dwelling in both sources. The EOD and Census inputs were loaded with `eodgdl` and `mxcensus`, respectively; no person-level attributes were joined into the model.

The source-selection notebook produced 39,238 Census dwellings and 17,901 EOD dwellings. Census `ID_VIV` was retained as `id_vivienda_censo`; EOD `folio_vivienda` identifies each EOD dwelling. The Census `FACTOR` was retained as `factor_expansion` and the EOD survey weight as `ponderador`. These weights are source-specific and are not interchanged.

## Outcome definition

The Census `CLAVIVP` field was mapped into the analysis outcome `dwelling_type`:

| `dwelling_type` | `clavivp_codigo` | Interpretation |
|---|---|---|
| `casa_unica` | 1 | House unique to its lot |
| `casa_compartida_duplex` | 2, 3 | House sharing a lot or duplex |
| `departamento` | 4 | Apartment in a building |
| `otra_vivienda` | 5–9 | Other specified private dwelling forms |

Code 99 denotes an unspecified dwelling type, not a fifth physical category. The 44 records carrying that unknown outcome were excluded from model fitting and evaluation, leaving 39,194 labeled Census dwellings. The grouping reduces the sparsity of individual `CLAVIVP` codes but also removes distinctions within the combined categories. Code 6 was not observed in the selected Census records, although it remains in the defined mapping.

Among labeled Census dwellings, the `factor_expansion`-weighted outcome proportions were 74.3400% `casa_unica`, 13.5087% `casa_compartida_duplex`, 11.3250% `departamento`, and 0.8263% `otra_vivienda`.

## Predictor harmonization

Seven dwelling-level predictors were used in both sources: `municipio`, `personas_en_vivienda`, `tenencia_vivienda`, `tiene_auto_camioneta`, `tiene_moto`, `tiene_bicicleta`, and `tiene_internet`. Municipalities were mapped to the same nine Spanish `snake_case` labels. Household size was numeric and capped at 10 in the Census; the EOD response `10 y +` was assigned 10. Tenure was mapped to `propia`, `hipotecada`, `rentada`, `prestada`, or `otra`. For owner-occupied Census dwellings, `deuda` separated fully paid ownership from mortgages or stopped payments; `Blanco por pase` for this owner-debt question was interpreted as `propia` by the implemented mapping. Vehicle ownership and Internet access were mapped to `si` and `no`; positive EOD vehicle counts indicated `si`.

The first notebook selected source fields and built the Census outcome; the second notebook performed the shared-predictor recoding. Thus the two saved “harmonized” input CSVs preserve selected source-coded predictors rather than the final model-ready recodings. Missing predictor values were handled within each model fit: the most frequent training value for categorical fields and the training median for household size. Categorical fields were one-hot encoded, with unseen categories ignored. This preprocessing was refitted separately within each cross-validation training partition.

Income was excluded because the available Census work-income and EOD household-income measures are not equivalent. `ageb`, `centralidad`, and parking were not shared supervised predictors; the first two were retained only as EOD output descriptors. Neither source weight, `upm`, dwelling identifier, nor target-provenance field entered the predictor matrix.

## Model development and evaluation

With random seed 2026, an outer `StratifiedGroupKFold` divided the labeled Census sample into five folds, approximately stratifying `dwelling_type` while keeping each `upm` entirely in one fold. Folds 1–4 formed the training and selection set (31,368 records across 150 `upm` groups); fold 5 was held out for a single final comparison (7,826 records across 36 groups). Approximate stratification does not guarantee identical `factor_expansion`-weighted outcome proportions across these sets.

Within the training and selection set, a separate five-fold `StratifiedGroupKFold` supplied pooled out-of-fold probabilities for every candidate. Identical grouped partitions were used across candidates. `factor_expansion` was passed as `sample_weight` for fitted estimators and for the evaluation metric; it was never a predictor. No class reweighting or resampling was used. The comparison comprised a weighted prior-only reference, multinomial logistic regression (`C` = 0.1, 1, or 10), histogram gradient boosting (`learning_rate` = 0.05 or 0.1; `max_leaf_nodes` = 7 or 15; `min_samples_leaf` = 20 or 50), and random forest (`n_estimators` = 400; `max_depth` = 10 or 15; `min_samples_leaf` = 20 or 50). The remaining estimator settings are recorded in the notebook.

Model selection minimized pooled, `factor_expansion`-weighted multiclass log loss among candidates that improved on the weighted prior-only reference. For observed class $y_i$, its predicted probability $p_{i,y_i}$, and expansion factor $w_i$, the score was

$$
L=-\frac{\sum_i w_i\log p_{i,y_i}}{\sum_i w_i}.
$$

Lower log loss indicates better probability assigned to observed outcomes. Balanced accuracy was reported on the held-out set as a secondary diagnostic of hard, maximum-probability classification; it did not determine model selection.

The internal cross-validation reference log loss was 0.782473. The lowest eligible candidate achieved 0.709591: histogram gradient boosting with `learning_rate=0.05`, `max_leaf_nodes=7`, `min_samples_leaf=50`, `l2_regularization=1.0`, `max_iter=200`, and `random_state=2026`. This configuration was fitted on the complete training and selection set before the held-out comparison.

| Held-out procedure | Weighted log loss ↓ | Weighted balanced accuracy ↑ |
|---|---:|---:|
| Weighted prior-only reference | 0.760589 | 0.250000 |
| Selected histogram gradient boosting | 0.679022 | 0.261994 |

This single held-out split supports an improvement in the model's probability score over a prior-only reference. The small balanced-accuracy gain also shows that maximum-probability classification remains weak for distinguishing all four categories. These results are preliminary and still under review. They do not measure accuracy after transfer to the EOD, where observed dwelling types are unavailable. No separate probability transformation was applied to the selected model's `predict_proba` output.

## EOD application and categorical sampling

For the current output, the selected preprocessing and estimator were refitted on all 39,194 labeled Census dwellings using `factor_expansion`. The fitted model then produced one four-element probability vector $p_i=(p_{i1},\ldots,p_{i4})$ for each of the 17,901 EOD dwellings. The model did not use `ponderador` when predicting; that weight was used only for EOD summaries. No spatial calibration, AGEB quota, or OpenStreetMap-derived variable was applied.

A single dwelling category was sampled independently from each vector, rather than selected by its maximum probability:

$$
Y_i\mid p_i\sim\operatorname{Categorical}(p_{i1},p_{i2},p_{i3},p_{i4}).
$$

The EOD rows were sorted by `folio_vivienda` and sampled with NumPy seed 2026 for reproducibility. The sampled `muestreo_tipo_vivienda` is therefore one possible realization, not an observed dwelling type or a certainty claim. The complete probability vector is preserved as JSON in `probabilidad_tipo_vivienda`; `fuente_tipo_vivienda` identifies the model-and-sampling source.

## Descriptive distribution comparison

The table compares (i) observed, expansion-weighted Census categories, (ii) `ponderador`-weighted mean EOD probabilities, and (iii) `ponderador`-weighted frequencies from this one EOD sample.

| Category | Census observed | EOD expected | EOD sampled |
|---|---:|---:|---:|
| `casa_unica` | 74.3400% | 77.1005% | 77.1533% |
| `casa_compartida_duplex` | 13.5087% | 12.4269% | 12.4375% |
| `departamento` | 11.3250% | 9.8388% | 9.7424% |
| `otra_vivienda` | 0.8263% | 0.6338% | 0.6668% |

The notebook also reports total variation distance, $D_{\mathrm{TV}}(p,q)=\tfrac12\sum_k|p_k-q_k|$, for Census versus expected EOD shares and for expected versus sampled EOD shares. The first is a descriptive difference between surveys from different years; the second quantifies sampling variation in this realization. Neither is an accuracy measure for EOD dwelling types. A transparent-background comparison figure is saved in PNG and PDF.

## Reproducibility and outputs

The two notebooks are `notebooks/01_harmonizacion_datos.ipynb` and `notebooks/02_modelo_individual.ipynb`. Their principal outputs are:

| File | Contents |
|---|---|
| `outputs/censo_viviendas_armonizado.csv` | Selected Census dwelling fields and grouped outcome |
| `outputs/eod_viviendas_armonizado.csv` | Selected EOD dwelling fields |
| `outputs/modelo_dwelling_type_individual.joblib` | Fitted pipeline, class order, predictor list, seed, and EOD mappings |
| `outputs/eod_probabilidades_individuales.csv` | EOD identifiers/descriptors, probability JSON, sampled category, and source label |
| `outputs/figures/distribucion_dwelling_type_censo_eod.png` and `.pdf` | Weighted distribution comparison |

The EOD result CSV has one row per `folio_vivienda` and the columns `folio_vivienda`, `ponderador`, `municipio`, `ageb`, `centralidad`, `probabilidad_tipo_vivienda`, `muestreo_tipo_vivienda`, and `fuente_tipo_vivienda`. The saved model is a Python joblib artifact and should be loaded only from a trusted source with a compatible software environment. The source-loading notebook records the `eodgdl`, `mxcensus`, and `pandas` versions used when its outputs were generated.

## Interpretation and remaining uncertainty

The EOD outcome is imputed from relationships learned in 2020 Census data, then sampled. Year-to-year change, differences in survey coverage and response, incomplete predictor equivalence, and weak separation of rare categories may affect transfer. `casa_compartida_duplex` combines two Census structures, while `otra_vivienda` combines several uncommon structures; the model does not recover distinctions inside those groups. The resulting probability vectors and categorical draw should be described as estimates, not directly observed housing types. Further review of source harmonization, held-out results, and output interpretation is pending before treating this as a finalized methodology.
