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
- `dist_min_ci_0_5h`
  
**2. Fire Growth**
- Initial Size:
- Growth Rates:
- Transformed:

**3. Fire Movement and Direction**
- Bearing:
- Speed:
- Alignment:
- Closing speed:
**4. Temporal Coverage**
- 

**5. Temporal Context**
- 

## More about Survival Analysis
### Why Use?
1. Handling Censoring
3. Providing Actionable Time Estimates
4. Realistic Modeling of Real-World Processes

### Math behind the concept
1. Survival Function: S(t)
2. Hazard Function: h(t)
3. Relationship between S(t) and h(t)

### Which survival models would best suit this data?

### How to evaluate performance of survival models?
