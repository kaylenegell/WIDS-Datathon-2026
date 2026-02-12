
---

*Add your detailed notes from Evernote here*

# Centroid Kinematics Features (5 features)

Track where the fire is moving - crucial for predicting if it will reach evacuation zones.

## Features List

### 1. centroid_displacement_m - Total Distance Moved ⭐⭐

**What it measures**: How far the fire's center point moved during the 5-hour window (meters).

**Examples from data**:
- Row 2: 8.3 m (barely moved)
- Row 20: 169.3 m (significant movement)
- Row 28: 450.4 m (substantial movement!)
- Many rows: 0.0 (single perimeter, can't calculate)

**Why it matters**:
- **Stationary fires** (low displacement) may burn in place
- **Moving fires** (high displacement) are actively spreading
- Movement suggests wind-driven or terrain-influenced spread
- Only available when num_perimeters_0_5h > 1

**Key insight**: A fire can grow (area increase) without much movement (centroid stays put), or it can move significantly while maintaining similar size. Both patterns matter!

---

### 2. centroid_speed_m_per_h - Speed of Movement ⭐⭐⭐

**What it measures**: Meters per hour the fire centroid is traveling.

**Examples**:
- Row 2: 1.9 m/h (very slow)
- Row 20: 34.8 m/h (moderate speed)
- Row 28: 90.2 m/h (fast!)

**Why it matters**:
- Speed indicates fire behavior:
  - **Slow**: Smoldering, contained, or burning in circles
  - **Fast**: Wind-driven, running, or spotting ahead
- **Faster fires** are harder to contain and more unpredictable
- Combined with direction, predicts trajectory
- Likely highly predictive - fast-moving fires toward zones are dangerous

**Calculation**: `centroid_speed_m_per_h = centroid_displacement_m / dt_first_last_0_5h`

---

### 3. spread_bearing_deg - Direction of Movement ⭐

**What it measures**: Compass bearing (0-360°) of fire spread direction.

**Examples**:
- Row 2: 70.1° (roughly ENE - East-Northeast)
- Row 20: 60.4° (ENE)
- Row 26: 132.7° (SE - Southeast)
- Row 28: 95.7° (E - East)
- Many rows: 0.0 (no movement or single perimeter)

**Why it matters**:
- Direction **matters** when combined with evacuation zone locations
- Fire moving **toward** zones is threatening
- Fire moving **away** from zones is less concerning
- Wind direction often drives fire direction
- Raw degrees are problematic for ML (359° and 1° are close but numerically far)

**Note**: 0° typically means North, 90° = East, 180° = South, 270° = West

---

### 4 & 5. spread_bearing_sin and spread_bearing_cos - Circular Encoding ⭐⭐

**What they measure**: Sine and cosine of the bearing angle.

#### Why circular encoding?

**The problem with raw degrees**:
- 359° and 1° are only 2° apart, but numerically 358 units apart
- ML models see them as very different
- Can't capture "circular" nature of compass directions

**Solution**: Convert to (sin, cos) coordinates:
- `spread_bearing_sin = sin(spread_bearing_deg)`
- `spread_bearing_cos = cos(spread_bearing_deg)`

**Examples**:
- Row 2: sin=0.940, cos=0.340 (ENE direction)
- Row 20: sin=0.870, cos=0.494 (ENE direction)
- Row 26: sin=0.735, cos=-0.678 (SE direction - note negative cos)
- Row 28: sin=0.995, cos=-0.100 (E direction)

**Why it matters**:
- Preserves circular relationships: North (0°) and North (360°) have same sin/cos
- ML-friendly: Models can learn directional patterns
- 2D representation: (sin, cos) forms a unit circle
- Together with evacuation zone directions, enables alignment calculations

**Visual interpretation**:
- sin > 0, cos > 0: NE quadrant (0-90°)
- sin > 0, cos < 0: SE quadrant (90-180°)
- sin < 0, cos < 0: SW quadrant (180-270°)
- sin < 0, cos > 0: NW quadrant (270-360°)

---

## Key Patterns & Insights

### Pattern 1: No Movement Data

Many fires have centroid features = 0.0 (rows 3-9, 11-18, 21-25, etc.)

**Why**:
- Single perimeter observations (num_perimeters_0_5h = 1)
- Can't calculate movement from one point
- Models must rely on other features

### Pattern 2: Fast Movement + Hit

**Row 28 (event_id 20620516)**:
- Displacement: 450.4 m
- Speed: 90.2 m/h (very fast!)
- Bearing: 95.7° (moving East)
- **Result**: Hit in 1.4 hours! (event=1)

**Row 20 (event_id 18292206)**:
- Displacement: 169.3 m
- Speed: 34.8 m/h (moderate)
- Bearing: 60.4° (moving ENE)
- **Result**: Hit in 5.8 hours (event=1)

### Pattern 3: Minimal Movement

**Row 2 (event_id 10892457)**:
- Displacement: 8.3 m (barely moved)
- Speed: 1.9 m/h (very slow)
- **Result**: Censored at 18.9 hours (event=0)

---

## Relationship to Other Features

### Connection to Distance Features

The centroid kinematics become most powerful when combined with distance features:

**Example**: If a fire is:
- Moving at 50 m/h (`centroid_speed_m_per_h`)
- Currently 3,000 m from nearest zone (`dist_min_ci_0_5h`)
- Moving **toward** the zone (alignment features)
- **Estimated time to reach**: ~60 hours (3000m / 50m/h)

### Connection to Directionality Features

- `spread_bearing_sin/cos` feeds into alignment calculations
- Alignment features (coming next) compare fire direction to evacuation zone direction
- This determines if fire is moving **toward** or **away** from zones

---

## Feature Importance Predictions

### Likely MOST Important ⭐⭐⭐

**1. centroid_speed_m_per_h** - Speed of fire movement
- Fast-moving fires are more threatening
- Enables time-to-threat calculations
- Likely very predictive for all time horizons

### Likely MODERATELY Important ⭐⭐

**2. centroid_displacement_m** - Total movement
- Indicates fire mobility
- Correlates with speed but adds information about total distance traveled

**3. spread_bearing_sin & spread_bearing_cos** - Direction encoding
- Critical when combined with evacuation zone locations
- Enables directional analysis
- Most useful in interaction terms with distance/alignment features

### Likely LESS Important (Standalone) ⭐

**4. spread_bearing_deg** - Raw bearing
- Problematic for ML due to circular nature
- Likely ignored by models in favor of sin/cos encoding
- Useful for human interpretation only

---

## Modeling Considerations

### For Feature Engineering

**Speed × Distance ratio**:
- `centroid_speed_m_per_h / dist_min_ci_0_5h` (how fast relative to distance)

**Time to reach**:
- `dist_min_ci_0_5h / centroid_speed_m_per_h` (estimated hours to zone)

**Movement flag**:
- `is_moving = (centroid_displacement_m > threshold)`

**Direction interactions**:
- Combine bearing with evacuation zone directions

### For Handling Missing Movement Data

When `num_perimeters_0_5h = 1`:
- Movement features are 0.0
- Models should learn: "0 movement = unknown movement, not stationary"
- Consider imputation or separate modeling for single-perimeter fires

### For Model Interpretation

- **Speed matters more than displacement** for predictions
- **Direction only matters relative to evacuation zones** (hence alignment features)
- Fast + toward zones = high threat
- Slow or away from zones = lower threat

### Expected Correlations

- `centroid_speed_m_per_h` and `centroid_displacement_m` will correlate (speed = displacement / time)
- Movement features correlate with growth features (active fires do both)
- Movement features correlate with `num_perimeters_0_5h` (data availability)

---

## Real-World Interpretation

### Fire Behavior Insights

**High speed + high displacement**: Wind-driven fire, running behavior

**Low speed + high growth**: Fire burning in place, expanding radially

**High speed + low growth**: Fire moving but not expanding (unusual, possibly spotting)

**Consistent bearing**: Steady wind direction, predictable spread

**Variable bearing**: Changing conditions, less predictable

---

## Summary

The centroid kinematics features capture **fire mobility and trajectory** - essential for predicting if and when the fire will reach evacuation zones. They're most powerful when combined with distance and alignment features to answer: **"Is this fire moving toward the zones, and how fast?"**
