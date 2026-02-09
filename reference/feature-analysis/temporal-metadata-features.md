# Temporal Metadata Features (3 features)

Time-based context - when the fire occurred and how long it's been burning.

## Features List

1. **fire_duration_hours** - Time from first to last perimeter observation (hours) ⭐⭐
2. **day_of_year** - Julian day (1-366) when fire started
3. **year** - Calendar year of fire event

## Key Insights

**Fire Duration**:
- Longer duration = more data points = better trajectory understanding
- BUT: Duration is observation window, not total fire lifetime
- Short duration fires may be:
  - Recently started (still growing)
  - Quickly contained (less threat)
  - Under-observed (data quality issue)

**Seasonality (day_of_year)**:
- Fire behavior varies by season
- Summer fires (days 150-250) typically more severe
- Fall fires (days 250-330) can be wind-driven
- Winter/Spring fires (days 1-100, 330-366) less common but possible

**Year**:
- Climate trends over time
- Different fire seasons have different characteristics
- 2020-2024 data in training set
- May capture multi-year climate patterns

## Temporal Patterns

**High Fire Season** (Western US):
- June-September (days ~150-270)
- Hot, dry conditions
- Higher fire intensity
- More resources deployed

**Shoulder Seasons**:
- April-May (days ~90-150): Spring fires, vegetation drying
- October-November (days ~270-330): Fall fires, wind events

**Low Season**:
- December-March (days ~330-366, 1-90)
- Cooler, wetter conditions
- Fewer but potentially dangerous fires

## Duration Considerations

**Short Duration (<12 hours)**:
- Limited observation window
- May be early-stage fire (high uncertainty)
- Or quickly contained fire (lower threat)
- Fewer perimeters = less growth/movement data

**Medium Duration (12-48 hours)**:
- Typical observation window
- Enough data for trend analysis
- Matches prediction horizons (12h, 24h, 48h)

**Long Duration (>48 hours)**:
- Extended fire event
- More perimeters = better trajectory
- May indicate difficult-to-contain fire
- But observation ends before 72h prediction

## Feature Engineering Ideas

**Duration-based**:
- Duration bins (short/medium/long)
- Duration relative to prediction horizon
- Observations per hour (data density)

**Seasonality**:
- Month (1-12) from day_of_year
- Season (Spring/Summer/Fall/Winter)
- Peak fire season indicator (binary)
- Days since/until peak season

**Cyclical Encoding**:
- day_of_year_sin = sin(2π × day_of_year / 365)
- day_of_year_cos = cos(2π × day_of_year / 365)
- Preserves circular nature (Dec 31 → Jan 1)

## Expected Feature Importance

**Highest**:
- fire_duration_hours (data quality + fire stage indicator)

**Moderate**:
- day_of_year (seasonality effects)

**Lower**:
- year (limited range, may not capture trends)

## Interaction Effects

**Duration × Growth Rate**:
- Fast growth over long duration = aggressive fire
- Slow growth over long duration = contained fire

**Season × Distance**:
- Summer fires far away may reach zones faster
- Winter fires close may be less threatening

**Duration × Number of Perimeters**:
- Many perimeters over short duration = rapidly evolving
- Few perimeters over long duration = sparse data

## Data Quality Implications

**Duration as Proxy**:
- Longer duration → more perimeters → better features
- Relates to temporal_coverage_hours
- Both indicate observation quality

**Missing Movement Data**:
- Single perimeter fires have duration = 0
- No growth or kinematics features
- Must rely on distance and static features

---

*Add your detailed notes from Evernote here*