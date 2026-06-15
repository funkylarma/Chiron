# PROCESS_W - Workout Prescription Ontology - v2

> **Document status:** Active - supersedes PROCESS_W v1  
> **Supersedes:** PROCESS_W.md (v1, ladder/rung mechanic)  
> **Key changes from v1:** Progression mechanic replaced entirely by the Chiron Development Engine (see `DEVELOP_E.md`); zone crosswalk to Development Engine branches formalised; handoff contract between Stimulus Engine and Workout Generator defined; Layer 4 (ladder/rungs/advancement tiers) removed; Layer 3 feedback vocabulary updated to Development Engine terminology; extensive vs intensive progression logic retained and mapped to branch-level prescription axis

---

## Scope and System Context

This document is the **ontology and grounding logic** layer for workout prescription. It defines concepts from first principles so that the reasoning layer has something precise to reason over. It is consumed by the Workout Generator, which translates a `prescription` object from the Chiron Development Engine's Stimulus Engine into a specific session using the structural patterns and zone definitions here.

**What this document owns:**

- Zone definitions (Z1–Z7) and their physiological foundations
- Effort block primitives — intensity, duration, recovery, session shape
- Structural session patterns (sustained power, intervals, on-offs, attacks, etc.)
- The extensive vs intensive progression axis and how it maps to branch targets
- The handoff contract between the Stimulus Engine and the Workout Generator
- Session feedback vocabulary and completion quality assessment

**What this document does not own:**

- Readiness and daily go/modify/skip decisions — **Section 11 is authoritative**
- Capability scoring, Athlete Levels, Mastery, and the Development Tree — **the Development Engine spec is authoritative**
- Progression mechanic (when and how stimulus difficulty advances) — **the Development Engine spec is authoritative**
- Load metrics (CTL/ATL/TSB, ACWR, TID, durability measurement) — **Section 11 is authoritative**

**One rule resolves every boundary conflict:**
> Measurement and readiness follow Section 11. Prescription structure follows this document. Progression targeting follows the Development Engine.

---

## Relationship to Section 11

Section 11 (`SECTION_11.md`) owns how training data is interpreted: load metrics, readiness, TID, durability, and the sync layer. This document owns how sessions are structured. The two dovetail at a clean boundary.

**Priority rule:** Section 11's Readiness Decision (P0–P3) takes absolute precedence over any prescription this document supports. If Section 11 returns `skip` or `modify`, the planned session is deferred, reduced, or swapped for recovery — regardless of what the Development Engine has prescribed.

| Section 11 verdict | Effect on prescription |
|-------------------|----------------------|
| P0 — blocked | No session. Development Engine stimulus targeting pauses. |
| P1 — modify | Session reduced or swapped for recovery. Stimulus Engine target holds; credits are recalculated on actual completion. |
| P2 — normal | Prescribe as planned. |
| P3 — high readiness | Prescribe as planned. Stimulus Engine may target the upper edge of the Productive Progression Window. |

**Zone vocabulary:** This document uses granular **Z1–Z7** (Coggan-style). Section 11 uses Seiler three-zone aggregates (`Z1/Z2/Z3`) with different meanings. The symbols do not align:

| Symbol | This document (granular) | Section 11 (Seiler aggregate) |
|--------|--------------------------|-------------------------------|
| Z1 | Recovery, < 55% FTP | Easy: granular Z1–Z2, below LT1 |
| Z2 | Endurance, 55–75% FTP | Grey zone: granular Z3, LT1–LT2 |
| Z3 | Tempo, 76–87% FTP | Hard/quality: granular Z4–Z7, above LT2 |

Within this document, `Zn` always means the granular zone. When referencing Section 11's distribution buckets, use "Seiler easy / grey / hard" rather than zone numbers.

**Subjective inputs** follow Section 11's scales: RPE 1–10; Feel 1–5 inverted (1 = Strong); wellness 1–4 inverted (1 = best).

**Durability measurement** belongs to Section 11: the Durability Index (average power last hour ÷ first hour) and aggregate durability (mean decoupling on qualifying steady sessions ≥ 90 minutes). The conceptual framing of durability as a fatigued power curve in section 1.7 below is descriptive; Section 11's metrics are the operational signals.

---

## Relationship to the Chiron Development Engine

The Development Engine (`DEVELOP_E.md`) owns the progression layer. This document owns the prescription structure layer. The boundary is the `prescription` object: the Development Engine's Stimulus Engine produces it; this document's Workout Generator consumes it.

**Branch-to-zone mapping**

Development Engine branches map to granular zones as follows. The Workout Generator uses this mapping to select appropriate structural patterns and intensity targets.

| Branch | Primary granular zones | Dominant structural patterns |
|--------|----------------------|------------------------------|
| Aerobic Foundation | Z2, Z3 | Sustained power, endurance with bursts |
| Threshold | Z4, Sweet Spot (Z3–Z4 boundary) | Sustained power, intervals, over-unders |
| VO2max | Z5 | Intervals, on-offs, float sets, traditional (5×5) |
| Anaerobic | Z6 | Intervals, steps, attacks |
| Neuromuscular | Z7 | Sprint efforts, attacks (Phase 1 only) |

