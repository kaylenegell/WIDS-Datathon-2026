# Feature Analysis

Detailed analysis of all 34 features organized by category.

*Add your detailed feature notes from Evernote to the individual category files.*

# Summary: All 34 Features Analyzed

## We've now covered all feature categories:
1. Temporal Coverage (3): Data quality indicators
2. Growth (10): Fire expansion metrics
3. Centroid Kinematics (5): Fire movement
4. Distance (9): Proximity to zones
5. Directionality (4): Approach vectors
6. Temporal Metadata (3): Timing context

## Most Predictive Features (Expected):
- dist_min_ci_0_5h(distance)
- along_track_speed(directionality)
- closing_speed_m_per_h(distance)
- area_growth_rate_ha_per_h(growth)
- centroid_speed_m_per_h(kinematics)
- event_start_month(temporal)

## Key Challenges:
- Many features = 0 for single-perimeter fires
- Feature redundancy (multiple versions of same concept)
- Small dataset (221 samples) limits complex modeling
- Centroid-based features don't capture asymmetric growth
