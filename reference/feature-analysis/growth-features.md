
---

*Add your detailed notes from Evernote here*

# Growth Features (10 features)

Capture how the fire expanded during the first 5 hours. These are likely to be among the most predictive features.

## Features List

### 1. area_first_ha - Initial Fire Size ⭐⭐⭐

**What it measures**: Fire area at t0 (first observation) in hectares.

**Range**: 0.04 → 98.5 ha

**Why it matters**:
- Larger initial fires may have more momentum and resources
- Small fires might be easier to contain OR could explode rapidly
- Available for ALL fires (even single perimeter observations)
- Likely highly predictive - larger fires are generally more threatening

---

### 2. area_growth_abs_0_5h - Absolute Area Growth ⭐⭐⭐

**What it measures**: Hectares gained during the 5-hour window.

**Why it matters**:
- Direct measure of fire expansion
- Larger absolute growth = more aggressive fire
- Only available when num_perimeters_0_5h > 2 (and still 0 sometimes for larger num_perimeters!)
- Likely very predictive when available

---

### 3. area_growth_rel_0_5h - Relative Area Growth ⭐⭐⭐

**What it measures**: Proportional growth (growth / initial area).

**Examples**:
- Row 2: 0.036 (3.6% growth)
- Row 20: 0.803 (80.3% growth - nearly doubled!)
- Row 28: 0.506 (50.6% growth)

**Why it matters**:
- Normalizes for initial fire size
- Small fire growing 50% might be more concerning than large fire growing 5%
- Shows growth rate relative to starting point
- Likely highly predictive - fast relative growth = aggressive fire

---

### 4. area_growth_rate_ha_per_h - Area Growth Rate ⭐⭐⭐

**What it measures**: Hectares per hour expansion rate.

**Examples**:
- Row 2: 0.67 ha/h (slow)
- Row 20: 4.16 ha/h (moderate)
- Row 28: 10.78 ha/h (fast!)

**Why it matters**:
- Speed of expansion
- Higher rate = fire is accelerating
- Useful for extrapolating future growth
- Likely very predictive for near-term threats (12h, 24h horizons)

**Note**: The above 3 growth rates - if 0 for one, 0 for the others.

---

### 5. log1p_area_first - Log-Transformed Initial Area ⭐⭐

**What it measures**: log(1 + area_first_ha) - handles skewness.

**Why it matters**:
- Fire sizes span huge range (4 ha to 6,000+ ha)
- Log transformation reduces impact of extreme values
- Helps models handle the scale better
- Many ML algorithms prefer log-transformed features for skewed distributions

---

### 6. log1p_growth - Log-Transformed Absolute Growth ⭐

**What it measures**: log(1 + area_growth_abs_0_5h).

**Why it matters**:
- Same reasoning as log1p_area_first
- Growth also spans wide range
- Reduces influence of outliers
- Better for linear models and distance-based algorithms

---

### 7. log_area_ratio_0_5h - Log Area Ratio ⭐⭐

**What it measures**: log(final_area / initial_area).

**Examples**:
- Row 2: 0.035 (small ratio, modest growth)
- Row 20: 0.590 (large ratio, significant growth)
- Row 28: 0.409 (substantial growth)

**Why it matters**:
- Multiplicative growth measure
- Captures proportional change on log scale
- Useful for understanding exponential growth patterns
- Different perspective than relative growth

---

### 8. relative_growth_0_5h - Relative Growth (Duplicate?) ⚠️

**What it measures**: Appears to be same as area_growth_rel_0_5h.

**Note**: Looking at the data, this seems to be a duplicate feature. Values match area_growth_rel_0_5h exactly. This might be:
- An error in feature engineering
- Intentional redundancy
- Or they're calculated slightly differently (would need to verify)

**Recommendation**: **DROP!!** - Likely redundant

---

### 9. radial_growth_m - Radial Expansion ⭐⭐

**What it measures**: Change in effective radius (meters) - assumes circular fire.

