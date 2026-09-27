# Methodology

The EOD records where people travel from and to, but it does not identify the physical type of their dwelling. We estimate that attribute in four categories. The procedure first learns dwelling-level probabilities from the 2020 Census Expanded Questionnaire and then adjusts those probabilities to a Census-derived dwelling-type distribution for each AGEB. The assigned type is a draw from the adjusted probabilities, not an observed EOD response.

## 1. Harmonize the Census and EOD dwelling data

We work at the dwelling level in both sources: the 2020 Population and Housing Census Expanded Questionnaire and the 2023 Guadalajara Metropolitan Area Origin–Destination Survey (EOD). The analysis covers the nine municipalities selected in the notebooks. The first notebook loads the source tables, retains the fields needed for this analysis, preserves each source's dwelling identifier and expansion factor, and constructs the observed Census outcome. The second notebook completes the recoding of predictors shared by the two surveys.

The Census variable CLAVIVP distinguishes detailed forms of private dwelling. We group codes 1–9 into four categories that can be used throughout the analysis:

| Analysis category | CLAVIVP codes | Included dwelling forms |
|---|---:|---|
| casa_unica | 1 | House unique to its lot |
| casa_compartida_duplex | 2, 3 | House sharing a lot or duplex |
| departamento | 4 | Apartment in a building |
| otra_vivienda | 5–9 | Other specified private dwelling forms |

Code 99 is an unspecified type. We leave it without a target label and exclude it from supervised fitting; it is not a fifth physical category. Grouping the detailed codes makes the outcome usable across the workflow, but it also means that we do not recover the distinctions inside the combined categories.

The individual model uses seven predictors with corresponding meanings in both surveys: municipality, persons in the dwelling, tenure, access to a car or pickup truck, motorcycle, bicycle, and internet. We map municipalities to the same labels, cap Census household size at 10 to match the EOD's “10 or more” response, align tenure categories, and convert vehicle availability and internet access to yes/no indicators. We do not use income because the available Census work-income and EOD household-income measures are not equivalent. AGEB identifies where an EOD dwelling belongs for the later spatial steps; it is not a supervised predictor. Person-level attributes, identifiers, survey weights, and UPM are also excluded from the predictor matrix.

The Census expansion factor and the EOD ponderador serve different purposes. The first weights Census model fitting and evaluation; the second weights EOD summaries and spatial calibration. Neither is used as a predictor.

## 2. Train and apply the individual probability model

We split labeled Census dwellings with StratifiedGroupKFold. Stratification approximately preserves the four-category outcome mix, while grouping keeps every UPM entirely within one partition. Four of five outer folds form the training and model-selection set; the fifth is left untouched for final evaluation. Within the training set, a second five-fold grouped split generates out-of-fold probabilities for every candidate under the same partitions.

The candidates include a weighted prior-only reference, multinomial logistic regression, histogram gradient boosting, and random forest. Missing categorical values are imputed from the most frequent value in the current training fold; missing household size is imputed from that fold's median. Categorical predictors are one-hot encoded. We refit this preprocessing inside every training partition so validation records do not inform their own preprocessing.

We compare complete probability vectors using expansion-factor-weighted multiclass log loss. If $y_i$ is the observed Census category, $p_{i,y_i}$ is the probability assigned to it, and $d_i$ is its Census expansion factor, the score is

$$
L=-\frac{\sum_i d_i\log p_{i,y_i}}{\sum_i d_i}.
$$

The selected candidate must improve on the weighted prior-only reference and have the lowest pooled out-of-fold log loss among eligible candidates. We also report weighted balanced accuracy for the category with the largest predicted probability, but that hard-classification measure does not select the probability model. The locked candidate is fitted on the outer training set and compared once with the reference on the untouched outer test set.

After this comparison, we refit the selected preprocessing and estimator on all labeled Census dwellings. Its direct probability output gives each EOD dwelling a vector $p_i=(p_{i1},\ldots,p_{i4})$ in the fixed category order. No AGEB information or spatial quota enters this supervised model. An initial seeded categorical draw from $p_i$ is retained so we can compare the individual-only and spatially adjusted versions.

## 3. Estimate the AGEB dwelling-type proportions

The Census publishes AGEB counts for inhabited private dwellings and selected dwelling characteristics, but the workflow does not have an observed four-category dwelling-type count for every AGEB. We therefore estimate that mix rather than treating it as an official count.

For the AGEBs present in the EOD, we load two distinct Census totals: $V_g$, all inhabited private dwellings in AGEB $g$, and $C_g$, dwellings covered by the reported characteristics. We also load candidate counts for bedrooms, rooms, vehicles, bicycles, and internet access. From these, the calibration actually uses up to eight controls: $C_g$, dwellings with one bedroom, dwellings with one room, dwellings with two rooms, and dwellings with a car or pickup truck, motorcycle, bicycle, or internet. Other loaded Census fields do not enter the fit.