**Sweet Spot** is the Z3–Z4 boundary intensity (approximately 88–93% FTP). It is not a branch; it is a prescription methodology the Stimulus Engine may specify when targeting Aerobic Foundation or Threshold. When `methodology: "sweet_spot"` appears in the prescription object, the Workout Generator constrains primary interval intensity to the 88–93% FTP range and selects sustained power or interval patterns accordingly.

**Progression axis mapping**

Each branch carries a progression axis (extensive or intensive) set by the Development Engine based on event demand and training phase. The Workout Generator applies this axis when structuring the main set:

- **Extensive axis** — hold intensity at the branch target level; extend effort block duration toward the branch's duration ceiling
- **Intensive axis** — hold effort block duration; raise intensity within the branch's zone range

The axis is passed in the `prescription` object. The Workout Generator does not determine it independently.

---

## The Prescription Handoff Contract

The Stimulus Engine outputs a `prescription` object. The Workout Generator receives it and produces a `workout` object. This section defines what the Workout Generator does with each field.

### Prescription object (input)

```
prescription {
  target_branch: string,              // Aerobic Foundation | Threshold | VO2max | Anaerobic | Neuromuscular
  target_stimulus_level: float,       // 1.0–10.0, normalised stimulus intensity for this branch
  difficulty_classification: string,  // Recovery | Easy | Productive | Stretch | Breakthrough
  secondary_branch: string | null,    // Optional compatible secondary branch
  methodology: string | null,         // "sweet_spot" when specified; otherwise null
  credit_split: object | null,        // Only present when methodology is "sweet_spot"
  available_time_minutes: int,        // Athlete's available session time
  progression_axis: string,           // "extensive" | "intensive"
  session_notes: string               // Coach-readable rationale from the Stimulus Engine
}
```

### Workout Generator rules

**1. Branch → zone selection**

Use the branch-to-zone mapping above. For VO2max, select Z5. For Threshold, select Z4 unless `methodology: "sweet_spot"` is present, in which case constrain to 88–93% FTP.

**2. Stimulus level → session structure**

`target_stimulus_level` (1.0–10.0) determines effort block duration and/or intensity within the target zone, depending on `progression_axis`:

- Extensive axis: stimulus level drives total time-in-zone. Higher levels = more accumulated time in zone per session.
- Intensive axis: stimulus level drives interval intensity within the zone. Higher levels = higher power targets within the zone band.

The Workout Generator translates stimulus level to concrete session parameters using the normalisation tables defined in the Implementation Notes of the Development Engine spec.

**3. Difficulty classification → structural constraints**

| Classification | Structural constraint |
|---------------|----------------------|
| Recovery | No branch target. Z1 session only. Full session duration. |
| Easy | Branch zone, low-mid position. Shorter effort blocks. Extended recovery. |
| Productive | Branch zone, mid position. Standard effort blocks and recovery per pattern. |
| Stretch | Branch zone, mid-high position. Longer or harder blocks. Moderate recovery reduction. |
| Breakthrough | Branch zone, high position. Maximum effort block duration/intensity within the branch ceiling. |

**4. Available time → session scaling**

Total session duration must not exceed `available_time_minutes`. Scale effort block count or duration accordingly. Warm-up and cool-down are mandatory and not negotiable; they are scaled within the available time before the main set is reduced.

**5. Secondary branch**

When `secondary_branch` is present, append compatible secondary work after the primary main set. Secondary work must not compromise the primary stimulus. Suitable combinations:

| Primary branch | Compatible secondary |
|---------------|---------------------|
| Threshold | Aerobic Foundation (Z2 volume) |
| VO2max | Threshold (brief, moderate) |
| Anaerobic | Neuromuscular (sprint count, low volume) |
| Aerobic Foundation | Threshold (short, easy) |

Secondary work is always appended, never interleaved with primary intervals.

**6. Session variety**

Adjacent sessions targeting the same branch must vary their structural pattern even when stimulus type and level are the same. The Workout Generator tracks recent session patterns per branch and rotates through available patterns to prevent repetition.

### Workout object (output)

```
workout {
  warm_up: {
    duration_minutes: float,
    intensity_target_pct_ftp: float
  },
  main_set: {
    pattern: string,                    // structural pattern name from section 1.6
    intervals: [
      {
        duration_minutes: float,
        power_target_pct_ftp: float,
        power_target_zone: string,      // e.g. "Z4 mid"
        rest_duration_minutes: float,
        rest_character: string          // "active" | "passive"
      }
    ],
    secondary_intervals: [] | null      // Same structure; null if no secondary branch
  },
  cool_down: {
    duration_minutes: float,
    intensity_target_pct_ftp: float
  },
  total_duration_minutes: int,
  expected_tss: float,
  primary_stimulus_branch: string,
  primary_stimulus_level: float,
  secondary_stimulus_branch: string | null,
  progression_axis: string,
  prescription_rationale: string        // Human-readable; drawn from prescription.session_notes
}
```

