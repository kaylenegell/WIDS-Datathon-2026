# modeling-strategies

# Modeling Strategies

## Approach Options

### 1. Direct Survival Models
**Pros**: Designed for censored data, theoretically sound
**Cons**: May need calibration, limited by sample size

**Models to try**:
- Cox Proportional Hazards
- Random Survival Forests
- Gradient Boosted Survival (XGBoost, LightGBM with survival objective)

### 2. Binary Classification per Time Horizon
**Pros**: Simple, many model options
**Cons**: Ignores censoring, treats as independent predictions

**Approach**:
- 4 separate models (12h, 24h, 48h, 72h)
- Post-process for monotonicity
- May need IPCW weighting

### 3. Regression on Time-to-Event
**Pros**: Direct time prediction
**Cons**: Censored data handling tricky

**Approach**:
- Predict time_to_event_hours
- Convert to probabilities at each horizon
- Handle censored observations carefully

### 4. Ensemble Approach
**Pros**: Combines strengths of multiple models
**Cons**: More complex, risk of overfitting

**Approach**:
- Blend survival models + classification models
- Stack predictions
- Weighted average based on validation performance

---

## Feature Engineering Priorities

### High Priority (Likely Most Predictive)

**Distance features**:
- dist_min_ci_0_5h (THE most important)
- closing_speed_m_per_h
- dist_accel_m_per_h2

**Growth features**:
- area_growth_rate_ha_per_h
- area_growth_rel_0_5h
- radial_growth_rate_m_per_h

**Directionality features**:
- along_track_speed (direct approach speed)
- alignment_cos (toward/away indicator)

### Medium Priority

**Kinematics features**:
- centroid_speed_m_per_h
- spread_bearing_sin/cos

**Temporal metadata**:
- event_start_month (seasonality)
- event_start_hour (diurnal patterns)

**Data quality**:
- low_temporal_resolution_0_5h (reliability flag)
- num_perimeters_0_5h (observation quality)

### Feature Engineering Ideas

**Interaction terms**:
- distance × growth_rate (close + fast growing = danger)
- distance × alignment (close + approaching = danger)
- speed × distance (time-to-threat estimate)

**Derived features**:
- time_to_5km = (dist_min - 5000) / closing_speed
- threat_score = growth_rate / distance
- has_movement_data = (num_perimeters > 1)

**Transformations**:
- Log transforms for skewed features
- Circular encoding for hour/month
- Polynomial features for key predictors

---

## Handling Data Challenges

### Challenge 1: Small Sample Size (221)

**Strategies**:
- Simple models over complex ones
- Strong regularization
- Careful cross-validation
- Feature selection critical

### Challenge 2: High Censoring (69%)

**Strategies**:
- Use survival models designed for censoring
- IPCW weighting if using classification
- Don't treat censored as negative class

### Challenge 3: Single-Perimeter Fires (72%)

**Strategies**:
- Separate models for single vs. multiple perimeter?
- Interaction terms with low_temporal_resolution flag
- Rely on distance and initial size features

### Challenge 4: Class Imbalance (31% hits)

**Strategies**:
- Stratified sampling in CV
- Class weights in models
- Evaluation metrics already handle this (C-index, Brier)

---

## Validation Strategy

### Cross-Validation Approach

**Stratified K-Fold** (k=5 or k=10):
- Stratify by event (hit/censored)
- Ensures balanced splits
- Track both C-index and Brier score

**Time-aware considerations**:
- Fires from same event_id should stay together
- Consider temporal ordering if relevant

### Metrics to Track

**During development**:
- C-index (ranking ability)
- Brier Score (calibration)
- Hybrid Score (final metric)
- Monotonicity violations

**Diagnostic plots**:
- Calibration curves
- Predicted vs. actual probabilities
- Feature importance
- Residual analysis

---

## Calibration Strategies

### Why calibration matters
- 70% of final score is Brier (calibration-focused)
- Predictions must be accurate probabilities
- Many models output uncalibrated scores

### Calibration methods

**1. Platt Scaling**:
- Logistic regression on model outputs
- Simple, effective
- Fits sigmoid to predictions

**2. Isotonic Regression**:
- Non-parametric calibration
- More flexible than Platt
- Risk of overfitting with small data

**3. Beta Calibration**:
- Extension of Platt scaling
- Better for extreme probabilities
- Good for imbalanced data

**4. Temperature Scaling**:
- Single parameter calibration
- Common in neural networks
- Simple and effective

### Implementation
- Use separate calibration set (not training or test)
- Apply after main model training
- Preserve monotonicity after calibration

---

## Ensemble Strategies

### Model diversity
- Survival models (Cox, RSF)
- Gradient boosting (XGBoost, LightGBM)
- Neural networks (if enough data)
- Different feature sets

### Combination methods
- Simple average
- Weighted average (based on validation performance)
- Stacking (meta-model on predictions)
- Rank averaging (for C-index optimization)

### Considerations
- Don't overfit to validation set
- Maintain monotonicity in ensemble
- Calibrate ensemble output

---

## Monotonicity Enforcement

### Post-processing approach
```python
# Ensure prob_12h ≤ prob_24h ≤ prob_48h ≤ prob_72h
probs_sorted = np.sort([prob_12h, prob_24h, prob_48h, prob_72h])
prob_12h, prob_24h, prob_48h, prob_72h = probs_sorted
```

### Model-based approach
- Cumulative models (predict increments)
- Constrained optimization
- Monotonic neural networks

### Validation
- Check all predictions satisfy constraint
- Penalize violations in validation metric

