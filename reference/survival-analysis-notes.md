# survival-analysis-notes

# Survival Analysis Notes

## What is Survival Analysis?

**Definition**: Statistical methods for analyzing time-to-event data, especially when some observations are censored.

**Key concept**: Not just "did it happen?" but "when did it happen?"

---

## Right-Censored Data

### Definition
Observation ends before the event occurs - we know the event **hasn't happened yet**, but don't know **if or when** it will happen.

### In this competition
- **Event**: Fire reaches within 5km of evacuation zone
- **Censored**: Fire didn't reach zones during observation window (up to 72h)
- **69% censored** in training data (152 out of 221 fires)

### Why it matters
- Can't treat censored as "negative" examples
- They're "not yet" not "never"
- Standard classification ignores this information
- Survival models use censored data properly

---

## Survival Function S(t)

### Definition
```
S(t) = P(T > t) = Probability of surviving past time t
```

In our context:
```
S(t) = P(fire hasn't reached zones by time t)
```

### Properties
- S(0) = 1 (at start, no fires have hit)
- S(∞) = 0 (eventually all fires either hit or are contained)
- S(t) is non-increasing (monotonic)

### Relationship to our predictions
```
P(hit by time t) = 1 - S(t)

prob_hit_12h = 1 - S(12)
prob_hit_24h = 1 - S(24)
prob_hit_48h = 1 - S(48)
prob_hit_72h = 1 - S(72)
```

---

## Hazard Function h(t)

### Definition
```
h(t) = Instantaneous risk of event at time t
```

**Interpretation**: Given fire hasn't hit yet, what's the risk it hits in the next instant?

### Relationship to survival
```
S(t) = exp(-∫h(u)du from 0 to t)
```

### In practice
- High hazard = high immediate risk
- Hazard can vary over time
- Some models estimate hazard directly

---

## Censoring Mechanisms

### Types of censoring

**1. Right censoring** (our case):
- Observation ends before event
- Know: T > t_censored
- Don't know: actual event time

**2. Left censoring**:
- Event occurred before observation started
- Not relevant for this competition

**3. Interval censoring**:
- Event occurred between two observation times
- Partially relevant (perimeter observations)

### Handling censored data

**Wrong approach**:
- Treat censored as negative class
- Ignore censored observations
- Impute event times

**Right approach**:
- Use survival models (Cox, AFT, etc.)
- Incorporate censoring in loss function
- IPCW (Inverse Probability of Censoring Weighting)

---

## Common Survival Models

### 1. Kaplan-Meier Estimator
- Non-parametric survival curve
- Baseline for comparison
- Doesn't use features

### 2. Cox Proportional Hazards
- Semi-parametric model
- Estimates hazard ratios
- Assumes proportional hazards

### 3. Accelerated Failure Time (AFT)
- Parametric model
- Models time-to-event directly
- Assumes distribution (Weibull, log-normal, etc.)

### 4. Random Survival Forests
- Non-parametric ensemble
- Handles non-linear relationships
- Good for complex interactions

### 5. Neural Survival Models
- Deep learning approaches
- DeepSurv, DeepHit, etc.
- Flexible but need more data

---

## Key Concepts for This Competition

### 1. Time-dependent predictions
- Must predict at 4 specific time points
- Not just single survival probability
- Monotonicity constraint applies

### 2. High censoring rate (69%)
- Most fires didn't hit zones
- Censored data is informative
- Models must handle this properly

### 3. Small sample size (221)
- Limits complex model choices
- Cross-validation critical
- Feature engineering important

### 4. Evaluation metrics
- C-index: Ranking ability
- Brier Score: Calibration quality
- Both matter for final score

---

## Practical Considerations

### Feature engineering
- Time-varying covariates (growth, movement)
- Baseline features (initial distance, size)
- Interaction terms

### Model selection
- Balance complexity vs. sample size
- Ensemble approaches
- Calibration layer may be needed

### Validation strategy
- Time-aware splits
- Stratify by event/censored
- Track both C-index and Brier score