---

## Preamble: Purpose and Philosophy

### Why This System Exists

*"The best predictor of performance is performance itself."* — Andrew Coggan

The most accurate way to predict a 40km time trial is to ride a 40km time trial. Training cannot replicate this. What training can do is construct the most faithful simulation of the event's demand, applied progressively and repeatedly, until the athlete arrives at race day having built the specific physiological qualities the event will test.

Everything in this system — the zones, the structural patterns, the progression logic — is a proxy for that goal. The intervals are the mechanism. The power curve is the measure. The race is the point.

### The Purpose of Interval Training

The fundamental premise of interval training is that it allows an athlete to accumulate a stimulus that would be unsustainable as a continuous effort. A rider cannot hold sixty minutes at threshold on day one. They can hold three efforts of twelve minutes. Over time those blocks extend, the recoveries shorten, and the athlete moves toward the target duration at target intensity through a structured progression.

This is not a workaround — it is the mechanism. The interval is a tool for building time-in-zone without requiring the athlete to spend the entire session at that intensity. The goal is always the target duration at target intensity; the Development Engine's progression model is the path to get there.

### Two Modes of Curve Modification

Every prescription attempts to modify the athlete's power curve in a specific direction.

**Extending the curve** — the same intensity sustained for longer. The athlete can already produce this power; the adaptation is the ability to hold it further into duration. The curve moves rightward. This is the extensive axis.

**Raising the curve** — more power at the same duration. The athlete can already sustain this length of effort; the adaptation is the ability to produce more across it. The curve moves upward. This is the intensive axis.

The Development Engine determines which axis applies to each branch based on event demand and training phase. The Workout Generator applies it.

### Limiter Removal vs Blade Sharpening

Every prescription operates in one of two modes:

**Limiter removal** targets a branch where the athlete's Capability or Athlete Level is meaningfully below their best branches relative to their target event's demands. The prescription addresses the gap directly.

**Blade sharpening** reinforces existing strengths in the specific context of the target event. The athlete is already competitive in their target domain; the prescription fine-tunes the systems that will decide the outcome.

The Development Engine's branch priority scoring and periodisation phase together determine which mode the Stimulus Engine is in. The Workout Generator receives the output and structures the session accordingly.

### Individual Response Variability

No two athletes respond identically to the same prescription. The Development Engine accounts for this through the Confidence Model and the Productive Progression Window — prescribing conservatively until the system has sufficient data to understand how the individual athlete responds. This document's structural patterns are the execution layer for that conservative-to-confident progression.

---

## Layer 1: Workout Primitives

*The atoms from which all sessions are built.*

### 1.1 The Effort Block

The effort block is the atomic unit of a workout — a single continuous effort at a defined intensity for a defined duration.

#### Intensity Expression

**Absolute** — a specific wattage. Used when the specific number matters: test efforts, race simulation.

**Zone-relative with descriptor** — the primary expression language of this document. A zone combined with a positional descriptor communicates physiological intent without over-specifying. Position descriptors: *low*, *mid*, *high*, and *into* for zone boundaries (e.g. "high tempo into sweet spot").

**Percentage of FTP** — the translation layer for platforms and devices. Zone-relative language is converted to FTP percentages by the prescription builder, not defined as such here.

**RPE** — sanity check and fallback. The subjective experience of effort (1–10) validates power data and serves as the primary prescription tool when power is unavailable. Also the primary language of the session feedback layer.

#### Duration

An effort block has a target duration. A block too short for its zone does not produce the intended stimulus. A block too long collapses into a lower zone as the athlete fades. The valid duration range for each branch and pattern is defined in the structural patterns below.

#### Block Validity

A block is valid if the athlete held the intended intensity for the intended duration. A block where intensity drifted significantly below the target zone for more than a brief period is not valid for that zone, regardless of average power. Average power can be misleading — a threshold block that starts at FTP and collapses into tempo has a threshold-range average but did not produce a threshold stimulus.

Validity determines whether the session counts toward the Development Engine's Completion Quality assessment. Gradations (held it but it cost more than expected; faded the last minute; exceeded target) inform the Difficulty Factor and Completion Quality multipliers.

### 1.2 Recovery Intervals

A recovery interval is a prescribed component of the session, not passive time. Getting recovery wrong changes the stimulus as much as getting the effort wrong.

#### Recovery Character

**Active recovery** — pedalling at Z1. Maintains circulation and accelerates lactate clearance without adding meaningful physiological stress. The default for most quality sessions.

**Passive recovery** — complete rest. Reserved for neuromuscular sessions where phosphocreatine replenishment is the priority, and for sessions where even light pedalling would compromise the next effort.

#### Recovery Duration

| Work:rest ratio | Character | Primary use |
|-----------------|-----------|-------------|
| 1:1 or shorter | Short — incomplete recovery | On-offs, float sets; sustained elevation near ceiling |
| 1:1 to 1:2 | Moderate — partial recovery | Threshold, Sweet Spot; repeatability under fatigue |
| 1:3 or longer | Long — near-full recovery | Neuromuscular, Anaerobic, VO2max; quality of each effort paramount |

