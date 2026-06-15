# DEVELOP_E — Chiron Development Engine Specification v6

> **Document status:** Working specification — not yet approved for build  
> **Supersedes:** Chiron Development Engine Specification v5  
> **Key changes from v5:** Phase transition Method 2 (level threshold) strengthened with a Mastery-modulated minimum duration gate; Mastery tiers defined for use in phase transition logic; hard floor of 4 weeks retained regardless of Mastery to allow physiological re-expression in returning athletes

---

## Executive Summary

The Chiron Development Engine is an adaptive coaching system that models athlete development as a persistent progression tree.

Unlike traditional training platforms that prescribe workouts from a predefined library, the Development Engine prescribes physiological stimuli and dynamically generates workouts to deliver those stimuli.

The system connects to intervals.icu as its primary data source and combines:

1. Athlete data ingestion (intervals.icu)
2. Capability assessment
3. Fatigue modelling
4. Development Tree progression
5. Adaptation modelling
6. Stimulus prescription
7. Dynamic workout generation

The objective is to answer four questions continuously:

**How strong is the athlete?** (Capability)  
**How recovered is the athlete?** (Fatigue)  
**How developed is the athlete?** (Progression)  
**What should the athlete do next?** (Prescription)

---

## Core Philosophy

Traditional coaching platforms focus on current performance metrics:

- FTP
- VO2max
- CTL
- Power Profile

These metrics describe current fitness.

The Development Engine focuses on accumulated adaptation.

The athlete is not simply becoming fitter. The athlete is developing physiological systems over time.

**The three layers operate on fundamentally different timescales.**

This is the central architectural insight.

| Layer | Timescale | Can decrease? |
|-------|-----------|---------------|
| Capability | Days to weeks | Yes, rapidly |
| Athlete Level | Weeks to months | Yes, slowly |
| Mastery | Months to years | Rarely |

Injury, illness, or detraining affects Capability and Athlete Level. It does not affect Mastery.

Two athletes with identical current power profiles may have very different Development Trees.

---

## Data Architecture

### Source: intervals.icu

The system connects to intervals.icu as its primary data source.

**Power curve data**
- Best power at standard durations: 5 s, 30 s, 1 min, 2 min, 5 min, 8 min, 20 min, 60 min, 90 min, 2 h, 3 h
- Updated daily and on each new activity
- Endpoint: `/api/v1/athlete/{id}/power_curves`

**Athlete profile**
- Body weight (required for w/kg calculations)
- FTP (athlete-set or intervals.icu estimate)
- Endpoint: `/api/v1/athlete/{id}`

**Training load**
- ATL, CTL, TSB (intervals.icu native model)
- Endpoint: `/api/v1/athlete/{id}/fitness`

**Activity history**
- Individual workout data: power, HR, duration, TSS, IF
- Interval structure (for stimulus scoring)
- Endpoint: `/api/v1/athlete/{id}/activities`

**Wellness data**
- HRV, sleep duration, resting HR
- Subjective fatigue and mood scores
- Endpoint: `/api/v1/athlete/{id}/wellness`

**Events**
- Target events and race calendar
- Endpoint: `/api/v1/athlete/{id}/events`

### Sync Model

| Data type | Sync trigger |
|-----------|-------------|
| Activity data | On new activity post |
| Power curve | Daily and after each activity |
| Athlete profile | On change or weekly |
| Training load (ATL/CTL/TSB) | Daily |
| Wellness | On athlete entry |
| Events | On change |

### Key Derived Calculations

**w/kg conversion:**
```
wkg(duration) = power_curve[duration] / athlete_weight_kg
```

**FTP (if not athlete-set):**
```
ftp_estimated = power_curve[p1200] * 0.95
```
Athlete-set FTP takes priority over the estimated value at all times.

**W' estimation (Critical Power model):**

The 2-parameter CP model can be fitted to the power curve:
```
P(t) = W_prime / t + CP
```

Where CP approximates FTP. W' is estimated by fitting this model across qualifying intentional maximal efforts.

Research consistently shows CP can be estimated reliably from field data (prediction error ~2.5–5%), but W' is significantly less reliable when derived from unstructured training data (prediction error ~25–33%). W' enrichment is therefore gated behind intentional test data rather than opportunistic best efforts.

**Qualifying efforts for W' estimation:**
- Minimum 3 intentional near-maximal efforts
- Efforts must span meaningfully different durations (recommended: ~3 min, ~5 min, and ~12 min)
- Efforts must be flagged as intentional test efforts, not extracted from general training data
- Standard error of estimate must be below 10% for W' to be considered reliable

*Flag W' calibration as a structured onboarding step. Until qualifying efforts exist, the Anaerobic branch operates in Phase 1 (p60-based scoring) regardless of overall training history.*

---

## System Architecture

```
intervals.icu
    |
    v
Data Ingestion Layer
    |
    +---> Capability Model <---> Fatigue Model
    |              |                    |
    |              v                    |
    +---> Athlete Progression Model <---+
                   |
                   v
           Development Tree
                   |
                   v
          Stimulus Engine <--- Periodisation Context
                   |
                   v
          Workout Generator
                   |
                   v
         Workout Execution
                   |
                   v
        Activity Analysis
                   |
                   v
          Model Update (all layers)
```

All models update after every completed session.

---

