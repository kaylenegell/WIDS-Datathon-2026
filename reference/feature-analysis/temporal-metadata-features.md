
---

*Add your detailed notes from Evernote here*

# Temporal Metadata Features (3 features)

These capture WHEN the fire started, which can be surprisingly important due to environmental and operational factors.

## The 3 Temporal Metadata Features

### 1. event_start_hour - Hour of Day (0-23) ⭐⭐

**What it measures**: Hour when fire was first detected (24-hour format).

**Examples from data**:
- Row 2: 19 (7 PM)
- Row 3: 4 (4 AM)
- Row 4: 22 (10 PM)
- Row 8: 21 (9 PM)
- Row 20: 22 (10 PM)
- Row 28: 19 (7 PM)

**Distribution patterns to explore**:
- **Daytime fires** (6 AM - 6 PM): Hours 6-18
- **Nighttime fires** (6 PM - 6 AM): Hours 18-23, 0-5
- **Peak fire hours**: Typically afternoon (2-5 PM) when temperatures highest
- **Low fire hours**: Early morning (2-6 AM) when temperatures lowest

**Why it matters**:

**1. Environmental Conditions**:
- **Afternoon (14-18)**: Hottest, driest, windiest → fires spread faster
- **Evening (18-22)**: Cooling down, humidity rising → fires may slow
- **Night (22-6)**: Coolest, highest humidity → slowest spread
- **Morning (6-14)**: Warming up, drying out → fires accelerate

**2. Detection Timing**:
- **Daytime**: Fires detected earlier (visibility, activity)
- **Nighttime**: Fires may burn undetected longer before discovery
- Late detection = larger initial size when first observed

**3. Response Capacity**:
- **Daytime**: Full crews available, air support possible
- **Nighttime**: Limited crews, no air support → fires may grow more
- **Evening**: Shift changes, resource transitions

**4. Human Activity**:
- **Afternoon/Evening**: More human-caused ignitions (activities, accidents)
- **Night**: Fewer ignitions but also fewer people to detect/report

**Expected patterns**:
- Fires starting in afternoon (14-18) might spread faster initially
- Fires starting at night might be larger when detected
- Hour might interact with month (seasonal daylight patterns)

---

### 2. event_start_dayofweek - Day of Week (0-6) ⭐

**What it measures**: Day of week when fire started.
- 0 = Monday
- 1 = Tuesday
- 2 = Wednesday
- 3 = Thursday
- 4 = Friday
- 5 = Saturday
- 6 = Sunday

**Examples from data**:
- Row 2: 4 (Friday)
- Row 3: 4 (Friday)
- Row 4: 4 (Friday)
- Row 8: 1 (Tuesday)
- Row 20: 6 (Sunday)
- Row 28: 2 (Wednesday)

**Why it matters**:

**1. Human Activity Patterns**:
- **Weekends (5-6)**: More recreational activity → more human-caused fires
- **Weekdays (0-4)**: More work-related activity, different ignition patterns
- **Friday-Sunday**: Peak outdoor recreation (camping, BBQs, fireworks)

**2. Resource Availability**:
- **Weekdays**: Full staffing, all resources available
- **Weekends**: Potentially reduced staffing, especially holidays
- **Monday**: Fresh crews after weekend
- **Friday**: Crews may be fatigued from week

**3. Response Timing**:
- **Weekend fires**: May get slower initial response
- **Weekday fires**: Faster mobilization of resources

**Expected patterns**:
- Weekend fires might have different characteristics
- Day of week likely less important than hour or month
- Might show regional patterns (e.g., weekend recreation areas)

---

### 3. event_start_month - Month (1-12) ⭐⭐⭐

**What it measures**: Month when fire started.
- 1 = January, 2 = February, ..., 12 = December

**Examples from data**:
- Row 2: 5 (May)
- Row 3: 6 (June)
- Row 4: 8 (August)
- Row 8: 3 (March)
- Row 20: 8 (August)
- Row 28: 7 (July)

**Distribution to explore**:
- Fire season varies by region:
  - **Western US**: June-October (dry season)
  - **Southern US**: March-May, October-November
  - Different regions: Different peak months

**Why it matters**:

**1. Seasonal Weather Patterns**:
- **Summer (6-8)**: Hot, dry, high fire danger → faster spread
- **Spring (3-5)**: Variable conditions, green-up or dry-out
- **Fall (9-11)**: Cooling, but vegetation dry from summer
- **Winter (12-2)**: Generally lower fire danger (but regional variation)

**2. Vegetation State**:
- **Late summer**: Driest vegetation, maximum fuel load
- **Spring**: New growth (less flammable) or dead winter vegetation
- **Fall**: Cured grasses, fallen leaves
- **Winter**: Dormant vegetation, often wetter