Recovery duration is part of the prescription. Cutting it short corrupts the subsequent effort block regardless of what the power numbers show.

### 1.3 Session Shape

A session has four components. Skipping any of them changes what the session does.

**Warm-up** — elevates heart rate, increases muscle temperature, primes cardiovascular and neuromuscular systems. Begins at Z1 and builds gradually into the lower end of the target zone. Duration scales with session intensity — neuromuscular and anaerobic sessions require longer warm-ups.

**Settle** — brief transition (2–5 minutes, easy intensity) between warm-up and first effort block. Allows the athlete to find position, take on nutrition, and make the mental transition to work. Not required for endurance or recovery sessions.

**Main set** — the prescription itself. Everything before it is preparation; everything after it is conclusion.

**Cool-down** — returns the body to resting state gradually. Duration proportional to session intensity.

| Session type | Warm-up | Settle | Main set | Cool-down |
|-------------|---------|--------|----------|-----------|
| Quality (intervals, structured) | Full, zone-building | Yes | Effort blocks + recovery | Full |
| Endurance | Short, easy | No | Sustained Z2/Z3 | Short |
| Recovery | Minimal | No | Entire ride at Z1 | Minimal |
| Neuromuscular | Long, thorough | Yes | Short maximal efforts, full recovery | Moderate |

### 1.4 Session Load

TSS (Training Stress Score) is the standard currency for quantifying session load and feeds directly into Section 11's ATL/CTL/TSB infrastructure. It remains a valid tool for managing overall load trajectory.

**TSS does not capture stimulus type.** Two sessions with identical TSS can produce completely different fatigue profiles, require different recovery timelines, and drive different adaptations. This system accounts for this through the Development Engine's branch-specific stimulus scoring and system-specific fatigue tracking — each branch accumulates its own fatigue independently of global TSS.

**Fatigue type matters for recovery planning:**

| Fatigue type | Primary driver | Recovery timeline |
|-------------|----------------|-------------------|
| Metabolic / cardiovascular | Endurance volume, threshold work | 24–48 hours |
| Neuromuscular / CNS | Sprints, maximal efforts, high-force work | 48–72 hours |
| Muscular damage | High-force contractions, standing efforts | 48–96 hours |

The Development Engine's system-specific fatigue model (7-day half-life per branch) translates this into branch readiness scores. The Workout Generator reads branch readiness as part of the `prescription` object.

### 1.5 Intensity Expression

| Mode | Primary use | Limitation |
|------|-------------|------------|
| Absolute watts | Test efforts, race simulation | Athlete-specific; requires current FTP |
| % FTP | Platform/device prescription | Approximation; FTP drift |
| Zone + descriptor | Ontology and coaching language | Requires zone model understanding |
| RPE | Sanity check, feedback, fallback | Subjective; affected by fatigue |

### 1.6 Structural Patterns

A structural pattern defines the shape of the main set — how effort blocks are arranged, how intensity moves, and what the work/recovery relationship looks like. Patterns are zone-agnostic: the same pattern at different zones produces different stimuli. The pattern is the architecture; the zone is the material.

**Sweet Spot** is the Z3–Z4 boundary intensity range (approximately 88–93% FTP). It is not a zone and not a branch — it is a named intensity range used as a training methodology by the Stimulus Engine when targeting Aerobic Foundation or Threshold. When Sweet Spot is prescribed, the Workout Generator applies sustained power or interval patterns with intensity constrained to 88–93% FTP.

---

#### Sustained Power

A single continuous effort held at target intensity for the duration of the main set. The simplest structural pattern and the most demanding test of pacing discipline. Used where the adaptation requires prolonged time-in-zone without recovery relief.

*Primary branches: Aerobic Foundation (Z2), Threshold (Z4, Sweet Spot)*  
*Progression axis: Extensive (extend duration at held intensity)*

---

#### Intervals

Repeated effort blocks at target intensity separated by defined recovery periods. Recovery duration and character are part of the prescription and directly affect the stimulus. Shorter recoveries accumulate fatigue across the set; longer recoveries preserve quality per effort.

*Primary branches: Threshold, VO2max, Anaerobic*  
*Progression axis: Extensive (add intervals or extend block duration) or Intensive (raise intensity, hold block count and duration)*

---

#### Hard Starts

The effort begins significantly above the target zone intensity then settles into it for the remainder of the block. Recruits fast-twitch fibres and elevates lactate before the steady-state portion begins. Trains the ability to find rhythm after an initial hard effort — a common race scenario.

*Primary branches: Threshold, VO2max*  
*Progression axis: Intensive (raise the opening surge; extend the settle duration)*

---

#### Over-Unders

Alternates within a single effort block between two intensities: one just below the target zone ceiling and one just above it. The athlete repeatedly crosses the threshold between clearance and accumulation, training the metabolic machinery to handle that transition. The classic tool for training at and around LT2.

*Primary branch: Threshold*  
*Progression axis: Extensive (extend total block duration); Intensive (raise the over intensity or extend the over duration relative to under)*

