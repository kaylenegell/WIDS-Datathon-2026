# Distance Features (9 features)

Proximity measurements - how close the fire is to evacuation zones.

## Features List

1. **min_distance_to_zone_m** - Closest point on fire to any zone (meters) ⭐⭐⭐
2. **max_distance_to_zone_m** - Farthest point on fire from zones (meters)
3. **mean_distance_to_zone_m** - Average distance across fire perimeter (meters)
4. **centroid_distance_to_zone_m** - Distance from fire center to zones (meters) ⭐⭐
5. **min_distance_to_zone_km** - Same as #1 but in kilometers
6. **max_distance_to_zone_km** - Same as #2 but in kilometers
7. **mean_distance_to_zone_km** - Same as #3 but in kilometers
8. **centroid_distance_to_zone_km** - Same as #4 but in kilometers
9. **distance_range_m** - Difference between max and min (fire size indicator)

## Key Insights

**Most Important Features**:
- **min_distance_to_zone_m** - Direct threat indicator
- **centroid_distance_to_zone_m** - Overall proximity

**Redundancy**: Meters vs kilometers versions are perfectly correlated (just unit conversion)
- Models should use one or the other, not both
- Meters preferred for precision

**Distance Range**: max - min gives fire elongation/spread
- Large range = fire stretched toward/away from zones
- Small range = compact fire

## The Centroid Paradox

**Critical Insight**: Fire centroid can move AWAY while perimeter reaches zones!

**Example**:
- Fire grows asymmetrically toward a zone
- Centroid shifts away (weighted by total area)
- But closest edge gets much closer
- **min_distance** captures threat, **centroid_distance** misses it

**Implication**: min_distance is more reliable than centroid_distance for threat assessment

## Distance Patterns

**High Threat Scenarios**:
- min_distance < 1000m (very close)
- min_distance decreasing over time
- Small distance_range (compact fire approaching)

**Lower Threat Scenarios**:
- min_distance > 10,000m (far away)
- min_distance increasing over time
- Large distance_range with high min_distance (fire spreading away)

## Feature Engineering Ideas

- **Distance velocity**: Change in min_distance over time
- **Approach rate**: (previous_min - current_min) / time_elapsed
- **Distance ratios**: min/centroid, min/mean (shape indicators)
- **Normalized distances**: Distance relative to fire size

## Expected Feature Importance

**Highest**: 
- min_distance_to_zone_m (or _km)
- centroid_distance_to_zone_m (or _km)

**Moderate**:
- mean_distance_to_zone_m
- distance_range_m

**Lower**:
- max_distance_to_zone_m (less relevant for threat)
- Redundant km versions if meters already used

---

*Add your detailed notes from Evernote here*

# Distance to Evacuation Zone Centroids Features (9 features)

Proximity measurements - how close the fire is to evacuation zones.

## Features List

### 1. dist_min_ci_0_5h - Minimum Distance to Nearest Zone ⭐⭐⭐

**What it measures**: Closest distance from fire to ANY evacuation zone centroid (meters).

**Examples from data**:
- Row 2: 6,166 m = 6.2 km (moderately close)
- Row 3: 2,931 m = 2.9 km (close!)
- Row 20: 1,135 m = 1.1 km (very close!)
- Row 28: 1,784 m = 1.8 km (close)
- Row 21: 495,662 m = 496 km (extremely far!)

**Why it matters**:
- **THE most important feature** - directly relates to the 5km threshold
- Fires already within 5km are immediate threats
- Fires far away (>50km) are unlikely to reach zones within 72h
- Available for ALL fires (even single perimeter)
- Likely **THE strongest predictor**

**Critical thresholds**:
- **< 5,000m**: Already within threat zone
- **5,000-20,000m**: Near-term threat (hours to days)
- **20,000-50,000m**: Medium-term threat (days)
- **> 50,000m**: Low probability of reaching zones in 72h

---

### 2. dist_std_ci_0_5h - Standard Deviation of Distances ⭐⭐

**What it measures**: Variability in distances to different evacuation zones.

**Examples**:
- Row 2: 0.21 (low variability - zones at similar distances)
- Row 28: 100.4 (high variability - zones at very different distances)
- Many rows: 0.0 (single perimeter OR all zones equidistant)

**Why it matters**:
- **High std**: Zones spread out in different directions
  - Fire might threaten one zone but not others
  - Directional spread matters more
- **Low std**: Zones clustered at similar distance
  - Fire threatens all zones equally
  - Distance matters more than direction
- Helps understand spatial distribution of evacuation zones

---

### 3. dist_change_ci_0_5h - Change in Distance ⭐⭐

**What it measures**: Distance at 5h minus distance at 0h (d5 - d0).

**Sign convention**:
- **Negative**: Fire is getting CLOSER (distance decreasing) ⚠️
- **Positive**: Fire is moving AWAY (distance increasing) ✓
- **Zero**: Distance unchanged OR single perimeter

