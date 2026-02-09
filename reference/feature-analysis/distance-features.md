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