## Layer 1: Capability Model

### Purpose

Capability represents current fitness.

Answers: *How strong is this athlete today?*

Capability can increase or decrease.

### Capability Scale

0 to 100, measured independently for each branch.

### Scoring Formula

Capability scores are derived from w/kg values mapped against Coggan normative data.

**Step 1: Pull power curve from intervals.icu**
```
wkg = power_curve[duration] / athlete.weight_kg
```

**Step 2: Map w/kg to a continuous Coggan position score**

The Coggan scale runs from 0 to 7, representing:

| Score range | Category |
|-------------|----------|
| 0.0 to 1.0 | Untrained |
| 1.0 to 2.0 | Fair |
| 2.0 to 3.0 | Moderate |
| 3.0 to 4.0 | Good |
| 4.0 to 5.0 | Very Good |
| 5.0 to 6.0 | Exceptional |
| 6.0 to 7.0 | World Class |

Within each category, interpolate linearly based on the athlete's position within the w/kg range for that category.

**Step 3: Convert to 0-100 capability score**
```
capability_score = (coggan_position_score / 7.0) * 100
```

### Coggan Normative Reference Data

*These values are approximate. They must be verified against the current published Coggan Power Profile norms before build. The system must support configurable normative tables to allow for future recalibration and gender-specific values.*

**Male approximate w/kg norms:**

| Duration | Untrained | Fair | Moderate | Good | Very Good | Exceptional | World Class |
|----------|-----------|------|----------|------|-----------|-------------|-------------|
| 5 s | < 10 | 10-14 | 14-17 | 17-21 | 21-25 | 25-30 | > 30 |
| 1 min | < 5 | 5-6.5 | 6.5-8 | 8-10 | 10-12 | 12-15 | > 15 |
| 5 min | < 3.5 | 3.5-4.3 | 4.3-5.0 | 5.0-5.7 | 5.7-6.4 | 6.4-7.6 | > 7.6 |
| 20 min | < 2.5 | 2.5-3.0 | 3.0-3.7 | 3.7-4.4 | 4.4-5.1 | 5.1-5.8 | > 5.8 |
| 60 min | < 2.0 | 2.0-2.6 | 2.6-3.2 | 3.2-3.9 | 3.9-4.6 | 4.6-5.3 | > 5.3 |

*Female normative data require a separate table. Collect from current Coggan sources prior to build.*

### Capability Branches

**Neuromuscular**
- intervals.icu source: `power_curves.p5`
- Metric: 5-second best power in w/kg

**Anaerobic**

The Anaerobic branch uses a two-phase stimulus model based on data availability and confidence.

*Phase 1 — p60 primary (default for new and developing athletes)*

- intervals.icu source: `power_curves.p60`
- Metric: 1-minute best power in w/kg, scored against Coggan norms
- Active when: Anaerobic branch confidence is below 75%, or fewer than 3 qualifying intentional efforts exist

*Phase 2 — W' enriched (developed athletes with sufficient test data)*

- Active when: Anaerobic branch confidence is at or above 75% AND at least 3 qualifying intentional efforts spanning different durations are available for CP model fitting
- Composite stimulus metric:
```
anaerobic_stimulus =
    (p60_score * 0.60)
  + (W_prime_expenditure_score * 0.40)
```

p60 remains the majority signal even in Phase 2, reflecting that W' carries inherent estimation noise (~25% prediction error from field data) even under favourable conditions.

When an athlete transitions to Phase 2, the UI displays a brief explanation of the model change. This prevents confusion from score shifts and frames the enrichment as a reward for accumulated training history.

**VO2max**
- intervals.icu source: `power_curves.p300`
- Metric: 5-minute best power in w/kg

**Threshold**
- Primary: athlete FTP from intervals.icu
- Fallback: `power_curves.p1200 * 0.95`
- Secondary metric: TTE (time to exhaustion at FTP)

TTE estimation: derived from training data by identifying the longest completed sustained efforts at or above 95% FTP. Requires a minimum of 3 qualifying efforts for a reliable estimate.

**Aerobic Foundation**
- Primary: `power_curves.p3600` in w/kg (60-minute best)
- Secondary metrics:
  - Efficiency Factor (EF): `normalised_power / avg_heart_rate` from long rides
  - Aerobic decoupling: Pa:HR drift from rides over 90 minutes

Composite score:
```
aerobic_capability =
    (ftp_wkg_score * 0.50)           // FTP as aerobic baseline
  + (p3600_wkg_score * 0.30)         // 60-minute endurance power
  + (efficiency_factor_score * 0.20) // Aerobic efficiency
```

*If heart rate data is unavailable, redistribute to FTP (0.65) and p3600 (0.35).*

### Capability Decay

During detraining, capability decays using exponential decay:

```
capability_t = capability_0 * exp(-(ln(2) / half_life) * days_inactive)
```

Branch half-lives:

| Branch | Half-life | Notes |
|--------|-----------|-------|
| Neuromuscular | 35 days | Slower decay |
| Anaerobic | 28 days | |
| VO2max | 21 days | Faster decay |
| Threshold | 28 days | |
| Aerobic Foundation | 42 days | Slowest decay |

Decay activates after 14 consecutive days without branch-relevant training.

---

## Layer 2: Athlete Level (Progression Model)

### Purpose

