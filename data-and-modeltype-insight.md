   # WiDS Datathon 2026: Survival Analysis Guide for Wildfire Evacuation Prediction 

## The Challenge
Predicting time to threat for evacuation zones using _survival analysis_. Four probabilities each fire: 12, 24, 48, and 72hr. 

### Why Survival Analysis?
Traditional classification analysis seeks to understand if the fire will hit the evacuation zone or not
_Survival Analysis_ seeks to understand WHEN the fire will hit and what the probability it hits by time t

This provides critical insights:
- Emergency managers need time estimates for evacuation decisions
- Not all fires that eventually hit do so within the 72-hour observation window (unnecesary displacement of individuals and families)
- The timing of impact is as important to responders and those affected as wether or not it happens

## Data Structures
### Target Variables 
1. `time_to_hit_hours`: (survival time)
   - Range: [0, 72] hours
   - For events that hit (event=1): actual time until fire came within 5km of evacuation zone
   - For censored events (event=0): last observed time, fire never hit within 72hr window
   - t0 = initial observation + 5 hours (model makes predicitons at t0 + 5h) <mark>need further clarification here</mark>
3. `event`: (event indicator)
   - 1: fire DID hit evacuation zone within 72 hours (uncensored)
   - 0: fire NEVER hit within 72 hours (right-censored)
   - Ditribiution: ~41% hit (event=1), ~59% censored (event=0)

### What is Censoring?
**Right-censoring** occurs when observation is stopped before the event happens:
- A fire may eventually hit, but not within the 72-hour window
- Know it survived AT LEAST 72 hours, but not the true event time
- Valuable as it is not missing data, but partial information

**Example Timeline:**
```
Event 1 (Uncensored):
t0: 5h  ----[observations]----> t=23h: Fire hits! 
time_to_hit_hours = 18h, event = 1

Event 2 (Censored):
t0: 5h  ----[observations]----> t=72h: Still >5km away
time_to_hit_hours = 72h, event = 0 (censored)
```

### Feature Categories (34 features)

All features based on the first 5 hours of fire behavior (t0 to t0+5h):
<mark>further understanding and context/details to come on all proceeding sections</mark>

**1. Distance Metrics**
- `dist_min_ci_0_5h`: minimum distance to evac zone centroid 
  
**2. Fire Growth**
- `area_first_ha`: initial size
- `area_growth_rate_ha_per_h`, `radial_growth_rate_m_per_h`: growth rates
- `log1p_area_first`, `log1p_growth`: transformed features
- !Note! 88.7% of fires show zero growth in first 5 hours

**3. Fire Movement and Direction**
- `spread_bearing_deg`, `spread_bearing_cos`, `spread_bearing_sin`: bearing 
- `centroid_speed_m_per_h`, `centroid_displacement_m`: speed
- `alignment_abs`, `alignment_cos`: alignment, how directly fire moved toward target
- `closing_speed_m_per_h`: closing speed on rate of approach
**4. Temporal Coverage**
- `num_perimeters_0_5h`: number of fire perimeter observations
- `dt_first_last_0_5h`: time span between first and last measurement
- `low_temporal_resolution_0_5h`: data quality flag (1=poor quality)

**5. Temporal Context**
- `event_start_hour`: hour of the day (0, 23)
- `event_start_dayofweek`: dat of week (0=Monday)
- `event_start_month`: month (1-12)

## More about Survival Analysis
### Why Use?
1. Handling Censoring
- Regular Regression Issues:
   - treats event=0 as a definitely won't hit secanrio
   - ignores that censored events might hit after 72 hr (similar to above issue)
   - loses information about partial survival times
- Survival Analysis Positives:
   - uses both event indicator and time information
   - correctly handles censored observations
   - provides time-dependent predictions
3. Providing Actionable Time Estimates
- Instead of a traditional approach with results like "70% of hitting," you get:
   - immediate evacuation needed: 20% chance of hitting in 6 hours
   - prepare for evacuation: 50% chance by 24 hours
   - monitor closely and stay aware: 70% chance by 48 hours
