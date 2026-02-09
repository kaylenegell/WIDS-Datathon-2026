# Growth Features (10 features)

Fire expansion metrics during the first 5 hours - **likely the most important category**.

## Features List

1. **area_first_ha** - Initial fire size (hectares) - Available for ALL fires
2. **area_growth_abs_0_5h** - Absolute area growth (hectares)
3. **area_growth_rel_0_5h** - Relative growth (proportion) ⭐
4. **area_growth_rate_ha_per_h** - Growth rate (ha/hour) ⭐
5. **log1p_area_first** - Log-transformed initial area
6. **log1p_growth** - Log-transformed growth
7. **log_area_ratio_0_5h** - Log ratio of final/initial area
8. **relative_growth_0_5h** - Relative growth (appears to be duplicate of #3)
9. **radial_growth_m** - Radial expansion (meters)
10. **radial_growth_rate_m_per_h** - Radial growth rate (m/hour) ⭐

## Key Insights

- **Most important category** for predicting fire behavior
- Many features = 0 for single-perimeter fires (can't calculate growth from 1 observation)
- Growth **rates** more predictive than absolute values
- **Relative growth** matters more than absolute for small fires
- Initial size available for all fires (no missing data)

## Important Patterns

- Fast growth + close distance = extreme danger
- Large initial size ≠ high threat (distance matters more)
- Radial growth connects to distance features (spatial reasoning)

## Expected Feature Importance

**Highest**: 
- area_growth_rate_ha_per_h
- area_growth_rel_0_5h  
- radial_growth_rate_m_per_h
- area_first_ha

**Moderate**: 
- area_growth_abs_0_5h
- radial_growth_m

**Lower** (redundant): 
- Log-transformed versions
- relative_growth_0_5h (duplicate)

---

*Add your detailed notes from Evernote here*