Athlete Level represents current training capacity.

Answers: *How much stimulus can this athlete successfully absorb?*

Scale: 1.0 to 10.0 per branch.

Example: `Threshold 5.4`

### Level Decimal Definition

The decimal represents progress toward the next integer level:

```
display_level = integer_level + (credits_since_last_integer / credits_to_next_integer)
```

Example: `Threshold 5.4` means the athlete is at Level 5 with 40% of the credits needed to reach Level 6.

From the credit table:
- Level 5 requires 550 total credits
- Level 6 requires 750 total credits
- Gap = 200 credits
- Level 5.4 = 550 + (200 * 0.4) = 630 total credits accumulated

### Credit Table

| Level | Total credits required | Credits gap from previous |
|-------|------------------------|--------------------------|
| 1 | 0 | -- |
| 2 | 100 | 100 |
| 3 | 225 | 125 |
| 4 | 375 | 150 |
| 5 | 550 | 175 |
| 6 | 750 | 200 |
| 7 | 975 | 225 |
| 8 | 1,225 | 250 |
| 9 | 1,500 | 275 |
| 10 | 1,800 | 300 |

### Productive Progression Window

For each branch, there is an optimal range of stimulus intensity relative to the athlete's current level.

The Productive Progression Window (PPW) defines this range and determines which Difficulty Factor applies to the session's Adaptation Credits.

The Stimulus-to-Level Ratio is:
```
ratio = stimulus_level / athlete_level
```

Where `stimulus_level` is the normalised intensity of the session's branch-specific stimulus, expressed on the same 1-10 scale as Athlete Level.

| Classification | Ratio range | Difficulty Factor |
|---------------|-------------|------------------|
| Recovery | < 0.75 | 0.50 |
| Easy | 0.75 to 0.89 | 0.75 |
| Productive | 0.90 to 1.10 | 1.00 |
| Stretch | 1.11 to 1.30 | 1.20 |
| Breakthrough | 1.31 to 1.50 | 1.30 |
| Overreach | > 1.50 | 0.00 |

*Note: The PPW boundaries are initial values. Treat as configurable parameters, to be calibrated against real athlete data after launch. The boundaries may differ by branch -- neuromuscular athletes, for example, may tolerate a wider productive window.*

### Adaptation Credit Formula

```
Credits = Stimulus_Score
        x Difficulty_Factor
        x Completion_Quality
        x Consistency_Multiplier
        x Periodisation_Modifier
```

All factors are described below.

#### Stimulus Score

Measures the physiological dose delivered to each branch in a session.

| Branch | Stimulus metric | Data source |
|--------|----------------|-------------|
| Aerobic Foundation | Steady aerobic minutes (LT1 to LT2) | Power and HR zone analysis |
| Threshold | Threshold Time in Zone (TIZ) | Power zone analysis |
| VO2max | Effective VO2 time (minutes above 108% FTP) | Power zone analysis |
| Anaerobic | Phase 1: 1-minute best power (p60 w/kg). Phase 2: composite of p60 score (60%) and W' expenditure in kJ (40%) | Phase 1: power curve. Phase 2: power curve + CP model |
| Neuromuscular | Sprint count weighted by intensity | Sprint detection |

For each branch, the raw metric is normalised to a 0-10 stimulus level, referenced against level-appropriate training norms for the athlete's current level. This normalised value is the `stimulus_level` used in the PPW ratio calculation above.

#### Difficulty Factor

Derived directly from the PPW classification (see table above).

#### Completion Quality

*Revised from v2. A failed or partial effort is not zero: any genuine attempt delivers partial adaptation stimulus.*

| Outcome | Multiplier | Definition |
|---------|-----------|------------|
| No attempt | 0.00 | Workout not started |
| Abandoned early | 0.25 | Less than 50% of target completed |
| Partial | 0.50 | 50 to 79% of target completed |
| Completed | 1.00 | 80 to 100% of target completed |
| Exceptional | 1.10 | Target exceeded; performance clearly above prescription |

*Exceptional applies only when the athlete genuinely exceeded the prescribed stimulus. Completing a comfortable workout does not qualify. The Confidence Model detects this pattern over time and adjusts future prescription accordingly.*

#### Consistency Multiplier

*Revised from v2. The range is widened to reflect meaningful differences in training adherence.*

```
consistency_multiplier = 0.80 + (adherence_score * 0.40)
```

Range: 0.80 to 1.20.

`adherence_score` (0.0 to 1.0) is derived from completed workouts as a percentage of prescribed workouts over the last 28 days, weighted toward recent weeks:

```
adherence_score =
    (completion_rate_week_1 * 0.40)
  + (completion_rate_week_2 * 0.30)
  + (completion_rate_weeks_3_4 * 0.30)
```

#### Periodisation Modifier

| Phase | Priority branches | Modifier (priority) | Modifier (other) |
|-------|------------------|--------------------|--------------------|
| Base | Aerobic Foundation | 1.20 | 0.85 |
| Build | Threshold, VO2max | 1.20 | 0.85 |
| Peak | Event-specific branches | 1.25 | 0.80 |
| Recovery | All branches | 0.50 | 0.50 |

### Athlete Level Advancement

Athlete Levels advance continuously as credits accumulate.

Display updates after every session.

Integer level changes trigger a notification to the athlete.

