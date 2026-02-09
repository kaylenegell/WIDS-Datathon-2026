# data-insights

## Dataset Overview

### Training Data Statistics
- **Total samples**: 221
- **Hits (event=1)**: 69 (31.2%)
- **Censored (event=0)**: 152 (68.8%)
- **Features**: 34 (across 6 categories)

### Test Data
- **Total samples**: 95
- **Public leaderboard**: 51 samples
- **Private leaderboard**: 43 samples

---

## Critical Data Patterns

### 1. Single-Perimeter Dominance

**Finding**: 160 out of 221 fires (72%) have only 1 perimeter observation

**Implications**:
- num_perimeters_0_5h = 1
- dt_first_last_0_5h = 0.0
- low_temporal_resolution_0_5h = 1
- ALL growth features = 0.0
- ALL movement features = 0.0

**Modeling impact**:
- Can't calculate rates of change
- Must rely on distance and initial size
- Consider separate models for single vs. multiple perimeter fires

### 2. High Censoring Rate

**Finding**: 69% of fires didn't reach evacuation zones

**Implications**:
- Not "negative" examples - they're "not yet"
- Standard classification inappropriate
- Survival models essential
- IPCW weighting if using classification

### 3. Distance Distribution

**Key thresholds**:
- **< 5km**: Already within threat zone (immediate danger)
- **5-20km**: Near-term threat (hours to days)
- **20-50km**: Medium-term threat (days)
- **> 50km**: Low probability within 72h

**Finding**: dist_min_ci_0_5h is likely THE strongest predictor

### 4. The Centroid Paradox

**Finding**: Fire centroid can move AWAY while perimeter reaches zones

**Examples**:
- Row 26: alignment_cos = -0.888 (moving away) but HIT in 0.52h
- Row 28: along_track_speed = -33.7 m/h (retreating) but HIT in 1.4h

**Explanation**:
- Asymmetric fire growth
- One side expands toward zone while centroid shifts away
- Distance to centroid ≠ distance to nearest fire edge

**Modeling impact**:
- Directionality features imperfect
- Must combine with radial growth
- Distance change alone insufficient

---

## Feature Correlations

### Expected Strong Correlations

**Distance features** (redundant):
- dist_change_ci_0_5h ≈ -projected_advance_m (exact opposites)
- dist_slope_ci_0_5h ≈ -closing_speed_m_per_h (exact opposites)

**Growth features** (related):
- area_growth_abs, area_growth_rel, area_growth_rate (all measure growth)
- relative_growth_0_5h = area_growth_rel_0_5h (duplicate)

**Movement features** (mathematical):
- along_track_speed = centroid_speed × alignment_cos
- centroid_speed² = along_track² + cross_track²

### Feature Selection Implications
- Drop exact duplicates
- Keep one version of redundant features
- Consider PCA or feature selection

---

## Temporal Patterns

### Seasonality (event_start_month)
- **Peak fire season**: June-September (months 6-9)
- **Shoulder season**: April-May, October-November
- **Off-season**: December-March

**Expected**: Summer fires more aggressive, faster-spreading

### Diurnal Patterns (event_start_hour)
- **Afternoon (14-18)**: Hottest, driest, windiest
- **Evening (18-22)**: Cooling, humidity rising
- **Night (22-6)**: Coolest, highest humidity
- **Morning (6-14)**: Warming, drying

**Expected**: Afternoon fires spread faster initially

### Day of Week (event_start_dayofweek)
- **Weekends**: More recreational activity
- **Weekdays**: Different ignition patterns

**Expected**: Weak signal, likely least important temporal feature

---

## Missing Data Patterns

### Growth/Movement Features
**When**: num_perimeters_0_5h = 1 (72% of data)
**Values**: All = 0.0
**Interpretation**: Not missing, but uninformative (can't calculate from 1 point)

### Distance Trend Features
**When**: num_perimeters_0_5h = 1
**Values**: All = 0.0
**Available**: Only dist_min_ci_0_5h and dist_std_ci_0_5h

### No True Missing Data
- All features have values for all fires
- Zeros indicate "can't calculate" not "missing"
- Models must learn this distinction

---

## Target Variable Insights

### Time to Event Distribution
**For hits (event=1)**:
- Range: 0.52 to 71.9 hours
- Median: ~20 hours
- Some very fast hits (<2 hours)
- Some slow burns (>60 hours)

### Censoring Times
**For censored (event=0)**:
- All censored at observation end
- Range: up to 72 hours
- Don't know if/when they would have hit

---

## Data Quality Indicators

### High Quality Fires
- num_perimeters_0_5h > 5
- dt_first_last_0_5h > 2 hours
- low_temporal_resolution_0_5h = 0
- Growth and movement features reliable

### Low Quality Fires
- num_perimeters_0_5h = 1
- dt_first_last_0_5h = 0
- low_temporal_resolution_0_5h = 1
- Limited information available

### Modeling Strategy
- Weight by data quality?
- Separate models by quality tier?
- Interaction terms with quality flags?

---

## Key Insights for Modeling

1. **Distance dominates**: Fires >100km away rarely hit
2. **Single-perimeter challenge**: 72% have limited data
3. **Centroid paradox**: Movement direction imperfect indicator
4. **Feature redundancy**: Several exact duplicates to remove
5. **Censoring critical**: Must use survival methods
6. **Small sample**: Feature engineering > model complexity