---

#### Mixed Intervals

A session containing intervals at two distinct intensity levels within the same main set. Discrete efforts — not continuous alternation. Used to combine stimuli or simulate the variable demands of a race. The harder intervals must not so compromise the athlete that the secondary set becomes junk volume.

*Primary branches: Threshold, VO2max*  
*Progression axis: Extensive or Intensive depending on which component is targeted*

---

#### Ramps

Intensity increases progressively across the main set, either continuously or in discrete steps. Used to train the ability to lift intensity as fatigue accumulates and to expose the athlete to a range of intensities within a single session. Diagnostic: where the athlete drops off reveals their current ceiling.

*Primary branches: Threshold, VO2max*  
*Progression axis: Intensive (raise the ramp ceiling)*

---

#### Attacks

A compound two-phase pattern simulating a race move. Phase 1: a short explosive effort (Z6/Z7) to simulate the accelerating move. Phase 2: a sustained effort at or above threshold to simulate holding the gap on depleted systems. The physiological demand is the combination — threshold-plus power on exhausted phosphocreatine with lactate already elevated.

*Primary branches: Anaerobic (Phase 1), Threshold or VO2max (Phase 2)*  
*Progression axis: Intensive (raise Phase 1 intensity or extend Phase 2 duration)*

---

#### On-Offs

Very short alternating work and rest intervals — typically 15–30 seconds on, 15 seconds off — accumulating time near a physiological ceiling at a lower per-rep cost than longer continuous efforts. The Ronnestad protocol (30s/15s) is the canonical example. Short recovery keeps the system elevated; brief effort avoids the deep per-rep cost of a full long interval.

*Primary branch: VO2max*  
*Progression axis: Extensive (add sets); Intensive (raise the on-interval intensity)*

---

#### Float Sets

Intervals where the recovery period is a reduced-intensity "float" rather than full rest. The system remains elevated throughout. More demanding than standard intervals with full recovery. Used at VO2max intensity to extend total time near the aerobic ceiling.

*Primary branch: VO2max*  
*Progression axis: Extensive (extend float set duration or add sets)*

---

#### Traditional (5×5)

Five efforts of five minutes at VO2max intensity with defined recovery. Long enough per effort to fully stress the aerobic ceiling; short enough to repeat. The benchmark quality session for aerobic power development.

*Primary branch: VO2max*  
*Progression axis: Extensive (extend individual effort duration toward 6 or 8 minutes)*

---

#### With Bursts

An endurance-paced ride with short neuromuscular efforts embedded. The base ride is the session; the bursts (10–15 second maximal sprints) activate the ATP-PC pathway and reinforce neuromuscular recruitment without disrupting the endurance stimulus. Used to maintain neuromuscular qualities during base phase.

*Primary branch: Aerobic Foundation (Z2 base) with Neuromuscular (Z7 bursts)*  
*Progression axis: Extensive (base ride duration); burst count and intensity held constant*

---

#### Steps

Intensity increases in discrete blocks within a single continuous effort, each step held for a defined duration before moving to the next. Anaerobic-specific. Each step takes the athlete deeper into glycolytic demand. Trains progressive lactate accumulation and the ability to sustain power as metabolic conditions deteriorate.

*Primary branch: Anaerobic*  
*Progression axis: Intensive (raise the step ceiling or add a final step)*

---

### 1.7 Extensive vs Intensive Progression

Every workout can be progressed in one of two directions. The Development Engine determines which applies to each branch; the Workout Generator applies it.

**Extensive progression** — intensity held constant; effort block duration increases. The adaptation is fatigue resistance and metabolic efficiency at the current intensity. The power curve extends rightward. Lower recovery cost per session. Primary mode in Base and early Build phases.

**Intensive progression** — duration held constant; intensity increases. The adaptation is more power at the same duration. The power curve shifts upward. Higher recovery cost per session. Primary mode in late Build and Peak phases, and for branches where event demand is power-dominant rather than duration-dominant.

**Phenotype and progression priority**

The Development Engine's power profile and branch Capability scores reveal the athlete's phenotype. The Workout Generator applies progression accordingly:

A punchy, anaerobic phenotype (strong at short durations, curve drops steeply) typically needs extensive progression at longer durations to develop aerobic endurance.

A diesel, aerobic phenotype (strong at long durations, relatively flat curve) typically needs intensive progression at shorter durations to develop top-end power.

**Durability**

The fresh power curve describes what the athlete can do rested. Durability is the degree to which that curve holds under accumulated fatigue. Training durability requires deliberately placing quality work on pre-fatigued legs — the fatigue resistance exception in the session design rules below. Its measurement belongs to Section 11 (DI and aggregate durability); its prescription belongs here.

---

## Layer 2: Zone and Stimulus Mapping

*How workout primitives produce a specific physiological stimulus, and why.*

### 2.1 The Training Zones

#### Physiological Foundation

Training zones are expressions of which energy system is dominant and which is being stressed toward its limits. The three systems always operate in parallel — the question is proportion, not switching.