### Athlete Level Decay

During extended inactivity, Athlete Levels decay slowly:

```
credits_today = credits_at_inactivity_start * exp(-(ln(2) / 42) * days_inactive)
```

Half-life of 42 days. An athlete returning from a 6-week break retains approximately 66% of their accumulated credits.

This is deliberately slower than Capability decay. The athlete retains most of their level through short breaks.

### Confidence Model

Each Athlete Level per branch carries a confidence score (0% to 100%).

Confidence reflects how certain the system is about the current level estimate.

Low confidence triggers more conservative prescriptions (the Productive Progression Window shifts downward).

**Formula:**
```
confidence =
    (0.40 * recency_score)
  + (0.30 * consistency_score)
  + (0.20 * data_quality_score)
  + (0.10 * volume_score)
```

**Recency score**
```
recency_score = exp(-(ln(2) / 28) * days_since_relevant_activity)
```
Half-life of 28 days. Score = 1.0 trained yesterday; 0.5 if no relevant training for 28 days.

**Consistency score**
```
if mean_completion_rate == 0 or insufficient_data (fewer than 4 sessions in last 28 days):
    consistency_score = 0.50  // neutral fallback; insufficient data to assess consistency
else:
    consistency_score = clamp(1.0 - (std_dev_completion_rate / mean_completion_rate), 0.0, 1.0)
```
Based on the last 28 days. Clamped to 0.0–1.0. Falls back to a neutral 0.50 when the athlete has fewer than 4 sessions in the window or a zero mean completion rate, to prevent division-by-zero and avoid penalising new athletes.

**Data quality score**

| Source | Score |
|--------|-------|
| Field test (FTP test, ramp test) | 1.00 |
| Near-maximal effort extracted from training | 0.75 |
| Sub-maximal estimation | 0.50 |
| Questionnaire or manual entry only | 0.25 |

**Volume score**
```
volume_score = min(1.0, relevant_sessions_last_90_days / 12)
```
Full confidence on volume requires 12 or more qualifying branch sessions in the last 90 days.

**Effect on prescription**

| Confidence | Prescription adjustment |
|-----------|------------------------|
| Greater than 85% | Standard productive window |
| 60% to 85% | Shift window lower by 0.10 x athlete_level |
| Less than 60% | Shift window lower by 0.20 x athlete_level; prioritise data-gathering sessions |

---

## Layer 3: Mastery

### Purpose

Mastery represents long-term accumulated development.

Answers: *How much development has this athlete built over time?*

Mastery powers the Development Tree visualisation.

### Mastery Accumulation

Mastery is accumulated per branch from the lifetime sum of adaptation credits, weighted by a slow long-term decay function:

```
mastery_points_branch = sum of all sessions:
    credits_earned * mastery_accumulation_weight(session_date)
```

Where:
```
mastery_accumulation_weight = exp(-(ln(2) / 5) * years_elapsed)
```

Recent credits carry full weight (weight = 1.0 at years_elapsed = 0). Credits from training completed 5 years ago carry approximately 50% weight. Credits are never fully lost.

*Note: The previous v3 formula `1.0 - exp(-(ln(2) / 5) * years_elapsed)` was inverted, producing zero weight for today's sessions and increasing weight for older ones. This has been corrected.*

### Mastery Decay

Mastery decays at approximately 1% per month of complete branch inactivity:

```
mastery_today = mastery_at_inactivity_start * exp(-(ln(2) / 60) * months_inactive)
```

Half-life of 60 months (5 years). Mastery is practically permanent for any athlete with meaningful training history.

### Display

Mastery is displayed exclusively through the Development Tree.

It is not shown as a numerical score to the athlete.

### Mastery Tiers

Mastery points are grouped into five tiers for use in system logic (currently: phase transition minimum duration). Tiers are per-branch.

| Tier | Label | Mastery points | Meaning |
|------|-------|---------------|---------|
| 0 | None | 0 | No meaningful prior history in this branch |
| 1 | Developing | 1 to 499 | Early accumulated history |
| 2 | Established | 500 to 1,499 | Solid training history in this branch |
| 3 | Experienced | 1,500 to 3,499 | Extensive branch-specific history |
| 4 | Elite | 3,500+ | Long-term high-volume branch development |

*Tier boundaries are initial values and should be calibrated against real athlete population data after launch.*

---

## Fatigue Model

### Purpose

The Fatigue Model answers: *How recovered is this athlete today?*

Fatigue affects prescription intensity and workout selection.

### Global Fatigue (from intervals.icu)

intervals.icu calculates ATL, CTL, and TSB natively. Pull these values directly from the fitness endpoint. Do not recalculate them.

TSB is used as a global readiness modifier:

| TSB range | Readiness state |
|-----------|----------------|
| Greater than 10 | Well rested; higher intensity available |
| -10 to 10 | Normal training state |
| -10 to -30 | Fatigued; reduce intensity |
| Less than -30 | Recovery required; no high-intensity work |

```
global_readiness = sigmoid((TSB + 10) / 20)
```
Returns a 0.0 to 1.0 score. Sigmoid function prevents step-change behaviour at the boundaries.

### System-Specific Fatigue

Global ATL/CTL does not distinguish between branch-specific fatigue. A neuromuscular session affects sprint capacity without meaningfully impacting threshold recovery.

