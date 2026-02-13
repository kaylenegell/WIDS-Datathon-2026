# WiDS Datathon 2026 - EDA Takeaways & Modeling Strategy

## Executive Summary

This wildfire evacuation prediction challenge is fundamentally a **proximity + direction problem** with survival analysis structure. Distance to evacuation zone dominates all other predictors, achieving 0.91 C-index alone. The dataset has 68.8% censored events (fires that never hit within 72h), requiring survival analysis methods rather than standard classification.


## Critical EDA Findings

### 1. **Distance is HIGHLY Important**

**Key Stats:**
- `dist_min_ci_0_5h` achieves **C-index: 0.915** as a single feature
- Correlation with target: **-0.481** (strongest by far)
- Fires <5km: **100% hit rate**
- Fires ≥5km: **0% hit rate**
- This is a clear threshold effect

**Implication:** Distance is not just the most important feature - it's the dominant signal in the data.

### 2. **Survival Analysis is Essential**

**Why not standard classification:**
- 68.8% of fires are censored (never hit within 72h observation window)
- These are not "negative" examples - they're partial information
- Time-to-event matters: hitting in 6h vs 60h has different implications
- Censored fires might have hit given more time

**Survival Analysis Benefits:**
- Properly handles censored observations
- Provides time-dependent predictions: P(hit by 12h), P(hit by 24h), etc.
- C-index (concordance index) is the right metric
- Can predict median time-to-hit for emergency planning

### 3. **Temporal Resolution Confounding**

**The Relationship:**
- 72.9% of fires have low temporal resolution (single perimeter observation)
- Low resolution fires: **20.5% hit rate**
- High resolution fires: **60.0% hit rate**
- BUT: Low resolution fires are **93km farther away** on average!

**Confounding Structure:**
- understanding the meaning of poor data quality DOES NOT mean inaccurate results 
- correlated meaning -> signals 
```
Far fires → Less tracking → Low temporal resolution
Far fires → Won't hit in 72h → Censored

NOT: Poor tracking → Missed events → Incorrectly censored
```

**Evidence:**
- When controlling for distance (stratified analysis), the effect largely disappears
- In closest quartile: Both 100% hit rate regardless of resolution
- In farthest quartiles: Both 0% hit rate regardless of resolution

**Modeling Implication:**
- Keep `low_temporal_resolution_0_5h` as a feature (still adds signal)
- But prioritize `dist_min_ci_0_5h` (primary driver)
- Consider interaction: `distance × temporal_resolution`

### 4. **High Sparsity in Growth Features**

**The Problem:**
- 88.7% of fires show **zero growth** in first 5 hours
- This affects 19 growth-related features
- Features like `radial_growth_m`, `log_area_ratio_0_5h` are mostly zeros

**Why This Happens:**
- Observation window is only 5 hours
- Many fires are slow-growing or far away
- Growth metrics are captured at t0+5h, not over extended period

**Modeling Implication:**
- Growth features have limited predictive value
- Consider removing features with >80% zeros
- Exception: `area_first_ha` (initial size) is useful

### 5. **High Multicollinearity**

**65 feature pairs with |correlation| > 0.8**

**Major redundancies:**
```
area_growth_rel_0_5h ≈ relative_growth_0_5h (r=1.00) - Identical
closing_speed_m_per_h ≈ projected_advance_m (r=0.998)
dist_std_ci_0_5h ≈ closing_speed_abs_m_per_h (r=0.997)
radial_growth_m ≈ centroid_displacement_m (r=0.989)
```

**Implication:**
- Remove one from each highly correlated pair
- Reduces overfitting risk
- Speeds up model training
- Improves interpretability

### 6. **Directional Features Matter (But LESS Than Distance)**

**Important directional features:**
- `spread_bearing_cos`: Correlation -0.32 (E-W component)
- `spread_bearing_deg`: Combined score 0.26
- `alignment_abs`: How directly fire moves toward target

**Why They Help:**
- Fire moving toward evacuation zone = higher risk
- Perpendicular movement = lower risk
- But effect is modest compared to distance

### 7. **Time Patterns**

**Hit time distribution (for fires that hit):**
- Mean: 10.0 hours
- Median: 3.5 hours
- 91% hit within first 24 hours
- Only 9% hit after 24h

