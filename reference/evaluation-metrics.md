# evaluation-metrics

# Evaluation Metrics

## Hybrid Score Formula

```
Final Score = 0.3 × C-index + 0.7 × (1 - Weighted Brier Score)
```

Higher is better (range: 0 to 1)

---

## C-index (Concordance Index)

### What it measures
**Ranking ability**: Can the model correctly order pairs of fires by threat level?

### How it works
For every pair of fires where one hit and one was censored:
- Does the model give higher probability to the fire that hit?
- C-index = proportion of correctly ordered pairs

### Range
- **0.5**: Random guessing (no better than coin flip)
- **0.7**: Decent discrimination
- **0.8**: Good discrimination
- **0.9**: Excellent discrimination
- **1.0**: Perfect ranking

### Why it matters (30% of score)
- Ensures model can identify which fires are more threatening
- Important for prioritization and resource allocation
- Less sensitive to calibration issues

### Limitations
- Doesn't measure probability accuracy
- Only cares about relative ordering
- Can have high C-index with poorly calibrated probabilities

---

## Weighted Brier Score

### What it measures
**Calibration quality**: Are the predicted probabilities accurate?

### How it works
For each fire and each time horizon:
```
Brier Score = (predicted_prob - actual_outcome)²
```

Weighted version accounts for:
- Censoring (IPCW - Inverse Probability of Censoring Weighting)
- Multiple time horizons (12h, 24h, 48h, 72h)

### Range
- **0.0**: Perfect calibration (predicted probabilities match reality)
- **0.25**: Random predictions (for binary outcome)
- **1.0**: Worst possible (always wrong)

### Why it matters (70% of score!)
- Ensures probabilities are meaningful
- Critical for decision-making (evacuation timing)
- Heavily weighted in final score

### Calibration vs. Discrimination
- **Well-calibrated**: If model says 30% probability, ~30% of those fires actually hit
- **Well-discriminated**: Model can separate high-risk from low-risk fires
- Need both for good Brier score!

---

## Monotonicity Constraint

### Requirement
```
prob_12h ≤ prob_24h ≤ prob_48h ≤ prob_72h
```

### Why it's required
- Longer time = more opportunity for fire to reach zones
- Probabilities must be non-decreasing
- Violating this is physically impossible

### Implementation strategies
1. **Post-processing**: Sort predictions after model output
2. **Constrained models**: Build monotonicity into model
3. **Cumulative approach**: Model incremental probabilities

---

## Optimization Strategy

### Balancing C-index and Brier Score

**C-index focus (30%)**:
- Get relative rankings right
- Identify high-risk vs. low-risk fires
- Feature importance for discrimination

**Brier Score focus (70%)**:
- Calibrate probabilities accurately
- Avoid overconfidence or underconfidence
- May need calibration layer

### Common pitfalls
1. **High C-index, poor Brier**: Model discriminates but probabilities are off
2. **Low C-index, good Brier**: Probabilities calibrated but can't rank fires
3. **Monotonicity violations**: Automatic score penalty

### Validation approach
- Track both metrics separately during development
- Ensure monotonicity in all predictions
- Calibration plots to check probability accuracy

*Add your additional evaluation metric notes from Evernote here*