**Examples**:
- Row 2: +0.44 m (slightly moving away - good!)
- Row 26: +40.7 m (moving away - good!)
- Row 28: +212.9 m (moving away significantly - seems good?)
- Many rows: 0.0 (single perimeter)

**Why it matters**:
- Negative values = DANGER - fire approaching zones
- Positive values = SAFER - fire moving away
- **But wait...** Row 28 moved AWAY (+212.9m) but still HIT in 1.4h! 🤔
  - This shows distance change alone isn't enough
  - Fire can move away from centroid while perimeter expands toward zone
  - Or fire can curve back

---

### 4. dist_slope_ci_0_5h - Rate of Distance Change ⭐⭐

**What it measures**: Linear slope of distance vs. time (meters/hour).

**Sign convention** (same as dist_change):
- **Negative**: Closing on zones (m/h toward) ⚠️
- **Positive**: Moving away from zones (m/h away) ✓

**Examples**:
- Row 2: +0.11 m/h (slowly moving away)
- Row 26: +4.41 m/h (moving away)
- Row 28: +56.8 m/h (moving away fast)

**Why it matters**:
- Rate of approach/retreat
- More refined than total change (accounts for time)
- Can extrapolate: "At this rate, when will fire reach 5km?"
- But again, Row 28 shows limitations (moving away but still hit)

---

### 5. closing_speed_m_per_h - Closing Speed ⭐⭐⭐

**What it measures**: Speed at which fire is closing distance (m/h).

**Sign convention** (OPPOSITE of dist_slope!):
- **Positive**: Fire IS closing (approaching) ⚠️
- **Negative**: Fire NOT closing (moving away) ✓

**Examples**:
- Row 2: -0.10 m/h (not closing, moving away)
- Row 26: -9.45 m/h (not closing, moving away fast)
- Row 28: -42.6 m/h (not closing, moving away)

**Why it matters**:
- Intuitive sign: Positive = danger, negative = safer
- Essentially `-dist_slope_ci_0_5h`
- Easier to interpret: "Fire closing at 50 m/h" vs "slope of -50 m/h"

---

### 6. closing_speed_abs_m_per_h - Absolute Closing Speed ⭐

**What it measures**: Absolute value of closing speed (always positive).

**Examples**:
- Row 2: 0.10 m/h
- Row 26: 9.45 m/h
- Row 28: 42.6 m/h

**Why it matters**:
- Magnitude of distance change regardless of direction
- High absolute speed = dynamic fire (moving significantly)
- Low absolute speed = stable fire (distance not changing much)
- Useful for identifying active vs. static fires

---

### 7. projected_advance_m - Projected Advance ⚠️ REDUNDANT

**What it measures**: Distance at 0h minus distance at 5h (d0 - d5).

**Sign convention** (OPPOSITE of dist_change):
- **Positive**: Fire advanced TOWARD zones ⚠️
- **Negative**: Fire retreated AWAY from zones ✓

**Examples**:
- Row 2: -0.44 m (retreated slightly)
- Row 26: -40.7 m (retreated)
- Row 28: -212.9 m (retreated significantly)

**Why it matters**:
- Intuitive sign: Positive = advanced toward threat
- Essentially `-dist_change_ci_0_5h` - **it is exactly!**
- Same information, different framing
- **Likely redundant with dist_change**

---

### 8. dist_accel_m_per_h2 - Distance Acceleration ⭐⭐

**What it measures**: Acceleration in distance change (meters/hour²).

**Examples**:
- Row 2: +0.07 m/h² (accelerating away)
- Row 26: +10.5 m/h² (accelerating away fast)
- Row 28: +9.81 m/h² (accelerating away)

**Why it matters**:
- Is the fire speeding up or slowing down?
- **Positive**: Fire accelerating away (good trend)
- **Negative**: Fire accelerating toward zones (bad trend)
- Captures changing behavior over the 5h window
- Useful for trend analysis: Is threat increasing or decreasing?

**Limitation**: Requires multiple perimeters to calculate reliably

---

### 9. dist_fit_r2_0_5h - R² of Distance Trend ⭐⭐

**What it measures**: How well a linear model fits distance vs. time (0 to 1).

**Examples**:
- Row 2: 0.89 (good linear fit - consistent trend)
- Row 26: 0.17 (poor fit - erratic movement)
- Row 28: 0.54 (moderate fit)
- Many rows: 0.0 (single perimeter OR no trend)

**Why it matters**:
- **High R² (>0.7)**: Fire moving consistently (predictable)
  - Linear extrapolation is reliable
  - Steady approach or retreat
- **Low R² (<0.3)**: Fire moving erratically (unpredictable)
  - Changing direction or speed
  - Linear extrapolation unreliable
  - More dangerous (unpredictable behavior)
- Data quality indicator: How reliable are the distance trend features?

---

## Key Patterns & Insights

### Pattern 1: Close + Approaching = Immediate Threat