**Examples**:
- Row 2: 9.0 m (small radial expansion)
- Row 20: 97.2 m (significant radial expansion)
- Row 28: 132.2 m (large radial expansion)

**Why it matters**:
- Geometric perspective on growth
- Radius matters for distance calculations
- Fire growing radially toward evacuation zones is more threatening
- Complements area-based measures

---

### 10. radial_growth_rate_m_per_h - Radial Growth Rate ⭐⭐⭐

**What it measures**: Meters per hour of radius expansion.

**Examples**:
- Row 2: 2.1 m/h (slow)
- Row 20: 20.0 m/h (moderate)
- Row 28: 26.5 m/h (fast)

**Why it matters**:
- Speed of perimeter expansion
- Useful for projecting when fire might reach evacuation zones
- Combined with distance features, can estimate time-to-threat
- Likely very predictive for time-based predictions

---

## Key Patterns & Insights

### Pattern 1: Zero Growth Features

Many fires have ALL growth features = 0.0 (rows 3-9, 11-18, 21-25, etc.)

**Why**:
- These are single perimeter fires (num_perimeters_0_5h = 1)
- Can't calculate growth from one observation
- Models must rely on:
  - Initial size (area_first_ha)
  - Distance features
  - Temporal metadata

### Pattern 2: High Growth = High Threat

**Row 20 (event_id 18292206)**:
- 80% relative growth
- 20 ha absolute growth
- 97m radial expansion
- **Result**: Hit in 5.8 hours! (event=1)

**Row 28 (event_id 20620516)**:
- 51% relative growth
- 54 ha absolute growth
- 132m radial expansion
- **Result**: Hit in 1.4 hours! (event=1, very fast)

### Pattern 3: Large Initial Size ≠ Growth

**Row 21 (event_id 18475932)**:
- Huge initial size: 3,639 ha
- But 0 growth (single perimeter)
- Far from zones: 495 km
- **Result**: Censored (event=0)

---

## Feature Importance Predictions

### Likely MOST Important ⭐⭐⭐

1. **area_growth_rate_ha_per_h** - Direct measure of expansion speed
2. **area_growth_rel_0_5h** - Proportional growth rate
3. **radial_growth_rate_m_per_h** - Perimeter expansion speed
4. **area_first_ha** - Initial threat level (available for all fires)

### Likely MODERATELY Important ⭐⭐

5. **area_growth_abs_0_5h** - Absolute expansion
6. **radial_growth_m** - Geometric expansion
7. **log1p_area_first** - Transformed initial size

### Likely LESS Important ⭐

8. **log1p_growth** - Redundant with absolute growth
9. **log_area_ratio_0_5h** - Redundant with relative growth
10. **relative_growth_0_5h** - Duplicate of #2

---

## Modeling Considerations

### For Feature Engineering

**Interaction terms**:
- `area_growth_rate × dist_min_ci_0_5h` (fast growth + close = very dangerous)

**Ratios**:
- `radial_growth_rate / dist_min_ci_0_5h` (expansion speed relative to distance)

**Flags**:
- `has_growth_data = (num_perimeters_0_5h > 1)`

### For Handling Missing Growth Data

When `num_perimeters_0_5h = 1`:
- Growth features are 0.0 (not truly missing, but uninformative)
- Models should learn: "0 growth + single perimeter = unknown growth, rely on other features"
- Consider separate models or feature sets for single vs. multiple perimeter fires

### For Model Interpretation

- Growth **rates** are more predictive than absolute values
- **Relative growth** matters more than absolute for small fires
- **Radial growth** connects to distance features (spatial reasoning)
- **Initial size** is baseline threat level

### Expected Correlations

- Growth features will correlate with each other (multicollinearity)
- Growth features will correlate with num_perimeters_0_5h (data availability)
- Fast growth likely correlates with hitting evacuation zones (event=1)

---

## Summary

The growth features capture the fire's **aggressiveness and momentum** - absolutely critical for predicting whether and when it will threaten evacuation zones!
