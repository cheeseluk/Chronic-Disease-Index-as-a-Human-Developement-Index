# Chronic Disease Index (CDI) as a Human Development Index

## Overview

The **Chronic Disease Index (CDI)** evaluates whether mortality distribution across major chronic diseases can serve as a robust proxy for global socio-economic development. 

While metrics like the **Human Development Index (HDI)**, **Life Expectancy**, and **Gross National Income (GNI) per capita** are difficult and slow to collect, healthcare and mortality data are often more readily available. CDI constructs a 9-dimensional conditional probability vector representing mortality counts per disease given an individual died from a chronic disease:

$$P(\text{Death from Disease } i \mid \text{Contracted a Chronic Disease})$$

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

---

## Datasets & Integration

* **Mortality Data:** Sourced from Kaggle via *Our World in Data’s Burden of Disease Study* (204 countries, 1990–2019).
* **Development Indicators:** Sourced from the United Nations Development Programme (UNDP) containing yearly HDI, GNI per capita, and Life Expectancy.
* **Merged Dataset (`merged_df`):** Inner-joined across country ISO3 codes and years, yielding **5,520 records** (184 complete-data countries across 29 years).

---

## Key Research Findings

### 1. Predicting Traditional Development Metrics (Regression)
Using **K-Nearest Neighbors (KNN)** regressor with feature selection, the death ratio vector explains **~89% of the variance** across traditional socio-economic metrics.

| Target Metric | Optimal # Features | Selected Disease Features | Mean Squared Error (MSE) | $R^2$ Score |
| :--- | :---: | :--- | :---: | :---: |
| **GNI per Capita** | 6 | Alzheimer's, Parkinson's, Cardiovascular, Neoplasms, Diabetes, Chronic Kidney | 36,773,520 | **0.8923** |
| **HDI** | 6 | Parkinson's, Cardiovascular, Diabetes, Chronic Kidney, Respiratory, Cirrhosis | 0.00313 | **0.8849** |
| **Life Expectancy** | 5 | Parkinson's, Cardiovascular, Diabetes, Chronic Kidney, HIV/AIDS | 11.3448 | **0.8797** |

* **Core Drivers:** Cardiovascular diseases, Diabetes, and Parkinson's consistently scale monotonically with development and aging populations.
* **Excluded Noise:** Diseases with regional spikes (HIV/AIDS) or non-linear trends (Cirrhosis) were automatically excluded by feature selection to optimize performance.

---

### 2. Regional Classification
Using death ratios, **KNN Classifiers** were trained to categorize nations into socio-economic tiers:

* **Global North vs. Global South** ($\text{HDI} \ge 0.8$ vs. $< 0.8$): **92.33% Accuracy**
* **Life Expectancy Tiers** ($>80$, $70\text{--}79$, $<70$ years): **85.60% Accuracy**
* **GNI Quartiles** (Q1–Q4): **72.15% Accuracy**

#### Regional Mortality Profiles (75th Percentile Proportions)
* **Alzheimer's & Dementias:** **6.2%** in Global North vs. **3.1%** in Global South (correlates directly with longevity).
* **HIV/AIDS:** **3.4%** in Global North vs. **11.6%** in Global South (correlates inversely with development).

---

### 3. Independent Death Ratio Index (IDRI)
An ideal target ratio vector was established by averaging the disease probabilities of the 5 nations with the highest life expectancy (excluding population skew):

```text
[Alzheimer's: 9.56%, Parkinson's: 1.48%, Cardiovascular: 37.12%, 
 Neoplasms: 37.86%, Diabetes: 2.63%, Kidney: 3.65%, 
 Respiratory: 5.24%, Cirrhosis: 2.38%, HIV/AIDS: 0.09%]
```

Measuring the Euclidean distance between a country's death ratio vector and this target profile yields strong negative Spearman rank correlations:
* **Distance vs. HDI:** $\rho = -0.568$
* **Distance vs. Life Expectancy:** $\rho = -0.648$

---

## Conclusion
The **Chronic Disease Index (CDI)** provides a biologically grounded, highly accurate stand-in for traditional development indicators. It effectively captures epidemiological transitions—specifically the shift from preventable or infectious conditions to diseases of longevity—making it an effective proxy for evaluation when economic data is unavailable.