The donors are dwelling records from the Census Expanded Questionnaire in the same municipality as the target AGEB. A donor need not have been observed inside that AGEB. We retain the six detailed dwelling forms eligible for this calibration, map them to the four analysis categories, and construct an indicator vector $x_i$ matching the available AGEB controls. Let $d_i$ be donor $i$'s original Census expansion factor and $T_g$ the vector of usable control totals for AGEB $g$. We adjust the donor weights separately for each AGEB:

$$
w^*_{gi}=d_i\exp(x_i^\top\lambda_g).
$$

The notebook finds the AGEB-specific parameters $\lambda_g$ by minimizing the dual objective

$$
F_g(\lambda_g)=\sum_i d_i\exp(x_i^\top\lambda_g)-T_g^\top\lambda_g,
\qquad
\nabla F_g(\lambda_g)=\sum_i w^*_{gi}x_i-T_g.
$$

This is the dual form of choosing weights close to their starting values under an entropy distance, subject to $\sum_i w^*_{gi}x_i=T_g$. A missing control is omitted for that AGEB. When a control is zero, donors whose corresponding indicator is one are given zero calibrated weight; the positive controls are fitted among the remaining eligible donors. The notebook compares fitted control totals with the Census totals to describe the numerical fit.

We sum the adjusted weights of donors in each category to estimate its AGEB count among dwellings with characteristics:

$$
\widehat C_{gk}=\sum_i w^*_{gi}\,\mathbf{1}(y_i=k).
$$

The difference $R_g=\max(0,V_g-C_g)$ represents dwellings outside that characteristics base. The implemented method allocates this entire residual to otra_vivienda. Thus $\widehat N_{gk}=\widehat C_{gk}$ for the other three categories and $\widehat N_{g,\mathrm{otra}}=\widehat C_{g,\mathrm{otra}}+R_g$. This is a modeling assumption, not an observed count of other dwellings. The estimated proportions are $\widehat N_{gk}/V_g$ and are saved as the spatial input for the next notebook.

## 4. Calibrate EOD probabilities and draw dwelling types

We align the four EOD probabilities with the four estimated AGEB proportions. Because small numerical differences can leave the estimated proportions slightly short of or above one, we divide each AGEB's four values by their sum. Call the resulting target $q_{gk}$. For comparison, we first calculate the original EOD mean within AGEB $g$, using its fixed EOD expansion weights $a_i$:

$$
r_{gk}=\frac{\sum_{i\in g}a_i p_{ik}}{\sum_{i\in g}a_i}.
$$

The notebook summarizes the difference between $r_{gk}$ and $q_{gk}$ across AGEBs using the Census dwelling totals $V_g$:

$$
D_k=\frac{\sum_g V_g\lvert r_{gk}-q_{gk}\rvert}{\sum_g V_g}.
$$

Multiplying $D_k$ by 100 expresses it in percentage points. This compares the original probabilities with the estimated spatial targets; it does not compare them with observed EOD dwelling types.

We then adjust each dwelling's probability vector as little as possible under a weighted relative-entropy criterion. With $\tilde p_{ik}$ denoting the adjusted probability, the problem is

$$
\min_{\tilde p}\sum_{i\in g}a_i\sum_k\tilde p_{ik}\log\!\left(\frac{\tilde p_{ik}}{p_{ik}}\right)
$$

subject to

$$
\sum_k\tilde p_{ik}=1
\quad\text{for every dwelling }i,
\qquad
\frac{\sum_{i\in g}a_i\tilde p_{ik}}{\sum_{i\in g}a_i}=q_{gk}
\quad\text{for every category }k.
$$

The EOD weights $a_i$ remain fixed. For categories with a positive target, the calibrated vector has a category-specific exponential shift followed by normalization within each dwelling:

$$
\tilde p_{ik}
=\frac{p_{ik}\exp(\lambda_{gk})}
{\sum_{\ell:q_{g\ell}>0}p_{i\ell}\exp(\lambda_{g\ell})}.
$$

The implementation fixes one shift as a reference and solves the remaining moment equations. A category with target zero is assigned zero probability in that AGEB. Such a zero follows the estimated donor-based target; it is not proof that the dwelling type is physically absent there. Recomputing $D_k$ after calibration checks whether the imposed AGEB constraints were met numerically. A near-zero value is expected by construction and is not independent validation of individual dwelling types.

Finally, we draw one category from each calibrated vector with a fixed random seed:

$$
Y_i\mid\tilde p_i\sim\operatorname{Categorical}(\tilde p_{i1},\ldots,\tilde p_{i4}).
$$

The calibrated probabilities reproduce the estimated AGEB proportions in expectation under the EOD weights. Independent draws need not reproduce them exactly in the realized sample, especially in AGEBs with few EOD dwellings. The final dwelling-level CSV keeps the identifier, AGEB, EOD weight, original and calibrated probability dictionaries, and both the original and calibrated sampled categories. The notebooks also save the comparison figures. These outputs remain estimates: differences between the 2020 Census and 2023 EOD, uncertainty in the donor-derived AGEB targets, and weak separation of uncommon dwelling types are not removed by matching the spatial constraints.