**Implication:**
- Early prediction is critical
- Model should focus on 0-24h timeframe
- Features captured at t0+5h must predict next 20h

### 8. **Data Quality is Good**

**Evidence:**
- No missing values
- Censored fires observed for full period (avg 51.7h for low res, 41.6h for high res)
- Not evidence of premature stopping
- Temporal resolution reflects fire characteristics, not data collection issues


## Feature Engineering Ideas

### 1. **Distance-Based Features** (Highest Priority)

```python
# Inverse distance (risk score)
train['distance_risk'] = 1 / (train['dist_min_ci_0_5h'] + 1)

# Log distance (compress scale)
train['log_distance'] = np.log1p(train['dist_min_ci_0_5h'])

# Distance bins (categorical risk levels)
train['distance_category'] = pd.cut(
    train['dist_min_ci_0_5h'],
    bins=[0, 5000, 20000, 50000, 1000000],
    labels=['Critical', 'High', 'Moderate', 'Low']
)

# Distance rate of change (if you have multiple time points)
train['distance_velocity'] = train['dist_change_ci_0_5h'] / train['dt_first_last_0_5h']
```

### 2. **Directional Risk Features**

```python
# Combined directional risk
train['directional_threat'] = (
    train['alignment_abs'] *  # How aligned with target
    np.abs(train['spread_bearing_cos'])  # Strength of directional movement
)

# Angle between fire movement and target
# (if both bearing and alignment available)
train['approach_angle'] = np.arccos(train['alignment_cos']) * 180 / np.pi

# Perpendicular distance component
train['lateral_distance'] = (
    train['dist_min_ci_0_5h'] * 
    train['cross_track_component']
)
```

### 3. **Urgency Scores** (Composite Features)

```python
# Primary urgency score
train['urgency_score'] = (
    1 / (train['dist_min_ci_0_5h'] + 1) *  # Proximity
    train['alignment_abs'] *                 # Alignment with target
    (1 - train['low_temporal_resolution_0_5h'])  # Data quality weight
)

# Time-normalized distance (how fast is gap closing)
train['time_to_contact'] = (
    train['dist_min_ci_0_5h'] / 
    (train['closing_speed_m_per_h'] + 1)  # Add 1 to avoid division by zero
)

# Threat level (categorical)
train['threat_level'] = pd.qcut(
    train['urgency_score'],
    q=4,
    labels=['Low', 'Moderate', 'High', 'Critical']
)
```

### 4. **Interaction Terms**

```python
# Distance × Temporal Resolution (confounding interaction)
train['dist_x_temporal'] = (
    train['dist_min_ci_0_5h'] * 
    train['low_temporal_resolution_0_5h']
)

# Distance × Direction
train['dist_x_bearing'] = (
    train['dist_min_ci_0_5h'] * 
    train['spread_bearing_cos']
)

# Distance × Alignment (close + aligned = very dangerous)
train['dist_x_alignment'] = (
    train['dist_min_ci_0_5h'] * 
    train['alignment_abs']
)

# Fire size × Distance (larger fires at moderate distance)
train['size_distance_interaction'] = (
    train['area_first_ha'] * 
    np.log1p(train['dist_min_ci_0_5h'])
)
```

### 5. **Temporal Features**

```python
# Time-of-day risk (categorical)
train['time_period'] = pd.cut(
    train['event_start_hour'],
    bins=[0, 6, 12, 18, 24],
    labels=['Night', 'Morning', 'Afternoon', 'Evening'],
    include_lowest=True
)

# Weekend flag
train['is_weekend'] = train['event_start_dayofweek'].isin([5, 6]).astype(int)

# Fire season (month-based)
train['fire_season'] = pd.cut(
    train['event_start_month'],
    bins=[0, 3, 6, 9, 12],
    labels=['Winter', 'Spring', 'Summer', 'Fall']
)

# Cyclical encoding for hour (preserves circularity)
train['hour_sin'] = np.sin(2 * np.pi * train['event_start_hour'] / 24)
train['hour_cos'] = np.cos(2 * np.pi * train['event_start_hour'] / 24)
```

### 6. **Growth Rate Features** (Handle Sparsity)

