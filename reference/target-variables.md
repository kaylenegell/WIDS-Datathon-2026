# Target Variables (Training Data Only)

The training dataset includes two target columns that define the survival analysis problem.

## The 2 Target Variables

### 1. time_to_hit_hours - Time to Event ⭐⭐⭐

**What it measures**: Time from t0+5h (end of observation window) until fire comes within 5 km of an evacuation zone centroid.

**Range**: 
- **For hits (event=1)**: 0.52 to 71.9 hours
- **For censored (event=0)**: Last observed time ≤ 72 hours

**Units**: Hours

**Key characteristics**:
- Measured from **t0+5h**, not from fire start
- Represents time from end of feature observation window to event
- For censored fires, this is the last time they were observed (not when they would have hit)

**Examples from data**:
- Row 20: 5.8 hours (hit quickly after observation window)
- Row 28: 1.4 hours (hit very quickly!)
- Row 2: 18.9 hours (censored - didn't hit within 72h)
- Row 21: 72.0 hours (censored at maximum observation time)

---

### 2. event - Event Indicator ⭐⭐⭐

**What it measures**: Binary indicator of whether the fire reached within 5km of evacuation zones.

**Values**:
- **1**: Fire HIT (came within 5km) during the 72-hour window
- **0**: Fire CENSORED (did not hit within 72h observation period)

**Distribution in training data**:
- **Hits (event=1)**: 69 fires (31.2%)
- **Censored (event=0)**: 152 fires (68.8%)

**Critical distinction**:
- **event=0 does NOT mean "will never hit"**
- It means "didn't hit within our 72-hour observation window"
- This is **right-censored data** - we don't know what happened after 72h

---

## Understanding the Relationship

### For Hits (event=1)

**Interpretation**: Fire reached within 5km at a specific time
- `time_to_hit_hours` = actual time when threshold was crossed
- Range: 0.52 to 71.9 hours in training data
- These are **observed events**

**Example**:
- event_id 20620516 (Row 28)
- event = 1 (hit)
- time_to_hit_hours = 1.4
- **Meaning**: Fire reached 5km threshold 1.4 hours after t0+5h

### For Censored (event=0)

**Interpretation**: Fire did not reach within 5km during observation
- `time_to_hit_hours` = last observed time (≤ 72h)
- Does NOT mean fire will never hit
- These are **censored observations**

**Example**:
- event_id 10892457 (Row 2)
- event = 0 (censored)
- time_to_hit_hours = 18.9
- **Meaning**: Fire was observed for 18.9 hours and hadn't hit by then

---

## Survival Analysis Context

### Why This is Survival Data

**Time-to-event**: We're modeling when (not just if) fires reach zones

**Right-censoring**: 69% of observations are censored
- Standard classification treats censored as "negative" (wrong!)
- Survival models use censored data properly

**Survival function**: S(t) = P(T > t)
- Probability fire hasn't hit by time t
- Our predictions: 1 - S(12), 1 - S(24), 1 - S(48), 1 - S(72)

---

## Prediction Task

### What We Must Predict (Test Data)

For each fire in test set, predict 4 probabilities:
- `prob_hit_12h`: P(fire hits within 12 hours from t0+5h)
- `prob_hit_24h`: P(fire hits within 24 hours from t0+5h)
- `prob_hit_48h`: P(fire hits within 48 hours from t0+5h)
- `prob_hit_72h`: P(fire hits within 72 hours from t0+5h)

**Constraint**: prob_12h ≤ prob_24h ≤ prob_48h ≤ prob_72h (monotonicity)

### Relationship to Targets

**For training fires with event=1**:
- If time_to_hit_hours = 10:
  - prob_hit_12h should be high (hit before 12h)
  - prob_hit_24h should be high (hit before 24h)
  - prob_hit_48h should be high (hit before 48h)
  - prob_hit_72h should be high (hit before 72h)

**For training fires with event=0**:
- If time_to_hit_hours = 30 (censored):
  - We know fire didn't hit by 30h
  - But don't know if it would have hit at 40h, 50h, etc.
  - Survival models handle this uncertainty

---

## Key Insights for Modeling

### 1. High Censoring Rate (69%)

**Challenge**: Most fires didn't hit within observation window
- Can't treat as "negative" examples
- Must use survival analysis methods
- IPCW (Inverse Probability of Censoring Weighting) if using classification

### 2. Time Distribution

**For hits**:
- Median: ~20 hours
- Fast hits: <5 hours (immediate threat)
- Slow hits: >50 hours (gradual approach)
- Wide range: 0.52 to 71.9 hours

**Implication**: Need to predict across diverse time scales

### 3. Prediction Horizons

**Why 12h, 24h, 48h, 72h?**
- **12h**: Immediate evacuation decisions
- **24h**: Next-day planning
- **48h**: 2-day resource allocation
- **72h**: 3-day strategic planning

**Challenge**: Must predict at all 4 horizons with monotonicity

### 4. Feature-Target Relationship

**Features observed**: First 5 hours (t0 to t0+5h)
**Target measured**: From t0+5h to t0+77h (up to 72h after feature window)

**Gap**: Predicting 12-72 hours into future based on first 5 hours
- Fire behavior can change
- Weather conditions evolve
- Containment efforts affect trajectory

---

## Validation Considerations

### Stratification

**Important**: Stratify by event status
- Ensure balanced hit/censored ratio in folds
- Prevents all-censored or all-hit validation sets

### Metrics to Track

**C-index**: Ranking ability
- Can model order fires by threat level?
- Handles censored data naturally

**Brier Score**: Calibration quality
- Are predicted probabilities accurate?
- Requires careful handling of censoring

### Time-Aware Validation

**Consider**:
- Temporal ordering if relevant
- Event_id grouping (fires from same event)
- Censoring distribution in splits

---

## Common Pitfalls to Avoid

### ❌ Wrong Approaches

1. **Treating censored as negative class**
   - Ignores that they might hit after 72h
   - Biases model toward underestimating risk

2. **Ignoring censored observations**
   - Throws away 69% of data
   - Severe information loss

3. **Predicting time_to_hit directly without handling censoring**
   - Regression on censored data is problematic
   - Need survival regression (AFT models)

### ✅ Right Approaches

1. **Use survival models**
   - Cox Proportional Hazards
   - Random Survival Forests
   - Gradient Boosted Survival

2. **IPCW weighting if using classification**
   - Weights observations by censoring probability
   - Corrects for censoring bias

3. **Proper evaluation metrics**
   - C-index for ranking
   - Time-dependent Brier Score for calibration
   - Both handle censoring correctly

---

## Summary

The target variables define a **survival analysis problem** with:
- **Time-to-event**: When fires reach 5km threshold
- **Right-censoring**: 69% didn't hit within 72h
- **Multiple horizons**: Must predict at 12h, 24h, 48h, 72h
- **Monotonicity**: Probabilities must increase with time

Success requires understanding survival analysis concepts and properly handling censored data throughout the modeling pipeline.