# Directionality Features (4 features)

Direction of fire movement relative to evacuation zones - is the fire heading toward or away?

## Features List

1. **bearing_to_zone_deg** - Direction from fire to zones (0-360 degrees)
2. **bearing_to_zone_sin** - Sine of bearing (circular encoding)
3. **bearing_to_zone_cos** - Cosine of bearing (circular encoding)
4. **alignment_score** - How aligned fire movement is with zone direction (-1 to 1) ⭐⭐⭐

## Key Insights

**Most Critical Feature**: alignment_score
- Combines movement direction with zone direction
- Positive = fire moving toward zones (HIGH THREAT)
- Negative = fire moving away from zones (LOWER THREAT)
- Zero = fire moving perpendicular to zones

**Bearing Features**: Show static direction to zones
- Less useful alone
- Important in combination with fire movement direction
- Circular encoding (sin/cos) needed for ML

## Alignment Score Deep Dive

**Formula**: Dot product of movement vector and zone direction vector
- alignment_score = cos(angle between vectors)
- Range: -1 (opposite) to +1 (same direction)

**Interpretation**:
- **+1.0**: Fire moving directly toward zones (MAXIMUM THREAT)
- **+0.5**: Fire moving at 60° angle toward zones (MODERATE THREAT)
- **0.0**: Fire moving perpendicular (NEUTRAL)
- **-0.5**: Fire moving at 120° angle away (LOWER THREAT)
- **-1.0**: Fire moving directly away from zones (MINIMUM THREAT)

**Why It Matters**:
- Fast fire moving toward zones >> Fast fire moving away
- Slow fire moving toward zones > Fast fire moving perpendicular
- Combines speed (from kinematics) with direction (from bearing)

## Threat Scenarios

**Highest Threat**:
- alignment_score > 0.7 (moving toward)
- High centroid_speed_m_per_h
- Low min_distance_to_zone_m
- **Interpretation**: Fast fire approaching zones

**Moderate Threat**:
- alignment_score 0.3 to 0.7 (angled approach)
- Moderate speed
- Moderate distance
- **Interpretation**: Fire may reach zones if it continues

**Lower Threat**:
- alignment_score < 0 (moving away)
- OR alignment_score near 0 (perpendicular)
- **Interpretation**: Fire trajectory not toward zones

**Ambiguous**:
- alignment_score undefined (fire not moving)
- Need to rely on distance and growth features instead

## Feature Interactions

**Critical Combinations**:
1. **alignment_score × centroid_speed_m_per_h** = Approach velocity
2. **alignment_score × min_distance_to_zone_m** = Directional threat
3. **alignment_score × perimeter_growth_rate_m2_per_h** = Expanding threat

**Example**:
- Fire 5km away, moving 100 m/h, alignment = 0.9
- Effective approach rate = 100 × 0.9 = 90 m/h toward zones
- Time to reach = 5000m / 90 m/h ≈ 55 hours

## Circular Encoding (Bearing)

**Why needed**: Same as kinematics features
- 359° and 1° are close but numerically far apart
- Sin/cos preserves circular relationships

**Usage**:
- bearing_to_zone_sin/cos for ML models
- bearing_to_zone_deg for human interpretation only

## Expected Feature Importance

**Highest**: 
- alignment_score (combines movement + direction)

**Moderate**:
- bearing_to_zone_sin/cos (in interactions)

**Lower**:
- bearing_to_zone_deg (raw degrees problematic)

## Missing Data Handling

**When alignment_score is undefined**:
- Fire has only 1 perimeter (no movement)
- Fire centroid hasn't moved
- Need alternative threat assessment:
  - Distance features (proximity)
  - Growth features (expansion)
  - Temporal features (time of year, duration)

---

*Add your detailed notes from Evernote here*

# Directionality Features (4 features)

The Directionality features are sophisticated - they answer the critical question: **"Is the fire moving TOWARD or AWAY FROM evacuation zones?"** These combine fire movement with spatial relationships to zones.

## The 4 Directionality Features

### 1. alignment_cos - Cosine of Alignment Angle ⭐⭐⭐

**What it measures**: Cosine of the angle between fire movement direction and direction to nearest evacuation zone.

**Value interpretation**:
- **+1.0**: Fire moving DIRECTLY TOWARD zone (0° angle) ⚠️⚠️⚠️
- **+0.5**: Fire moving at 60° angle toward zone (partially aligned)
- **0.0**: Fire moving PERPENDICULAR to zone (90° angle)
- **-0.5**: Fire moving at 120° angle (partially away)
- **-1.0**: Fire moving DIRECTLY AWAY from zone (180° angle) ✓

**Examples from data**:
- Row 2: -0.055 (nearly perpendicular, slight away component)
- Row 3: -0.569 (moving away at ~125° angle)
- Row 19: +0.954 (moving almost directly toward zone!) ⚠️
- Row 20: +0.438 (moving toward zone at ~64° angle)
- Row 26: -0.888 (moving strongly away at ~153° angle)
- Row 28: -0.374 (moving somewhat away at ~112° angle)