```python
# Binary: Any growth?
train['has_growth'] = (train['area_growth_abs_0_5h'] > 0).astype(int)

# Log-transformed growth (handles zeros better)
train['log_growth'] = np.log1p(train['area_growth_abs_0_5h'])

# Growth rate per unit area (normalized)
train['growth_rate_normalized'] = (
    train['area_growth_abs_0_5h'] / 
    (train['area_first_ha'] + 1)
)

# Interaction: Growth × Distance (growing fires closer = dangerous)
train['growth_proximity'] = (
    train['log_growth'] * 
    (1 / (train['dist_min_ci_0_5h'] + 1))
)
```

### 7. **Data Quality Indicators**

```python
# Comprehensive data quality score
train['data_quality_score'] = (
    (train['num_perimeters_0_5h'] >= 2).astype(int) * 0.5 +  # Multiple observations
    (train['dt_first_last_0_5h'] >= 1).astype(int) * 0.3 +   # Reasonable time span
    (1 - train['low_temporal_resolution_0_5h']) * 0.2         # Not flagged as low res
)

# Observation density (perimeters per hour)
train['observation_density'] = (
    train['num_perimeters_0_5h'] / 
    (train['dt_first_last_0_5h'] + 0.1)
)
```

### 8. **Polynomial Features** (For Linear Models)

```python
from sklearn.preprocessing import PolynomialFeatures

# Create polynomial features for top 5 predictors
top_features = [
    'dist_min_ci_0_5h',
    'spread_bearing_cos',
    'alignment_abs',
    'area_first_ha',
    'num_perimeters_0_5h'
]

poly = PolynomialFeatures(degree=2, include_bias=False, interaction_only=False)
poly_features = poly.fit_transform(train[top_features])
poly_feature_names = poly.get_feature_names_out(top_features)

# Add to dataframe
poly_df = pd.DataFrame(poly_features, columns=poly_feature_names, index=train.index)
train = pd.concat([train, poly_df], axis=1)
```


## Recommended Feature Sets

### **Minimal Set (9 features)** - Start Here
Best for: Fast iteration, avoiding overfitting, interpretability

```python
minimal_features = [
    'dist_min_ci_0_5h',              # Distance (dominant)
    'spread_bearing_deg',            # Direction
    'spread_bearing_cos',            # Directional component
    'low_temporal_resolution_0_5h',  # Data quality proxy
    'alignment_abs',                 # Target alignment
    'num_perimeters_0_5h',          # Temporal coverage
    'dt_first_last_0_5h',           # Time span
    'area_first_ha',                # Initial fire size
    'event_start_hour'              # Temporal context
]
```

**Expected Performance:** C-index 0.75-0.80

### **Extended Set (15 features)** - Better Performance
Best for: Balancing performance and simplicity

```python
extended_features = minimal_features + [
    'closing_speed_m_per_h',        # Rate of approach
    'centroid_speed_m_per_h',       # Fire movement speed
    'dist_slope_ci_0_5h',           # Distance trend
    'event_start_dayofweek',        # Day of week
    'alignment_cos',                # Directional alignment
    'log1p_area_first'              # Log-transformed size
]
```

**Expected Performance:** C-index 0.78-0.83

### **Engineered Set (20+ features)** - Maximum Performance
Best for: Competition winning, ensemble models

```python
engineered_features = extended_features + [
    'distance_risk',                # 1 / (distance + 1)
    'urgency_score',                # Composite risk
    'dist_x_temporal',              # Interaction term
    'dist_x_alignment',             # Interaction term
    'directional_threat',           # Composite direction
    'has_growth',                   # Binary growth flag
    'hour_sin', 'hour_cos',         # Cyclical time
    'data_quality_score',           # Quality composite
    'threat_level'                  # Categorical risk
]
```

**Expected Performance:** C-index 0.83-0.87

### **Distance-Only Baseline** - For Comparison
```python
baseline_features = ['dist_min_ci_0_5h']
```
**Expected Performance:** C-index 0.91 (surprisingly strong!)

---

## Modeling Recommendations

### **Phase 1: Baselines**

