# WiDS Datathon 2026 - EDA Presentation Script (15 minutes)

**Presenter:** [Your Name]  
**Duration:** ~15 minutes  
**Sections:** Automated Data Profiling, Understanding Features, Competition Evaluation Metrics

---

## INTRODUCTION (1 minute)

"Good evening everyone! Tonight I'll be walking you through the first part of our exploratory data analysis for the WiDS Datathon 2026. We'll cover three key areas: our automated data profiling approach, the features in our dataset, and the competition's evaluation metrics. My colleague will then continue with the deeper analysis.

Let's dive in!"

---

## SECTION 1: AUTOMATED DATA PROFILING (3 minutes)

### Opening

"First, let's talk about how we approached understanding this dataset. With 221 training samples and 34 features, we needed a systematic way to get a comprehensive overview quickly."

### The Tool

"We used **ydata-profiling** to generate three interactive HTML reports:
- A complete profile of our training data
- A profile of the test data  
- A comparison report between train and test sets

These reports are available in the repository as `train_data_profile.html`, `test_data_profile.html`, and `train_test_comparison.html`."

### What It Reveals

"The automated profiling immediately highlighted several critical insights:

**First**, we have a **small dataset** - only 221 training samples with 69 hits and 152 censored fires. This means feature engineering will be more important than model complexity.

**Second**, we discovered that **72% of fires have only a single perimeter observation**. This is huge - it means most of our growth and movement features are zero because you can't calculate rates of change from a single point.

**Third**, the profiling revealed **perfect separation at 5 kilometers** - ALL fires that hit evacuation zones were within 5km at the start, while ALL censored fires were beyond 5km. This makes distance features absolutely critical.

The train-test comparison also showed us that the distributions are generally similar, which is good news for model generalization."

### Transition

"Now that we understand the data quality, let's look at what these features actually represent."

---

## SECTION 2: UNDERSTANDING THE FEATURES (6 minutes)

### Opening

"Our dataset contains **34 features** organized into 6 logical categories. Each category captures a different aspect of wildfire behavior. Let me walk you through them."

### Category 1: Temporal Coverage (45 seconds)

"**Temporal Coverage** - these are data quality indicators:
- `num_perimeters_0_5h`: How many times the fire was observed (1-17 times)
- `dt_first_last_0_5h`: Time span of observations (0-5 hours)  
- `low_temporal_resolution_0_5h`: A flag indicating poor data quality -> 1 if dt < 0.5h or only 1 perimeter, else 0

Remember that 72% statistic? Most fires have `num_perimeters` equal to 1, which severely limits what we can calculate."

### Category 2: Growth Features (1 minute)

"**Growth Features** - measuring fire expansion:
- Initial fire size in hectares
- Absolute and relative area growth
- Growth rates in hectares per hour
- Radial growth in meters
- Plus log-transformed versions for better model performance

These features tell us **how aggressively the fire is growing**. Fast-growing fires are obviously more dangerous. But remember - for 72% of fires, these are all zero because we only have one observation."

### Category 3: Centroid Kinematics (1 minute)

"**Centroid Kinematics** - tracking fire movement:
- How far the fire's center moved
- Speed of movement in meters per hour
- Direction of movement in degrees
- Sine and cosine encodings of direction for ML models

Here's a **critical insight**: the fire centroid can move AWAY from evacuation zones while the fire perimeter still reaches them! This happens with asymmetric fire growth. So these features aren't perfect predictors on their own."

### Category 4: Distance Features (1.5 minutes)

"**Distance Features** - this is the heart of our prediction problem:
- `dist_min_ci_0_5h`: **THE most critical feature** - minimum distance to nearest zone
- Standard deviation of distances to different zones
- Change in distance over time
- Rate of distance change
- Closing speed
- Acceleration in distance change
- How predictable the distance trend is

Remember that perfect separation? **ALL hits are ≤5km, ALL censored fires are >5km**. This makes distance features the strongest predictors in our dataset."

### Category 5: Directionality Features (1 minute)

"**Directionality Features** - vector analysis:
- `alignment_cos`: Is the fire moving TOWARD or AWAY from zones?
  - +1.0 means directly toward zones - very dangerous
  - -1.0 means directly away - safer
- Speed components toward/away from zones
- Sideways drift

But again, remember the centroid paradox - these aren't perfect because the perimeter can reach zones even when the centroid moves away."

### Category 6: Temporal Metadata (45 seconds)

"**Temporal Metadata** - when the fire started:
- Hour of day (0-23)
- Day of week  
- Month (1-12)

Peak fire season is June-September, and afternoon fires (14-18h) tend to spread faster due to heat and wind conditions. These are weaker signals but still useful."

### The Timeline (30 seconds)

"One crucial point about timing: ALL features are measured during the **first 5 hours** after fire detection. Then we predict what happens in the NEXT 12, 24, 48, and 72 hours. So we're using early fire behavior to predict future threat."

### Transition

"Now that we understand what we're measuring, let's talk about how the competition will evaluate our predictions."

