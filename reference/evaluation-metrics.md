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
**Ranking ability**: Can the model correctly order pairs of fires by threat level and timing?

### How it works
C-index evaluates **comparable pairs** of fires:

1. **Event vs Event**: If both fires hit, did you predict higher probability for the fire that hit EARLIER?
   - Fire A hits at 10h, Fire B hits at 30h
   - ✅ Correct if: P(A) > P(B) at any time horizon

2. **Event vs Censored**: Did you predict higher probability for the fire that actually hit?
   - Fire A hits at 20h, Fire B censored at 72h (never hit)
   - ✅ Correct if: P(A) > P(B)

3. **Censored vs Censored**: NOT comparable (we don't know which would have hit first)
   - These pairs are excluded from C-index calculation

**Formula**:
```
C-index = (# correctly ordered pairs) / (# comparable pairs)
```

### Training Data Pairs (69 hits, 152 censored)
- **Event vs Event**: 2,346 pairs (comparing fires that both hit)
- **Event vs Censored**: 10,488 pairs (comparing hit vs non-hit)
- **Total comparable**: 12,834 pairs (52.8% of all possible pairs)
- **Censored vs Censored**: 11,476 pairs (EXCLUDED)

### Range
- **0.5**: Random guessing (no better than coin flip)
- **0.7**: Decent discrimination
- **0.8**: Good discrimination
- **0.9**: Excellent discrimination
- **1.0**: Perfect ranking

### Why it matters (30% of score)
- Ensures model can identify which fires are more threatening
- Uses time-to-event information for ranking (not just binary outcomes)
- Important for prioritization and resource allocation
- Less sensitive to calibration issues

### Limitations
- Doesn't measure probability accuracy
- Only cares about relative ordering
- Can have high C-index with poorly calibrated probabilities

---

## Weighted Brier Score

### What it measures
**Calibration quality**: Are the predicted probabilities accurate at multiple time horizons?

### How it works
For each fire and each time horizon:
```
Brier Score = (predicted_prob - actual_outcome)²
```

### Time Horizons and Weighting
Evaluated at **3 time horizons** (not 4):
- **24 hours** (30% weight)
- **48 hours** (40% weight) ⭐ Highest weighted!
- **72 hours** (30% weight)

**Final Weighted Brier Score**:
```
0.3 × Brier@24h + 0.4 × Brier@48h + 0.3 × Brier@72h
```

**Why 48h is weighted highest**:
- 24-48 hours is the strongest operational value zone
- Balances actionable lead time with decision urgency
- 72h included for extended planning but less operationally immediate

### Censor-Aware Evaluation
How actual outcomes are determined at each horizon H:

1. **Hits before H**: `actual_outcome = 1` (fire hit by this time)
2. **Hits after H**: `actual_outcome = 0` (fire will hit, but not yet at horizon H)
3. **Censored at/after H**: `actual_outcome = 0` (fire didn't hit by H)
4. **Censored before H**: **EXCLUDED** (we don't know what would have happened)

**Example for 48h horizon**:
- Fire hits at 30h → `actual_outcome = 1`
- Fire hits at 60h → `actual_outcome = 0` (not yet hit at 48h)
- Fire censored at 72h (never hit) → `actual_outcome = 0`
- Fire censored at 40h → **EXCLUDED** (censored before 48h horizon)

### Training Data Breakdown (221 fires: 69 hits, 152 censored)

**At 24h horizon**:
- **196 fires evaluated** (88.7%)
  - Outcome=1: 63 fires (hit by 24h)
  - Outcome=0: 133 fires (6 hit later + 127 censored ≥24h)
- **25 fires excluded** (censored before 24h)

**At 48h horizon**:
- **166 fires evaluated** (75.1%)
  - Outcome=1: 66 fires (hit by 48h)
  - Outcome=0: 100 fires (3 hit later + 97 censored ≥48h)
- **55 fires excluded** (censored before 48h)

**At 72h horizon**:
- **69 fires evaluated** (31.2%)
  - Outcome=1: 69 fires (ALL hits)
  - Outcome=0: 0 fires
- **152 fires excluded** (ALL censored fires)

### Critical Insight
The 72h horizon evaluates **ONLY fires that hit** - it measures temporal ordering (when fires hit), not discrimination (hit vs non-hit). This is why 24h and 48h horizons are more important for overall model discrimination.

### Range
- **0.0**: Perfect calibration (predicted probabilities match reality)
- **0.25**: Random predictions (for binary outcome)
- **1.0**: Worst possible (always wrong)

### Why it matters (70% of score!)
- Ensures probabilities are meaningful for decision-making
- Critical for evacuation timing at operationally relevant horizons
- Heavily weighted in final score
- Requires both discrimination AND calibration

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