System-specific fatigue (SFatigue) is tracked per branch:

```
SFatigue_branch(today) = sum over last 21 days:
    stimulus_score_branch(day) * exp(-(ln(2) / 7) * (today - day))
```

Where:
- `stimulus_score_branch(day)` is the stimulus delivered to this branch on that day (0-10 scale)
- The decay term applies a 7-day half-life to older sessions

Branch readiness:
```
readiness_branch = 1.0 - min(1.0, SFatigue_branch / SFatigue_max_branch)
```

`SFatigue_max_branch` is a per-branch calibration constant. Set initially from training norms; updated over time per athlete.

### Combined Readiness Score

```
readiness_combined =
    (global_readiness * 0.60)
  + (readiness_target_branch * 0.40)
```

Where `readiness_target_branch` is the system-specific readiness for whichever branch the Stimulus Engine has selected for this session.

If `readiness_combined < 0.50`, the session is automatically classified as a recovery session. No high-intensity prescription is issued.

### HRV Integration

HRV from wellness data will be incorporated as a readiness signal in a future version.

For v1: HRV data is stored but not used in the readiness formula. This is noted as a planned enhancement.

---

## Periodisation Model

### Phases

| Phase | Purpose | Default duration |
|-------|---------|-----------------|
| Base | Build aerobic foundation | 6 to 12 weeks |
| Build | Develop threshold and VO2max | 4 to 8 weeks |
| Peak | Sharpen for target event | 2 to 4 weeks |
| Recovery | Active rest between phases | 1 to 2 weeks |

### Phase Transition Triggers

Transitions are determined by one of three methods, applied in priority order.

**Method 1: Event-based (highest priority)**

When a target event is entered, the system back-calculates the phase schedule from the event date:

```
peak_start = event_date - peak_duration
build_start = peak_start - build_duration
base_start = build_start - base_duration
```

Minimum lead times enforced:
- Peak phase: 2 weeks
- Build phase: 4 weeks
- Full Base + Build + Peak cycle: 12 weeks minimum

If insufficient time exists before the event, compress phases proportionally rather than skipping them.

**Method 2: Level threshold with Mastery-modulated minimum duration**

In the absence of a target event, phase transitions are governed by two conditions that must both be satisfied:

```
eligible_for_transition =
    (athlete_level_branch >= level_threshold)
    AND (weeks_in_current_phase >= minimum_phase_duration_weeks)
```

**Level thresholds** (unchanged from v5):

| Condition | Level threshold |
|-----------|----------------|
| Transition to Build | Aerobic Foundation Athlete Level ≥ 4.0 |
| Transition to Peak | Threshold Athlete Level ≥ 5.0 |

**Minimum phase duration**

The minimum duration an athlete must spend in a phase before transitioning is modulated by their branch Mastery tier. Higher Mastery reflects prior physiological history in that system, reducing the time needed to re-establish readiness for the next phase.

```
minimum_phase_duration_weeks = max(4, 8 - (mastery_tier * 2))
```

| Mastery tier | Label | Minimum Base duration |
|-------------|-------|----------------------|
| 0 | None | 8 weeks |
| 1 | Developing | 6 weeks |
| 2 | Established | 4 weeks |
| 3 | Experienced | 4 weeks (floor) |
| 4 | Elite | 4 weeks (floor) |

The hard floor of 4 weeks applies regardless of Mastery. Even athletes with extensive prior history require sufficient time for physiological adaptations to re-express after a period of reduced training.

This ensures the system has a deterministic rule — not "it depends" — while remaining sensitive to individual history in a principled way. A brand new athlete serves the full 8 weeks. A returning athlete with high Mastery reaches the gate sooner. An athlete with limited time before an event should use Method 1 (event-based) to compress phases proportionally rather than bypassing Method 2's floor.

**Method 3: Manual override**

Coach or athlete can manually set the current phase.

Manual override remains in effect until the athlete enters a new target event, at which point event-based scheduling resumes.

### Phase Transition Behaviour

When a phase transition occurs:
- Periodisation modifiers update immediately
- Current Athlete Levels are retained
- Active prescriptions are recalculated
- A transition notification is sent to the athlete

---

## Development Tree

### Purpose

The Development Tree is the primary athlete-facing visualisation of Mastery.

It translates abstract credit accumulation into a visible, growing structure and creates long-term engagement with the platform.

### Visual Structure

**Trunk**

Represents Aerobic Foundation Mastery.

- Trunk width scales with Aerobic Foundation Mastery points
- Trunk height represents total training history (years active on the platform)
- The trunk is always visible; it never disappears

**Primary branches**

Five primary branches grow from the trunk:

| Branch | Position | Phenotype indicator |
|--------|----------|---------------------|
| Aerobic Foundation | Trunk | Core development |
| Threshold | Upper left | Sustained power |
| VO2max | Upper centre-left | Aerobic ceiling |
| Anaerobic | Upper centre-right | Short-duration power |
| Neuromuscular | Upper right | Sprint and peak force |

**Branch thickness**
```
branch_thickness = base_thickness + (mastery_points_branch / mastery_scale) * max_delta
```
All branches maintain a minimum visible thickness. No branch disappears.

**Branch length**

Proportional to each branch's Mastery points relative to the athlete's own highest-scoring branch. This keeps the tree proportionate regardless of absolute mastery level.