4. Realistic Modeling of Real-World Processes
- Wildfires do not just hit or miss, they can evolve over time with many things:
   - early vs late hitting fires can havew very diofferent charecterisitics
   - time-varying hazards (ex: when the wind will take effect or ease up)
   - competing risks (ex: will the fire be contained before hitting the evacuation zone)

### Math behind the concept
1. Survival Function: S(t) Probability of surviving beyond time t
- S(t) = P(T > t) = P(fire hasn't hit by time t)
- Properties:
   - S(0) = 1 (all fires start as "not hit")
   - S(infinity) = 0 (eventually all would hit, if given infinite time)
   - Monotonically decreasing (can't turn "un-hit")
   - For censored data: S(72) > 0 (some never observed hitting)
- Interpretation for this problem:
   - S(6) = 0.85 -> 85% of fires haven't hit evacuation zone by 6 hours

2. Hazard Function: h(t) Instantenous rate of event occurence at time t, given survival to t
- h(t) = lim[Δt→0] P(t ≤ T < t+Δt | T ≥ t) / Δt
- Interpretation:
   - h(6) = 0.05 -> 5% per hour risk of hitting at hour 6
   - high hazard = high immediate risk
   - can increase, decrease, or stay constant over time
- cumulative hazard:
   - H(t) = ∫[0 to t] h(u)du = -log(S(t))
3. Relationship between S(t) and h(t)
- S(t) = exp(-H(t)) = exp(-∫[0 to t] h(u)du)
- If hazard is constant (h(t) = λ): exponential survival: S(t) = exp(-λt)
- If hazard increases over time: weibull distribution (accelerating risk)

## Which survival models would best suit this data?

### 1. Cox Proportional Hazards Model (Recommended Starting Point)
When to use:
- understanding feature effects
- semi-parametric approach (no distributional assumptions)
- works well with censored data

**Model**
```
h(t|X) = h₀(t) × exp(β₁X₁ + β₂X₂ + ... + βₚXₚ)
```

Where:
- h₀(t) = baseline hazard (unspecified)
- X = features (distance, direction, etc.)
- β = coefficients (log hazard ratios)

**Interpretation:**
- β > 0: Feature increases hazard (shorter time to hit)
- β < 0: Feature decreases hazard (longer time to hit)
- exp(β) = hazard ratio

**Assumptions**
- Proportional hazards: effect of features is constant over time
- check with schoendeld residuals

### 2. Accelerated Failure Time (AFT) Models
When to use:
- need to specify time distribution
- want to model median survival time directly
- linear relationships with log(time)

**Available Distributions**
- weibull (most flexible - models increasing/decreasing hazard) 
- log-logistic (allows non-monotonic hazard)
- log-normal (symmetric on log scale)
- exponential (constant hazard - simplest)

### 3. Random Survival Forests (advanced)
When to use:
- non-linear relationships
- feature interactions important
- no distributional assumptions needed
- can handle complex patterns

### 4. Gradient Boosting Survival Models

### 5. Deep Learning Survival Models


## How to evaluate performance of survival models?

### 1. Concordance Index (C-Index): PRIMARY METRIC
Probability that model correctly ranks pairs of observations

Interpretation:
- C = 0.5 random predictions (coin toss)
- C = 1.0 perfect predictions
- C > 0.7 good model
- C > 0.8 excellent model

Calculation: 
For all comparable pairs (i, j) where tᵢ < tⱼ:
- Concordant: risk_scoreᵢ > risk_scoreⱼ (higher risk = shorter survival)
- Discordant: risk_scoreᵢ < risk_scoreⱼ
- C-index = concordant / (concordant + discordant)

### 2. Integrated Brier Score (IBS)
Mean squared error of survival probability predictions over time. Lower is better (similar to MSE)

### 3. Time-Dependent AUC
AUC at specific time points


## Possible approach

### 1. Baseline
1. Kaplan-Meier analysis for overall survival pattern
2. Cox model with top 9 features
3. Establish C-index baseline

### 2. Optimization 
4. Try AFT models (weibull, log-normal)
5. Random Survival Forest
6. Feature Engineering (distance ratios, interactions)

### 3. Advanced
7. Ensemble multiple survival models
8. Deep survival learning (python: DeepSurv)
9. Custom loss functions optimized for C-index