**Aerobic (oxidative phosphorylation)** — primary energy system for sustained effort. Draws on fat and glycogen; fat oxidation dominant at low intensities. Mitochondrial density, fat oxidation efficiency, and cardiac output are the primary adaptations. Never switches off; above a certain intensity it cannot meet demand alone.

**Glycolytic (fast glycolysis)** — faster than aerobic; glycogen-only. Becomes a meaningful contributor at threshold intensity. Produces lactate; aerobic system can clear it at moderate rates. When production outstrips clearance, lactate accumulates and effort becomes unsustainable. Threshold is precisely this equilibrium.

**Phosphocreatine / ATP-PC (alactic anaerobic)** — instantaneous power, trivial capacity (~10–15 seconds). Replenishes fully with adequate recovery. No lactate produced. Pure neuromuscular power expression.

#### Zone Model

| Zone | Name | Dominant system | FTP % (approx) | Character |
|------|------|----------------|---------------|-----------|
| Z1 | Recovery | Aerobic | < 55% | Trivial stress. Purpose is circulation and recovery. |
| Z2 | Endurance | Aerobic | 55–75% | Aerobic system stressed; fat oxidation high; glycolytic minimal. Mitochondrial development. |
| Z3 | Tempo | Aerobic + Glycolytic | 76–87% | Glycolytic now contributing. Lactate elevated but clearing. Above LT1. |
| Z4 | Threshold | Glycolytic / Aerobic at ceiling | 88–100% | Lactate equilibrium. LT2 sits near FTP. Sustainable ~30–60 min in trained athlete. |
| Z5 | VO2max | Aerobic at limit + Glycolytic high | 101–120% | Aerobic system at ceiling. Lactate accumulating. Raises the aerobic ceiling. |
| Z6 | Anaerobic Capacity | Glycolytic dominant | 121–150% | Glycolytic dominant. Lactate accumulating rapidly. Develops lactate production, tolerance, and recovery capacity. |
| Z7 | Neuromuscular | ATP-PC | > 150% | Maximal force and velocity. ATP-PC pathway. CNS demand dominant. |

#### Sweet Spot (named intensity range, not a zone)

Sweet spot sits at the Z3–Z4 boundary, approximately 88–93% FTP. Below LT2 but above the tempo zone. Maximises training load per unit of freshness cost by staying just below the lactate accumulation threshold. It is a prescription methodology available to the Stimulus Engine for Aerobic Foundation and Threshold branches. When prescribed, the Workout Generator constrains primary interval intensity to this range.

### 2.2 Branch-to-Zone Stimulus Mapping

For each Development Engine branch, this section defines the zone target, the primary physiological adaptation, the valid structural patterns, and the typical effort block duration range.

#### Aerobic Foundation

**Zone target:** Z2 primary; Z3 (tempo) secondary; Sweet Spot methodology when prescribed  
**Adaptation:** Mitochondrial density, fat oxidation efficiency, cardiac stroke volume, aerobic decoupling resistance  
**Stimulus metric:** Steady aerobic minutes in Z2–Z3 range (power and HR zone analysis)  
**Structural patterns:** Sustained power, endurance with bursts  
**Typical block duration:** 45–180 minutes continuous  
**Progression axis:** Extensive (extend duration at held intensity)

Aerobic Foundation is the trunk of the Development Tree. It is built through volume at moderate intensity, not through intervals. The adaptation timescale is long — weeks to months — which is why Aerobic Foundation Mastery is the slowest to decay and why it is the prerequisite for meaningful development in threshold and VO2max branches.

#### Threshold

**Zone target:** Z4 primary; Sweet Spot (88–93% FTP) when Stimulus Engine specifies  
**Adaptation:** LT2 elevation, time-to-exhaustion at FTP (TTE), lactate clearance efficiency  
**Stimulus metric:** Threshold Time in Zone (TIZ)  
**Structural patterns:** Sustained power, intervals, over-unders, hard starts  
**Typical block duration:** 8–30 minutes  
**Progression axis:** Extensive (extend TIZ) before Intensive (raise intensity within Z4)

TTE — the longest duration the athlete can sustain at FTP — is the key secondary metric for the Threshold branch. It is derived from training data (longest completed sustained efforts at or above 95% FTP). Sessions that target TTE extension use sustained power or long intervals at the lower end of Z4.

#### VO2max

**Zone target:** Z5 (101–120% FTP)  
**Adaptation:** Aerobic ceiling (VO2max), stroke volume, cardiac output  
**Stimulus metric:** Effective VO2 time (minutes above 108% FTP)  
**Structural patterns:** Intervals, on-offs, float sets, traditional (5×5)  
**Typical block duration:** 3–8 minutes  
**Progression axis:** Extensive (add sets or extend block duration) before Intensive (raise intensity within Z5)

VO2max branch requires the highest readiness of any branch. It should not be targeted when Section 11 returns P1 (modify) or when system-specific fatigue is elevated. The aerobic ceiling adapts more slowly than threshold and is more sensitive to accumulated fatigue.

#### Anaerobic