**Milestone nodes**

Each branch has milestone nodes at Levels 3, 5, 7, and 10.

- Locked: visible but dimmed until reached
- Unlocked: fully rendered and labelled
- Unlocked date is stored and displayed on hover

Example milestone labels:

| Branch | Level 3 | Level 5 | Level 7 | Level 10 |
|--------|---------|---------|---------|---------|
| Aerobic Foundation | Endurance base | Long ride engine | Durability | Aerobic elite |
| Threshold | Endurance pace | Threshold climber | Time trialist | Threshold elite |
| VO2max | Aerobic engine | VO2 builder | Pursuit power | Aerobic ceiling |
| Anaerobic | Punch | Attacking power | Race power | Anaerobic elite |
| Neuromuscular | Sprint awareness | Explosive burst | Raw power | Sprint specialist |

**Leaf density**

Leaf density on each branch represents recent activity on that branch:

- No recent training (28+ days): bare branches
- Light training: sparse leaves
- Consistent training: full foliage

Leaf density decays over 28 days of inactivity per branch. Inactive branches visually lose their leaves, giving the athlete a clear signal of which systems are being neglected.

```
leaf_density_branch = exp(-(ln(2) / 14) * days_since_branch_trained)
```
Half-life of 14 days.

### Phenotype Shape

The overall tree shape reflects the athlete's natural phenotype based on relative branch Mastery:

| Phenotype | Dominant branches | Tree shape |
|-----------|------------------|------------|
| Time Trialist | Aerobic, Threshold | Thick trunk; heavy left growth |
| Climber | Aerobic, Threshold, VO2max | Tall and balanced |
| Sprinter | Anaerobic, Neuromuscular | Strong upper right; thinner trunk |
| Rouleur | Threshold, Anaerobic | Balanced upper branches |
| All-rounder | All branches | Even proportions |
| Ultra endurance | Aerobic Foundation | Wide, dense trunk; sparse upper branches |

Phenotype is derived automatically from branch proportions. It is informational only and does not affect prescription.

### Data Model

```
development_tree {
  branches: {
    [branch_name]: {
      mastery_points: float,
      branch_thickness: float,        // derived
      branch_length: float,           // derived
      leaf_density: float,            // derived, 0.0 to 1.0
      milestones: {
        level_3:  { label, unlocked, unlocked_date },
        level_5:  { label, unlocked, unlocked_date },
        level_7:  { label, unlocked, unlocked_date },
        level_10: { label, unlocked, unlocked_date }
      }
    }
  },
  phenotype: string,                  // derived from branch proportions
  total_mastery_points: float,
  tree_age_days: int                  // days since first training record in system
}
```

---

## Stimulus Engine

### Purpose

The Stimulus Engine determines what physiological dose to prescribe next.

It prescribes adaptation targets, not workouts.

### Sweet Spot as a Prescription Methodology

Sweet Spot (88-93% FTP) is not a branch. It is a training methodology that the Stimulus Engine can prescribe when developing Aerobic Foundation or Threshold.

When the Stimulus Engine targets either of those branches and the athlete's current level and readiness support sustained sub-threshold work, it may prescribe a Sweet Spot session as the delivery mechanism.

Sweet Spot sessions earn credits across both branches simultaneously:

```
credits_aerobic_foundation = total_credits * 0.60
credits_threshold = total_credits * 0.40
```

The credit split is configurable. The 60/40 default reflects the greater aerobic demand of sustained Sweet Spot work relative to its lactate threshold contribution.

The prescription object for a Sweet Spot session identifies both target branches:

```
prescription {
  target_branch: "aerobic_foundation",
  secondary_branch: "threshold",
  methodology: "sweet_spot",
  credit_split: { aerobic_foundation: 0.60, threshold: 0.40 },
  ...
}
```

The Workout Generator interprets `methodology: "sweet_spot"` as a constraint that the primary interval intensity must fall in the 88-93% FTP range.



For each potential session, each branch receives a priority score:

```
priority_score_branch =
    (periodisation_modifier_branch * 0.40)
  + (readiness_score_branch * 0.35)
  + (recency_score_branch * 0.25)
```

Where `recency_score_branch` is 0.0 if the branch was trained yesterday, and 1.0 if it has not been trained in 14 or more days.

The branch with the highest priority score becomes the target for this session.

### Stimulus Level Targeting

The target stimulus level defaults to the centre of the Productive Progression Window:

```
target_stimulus_level = athlete_level_branch * 1.00
```

This target is then adjusted:

| Condition | Adjustment |
|-----------|-----------|
| Confidence less than 60% | Shift to 0.80 x athlete_level (data-gathering) |
| Confidence 60% to 85% | Shift to 0.90 x athlete_level (conservative) |
| Confidence greater than 85% | Standard (1.00 x athlete_level) |
| Three consecutive Exceptional completions | Shift to 1.10 x athlete_level (stretch) |
| readiness_combined less than 0.50 | Recovery session; no branch target |

### Stimulus Prescription Output

The Stimulus Engine outputs a prescription object passed to the Workout Generator:

```
prescription {
  target_branch: string,
  target_stimulus_level: float,
  difficulty_classification: string,  // Recovery / Easy / Productive / Stretch / Breakthrough
  secondary_branch: string | null,    // Optional compatible secondary branch
  available_time_minutes: int,
  session_notes: string               // Plain-language rationale for the prescription
}
```