#### 1.1 Distance-Only Model
```python
from sklearn.ensemble import RandomForestClassifier

# Simple baseline
rf_baseline = RandomForestClassifier(n_estimators=100, random_state=42)
rf_baseline.fit(train[['dist_min_ci_0_5h']], train['event'])

# Expected: 0.70-0.75 C-index
```

#### 1.2 Minimal Feature Set
```python
# 9 recommended features
rf_minimal = RandomForestClassifier(
    n_estimators=200,
    max_depth=10,
    min_samples_split=10,
    random_state=42
)
rf_minimal.fit(train[minimal_features], train['event'])

# Expected: 0.75-0.80 C-index
```

#### 1.3 Simple Survival Model: Cox proportional hazard 
```python
# Requires: pip install lifelines
from lifelines import CoxPHFitter

cph = CoxPHFitter(penalizer=0.1)
cph.fit(
    train[['time_to_hit_hours', 'event'] + minimal_features],
    duration_col='time_to_hit_hours',
    event_col='event'
)

# Expected: 0.75-0.80 C-index
```

### **Phase 2: Advanced Models**

#### 2.1 Random Survival Forest
```python
# Requires: pip install scikit-survival
from sksurv.ensemble import RandomSurvivalForest
from sksurv.util import Surv

# Prepare survival data
y = Surv.from_dataframe('event', 'time_to_hit_hours', train)

rsf = RandomSurvivalForest(
    n_estimators=500,
    max_depth=15,
    min_samples_split=10,
    min_samples_leaf=5,
    max_features='sqrt',
    n_jobs=-1,
    random_state=42
)

rsf.fit(train[extended_features], y)

# Expected: 0.80-0.85 C-index
```

#### 2.2 Gradient Boosting Survival
```python
from sksurv.ensemble import GradientBoostingSurvivalAnalysis

gbsa = GradientBoostingSurvivalAnalysis(
    n_estimators=300,
    learning_rate=0.05,
    max_depth=5,
    min_samples_split=10,
    subsample=0.8,
    random_state=42
)

gbsa.fit(train[extended_features], y)

# Expected: 0.82-0.86 C-index
```

#### 2.3 XGBoost (Time-to-Event)
```python
import xgboost as xgb
from sksurv.metrics import concordance_index_censored

# Convert to XGBoost format
dtrain = xgb.DMatrix(
    train[engineered_features],
    label=train['time_to_hit_hours']
)

# Train with Cox objective
params = {
    'objective': 'survival:cox',
    'eval_metric': 'cox-nloglik',
    'max_depth': 6,
    'eta': 0.05,
    'subsample': 0.8,
    'colsample_bytree': 0.8
}

xgb_model = xgb.train(
    params,
    dtrain,
    num_boost_round=500,
    verbose_eval=50
)

# Expected: 0.83-0.87 C-index
```

#### 2.4 LightGBM Survival
```python
import lightgbm as lgb

lgb_train = lgb.Dataset(
    train[engineered_features],
    label=train['time_to_hit_hours']
)

params = {
    'objective': 'regression',
    'metric': 'rmse',
    'boosting_type': 'gbdt',
    'num_leaves': 31,
    'learning_rate': 0.05,
    'feature_fraction': 0.8,
    'bagging_fraction': 0.8,
    'bagging_freq': 5
}

lgb_model = lgb.train(
    params,
    lgb_train,
    num_boost_round=500,
    valid_sets=[lgb_train]
)

# Expected: 0.82-0.86 C-index
```

### **Phase 3: Ensemble & Tuning**

#### 3.1 Weighted Ensemble
```python
from sklearn.ensemble import VotingRegressor

# Get predictions from multiple models
pred_cph = cph.predict_partial_hazard(test_features)
pred_rsf = rsf.predict(test_features)
pred_gbsa = gbsa.predict(test_features)
pred_xgb = xgb_model.predict(xgb.DMatrix(test_features))

# Weighted average (tune weights on validation set)
ensemble_pred = (
    0.25 * pred_cph +
    0.30 * pred_rsf +
    0.25 * pred_gbsa +
    0.20 * pred_xgb
)

# Expected: 0.85-0.89 C-index
```