**Zone target:** Z6 (121–150% FTP)  
**Phase 1 stimulus metric (default):** 1-minute best power (p60 w/kg), scored against Coggan norms  
**Phase 2 stimulus metric (W' enriched):** Composite of p60 score (60%) and W' expenditure in kJ (40%)  
**Transition to Phase 2:** Anaerobic branch confidence ≥ 75% AND at least 3 qualifying intentional efforts spanning different durations (approximately 3, 5, and 12 minutes)  
**Structural patterns:** Intervals, steps, attacks  
**Typical block duration:** 30 seconds – 3 minutes  
**Progression axis:** Intensive (raise intensity or step ceiling) after extensive base at current level

The two-phase stimulus model reflects the reliability characteristics of W' estimation from field data. p60 is always available and reliable. W' provides a richer physiological picture but requires dedicated intentional efforts to estimate reliably — training-data extraction alone carries ~25% prediction error. The Workout Generator applies the same structural patterns regardless of which phase is active; the phase affects scoring and credit calculation in the Development Engine, not session design here.

#### Neuromuscular

**Zone target:** Z7 (> 150% FTP)  
**Adaptation:** Peak force, peak velocity, ATP-PC capacity, motor unit recruitment  
**Stimulus metric:** Sprint count weighted by intensity  
**Structural patterns:** Attacks (Phase 1), sprints embedded in endurance sessions (with bursts)  
**Typical block duration:** 5–15 seconds  
**Progression axis:** Intensive (raise peak power or cadence at held duration)  
**Recovery requirement:** Passive; full phosphocreatine replenishment. Minimum 3–5 minutes between maximal sprints.

Neuromuscular work should be placed early in the session when the athlete is physiologically fresh. CNS fatigue from prior work is the primary limiter for sprint quality, not metabolic fatigue. A neuromuscular session preceded by heavy aerobic or threshold work will not produce the intended stimulus.

### 2.3 Fatigue Type by Branch

| Branch | Fatigue type | Recovery requirement | Effect on adjacent sessions |
|--------|-------------|---------------------|---------------------------|
| Aerobic Foundation | Metabolic / cardiovascular | 12–24 hours | Low; compatible with most next-day work |
| Threshold | Metabolic + glycolytic | 24–48 hours | Moderate; avoid threshold or above the next day |
| VO2max | Metabolic + CNS | 36–48 hours | High; avoid quality work within 48 hours |
| Anaerobic | Glycolytic + CNS | 48–72 hours | High; avoid anaerobic or VO2max within 48 hours |
| Neuromuscular | CNS + muscular damage | 48–72 hours | High CNS load; avoid sprint or maximal work within 48–72 hours |

The Development Engine's system-specific fatigue model (SFatigue per branch, 7-day half-life) captures this quantitatively and feeds into branch readiness. The Workout Generator does not need to compute this — it reads the readiness state from the prescription object.

### 2.4 Session Design Rules

**Quality before quantity.** Place the primary branch work first in the main set, when the athlete is physiologically fresh. Secondary branch work is always appended, never placed before primary intervals.

**Warm-up scales with intensity.** A neuromuscular session targeting Z7 efforts requires a longer warm-up that includes progressive accelerations and short Z5/Z6 priming efforts before the main set. A threshold session needs a shorter warm-up that builds smoothly into Z3/Z4.

**Recovery is not optional.** The recovery interval is part of the prescription. Cutting it short corrupts the subsequent effort block. The Workout Generator must enforce prescribed recovery durations, not treat them as guidelines.

**Fatigue resistance exception.** Deliberately placing quality work on pre-fatigued legs (for example, threshold intervals after a 3-hour endurance ride) is a specific training strategy targeting durability. It is prescribed intentionally and explicitly, not incidentally. Standard sessions avoid this. When the Development Engine prescribes a durability-specific session, the Workout Generator receives explicit instruction to sequence accordingly.

**Adjacent session variety.** The same branch trained on consecutive sessions must use different structural patterns. The Workout Generator tracks recent session patterns per branch and rotates through available options.

**Stimulus integrity over completion.** A session where the athlete cannot maintain the target zone should be terminated or reduced rather than completed at a lower zone. A Z5 session completed at Z3 does not earn Z5 credits. The Workout Generator communicates this to the athlete in the `prescription_rationale` field.

---

## Layer 3: Session Feedback and Completion Assessment

*How the system reads a completed session and what it does with the result.*

### 3.1 What the System Reads

After every completed session the Development Engine's update loop ingests activity data from intervals.icu. The Workout Generator's contribution to this loop is the completion assessment — translating the raw activity into a quality signal.

The following signals are read per session:

**Power compliance** — did each effort block hold the intended zone and intensity? A block is valid if intensity held within ±5% of the zone mid-point for at least 80% of the block duration.

**RPE vs expected** — how did the effort feel relative to the prescribed difficulty? RPE above expected indicates the session cost more than the Development Engine predicted (fatigue is higher than confidence suggests, or the stimulus level was too aggressive). RPE below expected indicates the athlete has capacity the system hasn't yet recognised.

**Completion rate** — what percentage of the main set was completed? Mapped to the Development Engine's Completion Quality multiplier (No attempt / Abandoned early / Partial / Completed / Exceptional).

**Heart rate response** — where HR data is available, HR:power relationship relative to baseline provides a secondary signal of physiological cost. Elevated HR for a given power output indicates higher fatigue or environmental stress than the stimulus score alone would suggest.

### 3.2 Completion Quality Assessment

The following maps session outcomes to the Development Engine's Completion Quality multiplier. These assessments are made per session and per branch where the session targeted multiple branches.

| Outcome | Multiplier | Assessment criteria |
|---------|-----------|---------------------|
| No attempt | 0.00 | Session not started |
| Abandoned early | 0.25 | Less than 50% of main set completed |
| Partial | 0.50 | 50–79% of main set completed |
| Completed | 1.00 | 80–100% of main set completed; effort blocks valid per zone compliance rule |
| Exceptional | 1.10 | Target exceeded; performance clearly above prescription; effort blocks completed at the high end of the zone or above with RPE below expected |

**Exceptional requires genuine exceedance.** Completing a comfortable session does not qualify. The signal is specifically: the athlete did more than asked and it felt easier than the prescription predicted. This is the system's primary signal that Athlete Level may be understated and Confidence should increase.

### 3.3 RPE Signal Interpretation

RPE relative to expected difficulty is one of the richest signals the system has. It captures information the power numbers alone cannot: fatigue state, motivation, environmental factors, and the subjective experience of the physiological stress.

| RPE vs expected | Signal | Development Engine response |
|----------------|--------|---------------------------|
| Much lower (2+ points below) | Athlete has capacity; stimulus may be below PPW | Confidence increases; future prescription may shift toward upper Productive or Stretch |
| Slightly lower (1 point below) | Session was appropriate; athlete managed it well | Standard; contributes to Confidence accumulation |
| On target | Prescription was well-calibrated | Standard |
| Slightly higher (1 point above) | Session was hard but achievable | Normal; fatigue modelling notes elevated cost |
| Much higher (2+ points above) | Overreach signal; athlete pushed beyond sustainable | Confidence decreases; next prescription shifts toward lower PPW; system-specific fatigue increases |

### 3.4 Maladaptation Signals

The system monitors for patterns that indicate the prescription is not producing the intended adaptation.

**Consistent partial completion** — three or more sessions in the same branch completed at Partial quality indicates the stimulus level is above the athlete's current capacity. The Development Engine reduces confidence and shifts prescription toward the lower end of the PPW.

**RPE consistently above expected** — the athlete is working harder than the prescription predicts for their current level. Indicates either that Athlete Level is overstated or that accumulated fatigue is higher than the branch-specific fatigue model reflects.

**Power:HR ratio declining** — for a given power output, HR is trending higher over a block of sessions. Indicates accumulated fatigue or early overreaching. The system flags this for Section 11's readiness layer to incorporate.

**No Exceptional completions in six or more sessions at the same level** — suggests the athlete is comfortably maintaining but not progressing. The Development Engine may need to shift toward the upper PPW or consider whether the branch normalisation tables need recalibration.

---

## Relationship to Section 11 — Crosswalk Reference

This section provides the explicit crosswalk between terminology used in this document and Section 11, for use by Claude Code when implementing.

| This document | Section 11 equivalent | Authority |
|--------------|----------------------|-----------|
| Z2 endurance (55–75% FTP) | Seiler "easy" (below LT1) | This doc for prescription; Section 11 for TID counting |
| Z3 tempo (76–87% FTP) | Seiler "grey zone" (LT1–LT2) | This doc for prescription intent; Section 11 flags as "minimise" |
| Z4 threshold (88–100% FTP) | Seiler "hard" (above LT2) | Both; threshold work counts in Section 11's hard bucket |
| Z5–Z7 | Seiler "hard" (above LT2) | Both |
| Effort block validity | Interval compliance | This doc defines; Section 11 measures via intervals.json |
| RPE vs expected | RPE Expectation Bands (v11.34) | Section 11 owns the bands; this doc owns the prescription interpretation |
| Completion Quality | n/a (Development Engine concept) | Development Engine spec |
| Branch fatigue | System-specific fatigue (SFatigue) | Development Engine spec |
| Branch readiness | Readiness Decision (P0–P3) modulated by branch | Section 11 P0–P3 takes priority; branch readiness is secondary |
| TTE (threshold) | Benchmark Index / sustained power observations | Section 11 tracks longitudinally; this doc uses for block duration targeting |
| Durability (fatigued curve) | Durability Index, aggregate durability | Section 11 owns measurement; this doc owns prescription implication |

---

## Document Version History

| Version | Key changes |
|---------|------------|
| v2.0 | Full rewrite. Ladder/rung mechanic (Layer 4) removed. Development Engine progression model adopted. Prescription handoff contract added. Branch-to-zone crosswalk formalised. Sweet Spot repositioned as prescription methodology. Completion assessment vocabulary aligned to Development Engine. |
| v1.x | Original PROCESS_W. Ladder and rung-based progression mechanic. Superseded by this document. |
