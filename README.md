# Chronic Disease Index (CDI) as a Human Development Index

## Overview

The **Chronic Disease Index (CDI)** evaluates whether mortality distribution across major chronic diseases can serve as a robust proxy for global socio-economic development.

While metrics like the **Human Development Index (HDI)**, **Life Expectancy**, and **Gross National Income (GNI) per capita** are difficult and slow to collect, healthcare and mortality data are often more readily available. CDI constructs a 9-dimensional conditional probability vector — the **death ratio** — representing the distribution of mortality across chronic diseases given that an individual has contracted a chronic disease:

$$P(\text{Death from Disease } i \mid \text{Contracted a Chronic Disease})$$

This project does not claim the death ratio is *easier* to collect than traditional indicators — only that it is a powerful and biologically grounded stand-in for them when economic or educational data is unavailable.

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

![Countries with missing data](images/countries_with_missing_data.png)
*Fig. 1 — Count of missing year-rows per country (out of a possible 29–30). Somalia, North Korea, Nauru, and Monaco are missing essentially all rows; San Marino follows closely, then a long tail of small nations and territories with partial gaps.*

- **Nauru, North Korea, and Somalia** were dropped entirely — each was missing all 29 possible year-rows.
- Checking average development metrics against the number of missing rows per country showed no strong systematic bias toward excluding less-developed nations, with one exception: a sharp spike at 27 missing rows, attributable to **San Marino** (small, highly developed, but with a very short data history).
- National reporting quality for mortality varies, particularly in low- and middle-income countries, and GNI figures can involve estimation error — both are limitations inherited from the source data rather than something this project corrects for.

---

## Key Research Findings

### 1. Predicting Traditional Development Metrics (Regression)
Using a **K-Nearest Neighbors (KNN)** regressor with integrated feature selection (grid search, 5-fold CV), the death ratio explains **~89% of the variance** across all three traditional socio-economic metrics.

| Target Metric | Optimal # Features | Selected Disease Features | MSE | $R^2$ |
| :--- | :---: | :--- | :---: | :---: |
| **GNI per Capita** | 6 | Alzheimer's, Parkinson's, Cardiovascular, Neoplasms, Diabetes, Chronic Kidney | 36,773,520 | **0.8923** |
| **HDI** | 6 | Parkinson's, Cardiovascular, Diabetes, Chronic Kidney, Respiratory, Cirrhosis | 0.00313 | **0.8849** |
| **Life Expectancy** | 5 | Parkinson's, Cardiovascular, Diabetes, Chronic Kidney, HIV/AIDS | 11.3448 | **0.8797** |

**Core drivers:** Cardiovascular disease, diabetes, and Parkinson's appear in *every* top model — each rises predictably with longevity and affluence, making them the most consistent development-linked features.

**Why certain diseases were excluded**, by feature selection, from one or more models:
- **Chronic respiratory diseases** — affects both high-income (smoking, aging) and low-income (pollution) populations, so it doesn't track monotonically with development.
- **Cirrhosis / liver disease** — peaks in *mid-development*, industrializing nations (alcohol use, hepatitis), producing an inverted-U relationship rather than a linear one.
- **HIV/AIDS** — mortality is highly regionally clustered (concentrated in parts of Africa), making it a poor general correlate of income or composite development, though still informative for life expectancy.
- **Alzheimer's/dementias** — likely underdiagnosed in low-resource settings, reducing its reliability outside GNI prediction.
- **Neoplasms (cancer)** — driven by many independent factors (tobacco use, environmental exposure) that don't cleanly track development level.

![Death ratio vs HDI](images/death_ratio_v_hdi.png)
*Fig. 3 — Each disease's death ratio plotted against HDI, with a linear fit. Neoplasms, cardiovascular disease, and Parkinson's rise clearly with HDI; cirrhosis, respiratory disease, and HIV/AIDS fall.*

![Death ratio vs GNI per capita](images/death_ratio_vs_gni_per_capita.png)
*Fig. 4 — The same disease ratios plotted against GNI per capita. Trends broadly echo the HDI plots, though noisier at high income levels.*

![Death ratio vs life expectancy](images/death_ratio_v_life_expectancy.png)
*Fig. 5 — The same disease ratios plotted against life expectancy — the closest-matching shape to the HDI plots, consistent with life expectancy being an HDI component.*

![Best model MSE vs. number of features](images/best_model_mse.png)
*Fig. 6 — Mean squared error of the best KNN model as more disease features are added, for each of the three targets. Error drops sharply through the first few features, then flattens (and for GNI, ticks back up) once enough diseases are included — the basis for the "optimal # features" column above.*

**Unexplained variance (~11%):** likely attributable to non-health factors excluded from this study (education, inequality), regional gaps in mortality-data quality, and a time lag between development gains and observable shifts in mortality patterns.

---

### 2. Regional Classification
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

**Results:** Both models achieved very low MSE — expected, given that individual death-ratio components are often small fractions (e.g., ~0.0005 for some diseases), which compresses squared error. Between the two, the **KNN regressor outperformed linear regression**, indicating the relationship between development trajectory and future death ratios is likely non-linear, and that near-term death ratios can be forecast with reasonable fidelity from historical development data.

---

## Conclusion
The **Chronic Disease Index (CDI)** provides a biologically grounded, highly accurate stand-in for traditional development indicators. Across four analyses, the death ratio:
- Explains ~89% of variance in HDI, GNI per capita, and life expectancy (Q1),
- Predicts regional/development classification with up to ~92% accuracy (Q2),
- Correlates meaningfully with a theoretically "ideal" disease profile derived from the highest-life-expectancy nations (Q3), and
- Can itself be forecast forward in time from historical development data (Q4).

Together, these results support the death ratio as an effective proxy for evaluation when traditional economic or educational data is unavailable — capturing the epidemiological transition from preventable/infectious conditions toward diseases of longevity as nations develop. Future work should examine the real-world difficulty of collecting death-ratio data at scale, and explore uses of the index beyond predicting location and traditional development metrics.