#### 3.2 Stacking
```python
from sklearn.linear_model import Ridge

# Level 1 models (base learners)
base_models = [
    ('rsf', rsf),
    ('gbsa', gbsa),
    ('cph', cph)
]

# Get out-of-fold predictions for level 1
# Then train Ridge meta-learner on top

# Expected: 0.85-0.90 C-index
```

#### 3.3 Bayesian Optimization for Hyperparameters
```python
from hyperopt import fmin, tpe, hp, Trials

def objective(params):
    model = RandomSurvivalForest(
        n_estimators=int(params['n_estimators']),
        max_depth=int(params['max_depth']),
        min_samples_split=int(params['min_samples_split']),
        random_state=42
    )
    
    # Cross-validation
    scores = []
    for train_idx, val_idx in kfold.split(X):
        model.fit(X[train_idx], y[train_idx])
        score = model.score(X[val_idx], y[val_idx])
        scores.append(score)
    
    return -np.mean(scores)  # Negative because we minimize

space = {
    'n_estimators': hp.quniform('n_estimators', 100, 1000, 50),
    'max_depth': hp.quniform('max_depth', 5, 30, 1),
    'min_samples_split': hp.quniform('min_samples_split', 5, 50, 5)
}

best = fmin(objective, space, algo=tpe.suggest, max_evals=50)
```

## Model Selection Strategy

### **Cross-Validation Setup**

```python
from sklearn.model_selection import StratifiedKFold

# Stratified by event to maintain censoring ratio
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

cv_scores = []
for fold, (train_idx, val_idx) in enumerate(skf.split(X, y_event)):
    print(f"Fold {fold + 1}/5")
    
    # Train on fold
    model.fit(X[train_idx], y[train_idx])
    
    # Validate
    val_score = model.score(X[val_idx], y[val_idx])
    cv_scores.append(val_score)
    print(f"  Validation C-index: {val_score:.4f}")

print(f"\nMean CV C-index: {np.mean(cv_scores):.4f} (+/- {np.std(cv_scores):.4f})")
```

### **Evaluation Metrics**

```python
from sksurv.metrics import (
    concordance_index_censored,
    integrated_brier_score,
    cumulative_dynamic_auc
)

# 1. Concordance Index (Primary Metric)
c_index = concordance_index_censored(
    y_test['event'],
    y_test['time'],
    risk_scores
)[0]
print(f"C-index: {c_index:.4f}")

# 2. Integrated Brier Score (Calibration)
times = np.percentile(y_train['time'], np.linspace(5, 95, 10))
ibs = integrated_brier_score(y_train, y_test, survival_preds, times)
print(f"IBS: {ibs:.4f}")

# 3. Time-dependent AUC
auc_times = [6, 12, 24, 48, 72]
auc_scores, mean_auc = cumulative_dynamic_auc(
    y_train, y_test, risk_scores, auc_times
)
print(f"Mean AUC: {mean_auc:.4f}")
```

### **Model Comparison Table**

| Model | Features | C-index | Training Time | Pros | Cons |
|-------|----------|---------|---------------|------|------|
| Distance Only | 1 | 0.91 | <1 sec | Simple, interpretable | Misses other signals |
| Cox PH | 9 | 0.75-0.80 | ~5 sec | Interpretable, standard | Linear assumptions |
| Random Survival Forest | 15 | 0.80-0.85 | ~30 sec | Non-linear, robust | Black box |
| Gradient Boosting Survival | 20 | 0.82-0.86 | ~60 sec | High accuracy | Requires tuning |
| XGBoost Cox | 25 | 0.83-0.87 | ~45 sec | Very accurate | Hyperparameter sensitive |
| Ensemble | All | 0.85-0.89 | ~2 min | Best accuracy | Complex, slow |

---

## Feature Importance Analysis

### **After Training, Always Check:**

```python
# For tree-based models
feature_importance = pd.DataFrame({
    'feature': feature_names,
    'importance': model.feature_importances_
}).sort_values('importance', ascending=False)

print("Top 10 Features:")
print(feature_importance.head(10))

# Plot
plt.figure(figsize=(10, 6))
top_20 = feature_importance.head(20)
plt.barh(range(len(top_20)), top_20['importance'])
plt.yticks(range(len(top_20)), top_20['feature'])
plt.xlabel('Importance')
plt.title('Top 20 Feature Importances')
plt.tight_layout()
plt.show()
```

