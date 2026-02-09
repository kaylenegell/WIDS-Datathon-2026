# Centroid Kinematics Features (5 features)

Fire movement patterns - where and how fast the fire is moving.

## Features List

1. **centroid_displacement_m** - Total distance fire center moved (meters)
2. **centroid_speed_m_per_h** - Speed of fire movement (m/hour) ⭐
3. **spread_bearing_deg** - Direction of movement (0-360 degrees)
4. **spread_bearing_sin** - Sine of bearing (circular encoding)
5. **spread_bearing_cos** - Cosine of bearing (circular encoding)

## Key Insights

- **Speed matters more than displacement** for predictions
- **Direction only matters relative to evacuation zones** (see directionality features)
- Circular encoding (sin/cos) is ML-friendly, raw degrees are problematic
- Many fires have 0 movement (single perimeter)
- Fire can grow without moving, or move without growing

## Circular Encoding

**Why needed**: 359° and 1° are only 2° apart, but numerically 358 units apart

**Solution**: Convert to (sin, cos) coordinates
- Preserves circular relationships
- ML models can learn directional patterns
- North (0°) and North (360°) have same sin/cos values

## Important Patterns

- **Fast-moving fires + toward zones = high threat**
- **Stationary fires** may burn in place
- Movement indicates wind-driven or terrain-influenced spread
- Speed combined with direction gives trajectory

## Expected Feature Importance

**Highest**: centroid_speed_m_per_h

**Moderate**: 
- centroid_displacement_m
- spread_bearing_sin/cos (in interactions with zone locations)

**Lower**: spread_bearing_deg (raw degrees problematic for ML)

---

*Add your detailed notes from Evernote here*