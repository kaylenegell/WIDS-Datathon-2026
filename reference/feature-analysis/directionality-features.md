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