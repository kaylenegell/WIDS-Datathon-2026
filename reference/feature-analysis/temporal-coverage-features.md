# Temporal Coverage Features (3 features)

These features describe the **quality and frequency of perimeter observations** during the first 5 hours.

## Features

### 1. num_perimeters_0_5h
**Description**: Number of perimeter observations in first 5 hours

**Range**: 1 to 12+

**Why it matters**:
- More perimeters = better data quality for calculating rates
- 1 perimeter = snapshot only, can't calculate growth/movement
- May indicate fire priority (more monitoring = higher threat)

---

### 2. dt_first_last_0_5h
**Description**: Time span between first and last perimeter (hours)

**Range**: 0 to ~5 hours

**Why it matters**:
- Longer span = better temporal coverage
- 0 hours = only one perimeter observation
- Indicates observation density

---

### 3. low_temporal_resolution_0_5h
**Description**: Binary flag for poor temporal resolution

**Values**: 
- 1 = Poor resolution (dt < 0.5h OR only 1 perimeter)
- 0 = Good resolution

**Why it matters**:
- Flags unreliable growth/movement calculations
- When = 1, most growth/movement features will be 0
- Models should handle these fires differently

---

## Key Patterns

- **Many fires have only 1 perimeter** → All growth/movement features = 0
- **Well-monitored fires** (high num_perimeters) may be higher priority
- **Data quality indicator** for other features

## Modeling Considerations

- Use as interaction terms with growth/movement features
- Consider separate models for low vs. high resolution fires
- Feature importance likely moderate (data quality signal)

---

*Add your detailed notes from Evernote here*

These are metadata about the quality and frequency of perimeter observations during the first 5 hours, which is crucial for understanding data reliability.
1. num_perimeters_0_5h- Number of Perimeter Observations
   
What it measures: How many times the fire perimeter was mapped during the first 5 hours.
Range: 1 to 17 perimeters (160 are 1, direct correlation to dt_first_last_0_5h)
Why it matters:
- More perimeters = better data quality for calculating growth rates and movement
- 1 perimeter = snapshot only - can't calculate rates of change
- Fires with frequent monitoring might be higher priority/threat fires


2. dt_first_last_0_5h - Time Span Between First and Last Perimeter

What it measures: Hours between the first and last perimeter observation (0 to ~5 hours).
Range: 0 to 4.99 (160 are 0!)
Why it matters:
- Longer span = better temporal coverage for trend analysis
- 0.0 hours means only a single observation (can't track changes)
- Combined with num_perimeters, tells you observation density


3. low_temporal_resolution_0_5h - Data Quality Flag

What it measures: Binary flag indicating poor temporal resolution.
Definition: 1 if either:
- Time span < 0.5 hours, OR
- Only 1 perimeter observation (161 data points)
Otherwise: 0 (60 data points)
Why it matters:
- Flags unreliable growth/movement calculations
- When low_temporal_resolution_0_5h = 1, all growth and movement features are likely 0.0 or unreliable

My observations before reading Bob's (Keely's):
When num_perims = 1, dt_first_last = 0, and low_temp = 1 (and same goes for the other 2, this combo is always present when any of the 3 are those values.) So there are only 60 data points that have more than 1 num_perims (non zero dt_first and low_temp = 0.)

Bob's observations:

Key Patterns to Notice

- num_perimeters_0_5h = 1
- dt_first_last_0_5h = 0.0
- low_temporal_resolution_0_5h = 1
- ALL growth features = 0.0 (can't calculate growth from 1 observation!)
- ALL movement features = 0.0 (can't track movement from 1 point!)


Feature Importance Implications


Why These Features Matter:

1. Data Quality Indicator
	- low_temporal_resolution_0_5h = 1 means limited information
	- Models need to handle fires with poor vs. good monitoring differently
2. Proxy for Fire Priority
	- Fires monitored more frequently (num_perimeters_0_5h high) might be:
		- Closer to populated areas
		- Larger or more threatening
		- Higher priority for emergency response
	- This could correlate with higher hit probability!
3. Feature Reliability Signal
	- When low_temporal_resolution_0_5h = 1:
		- Growth features are unreliable (often 0.0)
		- Movement features are unreliable (often 0.0)
		- Distance features might still be valid (based on single observation)
	- Models should learn to weight features differently based on this flag
4. Interaction Effects
	- A fire with low_temporal_resolution_0_5h = 1 but close distance (dist_min_ci_0_5h small) might still be high risk
	- A fire with good monitoring (low_temporal_resolution_0_5h = 0) and high growth rates is likely very dangerous 


Modeling Considerations


For Feature Engineering:

- Create interaction terms: low_temporal_resolution_0_5h × growth_features
- Consider separate models or feature sets for low vs. high resolution fires
- Use num_perimeters_0_5h as a weight or confidence score

For Model Interpretation:

- Don't over-rely on growth/movement features when low_temporal_resolution_0_5h = 1
- Distance features become MORE important for single-perimeter fires
- Initial fire size (area_first_ha) is reliable regardless of temporal resolution

Expected Feature Importance:

- low_temporal_resolution_0_5h: Likely moderately important as a data quality flag
- num_perimeters_0_5h: Could be important as a proxy for fire priority/threat level
- dt_first_last_0_5h: Likely less important directly, but useful for understanding data quality

These temporal coverage features are essentially metadata about data quality, which is crucial for a small dataset where every observation matters!