**Why it matters**:
- **Positive values = DANGER**: Fire heading toward zones
- **Negative values = SAFER**: Fire heading away from zones
- Magnitude matters: Closer to ±1 = more aligned (stronger directional component)
- Likely highly predictive - fires moving toward zones are much more threatening

---

### 2. alignment_abs - Absolute Alignment ⭐⭐

**What it measures**: Absolute value of alignment_cos (0 to 1).

**Value interpretation**:
- **1.0**: Fire moving DIRECTLY toward OR away (perfectly aligned)
- **0.5**: Fire at 60° or 120° angle (moderate alignment)
- **0.0**: Fire moving PERPENDICULAR (90° angle)

**Examples**:
- Row 2: 0.055 (nearly perpendicular)
- Row 3: 0.569 (moderately aligned)
- Row 19: 0.954 (highly aligned)
- Row 20: 0.438 (moderately aligned)
- Row 26: 0.888 (highly aligned)
- Row 28: 0.374 (somewhat aligned)

**Why it matters**:
- Measures directional certainty regardless of toward/away
- **High values (>0.7)**: Fire has strong directional component
  - If alignment_cos > 0: Heading toward (dangerous)
  - If alignment_cos < 0: Heading away (safer)
- **Low values (<0.3)**: Fire moving sideways/perpendicular
  - Less predictable
  - Could change direction
- Useful for interaction terms: `alignment_abs × alignment_cos`

---

### 3. cross_track_component - Sideways Drift ⭐

**What it measures**: Component of fire velocity perpendicular to the line toward evacuation zone (meters/hour).

**Interpretation**:
- **Large magnitude**: Fire drifting sideways significantly
- **Small magnitude**: Fire moving along the direct line (toward or away)
- **Sign**: Indicates which side of the line (less important than magnitude)

**Examples**:
- Row 2: -1.94 m/h (small sideways drift)
- Row 20: +31.3 m/h (significant sideways drift)
- Row 26: -4.88 m/h (small sideways drift)
- Row 28: +83.6 m/h (large sideways drift!)

**Why it matters**:
- **Large cross-track**: Fire not moving directly toward/away
  - Wind pushing fire sideways
  - Terrain effects
  - Less predictable trajectory
- **Small cross-track**: Fire on direct path
  - More predictable
  - Easier to extrapolate
- Combined with along-track speed, gives full velocity decomposition

**Calculation**: `cross_track = centroid_speed × sin(angle_to_zone)`

---

### 4. along_track_speed - Direct Approach/Retreat Speed ⭐⭐⭐

**What it measures**: Component of fire velocity along the line toward evacuation zone (meters/hour).

**Sign convention**:
- **Positive**: Fire moving TOWARD zone ⚠️
- **Negative**: Fire moving AWAY from zone ✓
- **Zero**: Fire moving perpendicular (all cross-track)

**Examples**:
- Row 2: -0.11 m/h (slowly moving away)
- Row 20: +15.2 m/h (moving toward zone!) ⚠️
- Row 26: -9.42 m/h (moving away from zone)
- Row 28: -33.7 m/h (moving away from zone)

**Why it matters**:
- **THE most direct measure of approach speed**
- Positive values = immediate concern
- Magnitude indicates how fast fire is closing
- Can calculate: `time_to_zone = dist_min_ci_0_5h / along_track_speed` (when positive)
- Likely very predictive - directly measures threat velocity

**Calculation**: `along_track = centroid_speed × cos(angle_to_zone)`

**Relationship to alignment_cos**:
- `along_track_speed = centroid_speed_m_per_h × alignment_cos`
- Combines speed with direction

---

## Mathematical Relationships

### Velocity Decomposition

Fire velocity can be broken into two perpendicular components:

```
centroid_speed² = along_track_speed² + cross_track_component²
```

**Example - Row 20**:
- centroid_speed_m_per_h: 34.8 m/h
- along_track_speed: 15.2 m/h (toward zone)
- cross_track_component: 31.3 m/h (sideways)
- Check: √(15.2² + 31.3²) = √(231 + 980) = √1211 ≈ 34.8 ✓

### Alignment Relationships

```
alignment_cos = along_track_speed / centroid_speed_m_per_h

alignment_abs = |alignment_cos|
```

---

## Key Patterns & Insights

### Pattern 1: Moving Toward + Close = Extreme Danger

**Row 20 (event_id 18292206)**:
- dist_min_ci_0_5h: 1,135 m (close)
- alignment_cos: +0.438 (moving toward)
- along_track_speed: +15.2 m/h (approaching)
- centroid_speed_m_per_h: 34.8 m/h (fast)
- **Result**: Hit in 5.8 hours (event=1)
- Time estimate: 1135m / 15.2m/h ≈ 75 hours (but hit in 5.8h!)
  - Fire expanded faster than centroid moved

