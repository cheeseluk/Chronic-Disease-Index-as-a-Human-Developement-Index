# Chronic Disease Index (CDI) as a Human Development Index

## Overview

The **Chronic Disease Index (CDI)** evaluates whether mortality distribution across major chronic diseases can serve as a proxy for global socio-economic development.

While metrics like the **Human Development Index (HDI)**, **Life Expectancy**, and **Gross National Income (GNI) per capita** are difficult and slow to collect, healthcare and mortality data are often more readily available. CDI constructs a 9-dimensional conditional probability vector — the **death ratio** — representing the distribution of mortality across chronic diseases given that an individual has contracted a chronic disease:

$$P(\text{Death from Disease } i \mid \text{Contracted a Chronic Disease})$$

This project does not claim the death ratio is *easier* to collect than traditional indicators. It tests whether the death ratio can stand in for them for a country whose economic or educational data is unavailable. That means the model has to work on countries it has never seen, which is how Question 1 is evaluated (see the evaluation note below).

> **Revised September 2026.** An earlier version of this README reported R² ≈ 0.89 for all three metrics. Those numbers came from a flawed evaluation and have been replaced. See [Evaluation note](#evaluation-note-what-changed-and-why).

### The 9 Chronic Diseases Studied
1. Alzheimer's Disease and Other Dementias
2. Parkinson's Disease
3. Cardiovascular Diseases
4. Neoplasms (Cancer)
5. Diabetes Mellitus
6. Chronic Kidney Disease
7. Chronic Respiratory Diseases
8. Cirrhosis and Other Chronic Liver Diseases
9. HIV/AIDS

Disease selection follows the CDC's definition of chronic disease.

---

## Datasets & Integration

* **Mortality Data:** *Cause of Deaths Around the World (Historical Data)*, sourced via Kaggle from Our World in Data's *Burden of Disease* study (Roser, Ritchie & Spooner, 2021). Covers 204 countries, 1990–2019, with mortality counts across 29 causes of death — **6,120 raw country–year observations**.
* **Development Indicators:** United Nations Development Programme (UNDP) Human Development Reports. Raw form: 193 countries × 880 metric–year columns, reshaped into long format and filtered to HDI, GNI per capita, and life expectancy for 1990–2019. Countries with any missing values across these metrics were dropped, reducing coverage to **184 countries**.
* **Merged Dataset (`merged_df`):** Inner-joined on country ISO3 code and year, yielding **5,520 records** (184 countries × up to 29 years), with 9 death-ratio features, 3 development metrics, and identifier columns.

### Data Quality Notes

<img width="3636" height="3020" alt="countries with missing data" src="https://github.com/user-attachments/assets/58eb9bf1-5fa0-4a3c-b1e2-0b0bf1817357" />

*Fig. 1 — Count of missing year-rows per country (out of a possible 29–30). Somalia, North Korea, Nauru, and Monaco are missing essentially all rows; San Marino follows closely, then a long tail of small nations and territories with partial gaps.*

- **Nauru, North Korea, and Somalia** were dropped entirely — each was missing all 29 possible year-rows.
- Checking average development metrics against the number of missing rows per country showed no strong systematic bias toward excluding less-developed nations, with one exception: a sharp spike at 27 missing rows, attributable to **San Marino** (small, highly developed, but with a very short data history).
- National reporting quality for mortality varies, particularly in low- and middle-income countries, and GNI figures can involve estimation error — both are limitations inherited from the source data rather than something this project corrects for.

---

## How to Run
 
**Setup:**
```bash
pip install -r requirements.txt
```
 
**Execution order matters:** the notebook for **Question 1** performs the EDA, data cleaning, and merge, and produces `merged_df` — the dataset every other question depends on. Questions 2–4 each just load `merged_df` and are independent of one another, so they can be run in any order (or in parallel) once it exists.
 
1. Run the **Question 1** notebook first.
   > ⚠️ **This one takes a while.** The feature-subset search near the END of the notebook runs a grouped-CV hyperparameter grid search over all 511 disease combinations for each of the three targets — roughly **~30 minutes** on a Ryzen 7900X. Budget time accordingly, or grab the pre-built `merged_df` included in the repo to skip the code cell or go straight to Questions 2–4.
2. Run **Question 2**, **Question 3**, and **Question 4** in any order — each finishes quickly.
**Data sources:** the mortality dataset is pulled automatically at runtime via `kagglehub` (`iamsouravbanerjee/cause-of-deaths-around-the-world`); no manual download needed. The HDI dataset (`Human Development Index - Full.csv`) is expected to already be present in the working directory, as it's not pulled programmatically.
 
---


## Key Research Findings

### 1. Predicting Traditional Development Metrics (Regression)

**Setup.** 20% of countries (37 of 184) are held out as a test set before any tuning. A **K-Nearest Neighbors (KNN)** regressor is tuned on the remaining 147 countries by grid search with 5-fold `GroupKFold` cross-validation (folds never split a country). The best disease subset for each target is chosen by that cross-validated R², then scored once on the held-out countries. A mean-value baseline (`DummyRegressor`) is scored on the same test countries for comparison.

| Target Metric | # Features | Diseases left out | CV $R^2$ (147 training countries) | **Test $R^2$ (37 unseen countries)** | Test MSE | Baseline test $R^2$ |
| :--- | :---: | :--- | :---: | :---: | :---: | :---: |
| **Life Expectancy** | 8 | Chronic Kidney | 0.802 | **0.822** | 14.51 | −0.033 |
| **HDI** | 9 | none | 0.775 | **0.800** | 0.00594 | −0.043 |
| **GNI per Capita** | 7 | Neoplasms, Diabetes | 0.407 | **0.130** | 390,518,700 | −0.077 |

**What this shows:**
- **Life expectancy and HDI:** the death ratio predicts both for countries the model has never seen, explaining about 80% of the variance, where the baseline explains none. The cross-validated and test scores agree, so this result is stable.
- **GNI per capita:** the death ratio does **not** reliably predict GNI for new countries. The test R² of 0.13 is well below the CV score of 0.41, which suggests the result depends heavily on which countries are held out. A likely reason is that countries with similar disease profiles can have very different incomes (resource-exporting economies, for example), and the model cannot tell them apart.
- **Feature selection matters little.** Once the evaluation is fair, the best subsets keep 7–9 of the 9 diseases, and no small group of diseases stands out as the key predictors. (The earlier version's 5–6-disease subsets and "core drivers" were most likely artifacts of choosing subsets by in-sample scores.)

<img width="1039" height="1107" alt="death ratio v hdi" src="https://github.com/user-attachments/assets/f52eba70-b7f7-408b-9fc0-a871f55b6f7c" />

*Fig. 3 — Each disease's death ratio plotted against HDI, with a linear fit. Neoplasms, cardiovascular disease, and Parkinson's rise clearly with HDI; cirrhosis, respiratory disease, and HIV/AIDS fall.*

<img width="1040" height="1107" alt="death ratio vs gni per capita" src="https://github.com/user-attachments/assets/1e7967fe-4a35-41c5-ac33-882f5a9d63d8" />

*Fig. 4 — The same disease ratios plotted against GNI per capita. Trends broadly echo the HDI plots, though noisier at high income levels, which is consistent with GNI being the hardest target to predict.*

<img width="1039" height="1107" alt="death ratio v life expectancy" src="https://github.com/user-attachments/assets/5e9515d2-804d-4513-8d22-01b6aaecf957" />

*Fig. 5 — The same disease ratios plotted against life expectancy — the closest-matching shape to the HDI plots, consistent with life expectancy being an HDI component.*

<img width="790" height="1490" alt="best mode mse" src="https://github.com/user-attachments/assets/46e7f6b2-7882-4db8-99b9-86e2040d72c7" />

**Unexplained variance (~20% for HDI and life expectancy, most of it for GNI):** likely due to non-health factors not in the death ratio (education, income distribution, resource wealth), uneven mortality-data quality across regions, and a lag between development gains and shifts in mortality patterns.

#### Evaluation note: what changed and why

The first version of this analysis reported R² ≈ 0.88–0.89 for all three targets. That evaluation had two problems:

1. **Scores were computed on training data.** Cross-validation was used only to pick hyperparameters. `GridSearchCV` then refit on every row, and R² was computed by predicting those same rows.
2. **Random folds leaked country identity.** Each country contributes up to 29 nearly identical yearly rows. With random 5-fold CV, a validation row such as Afghanistan 1995 had Afghanistan 1994 and 1996 in the training folds, so a nearest-neighbor model could simply find that country's other years. That measures how well the model recognizes a country, not how well it predicts development from a disease profile.

Because the use case is a country *without* development data, the fix holds out whole countries: `GroupKFold` for tuning and a separate set of test countries for the final score. Under this evaluation, HDI and life expectancy stay strong (≈ 0.80), while GNI drops from 0.89 to 0.13.

**Predicting future years for known countries is a different question.** A time-based split (train on earlier years, test on later ones) would test it, and there the simple baseline of carrying each country's last known value forward is very hard to beat, because these metrics change slowly. A quick check found that baseline outperformed the death-ratio KNN for all three targets, so this project does not claim the index is useful for that purpose.

---

### 2. Regional Classification
> ⚠️ **Not yet re-evaluated.** These accuracies were measured with random (not country-grouped) cross-validation, so they are likely inflated for the same reason as the original Question 1 results. A preliminary re-check of the Global North/South classifier with country-grouped folds gave about 90% accuracy, against about 76% for always guessing the majority class (Global South). Treat the figures below as upper bounds until Question 2 is rerun.

Countries were grouped by three development-derived labels, then classified using **KNeighborsClassifier** models (grid-search tuned) trained solely on the death ratio:

* **Global North vs. Global South** (HDI ≥ 0.8 vs. < 0.8): **92.33% accuracy**
* **Life Expectancy Tiers** (High >80, Medium 70–79, Low <70 years): **85.60% accuracy**
* **GNI Quartiles** (Q1–Q4): **72.15% accuracy**

The two-class Global North/South split achieved the highest accuracy, consistent with binary classification generally outperforming multi-class splits. Within that model, precision and recall were both strongest on the Global South class, likely because it's the more populous class in the training data. For the Life Expectancy model, precision peaked on the "High" category and recall peaked on the "Low" category. The GNI quartile model was notably weaker across all four classes — quartile boundaries appear to be harder to resolve from mortality patterns alone than the coarser HDI or life-expectancy splits.

#### Regional Mortality Profiles (75th Percentile Proportions)
Across all three groupings, Alzheimer's/dementia share of chronic deaths rises with development, while HIV/AIDS share falls:

| Grouping | Alzheimer's (low → high) | HIV/AIDS (low → high) |
|---|---|---|
| Global South → North | 3.1% → 6.2% | 11.6% → 3.4% |
| Life expectancy: Low → Medium → High | 2.2% → 4.3% → 7.1% | 27% → 1.3% → 0.2% |
| GNI quartile: Q1 → Q2 → Q3 → Q4 | 2% → 3.2% → 4% → 6.2% | 36% → 3.1% → 2% → 0.4% |

This supports the underlying hypothesis: developed regions, with broader healthcare access, see proportionally more deaths from less-preventable, longevity-linked diseases (Alzheimer's), while less-developed regions see proportionally more deaths from more-preventable diseases (HIV/AIDS).

---

### 3. Independent Death Ratio Index (IDRI)
An ideal target death-ratio vector was constructed by averaging the disease probabilities of the 5 countries with the highest life expectancy. Two methodological choices matter here:
- **Monaco was excluded** from the top-5 despite its high life expectancy, due to its small population skewing the ratio.
- Probabilities were **not population-weighted**, so no single large country's disease burden could dominate the average.
- The 5 source countries were then **removed from the comparison set** before measuring distances, to avoid artificially small self-distances.

```text
[Alzheimer's: 9.56%, Parkinson's: 1.48%, Cardiovascular: 37.12%,
 Neoplasms: 37.86%, Diabetes: 2.63%, Kidney: 3.65%,
 Respiratory: 5.24%, Cirrhosis: 2.38%, HIV/AIDS: 0.09%]
```

Neoplasms and cardiovascular disease dominate this ideal profile (~37% each) — consistent with both being diseases whose relative share rises as populations age. Alzheimer's/dementias is a distant third at ~9.6%.

Measuring Euclidean distance between each country's death ratio and this target profile yields moderate negative Spearman rank correlations:
* **Distance vs. HDI:** ρ = −0.568
* **Distance vs. Life Expectancy:** ρ = −0.648

These correlations are meaningful but not overwhelming — the IDRI is a useful ranking signal for how well a nation's chronic-disease burden resembles that of the highest-life-expectancy countries, rather than a precise substitute for HDI itself.

---

### 4. Forecasting Future Death Ratios
A further question: given a country's development trajectory, can the death ratio itself be predicted forward in time?

**Design & Implementation:** Two model families were compared — linear regression (via `MultiOutputRegressor`, since ordinary linear regression cannot natively predict multiple outputs) and a KNN regressor, with the number of neighbors scaled proportionally to each country's available data. Both were evaluated with 5-fold cross-validation and compared on held-out MSE across four countries randomly selected from four different regions.

**Results:** Both models achieved very low MSE — expected, given that individual death-ratio components are often small fractions (e.g., ~0.0005 for some diseases), which compresses squared error. Between the two, the **KNN regressor outperformed linear regression**, suggesting the relationship between development trajectory and future death ratios is non-linear.

> ⚠️ **Caveat.** Random 5-fold cross-validation on a single country's time series lets the model train on years after the ones it is tested on, so it fills in gaps between known years rather than forecasting. A forward-in-time split (train on earlier years, test on later ones) compared against a carry-last-value-forward baseline is needed before calling this a forecast.

---

## Conclusion
The death ratio carries real information about development, but less than the first version of this project claimed. Across four analyses:
- **Q1:** For countries the model has never seen, it predicts HDI and life expectancy with R² ≈ 0.80 (vs. ≈ 0 for a mean baseline). It does **not** reliably predict GNI per capita (R² = 0.13).
- **Q2:** It separates development tiers well above chance, though the reported accuracies still need to be re-measured with country-grouped folds.
- **Q3:** Distance from a "high-life-expectancy" disease profile correlates moderately with HDI (ρ = −0.57) and life expectancy (ρ = −0.65).
- **Q4:** Death ratios can be modeled from development history, but this has not yet been tested as a true forward-in-time forecast.

Overall, the death ratio is a reasonable stand-in for health-linked development measures (life expectancy, and HDI through it) when a country has no development data, and a poor stand-in for income. It captures the epidemiological transition from preventable and infectious conditions toward diseases of longevity. Future work: re-evaluate Q2 and Q4 with grouped or time-based splits, test whether combining the death ratio with a country's past values beats simply carrying those values forward, and examine how hard death-ratio data is to collect at scale.