### **For Cox Model:**

```python
# Coefficient interpretation
coef_df = pd.DataFrame({
    'feature': feature_names,
    'coefficient': cph.params_,
    'hazard_ratio': np.exp(cph.params_),
    'p_value': cph.summary['p']
}).sort_values('coefficient', key=abs, ascending=False)

print("\nTop Features by Coefficient Magnitude:")
print(coef_df.head(10))

# Interpretation
for idx, row in coef_df.head(5).iterrows():
    direction = "increases" if row['coefficient'] > 0 else "decreases"
    pct_change = (row['hazard_ratio'] - 1) * 100
    print(f"{row['feature']}: {direction} hazard by {abs(pct_change):.1f}%")
```

## Common Pitfalls to Avoid

### 1. **Data Leakage**
```python
#  WRONG: Using future information
train['future_hit'] = (train['time_to_hit_hours'] < 24)  # This is leakage!

#  CORRECT: Only use information available at t0+5h
train['early_growth'] = train['area_growth_abs_0_5h']  # Observed in first 5h
```

### 2. **Treating Censored as Negative**
```python
#  WRONG: Standard classification
from sklearn.linear_model import LogisticRegression
lr = LogisticRegression()
lr.fit(X, train['event'])  # Treats censored=0 as "won't hit"

#  CORRECT: Use survival analysis
from lifelines import CoxPHFitter
cph = CoxPHFitter()
cph.fit(survival_df, duration_col='time_to_hit_hours', event_col='event')
```

### 3. **Ignoring Class Imbalance** (if using classification)
```python
#  WRONG: Ignoring imbalance
rf = RandomForestClassifier()

#  CORRECT: Address imbalance
rf = RandomForestClassifier(
    class_weight='balanced',  # Or 'balanced_subsample'
    # Or use SMOTE, etc.
)
```

### 4. **Not Scaling Features** (for linear models ONLY)
```python
#  WRONG: Cox with unscaled features
cph.fit(train[features + ['time_to_hit_hours', 'event']], ...)

#  CORRECT: Scale first
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
train_scaled = scaler.fit_transform(train[features])
```

### 5. **Overfitting to Small Dataset**
```python
#  WRONG: Too complex for 221 samples
xgb_params = {
    'max_depth': 20,  # Too deep!
    'n_estimators': 5000,  # Too many!
    'min_child_weight': 1  # Too flexible!
}

#  CORRECT: Conservative parameters
xgb_params = {
    'max_depth': 5,  # Shallow trees
    'n_estimators': 200,  # Moderate
    'min_child_weight': 10,  # More regularization
    'subsample': 0.8,
    'colsample_bytree': 0.8
}
```

### 6. **Not Using Cross-Validation**
```python
#  WRONG: Single train/test split
train, test = train_test_split(data, test_size=0.2)
model.fit(train)
score = model.score(test)  # Unreliable with small data!

#  CORRECT: K-fold CV
from sklearn.model_selection import cross_val_score
scores = cross_val_score(model, X, y, cv=5)
print(f"CV Score: {scores.mean():.4f} (+/- {scores.std():.4f})")
```

##  Additional Resources

### **Survival Analysis:**
- lifelines documentation: https://lifelines.readthedocs.io/
- scikit-survival: https://scikit-survival.readthedocs.io/
- "Survival Analysis" by Kleinbaum & Klein (textbook)

### **Feature Engineering:**
- "Feature Engineering for Machine Learning" by Alice Zheng
- Kaggle feature engineering courses

### **Wildfire Science:**
- Understanding fire behavior for domain-informed features
- Evacuation zone planning context


##  Expected Performance Targets

| Target | C-index | Status | Notes |
|--------|---------|--------|-------|
| Baseline | 0.70 | Easy | Distance only |
| Good | 0.75-0.80 | Achievable | Minimal features + Cox/RF |
| Strong | 0.80-0.85 | Requires effort | Extended features + RSF/XGB |
| Excellent | 0.85-0.89 | Challenging | Full engineering + ensemble |
| Winning | 0.89+ | Very hard | Novel features + deep learning |

Remember: **Distance is 90% of the signal.** Don't overcomplicate early - start simple, validate, then iterate!