**Row 19 (event_id 18150961)**:
- alignment_cos: +0.954 (almost directly toward!)
- dist_min_ci_0_5h: 2,965 m (moderately close)
- **Result**: Hit in 3.5 hours (event=1)

### Pattern 2: Moving Away = Safer (Usually)

**Row 26 (event_id 19771815)**:
- alignment_cos: -0.888 (strongly away)
- along_track_speed: -9.42 m/h (retreating)
- **Result**: Hit in 0.52 hours! (event=1) 🤔
- **Paradox**: Fire centroid moved away but perimeter reached zone
  - Asymmetric fire growth
  - Expanding toward zone on one side while centroid drifts away

**Row 28 (event_id 20620516)**:
- alignment_cos: -0.374 (moving away)
- along_track_speed: -33.7 m/h (retreating fast)
- **Result**: Hit in 1.4 hours (event=1)
- Same paradox: Perimeter expansion overcame centroid retreat

### Pattern 3: Perpendicular Movement

**Row 2 (event_id 10892457)**:
- alignment_cos: -0.055 (nearly perpendicular)
- along_track_speed: -0.11 m/h (barely moving toward/away)
- cross_track_component: -1.94 m/h (mostly sideways)
- **Result**: Censored at 18.9 hours (event=0)

### Pattern 4: No Movement Data

Many fires have directionality features = 0.0:
- Single perimeter observations
- Can't calculate movement direction
- Models must rely on distance alone

---

## The Centroid vs. Perimeter Paradox

**Critical insight from Rows 26 & 28**:

- Fire centroid can move AWAY from zones
- But fire perimeter can still reach zones
- This happens when:
  - Fire grows asymmetrically
  - One side expands toward zone while other side grows more
  - Centroid shifts away from zone, but nearest edge gets closer

**Implications**:
- Directionality features based on centroid movement are imperfect
- Must combine with:
  - **Radial growth**: Fire expanding in all directions
  - **Distance features**: Actual proximity changes
  - **Growth rate**: How fast perimeter is expanding

---

## Feature Importance Predictions

### Likely MOST Important ⭐⭐⭐

**1. along_track_speed** - Direct approach speed
- Most direct measure of threat velocity
- Positive values = immediate danger
- Enables time-to-threat calculations

**2. alignment_cos** - Direction toward/away
- Strong signal for threat assessment
- Positive = toward (dangerous), negative = away (safer)

### Likely MODERATELY Important ⭐⭐

**3. alignment_abs** - Directional certainty
- Measures how aligned fire movement is
- Useful in interactions with alignment_cos

**4. cross_track_component** - Sideways drift
- Indicates trajectory predictability
- Large values = less predictable

### Redundancy Note

- `along_track_speed` and `alignment_cos` are mathematically related
- `alignment_abs` is derived from `alignment_cos`
- Some redundancy, but each provides slightly different perspective

---

## Modeling Considerations

### For Feature Engineering

**Threat velocity**:
- `max(0, along_track_speed)` (only positive values matter)

**Time to zone**:
- `dist_min_ci_0_5h / max(along_track_speed, 0.1)` (when approaching)

**Trajectory stability**:
- `cross_track_component / centroid_speed_m_per_h` (proportion sideways)

**Combined threat**:
- `along_track_speed × (1 / dist_min_ci_0_5h)` (speed weighted by proximity)

### For Handling Missing Data

When `num_perimeters_0_5h = 1`:
- All directionality features = 0.0
- No movement information
- Models must rely on distance and initial conditions

### For Model Interpretation

- **Positive along_track_speed** is strongest danger signal
- **High alignment_abs + positive alignment_cos** = direct threat
- **High cross_track** = unpredictable trajectory
- **Negative along_track doesn't guarantee safety** (see paradox)

### Expected Correlations

- `along_track_speed` correlates with `closing_speed_m_per_h` (both measure approach)
- `alignment_cos` correlates with `along_track_speed` (mathematically related)
- Directionality features correlate with `num_perimeters_0_5h` (data availability)
- Positive `along_track_speed` strongly correlates with event=1 (hits)

---

## Real-World Interpretation

### Fire Behavior Scenarios

**1. Direct Threat** (alignment_cos > 0.7, along_track > 0):
- Fire heading straight toward zones
- Highest priority for evacuation

**2. Oblique Approach** (alignment_cos 0.3-0.7, along_track > 0):
- Fire approaching at angle
- Monitor closely, prepare evacuation

**3. Perpendicular Movement** (alignment_cos ≈ 0):
- Fire moving sideways
- Could change direction
- Unpredictable threat

**4. Retreating** (alignment_cos < -0.5, along_track < 0):
- Fire moving away
- Lower priority BUT watch for asymmetric growth

**5. High Cross-Track** (large cross_track_component):
- Wind-driven lateral spread
- Less predictable
- Could shift direction

---

## Summary

The directionality features provide **vector analysis of fire movement relative to evacuation zones** - essential for understanding not just IF a fire will reach zones, but HOW it's approaching and how predictable that approach is!
