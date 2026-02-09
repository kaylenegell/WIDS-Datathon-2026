# WiDS Datathon 2026 - Competition Overview

*Add your detailed competition notes from Evernote here*

## Key Points
- Predict wildfire threat probabilities at 12h, 24h, 48h, 72h
- 221 training samples (69 hits, 152 censored)
- 95 test samples (51 public, 43 private)
- Submission deadline: May 1, 2026

# WiDS Datathon 2026 - Competition Overview

## Challenge Description

Predict the probability that a wildfire will reach within 5km of evacuation zone centroids at four time horizons:
- 12 hours
- 24 hours  
- 48 hours
- 72 hours

## Problem Type

**Survival Analysis** with right-censored data
- Not standard classification (hit/no-hit)
- Must predict time-dependent probabilities
- Handle censored observations (fires that didn't hit within observation window)

## Dataset

### Training Data
- **221 samples** total
  - 69 hits (event=1) - 31%
  - 152 censored (event=0) - 69%
- **34 features** across 6 categories
- **2 target columns**: event (binary), time_to_event_hours

### Test Data
- **95 samples** total
  - 51 public leaderboard samples
  - 43 private leaderboard samples (final scoring)

### Data Challenges
- **Small dataset**: Only 221 training samples
- **High censoring rate**: 69% censored
- **Class imbalance**: 31% hits vs 69% censored
- **Single-perimeter fires**: 72% have only 1 observation (160/221)

## Submission Format

Must predict 4 probabilities per fire:
- `prob_hit_12h`: P(hit within 12 hours)
- `prob_hit_24h`: P(hit within 24 hours)
- `prob_hit_48h`: P(hit within 48 hours)
- `prob_hit_72h`: P(hit within 72 hours)

**Monotonicity constraint**: prob_12h ≤ prob_24h ≤ prob_48h ≤ prob_72h

## Evaluation Metric

**Hybrid Score** = 0.3 × C-index + 0.7 × (1 - Weighted Brier Score)

### C-index (30% weight)
- Measures ranking ability
- Range: 0.5 (random) to 1.0 (perfect)
- Assesses: Can model correctly order fires by threat level?

### Weighted Brier Score (70% weight)
- Measures calibration quality
- Range: 0 (perfect) to 1 (worst)
- Assesses: Are predicted probabilities accurate?
- Inverted in formula: (1 - WBS) so higher is better

## Timeline

- **Competition Start**: January 2026
- **Submission Deadline**: May 1, 2026
- **Winners Announced**: May 2026
- **WiDS Conference**: June 2026

## Key Constraints

1. **Monotonicity**: Probabilities must increase with time
2. **Survival modeling**: Must handle censored data properly
3. **Small sample size**: Careful validation strategy needed
4. **Feature engineering**: Critical due to limited data

## Success Factors

1. **Understanding survival analysis** concepts
2. **Proper handling of censored data**
3. **Feature engineering** from limited observations
4. **Calibration** of probability predictions
5. **Validation strategy** for small dataset
