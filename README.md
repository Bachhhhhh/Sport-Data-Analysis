# Sports Data Analysis: FIFA Player Statistics in R

A statistical modeling project on FIFA player data, built in **R** for the course *Statistical Data Processing* (Xử lý số liệu thống kê) at the University of Science, VNU-HCM. The project covers data cleaning, exploratory analysis, regression and classification modeling, and model diagnostics, with the goal of supporting data-driven decisions in football club recruitment.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Dataset](#dataset)
3. [Repository Structure](#repository-structure)
4. [Getting Started](#getting-started)
5. [Methodology](#methodology)
6. [Key Results](#key-results)
7. [Limitations and Lessons Learned](#limitations-and-lessons-learned)
8. [Team](#team)

---

## Project Overview

The project poses four analytical problems, each framed around a decision a football club might face:

| # | Problem | Type | Methods |
|---|---------|------|---------|
| 1 | Predict a player's market **value** (transfer fee) | Regression | Linear Regression, Gamma GLM, GAM |
| 2 | Predict a player's **position** and suggest alternative positions | Multi-class classification | Multinomial Logistic Regression, Naive Bayes |
| 3 | Predict a player's **overall** rating from technical attributes | Regression | Linear Regression, Gamma GLM (GK vs. outfield players) |
| 4 | Predict a player's **potential** tier | Classification | Multinomial Logistic Regression (3 classes), Logistic Regression (2 classes) |

## Dataset

- **Source:** FIFA Player Stats (compiled from the FIFA video game series)
- **Size:** 18,207 records × 57 variables (16,643 records remain after removing rows with missing values)
- **Variable groups:**
  - *Basic info:* name, age, nationality, club
  - *Performance:* `overall`, `potential`, `position`
  - *Financial:* `value`, `wage`, `release_clause`
  - *Physical/personal:* height, weight, preferred foot, body type
  - *Technical skills:* crossing, finishing, dribbling, stamina, strength, `gk_*` attributes, etc.

## Repository Structure

> Adjust the file names below to match your actual repository.

```
.
├── data/
│   └── fifa_data.csv
├── scripts/                # or .Rmd notebooks
│   ├── 01_preprocessing.R
│   ├── 02_eda.R
│   ├── 03_value_prediction.R
│   ├── 04_position_prediction.R
│   ├── 05_overall_prediction.R
│   └── 06_potential_prediction.R
├── report/
│   └── Final_DPS_Report_Group1.pdf
└── README.md
```

## Getting Started

### Requirements

- R ≥ 4.0
- RStudio (recommended)

### Packages

> Adjust to the packages actually used in your code.

```r
install.packages(c(
  "tidyverse",   # data manipulation and ggplot2 visualization
  "naniar",      # missing-value visualization
  "corrplot",    # correlation heatmaps
  "car",         # VIF (multicollinearity)
  "leaps",       # best-subset selection (regsubsets, Mallow's Cp)
  "mgcv",        # Generalized Additive Models
  "nnet",        # multinomial logistic regression
  "e1071",       # Naive Bayes
  "caret",       # data splitting, cross-validation, confusion matrices
  "pROC"         # ROC / AUC
))
```

### Run

```r
# 1. Clone the repository and open the project in RStudio
# 2. Place the dataset in data/
# 3. Run the scripts in order
source("scripts/01_preprocessing.R")
```

## Methodology

### 1. Data Preprocessing

- Converted `value`, `wage`, and `release_clause` from strings (e.g. `€110.5M`, `€565K`) to numeric values by removing `€` and scaling `M` (10⁶) and `K` (10³).
- Converted `height` (e.g. `5'7`) and `weight` (e.g. `159lbs`) to numeric.
- Dropped uninformative columns: `loaned_from`, `id`, `jersey_number`, `joined`.
- Analyzed missing-value patterns (proportion and pattern plots): most columns have only 48 missing values, while `release_clause` has 1,564 (below 9%). Since missingness was low and similar across columns, rows with missing values were removed with `na.omit()`.

### 2. Exploratory Data Analysis

- `overall` and `potential` are approximately normally distributed; `value` is strongly right-skewed.
- Player counts are highly imbalanced across positions, body types, and football federations (UEFA dominates).
- Correlation heatmaps show strong links among `value`, `wage`, `release_clause`, and `potential`, and clusters of related technical skills (attacking vs. defensive vs. goalkeeping).

### 3. Modeling Workflow

Applied consistently across the regression problems:

1. **Multicollinearity check** with VIF (thresholds of 5 or 10), dropping offending variables.
2. **Feature selection** with best-subset selection (Mallow's Cp) or stepwise regression with 5-fold cross-validation.
3. **Model diagnostics:** residuals vs. fitted (linearity), scale-location (heteroscedasticity), Cook's distance (influential points).
4. **Model extension** when assumptions failed: Gamma GLM and GAM (with partial-residual plots to decide which variables need smooth terms).
5. **Evaluation** on a 70/30 train/test split using RMSE and R² (regression), or accuracy, Kappa, AUC, and confusion matrices (classification).

## Key Results

### Problem 1: Transfer Value Prediction

**Approach 1: non-technical variables** (`overall`, `wage`, `international_reputation`, `release_clause`; selected via Mallow's Cp = 4.05)

| Model | Result |
|-------|--------|
| Linear Regression | R² = 0.9898, RMSE ≈ 577,181; diagnostics show non-linearity and heteroscedasticity |
| Gamma GLM (log link) | Improved fit statistics, but residuals still show a trend |
| **GAM** (smooth terms for `overall`, `wage`, `international_reputation`) | Adjusted R² = 0.991; **test R² = 0.9907, RMSE = 560,051, RMSE/Mean = 0.229** |

**Approach 2: technical variables only**, one linear model per position group:

| Position group | Test R² | Test RMSE |
|----------------|---------|-----------|
| Goalkeeper | 0.305 | 2,447,594 |
| Defender | 0.464 | 2,872,353 |
| Midfielder | 0.474 | 3,628,482 |
| Forward | 0.359 | 4,625,396 |

All four models violate linearity and homoscedasticity, so technical skills alone predict value poorly. Approach 1 was selected as the better solution.

### Problem 2: Position Prediction (goalkeepers excluded)

| Model | Accuracy | AUC |
|-------|----------|-----|
| **Multinomial Logistic Regression** | **43.24%** | high on train and test |
| Naive Bayes | 33.80% | 0.857 |

Accuracy is modest because many positions are similar (e.g. CAM vs. CM). As an extension, the model outputs **Top-k alternative positions** with probabilities, which is useful for judging a player's positional flexibility. For example, a CAM sample had a 49.3% probability for CAM, followed by CM (19.7%), CB (6.7%), and LM (4.1%).

### Problem 3: Overall Rating Prediction

| Group | Model | Test RMSE | Test R² |
|-------|-------|-----------|---------|
| Goalkeepers | Linear | 0.871 | 0.986 |
| Goalkeepers | Gamma GLM | 0.871 | 0.986 |
| Outfield players | Linear | 2.256 | 0.889 |
| Outfield players | Gamma GLM | 2.257 | 0.889 |

### Problem 4: Potential Prediction

- **3-class** (low / medium / high via z-score cutoffs of ±0.43): multinomial regression, accuracy = 0.79, Kappa = 0.69. Most confusion occurs in the medium class.
- **2-class** (high potential = potential ≥ mean), using `age`, `value`, `wage`:

| Metric | Value |
|--------|-------|
| Accuracy (test) | 89.24% |
| Kappa | 0.7817 |
| AUC (train / test) | 0.9576 / 0.9543 |

The near-identical train and test AUC suggests little overfitting.

## Limitations and Lessons Learned

- **Near-leakage in Problem 1.** `release_clause` and `wage` are financially derived and move almost in lockstep with `value`, so the very high R² (> 0.99) should not be read as pure predictive skill. A follow-up should compare models with and without these variables.
- **Listwise deletion.** `na.omit()` discarded about 1,500 rows. Imputation or missingness analysis by group could be explored.
- **GLM did not fully fix heteroscedasticity** in Problem 1, which motivated the GAM.
- **Class imbalance** across positions limits the accuracy of position prediction; resampling or grouping similar positions could help.
- **Problem 4 (binary)** uses `value` as a predictor, which is itself strongly related to potential; this may inflate performance.
- **Future work:** cross-validated comparison against tree-based models (Random Forest, Gradient Boosting), and position grouping for classification.

## Contributors

| Name |
|------|
| Nguyễn Phạm Anh Trí | 
| Hoàng Minh Hiển | 
| Bạch Ngọc Lê Duy | 
| Hoàng Thái Hoàng | 
| Hồng Đức Hoàng | 
| Lê Tuấn Minh Thành | 

> **Supervisor:** Dr. Tô Đức Khánh ·

**University of Science, Vietnam National University Ho Chi Minh City**

---
