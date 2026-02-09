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