**3. Wind Patterns**:
- **Fall**: Strong Santa Ana winds (California), Chinook winds
- **Summer**: Afternoon thunderstorms, dry lightning
- **Spring**: Variable wind patterns
- **Winter**: Storm systems

**4. Daylight Hours**:
- **Summer**: Long days (more detection time, more activity)
- **Winter**: Short days (less detection time)
- Affects both ignition patterns and detection

**Expected patterns**:
- **Peak fire months (July-September)** likely have:
  - More fires overall
  - Faster-spreading fires
  - More threatening fires
- **Off-season months (December-February)** likely have:
  - Fewer fires
  - Slower-spreading fires
  - Different fire characteristics

---

## Circular Encoding Considerations

### Problem with Raw Values

Both `event_start_hour` and `event_start_month` are circular:
- Hour 23 and Hour 0 are adjacent (11 PM and midnight)
- Month 12 and Month 1 are adjacent (December and January)
- But numerically they're far apart (23 vs 0, 12 vs 1)

### Solution: Circular Encoding

For better ML performance, consider creating:

**For Hour**:
```python
hour_sin = sin(2π × hour / 24)
hour_cos = cos(2π × hour / 24)
```

**For Month**:
```python
month_sin = sin(2π × month / 12)
month_cos = cos(2π × month / 12)
```

This preserves circular relationships:
- Midnight (0) and 11 PM (23) have similar sin/cos values
- December (12) and January (1) have similar sin/cos values

---

## Feature Importance Predictions

### Likely MOST Important ⭐⭐⭐

**1. event_start_month** - Seasonal patterns
- Strong correlation with weather, vegetation, fire behavior
- Peak fire season vs. off-season
- Likely moderately to highly predictive

### Likely MODERATELY Important ⭐⭐

**2. event_start_hour** - Diurnal patterns
- Affects initial fire behavior
- Detection timing
- Likely moderately predictive

### Likely LEAST Important ⭐

**3. event_start_dayofweek** - Weekly patterns
- Weakest signal of the three
- Some human activity patterns
- Likely weakly predictive

---

## Key Patterns to Explore in EDA

### Hour Analysis

- **Distribution**: Are fires evenly distributed or clustered?
- **Hit rate by hour**: Do afternoon fires hit more often?
- **Interaction with month**: Summer afternoon vs. winter afternoon

### Day of Week Analysis

- **Distribution**: Weekend vs. weekday fires
- **Hit rate by day**: Any day-of-week effects?
- Likely minimal effect unless strong regional patterns

### Month Analysis

- **Distribution**: Peak fire season identification
- **Hit rate by month**: Summer fires more threatening?
- **Interaction with distance**: Close fires in peak season = highest threat

---

## Modeling Considerations

### For Feature Engineering

**Circular encoding**:
- Create sin/cos versions of hour and month

**Season categories**:
- Peak season (June-September)
- Shoulder season (April-May, October-November)
- Off-season (December-March)

**Time of day categories**:
- Night (22-6)
- Morning (6-12)
- Afternoon (12-18)
- Evening (18-22)

**Weekend flag**:
- `is_weekend = (dayofweek >= 5)`

### For Interactions

- **Month × Hour**: Seasonal diurnal patterns
- **Month × Distance**: Seasonal threat assessment
- **Hour × Growth rate**: Time-of-day fire behavior

### For Model Interpretation

- Month likely most important temporal feature
- Hour provides additional context
- Day of week probably least important
- All three are available for every fire (no missing data)

### Expected Correlations

- Month correlates with fire characteristics (seasonal patterns)
- Hour might correlate with initial fire size (detection timing)
- Day of week likely independent of other features

---

## Real-World Context

### Operational Implications

**1. Peak Season + Afternoon + Weekend**:
- Highest fire danger conditions
- Maximum resource demand
- Potentially slower initial response

**2. Off-Season + Night + Weekday**:
- Lower fire danger
- Fewer competing incidents
- Faster resource availability

**3. Shoulder Season + Variable**:
- Unpredictable conditions
- Transition periods
- Mixed fire behavior

### Regional Variations

- **California**: Peak July-October, Santa Ana winds in fall
- **Southwest**: Peak May-June (pre-monsoon), September-October
- **Southeast**: Peak March-May, October-November
- **Pacific Northwest**: Peak July-September

---

## Summary

The temporal metadata features provide **contextual information about environmental conditions and operational factors** at fire start. While not as directly predictive as distance or growth features, they capture important seasonal and diurnal patterns that affect fire behavior and threat level!