**Row 20 (event_id 18292206)**:
- dist_min_ci_0_5h: 1,135 m (very close!)
- dist_change_ci_0_5h: 0.0 (no change, but already close)
- **Result**: Hit in 5.8 hours (event=1)

**Row 28 (event_id 20620516)**:
- dist_min_ci_0_5h: 1,784 m (close)
- dist_change_ci_0_5h: +212.9 m (moving away?!)
- But dist_std_ci_0_5h: 100.4 (high variability)
- **Result**: Hit in 1.4 hours! (event=1)
- **Insight**: Fire perimeter can reach zones even if centroid moves away

### Pattern 2: Far Away = Low Threat

**Row 21 (event_id 18475932)**:
- dist_min_ci_0_5h: 495,662 m (496 km!)
- **Result**: Censored (event=0) - too far to threaten

**Row 25 (event_id 19552816)**:
- dist_min_ci_0_5h: 754,124 m (754 km!)
- **Result**: Censored (event=0)

### Pattern 3: Single Perimeter Limitations

Many fires have distance trend features = 0.0:
- Can't calculate change, slope, acceleration
- Only `dist_min_ci_0_5h` and `dist_std_ci_0_5h` are available
- Models must rely on initial distance alone

---

## Feature Redundancy Analysis

Several features measure similar things:

### Distance Change (3 versions)

- **dist_change_ci_0_5h**: d5 - d0 (negative = closing)
- **projected_advance_m**: d0 - d5 (positive = closing)
- **These are exact opposites (redundant)**

### Closing Speed (2 versions)

- **closing_speed_m_per_h**: Signed speed (positive = closing)
- **closing_speed_abs_m_per_h**: Absolute speed
- Related but provide different information

### Slope vs. Closing Speed

- **dist_slope_ci_0_5h**: Rate of distance change
- **closing_speed_m_per_h**: Essentially `-dist_slope_ci_0_5h`
- **Redundant with opposite signs**

---

## Feature Importance Predictions

### Likely MOST Important ⭐⭐⭐

**1. dist_min_ci_0_5h** - Initial distance
- THE strongest predictor
- Available for all fires
- Direct relationship to 5km threshold

**2. closing_speed_m_per_h** (or dist_slope_ci_0_5h) - Rate of approach
- Combines distance with velocity
- Enables time-to-threat calculations
- Highly predictive when available

### Likely MODERATELY Important ⭐⭐

**3. dist_accel_m_per_h2** - Acceleration
- Captures changing behavior
- Trend information

**4. dist_fit_r2_0_5h** - Trend reliability
- Data quality indicator
- Predictability measure

**5. dist_std_ci_0_5h** - Zone distribution
- Spatial context
- Interaction with directional features

### Likely LESS Important (Redundant) ⭐

**6. dist_change_ci_0_5h** - Redundant with closing_speed

**7. projected_advance_m** - Redundant with dist_change

**8. closing_speed_abs_m_per_h** - Derived from closing_speed

---

## Modeling Considerations

### For Feature Engineering

**Time to 5km threshold**:
- `(dist_min_ci_0_5h - 5000) / closing_speed_m_per_h`

**Distance-to-speed ratio**:
- `dist_min_ci_0_5h / closing_speed_abs_m_per_h`

**Threat score**:
- Combine distance, speed, and acceleration

**Drop redundant features**:
- Keep one version of each measurement

### For Handling Missing Trend Data

When `num_perimeters_0_5h = 1`:
- Only `dist_min_ci_0_5h` and `dist_std_ci_0_5h` available
- Trend features are 0.0 (uninformative)
- Models must rely on initial distance + other features

### For Model Interpretation

- **Distance dominates** - fires >100km away rarely hit
- **Closing speed matters** when distance is moderate (5-50km)
- **Acceleration** indicates changing fire behavior
- **Low R²** suggests unpredictable fire (higher risk)

### Expected Correlations

- Distance features highly correlated with each other
- Distance correlates with event outcome (closer = more hits)
- Closing speed correlates with centroid speed
- Distance features correlate with `num_perimeters_0_5h` (data availability)

---

## Real-World Interpretation

### Critical Scenarios

1. **Close + Fast Closing**: Immediate evacuation needed
2. **Close + Stable**: Monitor closely, prepare evacuation
3. **Moderate Distance + Fast Closing**: Prepare resources
4. **Far + Any Speed**: Low priority (unless extreme speed)

### The Row 28 Paradox

Fire moved AWAY from zones but still hit quickly. **Why?**

- Centroid moved away but perimeter expanded toward zones
- Fire can be asymmetric - growing more on one side
- Highlights importance of radial growth + alignment features
- **Distance to centroid ≠ distance to nearest fire edge**

---

## Summary

The distance features are **the heart of the prediction problem** - they directly measure proximity to the threat threshold and rate of approach. Combined with growth and movement features, they enable time-to-threat calculations essential for emergency response!