---

## SECTION 3: COMPETITION EVALUATION METRICS (4.5 minutes)

### Opening

"The WiDS Datathon uses a **hybrid scoring system** that balances two different aspects of model performance. This is important because it's not just about predicting yes/no - we need accurate probabilities at multiple time horizons."

### The Formula (30 seconds)

"The final score is:
```
Final Score = 0.3 × C-index + 0.7 × (1 - Weighted Brier Score)
```

Higher is better, range is 0 to 1. Notice that the Brier Score is **heavily weighted at 70%** - this tells us that probability calibration is more important than just ranking fires correctly."

### C-index: Ranking Ability (1.5 minutes)

"**C-index measures ranking ability** - can your model correctly order fires by threat level?

It evaluates **comparable pairs**:

1. **Event vs Event**: If both fires hit, did you predict higher probability for the fire that hit EARLIER?
   - Fire A hits at 10 hours, Fire B at 30 hours
   - You're correct if P(A) > P(B)

2. **Event vs Censored**: Did you predict higher probability for the fire that actually hit?
   - Fire A hits, Fire B never hits
   - You're correct if P(A) > P(B)

3. **Censored vs Censored**: NOT comparable - we don't know which would have hit first, so these pairs are excluded.

In our training data, we have **12,834 comparable pairs** out of 24,310 total pairs.

**Interpretation**:
- 0.5 = random guessing
- 0.7 = decent discrimination  
- 0.8 = good
- 0.9 = excellent
- 1.0 = perfect

The C-index ensures our model can identify which fires are MORE threatening, which is critical for resource allocation."

### Weighted Brier Score: Calibration Quality (2 minutes)

"**Weighted Brier Score measures probability accuracy** - are your predicted probabilities actually meaningful?

For each fire and time horizon:
```
Brier Score = (predicted_probability - actual_outcome)²
```

We evaluate at **3 time horizons** (not 4!):
- **24 hours** (30% weight)
- **48 hours** (40% weight) ⭐ **Highest weighted!**
- **72 hours** (30% weight)

Why is 48h weighted highest? Because **24-48 hours is the strongest operational value zone** - it balances actionable lead time with decision urgency.

**Critical insight about evaluation coverage**:
- At 24h: 196 fires evaluated (88.7%)
- At 48h: 166 fires evaluated (75.1%)  
- At 72h: **Only 69 fires evaluated (31.2%)** - ALL hits, NO censored fires!

This means the 72h horizon only measures temporal ordering among fires that hit, not discrimination between hits and non-hits. This is why the earlier horizons are more important.

**Interpretation**:
- 0.0 = perfect calibration
- 0.25 = random predictions
- 1.0 = worst possible

**Well-calibrated** means if your model says 30% probability, then about 30% of those fires actually hit. This is crucial for evacuation decisions."

### Monotonicity Constraint (30 seconds)

"One more requirement: **Monotonicity**

Your predictions MUST satisfy:
```
P(12h) ≤ P(24h) ≤ P(48h) ≤ P(72h)
```

Why? Because longer time windows mean more opportunity for fire to reach zones. Probabilities must be non-decreasing. Violating this results in automatic penalties."

### Key Takeaways (30 seconds)

"**Bottom line for modeling**:

1. **Balance both metrics** - you need good ranking AND good calibration
2. **Focus on the 70%** - probability calibration is more heavily weighted
3. **48-hour horizon is king** - it gets the highest weight
4. **Distance features dominate** - remember that 5km perfect separation
5. **Calibration layer likely needed** - most models need post-processing to get probabilities right"

---

## CONCLUSION (30 seconds)

"To summarize:
- We used automated profiling to quickly understand our small, challenging dataset
- We have 34 features across 6 categories, with distance features being most critical
- The competition heavily weights probability calibration at operationally relevant time horizons

Now I'll hand it over to [colleague's name] who will dive deeper into the feature relationships and modeling strategies. Thank you!"

---

## TIMING BREAKDOWN

- Introduction: 1 min
- Automated Profiling: 3 min
- Understanding Features: 6 min
- Evaluation Metrics: 4.5 min
- Conclusion: 0.5 min
- **Total: 15 minutes**

---

## TIPS FOR DELIVERY

1. **Have the HTML reports open** in browser tabs to show if time permits
2. **Use the notebook visualizations** - especially the evaluation coverage chart
3. **Emphasize the "72% single perimeter" insight** - it's the biggest data quality issue
4. **Stress the "5km perfect separation"** - it's the most important feature insight
5. **Make the 70% Brier weight clear** - calibration matters more than ranking
6. **Practice the timing** - you can adjust depth based on audience questions

---

## KEY NUMBERS TO REMEMBER

- 221 training samples (69 hits, 152 censored)
- 34 features in 6 categories
- 72% have only 1 perimeter observation
- 5km perfect separation threshold
- 30% C-index, 70% Brier Score weights
- 48h horizon gets 40% of Brier weight
- 12,834 comparable pairs for C-index

Good luck with your presentation!