The `session_notes` field provides a coach-readable explanation of why this prescription was chosen.

---

## Workout Generation Engine

### Purpose

Generates a specific workout structure to deliver the prescribed stimulus.

Workouts are generated dynamically. They are not selected from a library.

### Inputs

| Input | Source |
|-------|--------|
| Stimulus prescription | Stimulus Engine |
| Available time | Athlete preference |
| Athlete Levels (all branches) | Progression Model |
| Capability Scores | Capability Model |
| Current FTP | intervals.icu |
| Fatigue status | Fatigue Model |
| Current phase | Periodisation Model |
| Recent workout history | Activity data |

### Generation Rules

1. Every session includes a warm-up and a cool-down
2. Target branch work is placed after warm-up while the athlete is physiologically fresh
3. Secondary branch work may be appended where compatible (for example, aerobic base following threshold intervals)
4. Total duration must not exceed available time
5. The target stimulus must be achievable given current capability and fatigue
6. Adjacent sessions should vary their structure even when the stimulus type is the same, to prevent repetition

### Stimulus Equivalency

Multiple workout structures can deliver the same physiological stimulus.

The Workout Generator selects structure based on available time and session history, not a fixed prescription.

Example: Threshold Level 5 stimulus

| Option | Structure |
|--------|-----------|
| A | 3 x 12 minutes |
| B | 2 x 18 minutes |
| C | 4 x 9 minutes |
| D | 1 x 30 minutes (if TTE supports it) |

### Output Format

```
workout {
  warm_up: { duration_minutes, intensity_target_pct_ftp },
  intervals: [
    {
      duration_minutes: float,
      power_target_pct_ftp: float,
      rest_duration_minutes: float
    }
  ],
  cool_down: { duration_minutes, intensity_target_pct_ftp },
  total_duration_minutes: int,
  expected_tss: float,
  primary_stimulus_branch: string,
  primary_stimulus_level: float,
  secondary_stimulus_branch: string | null,
  prescription_rationale: string
}
```

*Note: Validate this output format against the intervals.icu planned workout format prior to implementing workout push in v2.*

---

## Athlete Phenotypes

The shape of the Development Tree creates rider archetypes based on relative branch Mastery.

Phenotypes are informational. They do not directly affect prescription.

| Phenotype | Dominant branches |
|-----------|-----------------|
| Time Trialist | Aerobic Foundation, Threshold |
| Climber | Aerobic Foundation, Threshold, VO2max |
| Sprinter | Anaerobic, Neuromuscular |
| Rouleur | Threshold, Anaerobic |
| All-rounder | All branches balanced |
| Ultra endurance | Aerobic Foundation only |

---

## System Update Loop

The complete update sequence after every completed workout:

1. **Ingest activity** from intervals.icu
2. **Update power curve** and recalculate affected capability scores
3. **Calculate stimulus scores** per branch from activity data
4. **Classify difficulty** using the PPW ratio
5. **Assess completion quality** against the prescribed stimulus
6. **Calculate adaptation credits** using the full formula
7. **Update Athlete Levels** per branch (add credits, recalculate decimal)
8. **Update Confidence Model** per branch with the new data point
9. **Update system-specific fatigue** per branch
10. **Update Mastery** per branch
11. **Update Development Tree** (branch properties, leaf density, phenotype)
12. **Prepare next prescription** via the Stimulus Engine

---

## Competitive Benchmarking

The platform differs from systems such as TrainerRoad Athlete Levels.

TrainerRoad primarily models workout completion capability.

Chiron models:

1. Current physiological capability (today's fitness)
2. Current training capacity (stimulus that can be absorbed)
3. Lifetime accumulated development (mastery)
4. Branch-specific fatigue and readiness

This creates a richer representation of athlete development and allows the platform to explain not only what an athlete should do next, but why, and to show what they have built over time.

---

## Outstanding Design Decisions

The following decisions are required before implementation begins.

| # | Decision | Status | Outcome |
|---|----------|--------|---------|
| 1 | Sweet Spot branch model | Resolved | Cross-branch prescription methodology; no separate branch |
| 2 | Coggan normative tables | Open | Gender-specific from launch recommended |
| 3 | W' estimation and Anaerobic stimulus model | Resolved | Two-phase hybrid model: p60 w/kg primary for all athletes; W' enrichment (40% weighting) unlocked when confidence ≥ 75% and 3+ qualifying intentional efforts exist |
| 4 | Workout push to intervals.icu | Open | v2 recommended (v1 read-only) |
| 5 | HRV in readiness formula | Open | v2 recommended |
| 6 | Manual phase override duration | Open | Until next event entry recommended |

---

## v1 Design Constraints

Version 1 must remain:

- **Deterministic**: every credit, level, and prescription is derivable from the formula
- **Explainable**: a coach must be able to audit any recommendation from raw inputs
- **Conservative**: where data is insufficient, prescribe conservatively and gather data first
- **Read-only**: intervals.icu integration is read-only in v1

Future versions may incorporate:

- AI-generated progression curves
- Dynamic stimulus optimisation
- Individual adaptation rate modelling
- HRV integration in readiness
- Workout push to intervals.icu
- Population-level benchmarking
- Event-specific progression trees
- Recovery prediction modelling

---

## Implementation Notes

*This section is addressed to Claude Code and any developer implementing this specification. It defines scope, document relationships, and the values that must be seeded before the engine can run.*

### Purpose of This Specification

This document specifies a rewrite of the Chiron coaching system's progression and prescription layer. The existing ladder and rung mechanic in `PROCESS_W.md` is superseded by the Development Engine architecture defined here. The replacement addresses three specific weaknesses in the prior system:

- Insufficient logic for seeding an athlete's starting position on the scale
- No principled model for progression rate or stimulus targeting
- Vocabulary and structure that did not adequately communicate development to the athlete

### What Is Preserved

The following existing components remain intact and are consumed by the new system. Do not rewrite or replace them.

**`SECTION_11.md` — AI Coach Protocol**

Owns all readiness infrastructure: the P0–P3 readiness decision ladder, TID classification, durability metrics, ACWR, the sync layer, and the `latest.json` / `history.json` / `intervals.json` data mirror. The Development Engine reads from this infrastructure but does not replace it.

Priority rule: Section 11's Readiness Decision (P0–P3) takes absolute precedence over the Development Engine's Confidence Model. If Section 11 returns `skip` or `modify`, that overrides any stimulus the Stimulus Engine would otherwise prescribe, regardless of branch confidence or PPW classification.

**`PROCESS_W.md` — Workout Prescription Ontology**

Owns workout primitives: zone definitions (Z1–Z7), effort block structure, session shape (warm-up, intervals, cool-down), recovery interval rules, and structural session patterns. The Development Engine's Workout Generator consumes these definitions when building sessions. It does not redefine zones or session structure.

Zone vocabulary note: `PROCESS_W.md` uses granular Z1–Z7 (Coggan-style). `SECTION_11.md` uses Seiler three-zone aggregates with the same `Z1/Z2/Z3` symbols but different meanings. The Development Engine branches (Aerobic Foundation, Threshold, VO2max, Anaerobic, Neuromuscular) map to `PROCESS_W.md`'s granular zones, not Section 11's Seiler aggregates.

**`CHIRON.md` — Coaching Persona**

The system prompt and coaching voice are unchanged. The Development Engine outputs (`prescription_rationale`, `session_notes`, milestone labels, phenotype descriptions) must conform to the voice and posture defined there.

### What Is Replaced

The following are superseded by this specification and should not be carried forward:

- The ladder and rung progression mechanic in `PROCESS_W.md` (sections 4.x)
- Any fixed session template selection logic tied to rung position
- The prior concept of athlete "level" as a single undifferentiated number
- Any phase transition logic based solely on calendar duration without a fitness gate

The physiological foundations in `PROCESS_W.md` (zones, primitives, session patterns) remain. Only the progression and prescription selection mechanic is replaced.

### Values That Must Be Seeded Before the Engine Can Run

The following values are referenced in this spec but not fully defined. They must be resolved before implementation begins or the engine cannot produce valid output.

**Stimulus level normalisation function**

The spec requires branch stimulus metrics (e.g. Threshold TIZ in minutes, VO2max minutes above 108% FTP) to be normalised to a 0–10 stimulus level "referenced against level-appropriate training norms." The normalisation function and the training norm tables it references are not defined here. These must be specified before the Stimulus Score can be computed.

Recommended approach: define a per-branch normalisation table mapping raw metric values to stimulus levels at each integer Athlete Level (1–10). These tables should be seeded from published training load norms and calibrated against real athlete data post-launch.

**`SFatigue_max_branch` calibration constants**

The branch readiness formula requires a per-branch maximum fatigue constant:

```
readiness_branch = 1.0 - min(1.0, SFatigue_branch / SFatigue_max_branch)
```

Initial values must be set from training load norms before the fatigue model can run. Suggested starting values (to be calibrated per athlete over time):

| Branch | Suggested initial SFatigue_max |
|--------|-------------------------------|
| Neuromuscular | 8.0 |
| Anaerobic | 10.0 |
| VO2max | 12.0 |
| Threshold | 15.0 |
| Aerobic Foundation | 20.0 |

These reflect the relative volume capacity of each system and should be treated as configurable parameters, not hard constants.

**Mastery tier point boundaries**

The Mastery tier boundaries (0 / 500 / 1,500 / 3,500 points) are initial values with no population data behind them. They govern phase transition minimum duration and are the first values likely to need recalibration after real athlete data is available. Implement as configurable constants, not hard-coded values.

**Coggan normative tables**

Male approximate w/kg norms are included in the spec. Female norms are not yet sourced and must be added before the system serves female athletes. Both tables must be implemented as configurable data structures, not hard-coded values, to allow recalibration.

### Relationship Summary

```
SECTION_11.md          PROCESS_W.md
(readiness, data)      (zones, session structure)
        |                      |
        +----------+-----------+
                   |
        Development Engine (this spec)
        (capability, progression, prescription targeting)
                   |
           Workout Generator
        (consumes PROCESS_W primitives)
                   |
            intervals.icu MCP
              (data layer)
```

The Development Engine sits between the data and readiness layer (Section 11) and the prescription execution layer (PROCESS_W). It does not own data or session structure — it owns the logic that connects an athlete's development state to an appropriate physiological stimulus.