# Workout Prescription Ontology

A foundational reference for the workout prescription protocol. Defines concepts from first principles so that the logic layer has something precise to reason over. The protocol (`WORKOUT_PROTOCOL.md`) builds on top of this; where the two conflict, this document takes precedence on definitions.

---

## Scope and System Context

This document is the **ontology and grounding logic** layer. It defines the concepts, the physiology, the progression rules, and the decision thresholds the reasoning layer operates on. It is consumed by an LLM that produces actual prescriptions. It is not itself the prescription builder.

A note on naming: this file currently carries the filename `WORKOUT_PROTOCOL.md` while its title and role are the ontology, and it refers above to a separate `WORKOUT_PROTOCOL.md` that builds on top of it. That is a collision worth resolving in the system — the ontology and the protocol should not share a filename. Throughout this document, "this document" means the ontology; "the protocol" means the layer above it that turns these definitions into platform-specific prescriptions.

**What the system has access to.** The reasoning layer can read:
- The athlete's current power profile (FTP and the power curve across key durations), exposed by the system.
- Recent training activities and current load (CTL/ATL/TSB and per-activity files), via intervals.icu.
- A daily readiness signal — a Go/No-Go message derived from subjective scores and trending health data (HRV, resting HR, sleep, wellness), produced by a separate daily layer that runs before any prescription.

**The division of labour with the Go/No-Go layer.** This document describes how to build sessions and sequence blocks, and how to read what a completed session means. It does not compute daily readiness — that is the Go/No-Go layer's job. When the daily signal says No-Go or Caution, that decision overrides the planned prescription: the planned session is deferred, swapped for recovery, or reduced, regardless of where the ladder would otherwise step. The maladaptation and readiness material in sections 3.4 and 4.3 informs how the system interprets trends and session outcomes; it does not duplicate or override the daily gate.

**Two modes of use.** The system uses this document in one of two modes:
1. **Single session.** Given a known limiter and the current readiness signal, prescribe one appropriate session today. The limiter-removal vs blade-sharpening logic in the preamble, the zone and stimulus mapping in Layer 2, and the ladder state in Layer 4 together determine what that session should be.
2. **Block sequencing.** Given a target event and a timeframe, sequence a block of training that builds toward it — selecting phases, dominant systems, and the ladder progression within each, per Layer 4 and the phase logic in section 4.5.

**What is deliberately out of scope.** This is a personal system for a single athlete who manages the following elsewhere, by design:
- **Female-specific physiology and menstrual-cycle periodisation.** Not modelled. The athlete this system serves does not require it. A general-purpose version of this system would need to add it.
- **Off-bike strength and conditioning.** Programmed separately by the athlete. The system treats neuromuscular *on-bike* work within its framework but does not prescribe gym work, and should account for separately-programmed strength load only insofar as it appears in the readiness signal and recent-fatigue picture.

These exclusions are choices, not oversights. The system should not attempt to fill them and should not apologise for their absence.

---

## Preamble: Purpose and Philosophy

### Why This System Exists

*"The best predictor of performance is performance itself."* — Andrew Coggan

The most accurate way to predict a 40km time trial is to ride a 40km time trial. The most accurate way to predict a criterium result is to race a criterium. Training cannot replicate this — you cannot have an athlete perform their target event every day. What training can do is construct the most faithful simulation of that demand, applied progressively and repeatedly, until the athlete arrives at race day having built the specific physiological qualities the event will test.

Everything in this system — the zones, the structural patterns, the ladders, the progression logic — is a proxy for that goal. The intervals are the mechanism. The power curve is the measure. The race is the point.

### The Purpose of Interval Training

The fundamental premise of interval training is that it allows an athlete to accumulate a stimulus that would be unsustainable as a continuous effort. A rider cannot hold sixty minutes at threshold on day one. They can hold three efforts of twelve minutes. Over time those blocks extend, the recoveries shorten, and the athlete moves toward the target duration at target intensity through a structured progression that would have been impossible to reach directly.

This is not a workaround — it is the mechanism. The interval is a tool for building time-in-zone without requiring the athlete to spend the entire session in that zone. The goal is always the target duration at target intensity; the ladder is the path to get there.

### Two Modes of Curve Modification

Every prescription is attempting to modify the athlete's power curve in a specific direction. The curve has a shape — a profile of best power output across all durations — and that shape reveals both the athlete's current capabilities and their gaps. The prescription targets a specific part of that shape and applies one of two interventions:

**Extending the curve** — the same intensity sustained for longer. The athlete can already produce this power; the adaptation is the ability to hold it further into duration. The curve moves rightward.

**Raising the curve** — more power at the same duration. The athlete can already sustain this length of effort; the adaptation is the ability to produce more across it. The curve moves upward.

The choice between these modes is not arbitrary. It is determined by where the curve currently falls short relative to the demands of the target event, and by the phase of training the athlete is in. Getting this choice wrong — raising the curve when the athlete needs extension, or extending when they need elevation — produces fitness that does not transfer to race performance.

### Limiter Removal vs Blade Sharpening

Every prescription operates in one of two modes, sometimes both simultaneously:

**Limiter removal** targets a weakness that is actively costing the athlete performance. The limiter is identifiable on the power curve and measurable against the demands of the target event. A criterium rider dropped on repeated hard accelerations has an anaerobic capacity limiter. A time triallist who fades in the final kilometres has a threshold endurance limiter. The prescription targets the gap directly. You are not trying to make a good athlete great — you are removing the constraint that is preventing them from expressing the fitness they already possess.

**Blade sharpening** reinforces and extends existing strengths in the specific context of the target event. The athlete is already competitive in their target domain. The prescription fine-tunes the systems that will decide the outcome and peaks them at the right moment. A sprinter getting faster. A climber extending the duration they can hold their best watts per kilo. Here you are not filling a gap — you are honing an edge.

These two modes produce different prescriptions from the same athlete profile. The power curve and the race demands together determine which mode is appropriate — and sometimes a broken edge must be repaired before a blade can be sharpened. Phase planning exists to sequence these correctly.

### Individual Response Variability

No two athletes respond identically to the same prescription. Two athletes with the same FTP can have completely different ceilings for how many quality intervals they can sustain before the stimulus collapses. Athlete one completes five intervals at 105% with clean power and good cadence throughout. Athlete two is deteriorating by the third interval at the same intensity — the numbers may still be there but the physiological quality is gone. The same prescription has produced two different sessions.

This is not a failure of the prescription system — it is an expected property of individual athletes that the system must account for. The ladder exists precisely to resolve this. Both athletes start where they actually are, not where the numbers suggest they should be. The system seeds their entry point from their power profile, observes how they respond, and adjusts. Athlete one progresses faster. Athlete two builds the foundation they need before climbing. Both arrive at the same destination through different routes at different rates.

The power curve is the most honest input the system has. It does not describe what the athlete should be able to do. It describes what they have actually done. That distinction is the difference between a prescription built on evidence and one built on assumption.

---

## Layer 1: Workout Primitives

*What a workout is at its most fundamental level — the atoms from which all sessions are built.*

### 1.1 The Effort Block

The effort block is the atomic unit of a workout — a single continuous effort at a defined intensity for a defined duration. Everything else in a session is either context for the effort block (warm-up, cool-down, recovery) or repetition of it (intervals).

#### Intensity Expression

An effort block's intensity is expressed in one of two modes depending on what the session is trying to achieve:

**Absolute** — a specific wattage. Used when precision matters: race simulation, test efforts, or when the athlete has a known target that a zone descriptor would blur. "Complete five minutes at 320 watts" is unambiguous and leaves no room for interpretation. Absolute expression is appropriate when the number itself is the point.

**Zone-relative** — a zone combined with a qualitative position descriptor. Used when the intent is physiological rather than numerical. The zone sets the physiological target; the descriptor carries the nuance within it. "Hold steady at high tempo" and "push into the lower threshold range" are coach-legible instructions that translate to a band of acceptable power without committing to a specific number.

Position descriptors within a zone: *low*, *mid*, *high*, and for zones bordering a named range, *into* (e.g. "high tempo into sweet spot"). These are sufficient to communicate intent without over-specifying.

The ontology uses zone-relative expression throughout. Translation to platform-specific percentages of FTP is the responsibility of the prescription builder, not this document.

#### Duration

An effort block has a target duration. For quality sessions this is precise — "five minutes," "twenty minutes." For endurance and recovery sessions it may be a range — "sixty to ninety minutes at Z2." Duration combined with intensity determines the physiological demand of the block and whether the intended stimulus is achievable.

A block that is too short for its zone does not produce the intended stimulus — a thirty-second effort cannot train threshold regardless of the power output. A block that is too long collapses into a lower zone as the athlete fades. Both are prescription failures. The valid duration range for each zone and pattern is implicit in the session design and made explicit in the structural pattern definitions.

#### What Makes a Block Valid

A block is valid if the athlete held the intended intensity for the intended duration. Validity is binary for prescription purposes — the block either counts or it doesn't — but the feedback layer introduces gradations (held it but it cost more than expected; faded the last minute; exceeded target). Those gradations inform progression; validity determines whether the session counts toward the ladder.

A block where intensity drifted significantly below the target zone for more than a brief period is not a valid block for that zone, regardless of average power. Average power can be misleading — a block that starts at threshold and collapses into tempo has a threshold-range average but did not produce a threshold stimulus.

### 1.2 Recovery Intervals

A recovery interval is the period between effort blocks. It is not passive time — it is a prescribed component of the session with its own intensity, duration, and purpose. Getting recovery wrong changes the stimulus as much as getting the effort wrong.

#### Recovery Character

Recovery is either **active** or **passive**.

Active recovery is pedalling at Z1 — enough movement to maintain circulation and accelerate lactate clearance without adding meaningful physiological stress. This is the default for most quality sessions. The body is still working; it is just working at a rate that allows the systems stressed by the effort block to partially restore.

Passive recovery is complete rest — no pedalling. Reserved for neuromuscular sessions where phosphocreatine replenishment is the priority and even light pedalling would delay it, and for sessions where the effort blocks are so demanding that any additional output would compromise the next effort.

#### Recovery Duration

Recovery duration is prescribed relative to the effort block and the zone being trained. The relationship is not arbitrary — it determines whether the next effort block will be performed on partially-recovered or fully-recovered systems, and that distinction changes the stimulus entirely.

**Short recovery (work:rest ratio of 1:1 or less)** — the athlete begins the next effort block before full recovery. Lactate remains elevated, cardiovascular demand stays high. Accumulates fatigue across the set. Used in on-offs and float sets where sustained elevation near a ceiling is the target.

**Moderate recovery (1:1 to 1:2)** — partial recovery. The athlete can reproduce the effort but under progressively accumulating fatigue. The standard for threshold and sweet spot intervals where repeatability under moderate fatigue is part of the adaptation.

**Long recovery (1:3 or greater)** — near-full recovery between efforts. Each block is performed as fresh as possible. Used for neuromuscular and anaerobic sessions where quality of each individual effort is paramount and accumulated fatigue would compromise it. Also used for VO2 intervals where the effort duration is long enough that insufficient recovery would cause the athlete to fall below the target zone.

#### Incomplete Recovery as Stimulus Corruption

If recovery is cut short — by poor session design, by the athlete skipping recovery time, or by external factors — the subsequent effort block is not the same session. An athlete completing Z5 intervals on insufficient recovery is either working at a lower zone than prescribed or accumulating fatigue at a rate that will cause the set to collapse. The stimulus is corrupted regardless of what the power numbers show.

The recovery interval is part of the prescription. It is not negotiable.

---

### 1.3 Session Shape

A session has four components. Each exists for a reason. Skipping any of them changes what the session does.

#### Warm-Up

Elevates heart rate, increases muscle temperature, primes the cardiovascular and neuromuscular systems for effort. Begins at Z1 and builds gradually into the lower end of the day's target zone. Duration scales with session intensity — a neuromuscular or anaerobic session requires a longer, more thorough warm-up than an endurance ride because the systems being recruited are more sensitive to cold activation.

A warm-up that is too short means the first effort block is performed on unprepared systems. The athlete may hit the numbers but the physiological cost is higher than it should be and injury risk is elevated.

#### Settle

A brief period between the warm-up and the first effort block. The athlete has elevated heart rate and is physiologically prepared, but the settle allows them to find their position, take on nutrition or water, locate appropriate terrain, and make the mental transition to the work ahead. The warm-up should not run directly into the first hard effort.

The settle is not additional warm-up — it is transition time. It is short (typically two to five minutes) and at an easy intensity. Sessions without a structured main set (endurance, recovery) do not need a formal settle.

#### Main Set

The prescription itself — the effort blocks and recovery intervals that constitute the session's stimulus. Everything before it is preparation; everything after it is conclusion. The main set is what advances the ladder, generates the TSS, and produces the adaptation.

Session type determines main set character. Quality sessions have a defined main set of effort blocks and recoveries. Endurance sessions have a sustained target zone for the bulk of the ride. Recovery sessions have no main set in the quality sense — the entire ride is at Z1.

#### Cool-Down

Returns the body to resting state gradually. Clears residual lactate, reduces heart rate, begins the recovery process. Skipping the cool-down does not save meaningful time and delays recovery onset. Duration is proportional to session intensity — a hard quality session warrants a longer cool-down than a tempo ride.

#### Shape by Session Type

| Session type | Warm-up | Settle | Main set | Cool-down |
|---|---|---|---|---|
| Quality (intervals, structured) | Full, zone-building | Yes | Defined effort blocks + recovery | Full |
| Endurance | Short, easy | No | Sustained Z2/Z3 | Short |
| Recovery | Minimal | No | Entire ride at Z1 | Minimal |
| Neuromuscular | Long, thorough | Yes | Short maximal efforts, full recovery | Moderate |

---

### 1.4 Session Load

#### TSS as a Load Metric

Training Stress Score (TSS) is the standard currency for quantifying session load. It is calculated from intensity factor (IF, the ratio of normalised power to FTP) and duration, producing a single number that represents the physiological cost of a session relative to a maximal one-hour effort. A perfect one-hour effort at FTP produces 100 TSS. Sessions are compared, summed, and tracked on this scale.

TSS is useful precisely because it is simple — it allows load to be aggregated across days and weeks into the performance management chart metrics of ATL (acute training load, fatigue), CTL (chronic training load, fitness), and TSB (training stress balance, form). These are legitimate and valuable tools for managing volume and timing.

#### The Fundamental Limitation

**Not all TSS is created equal.**

TSS measures how much was spent. It says nothing about what was bought. Two sessions with identical TSS scores can produce completely different fatigue profiles, require different recovery timelines, drive different chronic adaptations, and leave the athlete in entirely different physiological states. The performance manager chart treats them as equivalent. They are not.

This is not a flaw to work around — it is a structural limitation to understand and account for explicitly. A coaching system that plans and evaluates sessions on TSS alone is missing the most important dimension of the prescription.

#### The Dimensions TSS Flattens

**Fatigue type.** Fatigue is not a single thing. Metabolic fatigue (glycogen depletion, cardiovascular stress) recovers differently from neuromuscular fatigue (CNS and motor unit depletion from high-force or high-velocity efforts) and from muscular damage (structural stress from sprints, big gear work, repeated high-force contractions). A 100 TSS endurance ride is primarily metabolic and cardiovascular — CNS largely intact, muscular damage minimal. A sprint session of equivalent TSS is dominated by neuromuscular and CNS fatigue — glycogen relatively spared, but the nervous system is genuinely taxed and high-force muscular stress is significant. Recovery requirements and timelines are completely different despite identical TSS.

**Substrate cost.** Endurance work draws on a mix of fat and glycogen. Anaerobic and sprint work is almost exclusively glycogen-dependent. Sessions with the same TSS can carry very different refuelling requirements, which directly affects readiness for the next session independent of the load number.

**Structural stress.** High-force efforts — sprints, standing accelerations, big gear work — create muscular damage that pure cardiovascular load does not. That damage requires repair time that TSS does not capture. An athlete can be cardiovascularly recovered from a sprint session while still carrying meaningful muscular fatigue.

**Adaptation signal.** CTL built from endurance miles and CTL built from threshold intervals represent different kinds of fitness. They are not interchangeable on race day, and a training plan that treats them as equivalent will produce an athlete with a misleading fitness number and gaps in actual race readiness.

#### How This System Accounts for It

TSS and the performance manager chart remain valid tools for managing overall load trajectory, acute fatigue, and timing of form peaks. This system does not discard them.

What this system adds is stimulus tagging. Every session carries not just a TSS value but a record of what kind of stress it produced — the zone, the structural pattern, the fatigue type, and the primary adaptation being targeted. The ladder system tracks progression by stimulus type, not by load accumulation. Two sessions of identical TSS but different stimuli advance different ladders, trigger different recovery expectations, and produce different readiness profiles for subsequent sessions.

The performance manager chart answers *how much have you done*. The stimulus record answers *what have you actually trained*. Both questions matter. Neither alone is sufficient.

### 1.5 Intensity Expression

Intensity is the most important variable in a prescription and the one most prone to imprecision. This system uses a hierarchy of expression modes, from most to least precise, with each mode appropriate in different contexts.

#### Absolute Watts

The most precise expression. Used when the specific number matters — test efforts, race simulation, or when the athlete has a known target power that a zone descriptor would approximate poorly. Absolute watts are appropriate when the prescription is *this number*, not *this physiological state*.

Limitation: absolute watts are only meaningful relative to the athlete's current fitness. A 300-watt threshold prescription is easy for one athlete and impossible for another. Absolute watts require the prescription to be athlete-specific rather than system-generic.

#### Percentage of FTP

The standard translation layer between zone-relative intent and platform-specific prescription. FTP percentage is how devices, training platforms, and workout builders receive intensity — it is the lingua franca of structured training software.

This is an expression tool, not a definition. The zone model is defined by physiology; FTP percentages approximate where that physiology occurs for a trained athlete. The ontology uses zone-relative language throughout and defers to the prescription builder to translate into FTP percentages when needed.

Limitation: FTP is itself a model, not a physiological constant. It drifts with fitness, varies by testing protocol, and is subject to individual variation in how cleanly it maps to LT2. FTP percentage ranges are approximations and should be treated as such.

#### Zone-Relative with Descriptor

The primary expression language of this document. A zone combined with a positional descriptor communicates physiological intent clearly without over-specifying a number. "High Z3" and "low Z4" are meaningfully different instructions that a coach would use naturally and an athlete would understand intuitively.

Position descriptors: *low*, *mid*, *high*, and *into* for zone boundaries (e.g. "high tempo into sweet spot"). These cover the full range of intra-zone nuance without requiring a specific percentage.

This is the right mode when the intent is a physiological state, not a number.

#### RPE

Rate of perceived exertion. The subjective experience of effort on a scale of 1–10. RPE is not a primary prescription tool in this system but it serves two important functions.

First, it is the **sanity check** against power data. A threshold session that feels like a Z2 effort, or a Z2 ride that feels like threshold, is a signal that something is wrong — FTP is outdated, fatigue is higher than acknowledged, or the athlete is misjudging effort. RPE catches what power data misses.

Second, it is the **fallback** when power data is unavailable or unreliable — outdoors in variable conditions, on climbs where power meters behave inconsistently, or during early base phases when rigid power targets are less important than time-in-zone feel.

RPE is also the primary language of the feedback layer — how the effort *felt* relative to how it *should* have felt is the core signal for rung advancement decisions.

#### Hierarchy Summary

| Mode | Primary use | Limitation |
|---|---|---|
| Absolute watts | Test efforts, race simulation | Athlete-specific, requires current FTP |
| % FTP | Platform/device prescription | Approximation, FTP drift |
| Zone + descriptor | Ontology and coaching language | Requires zone model understanding |
| RPE | Sanity check, feedback, fallback | Subjective, affected by fatigue and motivation |

### 1.6 Structural Patterns

A structural pattern defines the shape of the main set — how effort blocks are arranged, how intensity moves within the session, and what the work/recovery relationship looks like. Patterns are zone-agnostic: the same pattern applied at different zones produces different stimuli. The pattern is the architecture; the zone is the material.

#### Sweet Spot as an Intensity Range

Before defining patterns, one clarification on intensity: sweet spot is not a zone in this system. It is a named intensity range sitting between Z3 and Z4 — roughly the upper quarter of tempo through the lower portion of threshold, approximately 88–94% FTP. It is a deliberate training strategy that sits just below LT2 to maximise training load per unit of freshness cost. It appears as a target intensity within patterns, not as a zone that owns its own category.

---

#### Sustained Power

A single continuous effort held at target intensity for the duration of the main set. No interruption, no variation. The simplest structural pattern and the most demanding test of pacing discipline — there is nowhere to hide if intensity drifts. Used where the adaptation being sought requires prolonged time-in-zone without the recovery relief of intervals.

*Zones: Z2, Z3, Sweet Spot, Z4*

---

#### Intervals

Repeated effort blocks at target intensity separated by defined recovery periods. The canonical quality session structure. Recovery duration and character (active vs passive) are part of the prescription and directly affect the stimulus — shorter recoveries accumulate fatigue across the set; longer recoveries preserve the quality of each individual effort. The right recovery duration depends on what the session is training.

*Zones: Z3, Sweet Spot, Z4, Z5, Z6*

---

#### Hard Starts

The effort begins significantly above the target zone intensity then settles into it for the remainder of the block. The opening surge recruits fast-twitch fibres and elevates lactate before the steady-state portion begins, meaning the sustained effort is performed under conditions of already-elevated metabolic stress. Trains the ability to find rhythm after an initial hard effort — a common race scenario. The hard start is prescribed, not incidental.

*Zones: Sweet Spot, Z4, Z5*

---

#### Over-Unders

Alternates within a single effort block between two intensities: one just below the target zone ceiling and one just above it. The athlete repeatedly crosses the threshold between clearance and accumulation, training the metabolic machinery to handle that transition and recover from brief excursions above it without falling apart. The classic tool for training at and around LT2. Duration of the under and over phases are both prescribed.

*Zones: Sweet Spot, Z4, Z5*

---

#### Mixed Intervals

A session containing intervals at two distinct intensity levels within the same main set. Unlike over-unders, the efforts are discrete rather than continuous alternation — each interval is its own block. Used to combine stimuli in a single session, or to simulate the variable demands of a race. The pairing matters: the harder intervals should not so compromise the athlete that the secondary set becomes junk volume.

*Zones: Sweet Spot, Z4, Z5, Z6*

---

#### Ramps

Intensity increases progressively across the main set, either continuously or in discrete steps. The athlete is never at a fixed steady state — the session builds. Used to train the ability to lift intensity as fatigue accumulates, and to expose the athlete to a range of intensities within a single session. Ramps are also diagnostic: where the athlete drops off reveals their current ceiling.

*Zones: Z4, Z5*

---

#### Attacks

A compound two-phase pattern simulating a race move. Phase 1 is a short explosive effort — maximal or near-maximal, Zone 6/7 — to simulate the accelerating move that creates a gap. Phase 2 follows immediately: a sustained effort at or above threshold to simulate holding the gap under already-depleted conditions. The physiological demand is the combination — producing threshold-plus power on exhausted phosphocreatine with lactate already elevated from Phase 1. Neither phase in isolation replicates this stress.

*Zones: Z6 into Z4/Z5*

---

#### On-Offs

Very short alternating work and rest intervals — typically 15–30 seconds on, 15 seconds off — that accumulate time near a physiological ceiling at a lower per-rep perceived cost than longer continuous efforts. The Ronnestad protocol (30s/15s) is the canonical example. The short recovery keeps the system elevated; the brief effort avoids the deep per-rep cost of a full long interval. Used when the athlete needs the ceiling stimulus but the longer format is mentally or physically too costly.

*Zones: Z5, Z6*

---

#### Float Sets

Intervals where the recovery period is not full rest but a reduced-intensity "float" — enough to briefly lower demand without allowing full recovery. The system remains elevated throughout. Used at VO2 intensity to extend total time near the aerobic ceiling within a session. More demanding than standard intervals with full recovery; the float is prescribed as a specific intensity, not just "easy."

*Zones: Z5*

---

#### Traditional

The canonical long VO2max interval format. Five efforts of five minutes at VO2max intensity with defined recovery between. Long enough per effort to fully stress the aerobic ceiling; short enough to repeat. The benchmark quality session for aerobic power development. Named explicitly because it is a well-understood reference point for both athlete and coach.

*Zones: Z5*

---

#### With Bursts

An endurance-paced ride with short neuromuscular efforts embedded. The base ride is the session; the bursts are 10–15 second maximal sprints that activate the ATP-PC pathway and reinforce neuromuscular recruitment patterns without meaningfully disrupting the endurance stimulus. Equivalent to strides in running. The bursts are not the physiological target — the aerobic base ride is. The bursts maintain the top end during a base phase.

*Zones: Z2 base with Z7 bursts*

---

#### Steps

Intensity increases in discrete blocks within a single continuous effort, each step held for a defined duration before moving to the next. Anaerobic-specific. Each step takes the athlete deeper into glycolytic demand. Used to train progressive lactate accumulation and the ability to sustain power as metabolic conditions deteriorate. Distinct from ramps in that each step is a fixed intensity held for a defined period, not a continuous climb.

*Zones: Z6*

### 1.7 Extensive vs Intensive Progression

Every workout can be progressed in one of two directions. Understanding which direction is appropriate at any given time is one of the most important prescription decisions a coach makes.

#### The Two Modes

**Extensive progression** — the target intensity is held constant; the duration of the effort block increases. The athlete can already produce the required power at this intensity. The adaptation being sought is the ability to sustain it for longer — fatigue resistance, metabolic efficiency, and substrate utilisation at that intensity. The power curve extends rightward. The ceiling does not move but the floor drops.

**Intensive progression** — the duration is held constant; the target intensity increases. The athlete can already sustain efforts of this length. The adaptation being sought is the ability to produce more power across that duration. The power curve shifts upward. The ceiling moves but the duration runway stays the same.

These are not interchangeable. Choosing the wrong mode produces the wrong adaptation and — in the case of intensive progression applied too early — accumulates fatigue faster than the athlete can absorb it.

#### Why the Ladder System is Primarily Extensive

The rung system progresses by adding time-in-zone at a fixed intensity. This is a deliberate choice grounded in the extensive model. Extensive progression is more sustainable across a training block, carries a lower recovery cost per session, and builds the aerobic and metabolic base that intensive progression later relies on. An athlete cannot meaningfully push the ceiling up without first having a deep base at the current ceiling.

Intensive progression has its place — primarily in peak phase, when the athlete has accumulated sufficient extensive base and needs to push power upward before a target event. At that point the ladder transitions from adding duration to adding intensity, and the recovery cost per session rises accordingly.

#### The Power Curve as Diagnostic

The athlete's power curve — the best power output across all durations from seconds to hours — reveals the shape of their current fitness and indicates which progression mode is the priority.

A curve that drops off sharply at longer durations indicates that extensive progression is the priority. The athlete can produce the power but cannot sustain it. Adding duration at current intensity will develop the missing fatigue resistance.

A curve that is flat but lower than expected across all durations indicates that intensive progression is needed. The ceiling itself is the limiter, not the endurance of that ceiling.

A curve with a specific gap — strong at short durations, weak at a particular duration range — indicates targeted extensive work in that range. The athlete's phenotype shapes the prescription.

#### Phenotype and Curve Shape

The power curve also reveals the athlete's phenotype — whether they are naturally better suited to short explosive efforts or long sustained ones. This shapes both the limiter identification and the progression priority:

A **punchy, anaerobic phenotype** — strong at short durations, curve drops steeply — typically needs extensive progression at longer durations to develop the aerobic endurance their phenotype undersupplies.

A **diesel, aerobic phenotype** — strong at long durations, curve relatively flat — typically needs intensive progression at shorter durations to develop the top-end power their phenotype undersupplies.

Neither phenotype is prescribed away entirely. The goal is to reduce the gap between the athlete's strengths and their limiters relative to their target race demands, not to turn a sprinter into a climber.

#### Durability and the Fatigued Power Curve

The fresh power curve describes what the athlete can do rested. It says little about what they can do after three hours of riding — which, for most road and endurance events, is precisely what decides the result. Durability is the degree to which the power curve holds its shape under accumulated fatigue: the gap between fresh best power and best power produced after a defined energy expenditure (for example, best 5-minute power after 2000 kJ of work).

This is a second diagnostic dimension sitting alongside the fresh curve, and intervals.icu exposes the data to measure it. Two athletes with identical fresh curves can have very different fatigued curves, and the one whose curve collapses late is the one who loses races in the final hour despite matching fitness on paper. Durability is trainable — primarily through aerobic base volume and through deliberately placing quality work on pre-fatigued legs (the fatigue resistance exception in section 2.4).

For prescription purposes, durability is read as a limiter in its own right: a curve that holds fresh but decays sharply when fatigued points to extensive aerobic work and fatigue-resistance sessions, regardless of where the fresh curve sits. The fresh curve sets the ceiling; the fatigued curve sets how much of that ceiling survives to the part of the race that matters.

#### Progression Mode as a Ladder Property

Each ladder in this system carries a progression mode — extensive or intensive — as a property. The mode determines whether advancing a rung means longer efforts at the same intensity or harder efforts of the same duration. This is set at the phase level: base and build phases favour extensive; peak phase may shift selected ladders to intensive. The mode is explicit, never assumed.

---

## Layer 2: Zone and Stimulus Mapping

*How workout primitives produce a specific physiological stimulus, and why.*

### 2.1 The Training Zones

#### Physiological Foundation

Training zones are not arbitrary bands on a power meter. They are expressions of which energy system is dominant and, critically, which system is being stressed toward its limits. The three systems always operate in parallel — the question is one of proportion, not switching.

**Aerobic (oxidative phosphorylation).** The primary energy system for sustained effort. Slow to ramp, but enormous in capacity. Draws on both fat and glycogen as substrate, with fat oxidation dominant at low intensities and glycogen contribution rising as intensity increases. The system that underpins everything else — mitochondrial density, fat oxidation efficiency, and cardiac output are all aerobic adaptations. It never switches off, but above a certain intensity it cannot meet demand fast enough on its own.

**Glycolytic (fast glycolysis).** Faster than aerobic, limited in capacity. Glycogen-only. Becomes a meaningful contributor when aerobic supply falls short of demand — roughly around threshold intensity. Produces lactate as a byproduct. At moderate production rates the aerobic system can consume that lactate as fuel; when production outstrips clearance, lactate accumulates and the effort becomes unsustainable. Threshold is precisely this equilibrium point — the highest intensity at which production and clearance are matched.

**Phosphocreatine / ATP-PC (alactic anaerobic).** Instantaneous power, trivial capacity. Fuels roughly the first 10–15 seconds of a maximal effort from stored creatine phosphate before it is exhausted. Replenishes fully during adequate recovery. No lactate produced. Pure neuromuscular power expression.

#### Zone Model

The zones below are derived from the energy system proportions above. FTP percentage ranges are the *expression language* for prescription — they are not the definition of the zone. The physiology defines the zone; the FTP percentage is an approximation of where that physiology occurs for a trained athlete.

| Zone | Name | Dominant System | FTP % (approx) | Physiological character |
|------|------|----------------|----------------|------------------------|
| Z1 | Recovery | Aerobic | < 55% | Trivial stress on any system. Lactate at resting baseline. Purpose is circulation and recovery, not adaptation. |
| Z2 | Endurance | Aerobic | 55–75% | Aerobic system stressed, fat oxidation working hard, glycolytic contribution minimal. The mitochondrial development zone. Sustainable for hours. Lactate low and stable. |
| Z3 | Tempo | Aerobic + Glycolytic | 76–87% | Aerobic still dominant but glycolytic now meaningfully contributing. Lactate elevated but clearing. Aerobically efficient athletes can sustain this for extended periods. Sits at or above the first lactate threshold (LT1) — the point where lactate first rises above baseline. |
| Z4 | Threshold | Glycolytic / Aerobic at ceiling | 88–100% | Lactate production and clearance approach equilibrium across this zone, with the equilibrium point — LT2, the second lactate threshold (MLSS) — sitting at the top, near FTP. Maximum sustainable glycolytic contribution. The aerobic system is working at a high fraction of its capacity. Sustainable for roughly 30–60 minutes in a trained athlete. |
| Z5 | VO2max | Aerobic at limit + Glycolytic high | 101–120% | Aerobic system at or near its absolute ceiling (VO2max). Glycolytic contribution high. Lactate accumulating. Only sustainable in intervals of several minutes with recovery between. The zone that raises the aerobic ceiling. |
| Z6 | Anaerobic Capacity | Glycolytic dominant | 121–150% | Glycolytic system dominant. Aerobic contribution limited by the short duration these intensities are sustainable. Lactate accumulating rapidly. Develops the ability to produce and tolerate high lactate — and to recover between such efforts. |
| Z7 | Neuromuscular | Phosphocreatine dominant | > 150% | Efforts too short for meaningful aerobic or glycolytic stress. Pure phosphocreatine and recruited muscle fibre expression. Develops peak power and neuromuscular recruitment patterns. Recovery between efforts must be long enough for phosphocreatine to replenish fully. |

#### The Boundary Zones

The Z2/Z3 boundary (LT1) and the LT2 line near the top of Z4 (around FTP) carry the most physiological significance and the most prescription risk. LT1 marks where lactate first rises above baseline and glycolysis becomes a meaningful contributor — it sits at the top of endurance, where Z2 gives way to tempo. LT2 marks the highest intensity at which lactate production and clearance stay matched — the maximal lactate steady state, which for a trained athlete falls close to FTP at the top of the threshold zone. Tempo (Z3) and sweet spot live in the band between these two thresholds. Work just below LT2 accumulates training stress efficiently without the deep fatigue of true threshold; work just above it is a different beast entirely. Confusing LT1 with LT2 — or drifting between them unintentionally — is one of the most common sources of grey-zone training where intensity is neither easy enough to be restorative nor hard enough to be productive.

The Z5/Z6 boundary is similarly important. Long VO2max intervals (3–5 minutes) stress the aerobic ceiling via sustained demand. Short anaerobic efforts (30–60 seconds) stress glycolytic capacity and lactate tolerance. Both sit "above threshold" but they are training different things.

### 2.2 Intensity Domain vs Stimulus Type

Two sessions can occupy the same intensity domain — sit at similar points on the power curve, generate similar TSS, feel broadly comparable in effort — and yet produce meaningfully different stimuli, drive different adaptations, and belong at different points in a training phase. Intensity domain describes *where* a session sits. Stimulus type describes *what it is actually training*. They are related but not the same thing.

This distinction is the reason the prescription system tracks stimulus type explicitly rather than inferring it from intensity alone.

#### Worked Example 1: Sweet Spot vs Threshold

Sweet spot and threshold occupy adjacent intensity domains — roughly 88–94% FTP and 95–105% FTP respectively. On a power file they look similar. On a performance manager chart their TSS contribution is comparable. An athlete moving between them may not notice a dramatic difference in any single session. Over a training block the difference is significant.

**Sweet spot** is a high-volume, single-system strategy. The slightly reduced intensity means the athlete recovers faster between sessions, allowing greater frequency and therefore more total time-in-zone across a block. The adaptation is primarily aerobic efficiency and lactate threshold development through accumulated volume. The aerobic ceiling — VO2max — is not meaningfully touched. This is why Frank Overton coined the term: it is the sweet spot between training stress and recovery cost, optimised for volume.

**Threshold** is a lower-volume, multi-system strategy. The intensity is high enough that the aerobic system is working at a sufficient fraction of its ceiling to produce a secondary VO2max stimulus alongside the primary threshold adaptation. The VO2max needle moves. The cost of this broader stimulus is duration — the athlete cannot sustain true threshold for as long as sweet spot — and recovery, which is substantially higher. Frequency must come down.

The prescription implication is that they are not interchangeable rungs on the same ladder. They are different ladders serving different purposes at different phases. Sweet spot belongs in base phase where volume accumulation is the goal. Threshold belongs in build phase where the athlete is deliberately reaching toward higher systems and accepting the recovery cost. An athlete whose threshold intervals have drifted into sweet spot territory without realising it has lost the multi-system stimulus entirely — the intensity difference is small enough that average power may not catch it, but RPE relative to expected will.

#### Worked Example 2: VO2max Traditional vs Micro-Burst Formats

Both the traditional long-interval format and micro-burst formats such as Ronnestad-style on-offs target the VO2max ceiling. The cardiovascular adaptation sought is the same. The mechanism, the peripheral experience, the psychological demand, and the appropriate prescription context are all different.

**Traditional VO2max intervals** — the canonical format is five efforts of five minutes at VO2max intensity with defined recovery between. Each effort is long enough to fully stress the aerobic ceiling and demand sustained output at that level. The per-rep cost is high. The session is genuinely taxing and most athletes approach a VO2max block with a degree of dread. That dread is not irrational — the format is hard, the efforts are long enough to hurt continuously, and the recovery between them is rarely sufficient to feel ready for the next one. Traditional intervals are the most direct route to the VO2max adaptation but they carry the highest psychological and peripheral cost.

**Micro-burst formats** exploit a physiological lag in the cardiovascular system. The heart and lungs do not drop out of the VO2max range during a recovery of fifteen to thirty seconds the way they would during a full three-minute recovery. The athlete gets peripheral relief — the legs partially recover, lactate briefly stabilises, the burning sensation eases — while the central cardiovascular system remains elevated near its ceiling. Time at VO2max stimulus accumulates without the continuous per-rep cost of sustaining it. On any individual rep the athlete is not experiencing the full demand of a traditional interval. Across the set they are receiving a comparable cardiovascular stimulus.

**The confidence bridge.** The micro-burst format does something the traditional format cannot: it gives the athlete a manageable entry point into VO2max work. A 15-second effort followed by 45 seconds of relief is not frightening. The athlete can complete it, build a reference point, and develop confidence in their ability to work at that ceiling. The progression from 15/45 through 30/30 to the classic 40/20 split is not only a physical progression — increasing the work interval and reducing recovery ratio — it is a psychological one. Each step is grounded in the completion of the previous one. By the time the athlete arrives at 40/20 they have accumulated hours of experience near the VO2max ceiling and they arrive at traditional intervals, when the time comes, as a different athlete.

**The bridging function in prescription.** For an athlete for whom traditional five-minute VO2max efforts are currently out of reach — whether through insufficient fitness, insufficient confidence, or both — the micro-burst format is not a compromise. It is a legitimate stimulus delivery mechanism that also constructs the pathway toward the traditional format. The ladder can explicitly route through this bridge: micro-burst formats as early rungs, progressive tightening of the work:rest ratio as middle rungs, traditional intervals as the upper rungs. The athlete does not wait until they are fit enough for the hard version. They build toward it through a format that is already working.

#### The General Principle

These two examples illustrate the same underlying principle. Intensity domain sets the physiological neighbourhood. Stimulus type — determined by session structure, duration, recovery ratio, and pattern — determines what is actually being trained within that neighbourhood. Two sessions in the same neighbourhood with different structures are different prescriptions driving different adaptations. The prescription system must track both.

Where intensity domain and stimulus type diverge most sharply is at the zone boundaries — the Z3/Z4 boundary where sweet spot and threshold blur, and the Z5/Z6 boundary where VO2max long intervals and anaerobic capacity work sit close together on the power curve but train fundamentally different systems. These are the highest-risk zones for prescription drift and the most important to define precisely.

### 2.3 Stimulus Specificity

A session is specific to its intended stimulus when the athlete holds the prescribed intensity for the prescribed duration with appropriate recovery between efforts. Stimulus specificity is not a binary — sessions exist on a spectrum from fully intact to fully collapsed — but for progression purposes the system must make a judgement about whether the intended stimulus was achieved.

The most important principle: **a session that looks correct on average can still be stimulus-compromised in detail.** Average power is an unreliable judge of session quality. The signals that matter are in the shape of the effort, not the summary number.

#### How Sessions Lose Their Stimulus

**Zone drift — fatigue origin.** The most common failure mode. Power progressively falls below the target zone as the session continues, not through poor pacing but through genuine fatigue accumulating faster than the prescription anticipated. The athlete started correctly and faded. Heart rate decoupling is the key signal — power dropping while heart rate remains elevated or continues climbing indicates the cardiovascular system is still working hard but the muscles can no longer respond. This is a load management signal, not a pacing failure. The appropriate response is to assess whether the ladder rung was too ambitious or whether external fatigue was carried into the session.

**Hero watts — pacing origin.** The athlete feels strong early and exceeds the prescribed intensity in the opening efforts or intervals. The session degrades progressively as a result — later intervals fall short of target, power fades, the athlete is hanging on where they should be performing. The stimulus of the session collapses from the middle outward. This is a pacing failure with a specific cultural dimension: the athlete's instinct to capitalise on good form is not wrong, but acting on it by going harder than prescribed is. The correct response to feeling strong is to follow the plan and extend it at the end if capacity remains. Front-loading the effort robs the session of its integrity. This principle should be communicated to the athlete directly and reinforced consistently.

**Short recovery with targets met — progression signal.** The athlete cuts recovery intervals but continues to hit power targets across the full set. This is not a failure — it is positive data indicating the athlete has adapted beyond the current rung. The recovery that was prescribed is no longer needed to produce the next effort. This is one of the clearest signals that rung advancement is appropriate, and potentially that the ladder should consider both extending duration and tightening recovery as progression axes.

**Short recovery with targets missed — pacing origin.** Superficially similar to the above but the opposite meaning. The athlete cuts recovery and then fails to hit power targets. The reduced recovery has corrupted the subsequent efforts. This is a pacing failure: the athlete underestimated the recovery requirement and paid for it in interval quality. The prescription was correct; the execution was not. The appropriate response is to bring this back to the athlete, explain the relationship between recovery duration and effort quality, and reinforce that recovery is part of the prescription.

**Sudden cessation — welfare signal.** An interval that stops abruptly mid-effort, particularly if followed by continuation at significantly reduced intensity, steps outside normal prescription feedback into athlete welfare territory. Possible causes span a wide range: mechanical, physical, motivational, or something more serious. The system flags this for direct conversation with the athlete. If the session continues at reduced output the most likely cause is motivational — the athlete reached a wall and backed off rather than through it. If the session ends entirely the cause needs establishing before the next prescription is made.

#### Reading Intervals in Isolation

When evaluating session quality the intervals must be examined individually, not just in aggregate. The signals to look for within a single effort block:

**Consistency** is the primary marker of a well-executed interval. Power held steadily across the duration indicates the athlete found and maintained the target zone. Minor variation is normal and expected, particularly outdoors.

**Cadence stability** — unless a specific cadence target is prescribed, cadence should remain broadly stable across the effort. A progressive cadence drop within an interval indicates the athlete is losing neuromuscular freshness and beginning to grind. Not necessarily a failure, but a signal worth noting.

**Heart rate behaviour** — heart rate will naturally rise during a work interval. A spike that significantly exceeds the expected range for the zone suggests the effort was harder than the power number indicates, either due to external factors (heat, fatigue, illness) or because the intensity was genuinely above prescription. A heart rate that decouples from power — rising while power falls — indicates the aerobic system is under stress the muscular system can no longer match.

**End-of-interval fade** — a slow drift in power or cadence toward the final moments of an effort indicates an athlete near their limit. This is not inherently a failure. Going to the well — genuinely emptying the tank in the final portion of a hard interval — is productive struggle. The athlete held on, the stimulus was delivered, the adaptation signal is strong. The distinction between productive struggle and stimulus collapse is whether the target zone was broadly maintained throughout or whether the final portion fell significantly below it.

#### The Grey Zone Problem

The grey zone is the space between genuinely easy and genuinely hard — intensity that is too high to be restorative and too low to be productive. Accumulated grey zone training is one of the most effective ways to accumulate fatigue without driving meaningful adaptation. It feels like work. It generates TSS. It produces ATL. It does not reliably improve performance.

The grey zone is entered through three routes:

**Endurance rides drifting upward.** An athlete who rides their Z2 sessions at the top of Z3 is not doing extra work — they are doing the wrong work. The adaptation from Z2 comes from sustained aerobic stimulus at an intensity the body can absorb and repeat daily. Pushing into Z3 adds fatigue without meaningfully adding adaptation, and it compromises the quality of the next day's harder session. Hard days must be hard; easy days must be easy. An endurance ride that creeps into tempo is neither.

**Quality sessions drifting downward.** A threshold session where the athlete cannot hold the target zone and settles into sweet spot for the bulk of the set is not a threshold session. The multi-system stimulus is gone. The session has generated fatigue without delivering the intended adaptation. This is a rung that was too ambitious, or fatigue carried in from prior days.

**Unplanned accumulation.** Sometimes the grey zone is entered not through any single session failure but through a week where every session was slightly harder than easy and slightly softer than quality. No individual session was wrong, but the cumulative effect is a fatigued athlete who has done a lot of moderately hard riding and not enough of anything specific. The performance manager chart may look fine. The athlete is not progressing.

The prescription system guards against the grey zone by being explicit about intent. Every session has a defined zone and a defined purpose. An endurance session is Z2 — not "easy-ish." A threshold session is Z4 — not "hard but not too hard." The specificity is the protection.

#### Scoring a Session from the File

The principles above describe how sessions lose their stimulus. This subsection makes the classification operational: given a completed activity file (power, heart rate, cadence per interval, of the kind intervals.icu exposes), how does the system decide whether the session was on target, below target, or interrupted? The advancement logic in section 4.3 depends on this verdict, so it must be computed consistently rather than inferred loosely.

The numbers below are starting thresholds, not constants. They are the kind of values the framework's closing principle treats as tunable against observed reality.

**Step 1 — identify the prescribed work.** Every prescription carries its target zone (a %FTP band), its structural pattern, the number and duration of work intervals, and the recovery between them. Scoring compares the file against this, not against session averages.

**Step 2 — score each work interval.** A single work interval passes when both hold:
- Its average power for the interval falls within the target band, or above the band's lower edge. Brief overshoot is fine; the concern is falling short.
- It did not drift more than ~5% below the band's lower edge for more than a brief portion (roughly the final 10% of the interval or 15 continuous seconds, whichever is longer). A block that starts in zone and collapses fails even if its average is rescued by a strong opening — this is the average-power trap from the start of this section.

**Step 3 — score the session.**
- **On target / completed:** all prescribed work intervals pass, or all but one pass with the miss being marginal (within ~2% of the band edge). Full duration held.
- **Below target:** more than one interval fails, or a sustained block fell out of zone for a meaningful stretch, or the work was curtailed early through fading rather than a discrete stop. This routes to the escalation protocol.
- **Interrupted:** a discrete stop mid-effort, distinct from a progressive fade. Routes to the session interruption protocol in section 3.3, and cause is established before any ladder response.

**Step 4 — apply the shape signals on top of the pass/fail.** Average power passing is necessary but not sufficient; the shape of the effort refines the verdict and feeds the RPE comparison:
- **Power-to-HR decoupling.** Compute decoupling across the work portion (the drift between the power:HR ratio in the first half versus the second). Decoupling beyond roughly 5% indicates fatigue-origin drift even where average power passed — flag it as an at-the-limit or fatigue signal rather than a clean on-target.
- **Inter-interval trend.** Power holding across the set is on-target; a progressive decline across otherwise-passing intervals is the "completed with progression fade" outcome from section 3.3, scored as completed but flagged for cause.
- **Recovery adherence.** If recovery intervals were cut but targets still met, this is a positive progression signal (advance and consider tightening recovery). If recovery was cut and later targets missed, the miss is attributed to pacing, not to an over-ambitious rung.

**For sustained-power and endurance sessions** there are no discrete intervals to score, so the test is time-in-band: the proportion of the prescribed block held within the target zone. On target is roughly 90%+ of the work duration in band for a sustained quality block. For an endurance ride, the grey-zone guard inverts the test — flag the session if more than a small proportion of time crept above the Z2 ceiling into Z3, since for endurance, too hard is the failure mode, not too easy.

**Heart-rate-only fallback.** Where power is unavailable or unreliable (outdoor variability, power meter dropout, climbs), the same logic runs against heart-rate zones and RPE instead, with the understanding that HR lags power and is a coarser instrument — the verdict is held more loosely and RPE carries more weight, consistent with the expression hierarchy in section 1.5.

### 2.4 Stimulus Interactions

Stimuli interact — within a session, across a day, and across a week. Getting these interactions right is what separates a coherent training plan from a collection of hard sessions. Getting them wrong accumulates fatigue without accumulating adaptation.

#### One Session, One Stimulus

Every session targets a single zone or adaptation. This is not a limitation — it is a design principle.

A session that attempts to train VO2max, threshold, and neuromuscular power in the same main set is not a comprehensive session. It is a confused one. The body does not respond to multiple simultaneous stimuli by adapting to all of them — it responds by managing the competing demands at the cost of each individual signal. The adaptation from each system is diluted. The fatigue from all of them is not.

Kitchen sink sessions also fail the athlete practically. A session with a clear single purpose is executable and evaluable. The athlete knows what they are doing, they know what success looks like, and the feedback is clean. A session targeting three things simultaneously gives the athlete nothing clear to aim at and the system nothing clean to evaluate.

The prescription system enforces this constraint by design. Every session has one primary zone, one structural pattern, one adaptation target. Complexity lives in the plan across weeks, not within a single session.

#### Weekly Sequencing

Within a training week, sessions should be sequenced to give the hardest stimuli the freshest physiological state. The principle is progressive fatigue management — quality sessions are front-loaded into the week, with easier sessions following as the week accumulates fatigue.

A standard quality week might place VO2max work early — Tuesday, after a rest or easy day following the weekend — and threshold work mid-week when some fatigue has accumulated but quality is still achievable. Endurance and recovery sessions fill the remaining days, allowing the week to recover into the weekend. The week breathes: hard, moderate, easy, hard, moderate, easy.

The sequencing also respects the recovery requirement between hard sessions. Two maximal quality sessions on consecutive days without adequate recovery between them is not progressive overload — it is accumulated damage. The second session is performed on a compromised system and the stimulus is corrupted regardless of what the power numbers show.

**The fatigue resistance exception.** Sometimes the prescription deliberately applies load to a fatigued body. Fatigue resistance — the ability to produce power when already tired, which is a direct race demand for most events — is a trainable quality. Back-to-back hard days, or a quality session following a race effort, are legitimate tools in a build or peak phase. The distinction is that this is a deliberate prescription choice with a specific adaptation target, not an accident of poor scheduling. The system must flag it as intentional and monitor the response carefully.

#### Cross-System Fatigue Profiles

Different stimuli produce different fatigue profiles with different recovery timelines. This determines which sessions can coexist in a week and in what order.

Neuromuscular sessions — sprints, attacks — produce high CNS and muscular fatigue with relatively low metabolic cost. The legs may feel fine metabolically while being genuinely neuromuscularly depleted. A threshold session the day after a sprint session may feel aerobically manageable but neuromuscularly compromised — the athlete cannot recruit cleanly and power suffers.

VO2max sessions produce high cardiovascular and metabolic fatigue with significant glycogen cost. Recovery typically requires 48 hours minimum before another hard quality session. Placing a VO2 session too close to a threshold session in either direction risks one corrupting the other.

Endurance sessions at true Z2 produce low acute fatigue with high chronic adaptation value. They can sit adjacent to quality sessions without meaningful interference — this is precisely their value in a polarised training structure. An endurance session the day before a threshold block does not compromise the threshold stimulus if the endurance session was genuinely easy.

#### Overtraining, Burnout, and Under-Recovery

Athlete burnout is discussed frequently and feared disproportionately. True overtraining syndrome — the clinical condition of sustained performance decrements that persist through weeks of rest — is rare and requires sustained months of genuine overload to develop. What most athletes experience is under-recovery: insufficient sleep, inadequate nutrition, life stress layering on top of training stress until the system cannot absorb the load being applied.

The distinction matters for the prescription system. Overtraining requires a fundamental restructuring of the training approach. Under-recovery requires identifying and addressing the recovery deficit — sleep, nutrition, life stress reduction — while moderating load temporarily. Treating under-recovery as overtraining produces unnecessary detraining. Treating overtraining as under-recovery and pushing through produces genuine harm.

Tiredness and achy legs are training signals, not warning signs. They are the expected acute response to a correctly applied load. The system monitors them as part of the wellness picture but does not treat them as indicators of overreach unless they persist and accumulate.

The monitoring layer — HRV, sleep quality, wellness scores, mood, motivation — provides the early warning signal that load and recovery are out of balance before performance decrements appear. A single bad night is noise. A week of declining HRV, poor sleep, and flat affect is signal. The prescription system responds to the pattern, not the individual data point.

#### Training Age and the Mental Success Principle

Periodisation models typically address the physiological dimension of athlete development and treat the psychological dimension as implicit. It is not implicit — it is foundational, particularly with newer athletes.

A new athlete's first adaptation is not physiological. It is psychological: the habit of training, the confidence that hard sessions are survivable, the belief that the process works. An athlete who has never completed a structured interval session does not know they can. That uncertainty costs them performance before the first effort block begins.

Early in a new athlete's development, the prescription should lean toward achievable sessions that build the reference points confidence requires. This is not softness — it is sequencing. An athlete who has completed twenty sessions and knows what they can do will perform differently in session twenty-one than one who arrives at every session uncertain. The physiological adaptation and the psychological adaptation are both real and both necessary. The plan should develop both.

As training age increases and confidence is established, the prescription can progressively challenge the athlete closer to their actual limits. The mental success principle does not disappear — it shifts. An experienced athlete's psychological challenge is different: trusting the process through a hard block, holding back on easy days when they feel strong, accepting that fitness sometimes feels worse before it feels better. The principle adapts to the athlete's stage, but it never stops being relevant.

*"Trust the process"* — the plan is progressive by design. Disrupting it, whether by going too hard on easy days, skipping recovery, or abandoning a block that feels hard, disrupts the adaptation the plan was building. The process works when it is followed. The coach's job is to build a plan worth trusting and to help the athlete trust it.

---

## Layer 3: Expected Response and Adaptation

*Given a stimulus, what does the body do — acutely and chronically — and what does a successful session look like.*

### 3.1 Acute Responses

The acute response is what the body does in the hours immediately following a session. Understanding it determines recovery requirements, informs readiness assessment for the next session, and explains why two sessions of identical TSS can leave the athlete in completely different physiological states.

Acute responses operate across four dimensions simultaneously. The proportion of each depends on the stimulus type.

#### Glycogen Depletion

Glycogen — stored carbohydrate in muscle and liver — is the primary fuel for moderate to high intensity work. Its depletion after a session is proportional to both intensity and duration, but intensity has a disproportionate effect: high-intensity work burns glycogen faster than the aerobic system can process fat, so anaerobic and VO2max sessions deplete glycogen rapidly even at shorter durations.

A depleted athlete is not simply tired — they are genuinely fuel-limited. Training on low glycogen produces a different stimulus than training fuelled, and attempting quality work in a significantly glycogen-depleted state compromises both performance and adaptation. Refuelling in the window following a hard session is not optional recovery hygiene — it is part of the prescription.

Glycogen replenishment takes 24 hours with adequate carbohydrate intake, longer if intake is insufficient. This is one of the primary reasons hard sessions cannot be stacked daily without nutritional support.

*Note: individual variation in glycogen storage capacity and utilisation rate is significant. Athletes with greater aerobic development burn a higher proportion of fat at given intensities, sparing glycogen. This is one of the chronic adaptations of Z2 training and one reason aerobic base work improves performance at all intensities.*

#### Muscular Damage and Structural Stress

High-force efforts — sprints, standing accelerations, big gear work, repeated hard accelerations — produce structural damage to muscle fibres. This is not injury; it is the mechanical disruption that triggers the repair and reinforcement response that underlies strength and power adaptation. It is also the source of the deep muscular soreness that follows neuromuscular sessions.

Muscular damage has a longer recovery timeline than metabolic fatigue. An athlete can be cardiovascularly and metabolically recovered from a sprint session — heart rate back to normal, glycogen replenished — while still carrying significant muscular damage that will compromise force production in a subsequent hard session. This is why neuromuscular sessions need more recovery time than their TSS suggests.

Threshold and VO2max sessions produce less muscular damage than neuromuscular work but are not damage-free. Sustained high-force pedalling at threshold accumulates meaningful muscular stress over a long session.

#### Cardiovascular and Metabolic Stress

Prolonged aerobic work elevates cardiovascular demand — heart rate, stroke volume, cardiac output — for the duration of the session. The recovery from this stress is primarily overnight: heart rate variability (HRV) the following morning is one of the best available proxies for how completely the cardiovascular system has recovered.

VO2max sessions produce the highest acute cardiovascular stress of any training stimulus. The aerobic system is working at its ceiling for extended periods and the recovery demand reflects this. Threshold sessions produce significant but lower cardiovascular stress; the system is working hard but not at its absolute limit.

Metabolic stress — lactate accumulation, acid-base disruption, oxidative stress — is highest in anaerobic and VO2max work where the glycolytic system is working at high rates. The body clears lactate relatively quickly (within an hour of session end) but the downstream metabolic disruption persists longer.

#### Neuromuscular and CNS Fatigue

The nervous system controls muscle recruitment. High-intensity efforts — particularly maximal sprints, repeated accelerations, and long VO2max intervals — appear to tax not just the muscles but the capacity to drive them. This is conventionally attributed to central (CNS) fatigue, though the mechanism is contested: a meaningful share of what coaches label "CNS fatigue" may be peripheral or perceptual rather than truly central. The label matters less than the observable pattern, which is robust regardless of cause: reduced power at perceived maximal effort, reduced coordination, and a general flatness that athletes describe as "having nothing in the legs" even when they feel aerobically fresh.

This fatigue type is one of the least visible and one of the most practically significant. An athlete who reports feeling aerobically fine but flat and unable to produce power is likely carrying it. Pushing through does not accelerate recovery — it deepens the deficit.

Recovery typically requires 48-72 hours after a maximal neuromuscular session. This is longer than most athletes expect and longer than the TSS of a short sprint session would suggest. Treat the 48-72 hour figure as a coaching heuristic rather than a measured constant — individual variation is large and the underlying mechanism is not fully settled.

#### Acute Response Summary by Stimulus Type

| Stimulus | Glycogen cost | Muscular damage | Cardiovascular stress | CNS fatigue | Typical recovery |
|---|---|---|---|---|---|
| Z1 Recovery | Minimal | None | None | None | Ready next day |
| Z2 Endurance | Moderate (duration-dependent) | Low | Low-moderate | Low | 24 hours |
| Z3 Tempo | Moderate-high | Low-moderate | Moderate | Low | 24-36 hours |
| Sweet Spot | High | Moderate | Moderate-high | Low-moderate | 36-48 hours |
| Z4 Threshold | High | Moderate | High | Moderate | 48 hours |
| Z5 VO2max | Very high | Moderate | Very high | Moderate-high | 48-72 hours |
| Z6 Anaerobic | High | High | Moderate | High | 48-72 hours |
| Z7 Neuromuscular | Low | Very high | Low | Very high | 48-72 hours |

*These are indicative ranges for a trained athlete. Individual variation is significant. Training age, fitness level, nutrition, sleep quality, and accumulated fatigue all modify these timelines meaningfully.*

---

### 3.2 Chronic Adaptations

Chronic adaptations are the structural and functional changes the body makes in response to repeated stimulus over weeks and months. They are the point of the training. The acute response is the cost; the chronic adaptation is the return on that cost — but only if the stimulus is applied consistently, at the right intensity, with adequate recovery between sessions.

Adaptations are stimulus-specific. Consistent Z2 work does not develop the same qualities as consistent threshold work. The body adapts to what it is repeatedly asked to do, at the intensity it is repeatedly asked to do it. This is the physiological grounding for the specificity principle and the reason the prescription must be honest to its intended stimulus.

#### Aerobic Base Adaptations (Z2, Tempo)

Consistent aerobic base work drives a cluster of adaptations that underpin performance at all intensities:

**Mitochondrial density.** Mitochondria are the cellular machinery of aerobic energy production. More mitochondria means more aerobic capacity per unit of muscle — the muscle can produce more energy aerobically before needing to recruit glycolytic pathways. This is the primary adaptation of sustained Z2 work and one of the most valuable in endurance sport.

**Fat oxidation efficiency.** The body becomes better at burning fat as fuel at higher intensities, sparing glycogen for when it is genuinely needed. An athlete with high fat oxidation efficiency can ride at higher absolute power while remaining in a primarily aerobic state — their effective Z2 ceiling is higher.

**Cardiac output.** The heart becomes stronger and more efficient — stroke volume increases, meaning more blood per beat, meaning more oxygen delivered per minute. This is a slow adaptation that builds over months and years, not weeks.

**Capillary density.** More capillaries in trained muscle means better oxygen and substrate delivery and better lactate clearance. A supporting adaptation to mitochondrial density.

*Timescale: aerobic base adaptations are among the slowest to develop and the slowest to lose. Meaningful mitochondrial adaptation requires weeks of consistent stimulus; full expression takes months. They are also among the most durable — an athlete returning from a break retains aerobic base longer than high-end fitness.*

#### Threshold Adaptations (Sweet Spot, Z4)

**Lactate threshold shift.** With consistent training at and around LT2, the threshold itself moves upward — the athlete can sustain higher absolute power before lactate production outstrips clearance. This is the primary adaptation of threshold work and one of the most performance-relevant for most endurance events.

**Lactate clearance rate.** The muscles become more efficient at consuming lactate as fuel, raising the ceiling at which accumulation begins. This is partly a function of mitochondrial density (the aerobic system can process more lactate) and partly a direct adaptation to repeated threshold stimulus.

**Muscular endurance.** The ability to sustain high-force contractions over extended durations without degradation. Threshold work is the primary driver of this quality.

*Timescale: threshold adaptations develop over 4-8 weeks of consistent stimulus and are relatively responsive to training. They are also relatively quick to fade with detraining — typically faster than aerobic base adaptations.*

#### VO2max Adaptations (Z5)

**VO2max ceiling.** The maximum rate at which the body can consume oxygen rises with consistent VO2max training. This raises the absolute ceiling of aerobic power — every zone below it benefits indirectly because the ceiling has moved.

**Cardiac stroke volume at maximal effort.** The heart's ability to deliver blood at maximal intensity improves. This is the primary central adaptation of VO2max work.

**Buffer capacity.** The body's ability to tolerate and manage the acidic environment of high-intensity effort improves, allowing the athlete to sustain efforts above threshold for longer before the environment becomes limiting.

*Timescale: VO2max adaptations respond relatively quickly to targeted stimulus — meaningful gains are typically visible within 4-6 weeks of a focused VO2max block. They also fade relatively quickly, which is why VO2max work is typically placed in the build and peak phases rather than carried throughout the year.*

#### Neuromuscular Adaptations (Z6, Z7)

**Peak power.** The absolute ceiling of power output rises with consistent neuromuscular training. More motor units recruited, faster, with better coordination.

**Repeatability.** The ability to produce near-maximal efforts repeatedly with short recovery — a direct race demand in criteriums, mass start events, and any race with repeated accelerations.

**Phosphocreatine replenishment rate.** With training, the ATP-PC system replenishes faster, meaning shorter recovery is needed between maximal efforts.

*Timescale: neuromuscular adaptations can appear quickly — within 2-3 weeks — but peak power gains plateau relatively fast. Maintaining neuromuscular sharpness requires regular stimulus; even one maximal effort per week can preserve it during periods when volume is the focus.*

#### The Adaptation Hierarchy

Adaptations build on each other in a sequence that the phase structure reflects:

Aerobic base → Threshold development → VO2max ceiling → Neuromuscular sharpness

Each layer is more effective when the layer below it is developed. Threshold work produces better results on a strong aerobic base. VO2max work produces better results when threshold is developed. This is why base phase comes first — not because it is easy, but because it is foundational.

Attempting to shortcut this sequence by jumping to high-intensity work on an underdeveloped base produces short-term gains that plateau quickly and high injury and burnout risk. The adaptation hierarchy is not a preference — it is a physiological constraint.

*Note: experienced athletes with well-developed bases can compress this sequence and maintain multiple qualities simultaneously. Training age matters. A new athlete needs to build the layers sequentially; an experienced athlete is maintaining and refining a structure that already exists.*

#### Building the House — A Model for Sequencing

Training adaptation can be understood as building a house. The model is simple enough to explain to any athlete and precise enough to justify the sequencing logic of the entire plan.

**The foundation** is the aerobic base — all the Z2 and tempo work that builds mitochondrial density, fat oxidation efficiency, and cardiac output. Without a solid foundation nothing built above it is stable. A house on a weak foundation has a hard ceiling on how tall it can grow.

**The walls** are threshold development — the ability to sustain high power at LT2. Threshold sits at a percentage of VO2max, typically between 75-90% depending on training age and aerobic development. This is the critical structural relationship: threshold has a ceiling above it defined by where the VO2max roof sits. The walls can keep rising toward that roof for a long time — pushing the fraction of VO2max held at threshold upward is one of the most trainable qualities in endurance sport — but they cannot rise above it. The roof is the hard limit; the room beneath it is often large and worth exploiting before assuming the roof itself is the constraint.

**The roof** is the VO2max ceiling — the absolute limit of aerobic power. Threshold can only develop as far as the roof permits. Once the walls begin bumping against the roof, further threshold gains become thin regardless of how much threshold work is applied. The athlete has reached the structural limit of what threshold training alone can produce.

**Raising the roof** is a VO2max block — targeted work at Z5 that pushes the aerobic ceiling upward. Once the roof rises, the walls have room to grow again. Threshold work becomes productive once more because there is headroom above it. This is why VO2max blocks appear in build and peak phases: not as an alternative to threshold work but as the intervention that makes further threshold development possible.

**Thin threshold returns as a prescription signal.** When an athlete has been consistently training threshold and progress has stalled — power at threshold is not moving, sessions feel no easier, rung advancement has plateaued — this is often a signal that fractional utilisation is near its current ceiling and the roof needs raising. A VO2max block is a strong candidate response. It is not the only one: a genuine threshold plateau can also reflect an underdeveloped aerobic base that cannot support more threshold work, accumulated fatigue masking real fitness, or simple habituation to a stale session format (see the habituation stagnation diagnosis in this section). The system should rule out under-recovery and base adequacy before concluding the roof is the binding constraint. Where the diagnosis does point to the ceiling, threshold work resumes once it has been elevated and there is room to grow into it.

**The genetic ceiling.** VO2max is substantially heritable — there is a genetic upper limit to how high the roof can go. This is real and should be acknowledged honestly. However two things remain trainable regardless of genetic ceiling: the absolute ceiling itself can be raised meaningfully through targeted training, particularly in athletes who have not previously trained it systematically; and the percentage of VO2max the athlete can sustain at threshold — their fractional utilisation — is highly trainable. An athlete with a modest genetic VO2max ceiling and 90% fractional utilisation will outperform one with a high genetic ceiling and 75% utilisation. The goal is always to develop both levers as far as the athlete's physiology and training age allow.

### 3.3 Defining Session Outcomes

A session is never a failure. It is either a result or a lesson — usually both. Every session that does not go to plan is a data point that tells the coach something genuinely valuable: about the athlete's current ceiling, their fatigue state, their nutrition, their mental load. That information improves the next prescription. An athlete who internalises a session as a failure carries it into the next one. An athlete who understands it as information arrives at the next one with clarity and curiosity.

*"Every session is either a result or a lesson — usually both."*

A session has one of three top-level outcomes: it completed, it was below target, or it was interrupted. Each outcome feeds into the feedback layer differently and produces a different prescription response. The top-level classification determines which branch of the feedback logic applies.

#### Outcome 1: On Target — Completed as Prescribed

The athlete finished the prescribed main set. Within this outcome, the outcome exists on a spectrum:

**Above target.** Power targets met, RPE below what the zone should feel like, recovery felt sufficient. The athlete had more in reserve than the prescription assumed. This is a progression signal — the rung was appropriate for where the athlete was, and they have moved beyond it. Rung advancement is indicated, and if recovery was cut short while targets were still met, the prescription should also consider tightening recovery as a progression axis.

**On target.** Power targets met, RPE consistent with the zone's expected discomfort profile, recovery felt appropriate. The prescription was correctly calibrated. The rung holds and the ladder steps forward as planned.

**Completed but at the limit.** Power targets met but it cost more than the zone should require. RPE elevated above expected for the stimulus type. The athlete got through it but was genuinely at their ceiling. This is a consolidation signal — the rung was at or near the athlete's current limit. Hold the rung, do not advance. Repeating at this level until it normalises is the correct response. This is not a below-target outcome — the athlete delivered the prescribed work. It is valuable information about where the ceiling currently sits.

**Completed with progression fade.** Power targets broadly met in early intervals but progressively fell short in later ones. The session started correctly and faded. This is the most nuanced outcome — it could indicate a rung that is too ambitious, fatigue carried into the session, a pacing error, or a nutritional deficit. Context determines the response. See 2.3 for the specific degradation signals and their likely causes. The lesson here is in the shape of the fade, not in the outcome itself.

#### Expected Discomfort Profiles

A session outcome is partly defined by whether the discomfort the athlete experienced matches what the stimulus type should feel like. Unexpected discomfort — too much or too little — is itself a signal worth exploring.

The expected RPE for a correctly executed, correctly calibrated session at each zone is given below. RPE is on a 1–10 scale and refers to the representative effort of the work portion — the steady sensation of a sustained block, or the per-rep sensation of short efforts. This table is the reference point the advancement logic in section 4.3 compares against: "RPE below expected" and "cost more than the zone should require" are defined relative to these bands, not against an expectation the system has to invent.

| Zone / stimulus | Expected work RPE (1–10) | Notes |
|---|---|---|
| Z1 Recovery | 1–2 | Should feel like almost nothing. |
| Z2 Endurance | 2–3 | Conversational. If it climbs to 4+, the grey zone has been entered. |
| Z3 Tempo | 4–5 | Comfortably hard, sustainable, controlled breathing. |
| Sweet Spot | 5–6 | Hard but repeatable; you could hold a short sentence. |
| Z4 Threshold | 6–7 | Sustained discomfort that builds; talking reduced to a few words. |
| Z5 VO2max | 8–9 | Breathing maximal within 60–90s; sustaining commitment is the challenge. |
| Z6 Anaerobic | 8–9 per rep | Acute and intense but clears fast once the effort stops. |
| Z7 Neuromuscular | 9–10 per rep | Maximal but brief; the system clears within minutes. |

These bands are for a session that landed where it was meant to. The gap between actual RPE and the expected band is the signal — see section 4.3 for how the size of that gap maps to a level change.

**Short sharp intervals (Z6, Z7, On-Offs).** The discomfort is acute and intense but cardiovascular in character. The heart rate spikes hard, breathing becomes maximal, the sensation is one of immediate overwhelm. Critically — it passes quickly. Once the effort stops the system clears rapidly and the athlete recovers faster than the intensity of the effort would suggest. Legs feel functional again within minutes. The acute distress is real but transient. An athlete who reports feeling depleted hours after a sprint session is carrying more fatigue than the stimulus type alone explains — look for accumulated fatigue or inadequate nutrition.

**Threshold and sweet spot intervals (Z4, sweet spot).** The discomfort builds progressively and persists. The hydrogen ion accumulation — the burning sensation in the legs — develops over the duration of the effort and lingers after it ends. Dead legs into the following day are normal and expected after a hard threshold session. This is muscular recovery from sustained metabolic stress, not cardiovascular stress. An athlete who expected to feel fine the next day and doesn't needs to understand this is the correct response, not a sign something went wrong.

**VO2max intervals (Z5).** A combination of both profiles. The cardiovascular ceiling is hit quickly — breathing becomes maximal within 60-90 seconds — and the peripheral burn builds across the effort. The mental challenge is sustaining commitment for the full duration when the body is signalling distress on multiple channels simultaneously. Recovery between efforts is never quite enough; the athlete begins each rep already slightly fatigued. This is by design.

**Endurance (Z2).** Should feel genuinely easy — conversational pace, no burn, no cardiovascular strain. If it doesn't, the intensity is too high and the grey zone has been entered. An endurance session that feels like work is a signal to bring the intensity down, not a reason to push through.

#### Mental Framing as Part of Session Outcome

The athlete's psychological approach to a session is not separate from its physiological outcome — it directly affects pacing, commitment, and whether the prescribed intensity is maintained under discomfort.

For sessions with a significant mental demand — VO2max intervals in particular — the prescription should include framing for how to approach the effort, not just what the effort is. The most effective reframe for long hard intervals is duration decomposition: rather than committing to the full interval duration from the start, the athlete commits to one minute at a time.

*"You can do anything for a minute."*

Four minutes at VO2max is a daunting prescription. Four efforts of one minute, assessed one at a time, is survivable. The cardiovascular system reaches its ceiling within the first 60 seconds regardless — the remaining three minutes are not physiologically harder, only psychologically longer. Breaking the interval into minute-long commitments does not change the stimulus. It changes the athlete's relationship with it, which changes their ability to maintain the intensity that delivers it.

This principle extends beyond VO2max work. Any interval that feels beyond reach can be decomposed into a duration the athlete can genuinely commit to. The prescription system should surface this framing when prescribing sessions with known high psychological demand.

#### Outcome 2: Below Target

The session was attempted but did not complete as prescribed — power targets fell significantly short, the athlete reduced intensity substantially mid-session, or the session was curtailed early through diminishing returns rather than a discrete stop. This is not a verdict on the athlete — it is information. The lesson is in understanding why, not in the shortfall itself. See section 2.3 for the full signal taxonomy. The feedback response depends on whether the outcome reflects a pacing error, a fatigue signal, or a ceiling discovery.

#### Outcome 3: Interrupted — Session Interruption Protocol

A session that stops abruptly mid-effort is categorised separately from a below-target outcome. The stop is a discrete event, not a progressive fade. Before any prescription response is made, the cause must be established. Every interrupted session has a lesson — finding it requires understanding the cause first. Causes fall into five categories:

**Environmental.** Conditions made the session impossible or unsafe — excessive heat or cold, wrong terrain for the prescribed session, unsafe road conditions, trainer connectivity failure, head unit not charged. The session did not happen but the athlete is unaffected. The ladder is untouched. Reschedule the session at the next available opportunity. No rung change, no welfare flag. The lesson here is logistical — preparation for the next attempt.

**Mechanical.** Equipment failure — bike mechanical, trainer pairing failure, power meter dropout. As with environmental causes, the athlete is unaffected and the session simply did not happen. Reschedule. The lesson is in the preparation: equipment checks before quality sessions prevent avoidable interruptions.

**Nutritional.** The athlete did not fuel or hydrate adequately for the session demand, or over-fuelled and experienced GI distress. This is a correctable and instructive outcome. The ladder holds. A conversation about pre-session nutrition and in-session fuelling protocol is warranted before the next quality session. The lesson is specific and actionable.

**Physical.** Pain, injury, or illness caused the stop. This is a welfare signal that takes priority above all prescription considerations. The ladder pauses. No rescheduling until the physical status is understood. The appropriate response is to establish the nature and severity of the physical issue and seek appropriate support. Training resumes only when it is safe to do so. This category should always be escalated for direct follow-up.

**Mental.** Motivation was absent, family or work obligations intruded, life stress made the session feel impossible. This category sits on a spectrum and requires the most careful and compassionate handling. At one end — the athlete simply wasn't feeling it on a given day — the response is a conversation about lowering the barrier to entry: start the warm-up, commit to ten minutes, and reassess. Motivation often returns once the body is moving. At the other end — persistent inability to engage, loss of enjoyment, significant life stress accumulating alongside training stress — the response is load reduction and genuine inquiry into the athlete's overall state. The prescription system flags mental interruptions for human follow-up rather than attempting to resolve them algorithmically. This is a coaching conversation, not a data problem. The lesson here belongs to the coach as much as the athlete.

### 3.4 Maladaptation, Overreach, and the Supercompensation Model

#### The Athlete Gets Faster in Recovery

The interval session is the stimulus. The adaptation — the faster, stronger, more resilient athlete — is built in the recovery that follows. During a hard session the athlete is, physiologically speaking, moving backwards: glycogen depleting, muscle fibres damaged, cardiovascular system stressed, performance temporarily reduced. A good successful session leaves the athlete slower than when they started.

The body then responds to that disruption. In the hours and days that follow — given adequate sleep, nutrition, and reduced load — it rebuilds. Not back to the previous baseline, but above it. This is the adaptation. This is where the work becomes fitness.

Recovery is not the absence of training. It is the second half of the prescription. Sleep, nutrition, and easy days are not rewards for doing the hard work — they are where the hard work becomes performance. An athlete who does not respect recovery is not training hard. They are applying stimulus without allowing the adaptation it was designed to produce. The work without the recovery is physiological cost without physiological return.

This reframes the taper that athletes often find psychologically uncomfortable. Reducing load before a target event feels like losing fitness. The opposite is true. The fitness was built during the training block. The taper gets out of the way and lets the body fully express the adaptation it has been accumulating. Performance peaks not despite the reduced load but because of it.

#### The Supercompensation Model

The adaptation process follows a predictable pattern known as supercompensation:

**Phase 1 — Stimulus.** Training load is applied. Performance capacity temporarily decreases. The athlete is fatigued.

**Phase 2 — Recovery.** Load is reduced or removed. The body returns toward baseline, repairing the disruption caused by the stimulus.

**Phase 3 — Supercompensation.** The body overshoots the previous baseline, rebuilding above it. This is the adaptation window — the athlete is temporarily capable of more than before the stimulus was applied.

**Phase 4 — Return to baseline.** If no further stimulus is applied, the supercompensation peak fades and the athlete returns to their previous level.

**A note on what this model is and is not.** Supercompensation is a teaching model — a clean single-curve story that makes the timing intuition vivid and is genuinely useful for explaining why recovery is part of the prescription. It is not how the system reasons quantitatively, and it should not be taken literally. The discrete "peak" cannot be reliably detected in an individual on any given day, and real adaptation does not follow one tidy curve per session — it is the summed response to weeks of overlapping stimulus and recovery. The model the system actually runs on is the fitness-fatigue (impulse-response) model that underpins the performance management chart: every load simultaneously adds to a slow-decaying fitness trace (CTL) and a fast-decaying fatigue trace (ATL), and readiness is the gap between them (TSB). This is the more defensible framework and it is the one already endorsed in section 1.4. Where the two models appear to conflict, fitness-fatigue governs the system's decisions; supercompensation survives only as the metaphor used to explain those decisions to the athlete.

The practical goal is the same under either lens: apply the next quality stimulus when the body has absorbed the last one and is trending toward readiness, rather than while still in deficit or long after the adaptation window has closed. The daily Go/No-Go layer — built from subjective scores and trending health data — is the system's real-world proxy for "is the body ready for the next stimulus," and it does the job the idealised supercompensation peak only gestures at.

#### The Timing Errors

**Too early.** The next stimulus arrives before the athlete has recovered to baseline, let alone supercompensated. The body is loaded while still in deficit. The hole gets deeper with each repetition. Acute fatigue accumulates faster than it can be cleared. Performance drops, sessions go below target, RPE rises for a given power output. Applied consistently this is the pathway to overreaching and, if sustained, to genuine overtraining.

**Too late.** The supercompensation peak has passed. The body has returned to its previous baseline. The stimulus produces a training response but not a progressive one. The athlete is working consistently but not building. Fitness plateaus despite regular training. This is less damaging than loading too early but equally unproductive — the timing of the rhythm matters as much as the load itself.

**Right time.** The stimulus meets the supercompensation peak. The next baseline is higher than the last. The staircase goes up. This is what the plan is trying to achieve with every quality session.

#### The Load Errors

**Too little.** The stimulus does not disturb homeostasis sufficiently to trigger a meaningful adaptation signal. The body absorbs it without needing to rebuild above the previous level. The athlete trains consistently, accumulates TSS, and does not progress. This is the athlete who is always comfortable — comfortable sessions do not build fitness, they maintain it at best.

**Too much.** The stimulus exceeds the body's capacity to absorb and recover from it. The acute training load outstrips the chronic training load faster than fitness can build. The athlete cannot return to baseline before the next session arrives. Each session digs the hole deeper. Performance drops, motivation falls, and the signals of overreaching begin to appear.

#### Productive Overreach vs Unproductive Overtraining

Not all accumulated fatigue is pathological. There is a meaningful distinction between productive short-term overreach and unproductive chronic overtraining.

**Productive overreach** is deliberate and time-limited. A block of training in which load intentionally exceeds what the athlete can fully recover from day to day, followed by a planned reduction in load that allows supercompensation to occur at a higher level than a more conservative approach would produce. The fatigue is real; the performance dip is real; the subsequent adaptation is greater than steady-state training would achieve. This is the mechanism behind build blocks and pre-competition loading phases. The key word is planned — the overreach is prescribed and the recovery is prescribed in equal measure.

**Unproductive overtraining** is sustained overreach without adequate recovery. Performance decrements persist through weeks of rest. Mood deteriorates, motivation disappears, resting heart rate elevates, HRV depresses chronically. Unlike productive overreach, which resolves within days to weeks of appropriate recovery, genuine overtraining syndrome requires months to reverse and in severe cases may never fully resolve. It is rare precisely because it requires sustained months of genuine overload — most athletes who think they are overtrained are under-recovered.

The prescription system monitors the signals that distinguish these states rather than treating all fatigue as a warning.

#### Signals of Maladaptation

When the stimulus is not producing adaptation — when the staircase has stopped going up — the signals manifest across multiple dimensions. A single bad session is noise. A pattern across sessions and weeks is signal.

**Performance stagnation.** Power at a given RPE is not improving. Rung advancement has plateaued. Sessions that should feel easier with repetition do not. The athlete is working but not progressing.

**Elevated RPE for given power.** The same watts cost more effort than they previously did. The athlete is working harder to maintain what they could previously sustain comfortably. This is one of the earliest and most reliable signals that something is wrong.

**HRV depression.** Resting heart rate variability trending downward over days and weeks indicates the autonomic nervous system is under sustained stress it is not recovering from. A single low HRV morning is noise. A week of declining HRV is a load management signal.

**Sleep disruption.** Paradoxically, overreaching often disrupts sleep — the athlete is tired but cannot rest deeply. Poor sleep quality compounds recovery failure and accelerates the descent into deeper overreach.

**Mood and motivation deterioration.** Persistent flatness, loss of enjoyment in training, irritability, and reduced motivation are among the most reliable early indicators of accumulated overreach. These are physiological signals, not character flaws. The nervous system and endocrine system under sustained stress produce exactly these responses. They should be treated as data, not dismissed.

**Persistent muscular heaviness.** Legs that feel heavy session after session, without the normal recovery between efforts, indicate accumulated muscular fatigue that is not clearing. Different from the expected post-session heaviness of threshold work — this heaviness does not resolve with a day's rest.

#### Reading Heart Rate as an Adaptation Signal

Heart rate during efforts provides diagnostic information beyond what power data alone reveals. The key is always the relationship between cardiovascular cost and mechanical output — when these decouple, something worth investigating is happening.

**Elevated heart rate at submaximal effort.** The cardiovascular system is working harder than the power output justifies. More cardiovascular cost for the same mechanical output. This decoupling indicates the body is under stress beyond what the session alone explains — accumulated fatigue, illness developing, heat, dehydration, or life stress elevating sympathetic tone. The power numbers may look acceptable while the physiological cost of producing them is significantly higher than normal. This is one of the earlier warning signals that the recovery half of the prescription is not keeping pace with the load half.

**Suppressed heart rate at maximal effort.** The athlete is working as hard as they can subjectively but the cardiovascular system is not responding to match it. Two opposite interpretations are possible and context separates them. In a fatigued athlete this indicates deep accumulated fatigue — the system is too depleted to elevate appropriately. In a well-rested athlete who has been training consistently it may indicate genuine fitness adaptation — the cardiovascular system has become more efficient and the effort that previously demanded a high heart rate no longer does. The signal is identical; the meaning is opposite. Subjective feedback, recent load history, and HRV trend together distinguish which interpretation is correct.

**Elevated resting heart rate and suppressed HRV.** Resting metrics are more reliable than effort metrics for detecting accumulated fatigue because they are not confounded by the session itself. A resting heart rate trending upward over days and an HRV trending downward over the same period is a consistent pattern indicating the autonomic nervous system is under sustained stress. Individual daily variation is noise. A directional trend across a week is signal. The prescription response is load reduction before performance decrements become apparent — catching it here is earlier and less costly than catching it when sessions start going below target.

#### Habituation Stagnation — When Comfort Becomes the Problem

The stagnation described earlier in this section — performance not improving despite consistent training — has two distinct causes that require opposite prescription responses. Distinguishing them is one of the most important diagnostic tasks the coaching system performs.

**Overreach stagnation** occurs when load exceeds recovery capacity. The body is too stressed to complete the adaptation process. The prescription response is reduced load and extended recovery.

**Habituation stagnation** occurs when the stimulus has become too familiar. The body has adapted to a given session type and no longer needs to rebuild above baseline to handle it — it already can. The adaptation signal disappears. More of the same produces diminishing returns and eventually no return at all.

Humans — and their physiology — are creatures of habit. A stimulus that was genuinely challenging three months ago may produce minimal adaptation today not because the athlete is fatigued but because the body has become efficient at handling exactly that stress. It no longer needs to shock the system into adaptation because the system is no longer shocked.

The signals of habituation stagnation differ subtly from overreach stagnation. The athlete is not particularly fatigued. RPE for familiar sessions may actually be lower than expected. HRV and resting heart rate are unremarkable. Sessions complete as prescribed. Progress simply stops.

The prescription response to habituation stagnation is the opposite of the response to overreach stagnation — not less load but a different stimulus entirely. Two interventions are available:

**Increase overall training volume.** The athlete may have reached the ceiling of what the current load can produce. The leash has run out. Extending total training time — more hours, more TSS across the week — extends the stimulus beyond where the body has adapted and forces a new response. This is appropriate when the athlete has capacity to absorb more load and the phase supports it.

**Change the stimulus type.** Introduce something the body has not adapted to. The most powerful intervention here is a VO2max block when threshold has plateaued — raising the roof to give the walls room to grow, as described in section 3.2. But any meaningful change to the stimulus type — different structural patterns, different zones, different interval formats — can disrupt habituation and restart the adaptation signal. The body does not adapt to variety; it adapts to specific repeated stresses. Variety is the antidote to habituation.

The critical diagnostic question when stagnation appears is always: *is the athlete too tired to adapt, or too comfortable to adapt?* The answer determines everything about the prescription response.

#### The Prescription Response to Maladaptation Signals

When the pattern of signals indicates that adaptation has stalled or reversed, the prescription response depends entirely on which type of stagnation is present. Overreach stagnation demands reduced load and extended recovery. Habituation stagnation demands a changed or intensified stimulus.

The temptation — for athlete and coach alike — is to respond to any stagnation with more work. When the cause is overreach this is exactly wrong. Stagnation under high load is not a signal to add more load. It is a signal that the recovery half of the prescription has been insufficient and the adaptation process has been interrupted. The path forward is through recovery, not around it.

When the cause is habituation, more of the same work is equally wrong — just for the opposite reason. The body has stopped responding to that stimulus. The path forward is through change.

*Is the athlete too tired to adapt, or too comfortable to adapt?* Every stagnation diagnosis starts here.

---

## Layer 4: Ladder Logic and Progression

*How stimulus, response, and adaptation combine into a structured progression system.*

### 4.1 The Ladder as Progression Unit

The ladder is the mechanism by which progressive overload is delivered systematically and individually. Every trainable system — threshold, VO2max, sweet spot, anaerobic, neuromuscular, endurance — has its own ladder. Each ladder is a sequence of levels through which the athlete progresses as their capability in that system develops.

#### The Level Scale

Levels run from 1.0 to 10.0, expressed to one decimal place.

**Integer levels** are the meaningful milestones — the psychological wins, the genuine markers of development. A level 6 threshold rider and a level 7 threshold rider are in meaningfully different places in their development. Integer transitions represent structural progression: a new interval format, a significant duration increase, a pattern change that demands a qualitatively different effort.

**Decimal increments** provide the granularity that keeps progression smooth and steps manageable. A nudge from level 6.0 to 6.1 might be a small intensity increase within the zone — 88% FTP to 90% FTP — or a single minute added to an interval duration, or recovery tightened slightly. The change is real but the demand increase is deliberately small. The athlete should barely feel the difference on any single step. Over ten decimal increments the cumulative change is significant; at each individual step it is achievable.

This structure serves both the physiological and psychological dimensions of progression. The decimal increments keep the body adapting without overstretching it. The integer milestones provide the motivational markers that make the process feel meaningful.

**Natural level bands:**

| Band | Levels | Character |
|---|---|---|
| Foundational | 1–3 | Building basic capacity to sustain the zone. Short intervals, generous recovery, lower intensity within the zone. |
| Development | 4–6 | Sustaining the zone and building duration and repeatability. The bulk of a training block lives here. |
| Proficiency | 7–9 | Pushing toward the ceiling of what the zone can produce. Longer efforts, tighter recovery, higher intensity within the zone. |
| Expression | 10 | At or near the current physiological ceiling for that system. Progression here means raising the roof before returning. |

#### Progression Mode and the Ladder

Each ladder carries a progression mode — extensive or intensive — as defined in section 1.7. Within a phase, the mode determines whether advancing a decimal increment means a longer effort at the same intensity or a harder effort of the same duration. Integer level transitions may shift the progression mode or introduce a new structural pattern.

#### The Feedback Resolution

The decimal scale gives the feedback layer matching resolution. Rather than binary advance or hold decisions, the system can advance by 0.1, 0.2, or 0.3 depending on how the session went:

- **Above target** — completed comfortably with reserve: +0.2
- **On target** — completed as prescribed: +0.1
- **At the limit** — completed but at the ceiling: hold, consolidate
- **Below target** — did not complete as prescribed: hold or step back depending on cause

The step back is always by the same increment as the most recent advance — the system undoes one step, not many.

---

### 4.2 Rung Seeding and the Signature Profile

#### The Signature Profile

The athlete's power profile is their physiological fingerprint — a unique expression of their current capabilities across all durations and systems. No two athletes have the same profile even at the same FTP. The shape of the curve reveals phenotype, training history, strengths, and limiters simultaneously.

The signature profile is the living record of that fingerprint. It is not a starting point that gets discarded once training begins — it is the foundation the entire level system is built on and the reference point against which all progression is measured. When the profile changes, the levels recalibrate against the new reality.

The signature profile contains:
- Current FTP and the date it was established
- Power curve across key durations (5s, 30s, 1min, 5min, 20min, 60min)
- A durability marker — best power at key durations after a defined energy expenditure, compared to the fresh value (see section 1.7)
- Current level for each trainable system
- Level history — how each system's level has moved over time
- Phenotype characterisation derived from curve shape

#### Seeding from the Profile

When an athlete enters the system, levels are not assigned at 1.0 by default. A trained athlete who has been riding for years should not begin threshold work at level 1 — they would spend weeks climbing through sessions they have already mastered, producing no meaningful adaptation and no motivation.

Instead, levels are seeded from the power curve. Each system maps to a characteristic duration and intensity on the curve. The athlete's best power at that duration and intensity, expressed relative to FTP, indicates where on the ladder they currently sit.

Seeding is deliberately capped in the lower half of the relevant band — a rider whose curve suggests level 7 capability is seeded at level 5 or 6. This is intentional. The seed gets the athlete into the right neighbourhood; it does not jump them to their apparent ceiling. There must always be a meaningful runway of progression above the starting point. An athlete seeded too high has nowhere to go and no room to learn.

The cap also accounts for the difference between peak performance and training performance. A power curve reflects best efforts, often in favourable conditions. The ability to reproduce that stimulus consistently in training, across a block of sessions, is a different and lower bar. The seeded level reflects trainable capability, not peak expression.

#### Seeding from the Coggan Power Profile

The system does not have access to a large proprietary dataset of athlete rides from which to derive natural level distributions. Instead it anchors the level scale to Coggan's published power profile table — an established, physiologically grounded reference for where athletes sit relative to the broader population across all key durations.

The Coggan table provides performance bands — untrained through world class — expressed in watts per kilogram across the durations that map to each trainable system. These bands serve as the anchor points for level seeding:

| Coggan band | Approximate level |
|---|---|
| Untrained | 1.0 – 2.0 |
| Fair | 2.0 – 3.5 |
| Moderate | 3.5 – 5.0 |
| Good | 5.0 – 6.5 |
| Very Good | 6.5 – 7.5 |
| Excellent | 7.5 – 8.5 |
| Exceptional | 8.5 – 9.5 |
| World Class | 9.5 – 10.0 |

Each system is seeded independently from the relevant part of the power profile. Neuromuscular level seeds from 5-second and 30-second power relative to the Coggan bands. VO2max level seeds from 5-minute power. Threshold level seeds from FTP. Endurance level seeds from 60-minute and longer power relative to FTP. The shape of the curve seeds each system from its characteristic duration — the profile is read as a whole, not reduced to a single number.

This approach means the level scale is meaningful beyond the individual athlete. A level 6 threshold is not an arbitrary internal score — it maps to a understood performance band relative to the trained population. For a self-coached amateur athlete the realistic operating range across most systems is approximately 4.0–7.0, which is precisely where the ladder needs the most resolution and where the decimal granularity earns its place.

#### Watts per Kilogram for Seeding; FTP-Relative for Training

The Coggan table uses watts per kilogram, normalising for body weight to enable population comparison. This is appropriate for seeding — it is comparing the athlete to a reference population and needs a normalised metric to do so fairly.

Once seeding is complete, w/kg is put away. All session prescriptions are expressed in FTP-relative terms — percentage of FTP, zone-relative language, or absolute watts anchored to the current FTP. Body weight never appears in a session prescription. The athlete is told to hold 95% FTP for 12 minutes, not to produce 3.8 w/kg. The daily work is anchored to what the athlete can produce today, full stop.

**On body weight and body image.** Endurance sport has a well-documented and unhealthy relationship with body weight. The w/kg metric, while physiologically useful for population comparison, can feed an unhealthy focus on weight as a performance lever. This system uses w/kg only where it is genuinely necessary — the seeding calculation — and keeps it out of every other layer of the prescription. The system does not track body weight as a performance metric. It does not comment on weight changes. It notices FTP changes and recalibrates accordingly. If an athlete's body composition changes naturally over a training block and FTP moves with it, the recalibration handles the update. Weight itself is never the subject.

Body weight is an optional input, provided by the athlete at their discretion, used once for the seeding calculation, and not stored as a tracked metric or surfaced in feedback, progression commentary, or session prescription.

When FTP increases — through a new test, a Ramp test, or a meaningful performance update — the power curve is re-read and levels recalibrate against the new baseline.

An FTP increase nudges levels downward. This is not a punishment or a reset — it is the system acknowledging that the athlete is now stronger and that the work which represented a given level of demand at the previous FTP represents proportionally less demand at the new one. The absolute watts of a level 6.4 session do not change. What changes is where those watts sit relative to the new ceiling.

The correct framing for the athlete: *your FTP went up, the work got easier — that is the point.* The level nudge down is confirmation that the training worked. The athlete is a stronger rider for whom that session is now appropriately less demanding. They will climb back through the decimal increments quickly because they have the fitness — and this time the sessions will feel like consolidation rather than struggle.

The magnitude of the nudge is proportional to the FTP increase. A small FTP gain produces a small recalibration. A significant jump produces a more meaningful one. The system calculates this relative to the percentage change in FTP rather than applying a fixed offset.

#### Phase Transitions and Level Preservation

Levels are never deleted. When a phase transition moves the dominant system away from threshold and into aerobic base work, the threshold level is preserved in the signature profile in a dormant state. When threshold returns as the dominant system in the next build phase, the level picks up where it left off — adjusted only if an FTP recalibration has occurred in the interim.

This means the signature profile accumulates history. The shape of how levels have moved across phases and seasons tells a story — a rider who has been consistently raising threshold levels while neuromuscular levels have stayed flat has a clear prescription signal in that pattern. The profile is not just current state; it is training biography.

### 4.3 Rung Advancement

The advancement logic is the decision engine at the heart of the ladder system. It translates session outcomes — as defined in section 3.3 — into level changes, and it ensures that progression is earned, proportional, and responsive to the full picture of the athlete rather than just the power numbers.

#### The Core Principle

A completed session advances the athlete. The question is not whether to advance but by how much — and that answer is tiered, because a session ripped apart with significant reserve is a fundamentally different signal from one scraped through at the limit. Treating them identically wastes the information the session just produced and risks either under-loading an athlete who has outgrown the rung or over-loading one who is at their ceiling.

#### Advancement Tiers

| Outcome | Level change | Condition |
|---|---|---|
| Exceptional | +0.3 | Exceeded all targets with clear reserve. RPE significantly below expected for the zone. Could have done more. |
| Above target | +0.2 | Completed comfortably. Targets met, RPE below expected, some reserve remaining. |
| On target | +0.1 | Completed as prescribed. Targets met, RPE consistent with expected zone feel. |
| At the limit | +0.0 | Completed but at the ceiling. Targets met but cost more than the zone should require. Consolidate at this level. |
| Below target | Escalation protocol | Did not complete as prescribed. See below. |

#### Defining the Tier Thresholds

The tiers above turn on two judgements: were the power/duration targets met, and how did RPE compare to expected. Both are made precise here so the same session produces the same level change every time it is evaluated.

**Targets met** is determined by the session outcome scoring rules in section 2.3 ("Scoring a Session from the File"). A session that passes those rules counts as completed; one that does not is below target and enters the escalation protocol. The RPE comparison is only made on sessions that completed.

**The RPE comparison** uses the expected work-RPE band for the session's zone from section 3.3. Let the gap be the distance between the athlete's reported work RPE and the nearest edge of that band:

- **Significantly below (Exceptional, +0.3):** reported RPE is 2 or more points below the lower edge of the expected band, with all targets met and clear reserve. The rung is well under the athlete's current ceiling.
- **Below (Above target, +0.2):** reported RPE is roughly 1 point below the lower edge (0.5 to 1.9 below). Comfortable, some reserve.
- **Within band (On target, +0.1):** reported RPE falls inside the expected band. Correctly calibrated.
- **Above band (At the limit, +0.0):** reported RPE is 1 or more points above the upper edge despite targets being met. The athlete delivered the work but it cost more than the zone should. Consolidate.

Where reported RPE and the power file disagree — for example, on-target power at a much higher RPE than expected — the higher-cost reading wins for the advancement decision, and the divergence is itself flagged per the RPE sanity-check role in section 1.5 (outdated FTP, hidden fatigue, illness, heat). An unexpectedly easy RPE at on-target power is a fitness signal; an unexpectedly hard one is a fatigue or calibration signal. The system does not simply average them.

**A cap of +0.3 applies regardless of how exceptional the session felt.** The body adapts on its own timeline. A single outstanding session does not accelerate the physiological adaptation process — it only reveals that the current rung is underdemanding. The system advances faster, but not recklessly.

**Consolidation is not failure.** An at-the-limit completion is a well-calibrated session — the rung was correctly placed at or near the athlete's current ceiling. Repeating at this level until it normalises before advancing is the correct response. The athlete is building the foundation for the next step, not stalling.

#### The Below Target Escalation Protocol

A session that goes below target is not automatically a step back. One miss is a data point — it could reflect a bad night's sleep, a nutritional deficit, accumulated life stress, or simply a local event with no systemic significance. The system's response scales with the pattern, not the single event.

**Cause is read before count.** The interruption and outcome cause categories from section 3.3 modify the escalation at every tier. The same miss count means different things depending on why the misses are happening.

**First below-target session:**
Hold the level. Do not step back. Note the cause and explore it with the athlete — a brief check-in, not an intervention. Go again at the same level. The message to the athlete: one session tells us something but not everything. Let's see what the next one says.

**Second consecutive below-target session:**
Hold the level. Flag the pattern. This is worth a genuine conversation — what is going on? Is the rung too ambitious? Is there accumulated fatigue? Is something happening outside training that is affecting sessions? Offer a modification — a reduced version of the session at the same level, or a deliberate step to a recovery week before returning. The message: something is worth looking at here. Let's address it rather than push through it.

**Third consecutive below-target session:**
Pause the ladder. Trigger the recovery and reassessment protocol. The athlete is not progressing at this level and continuing to attempt it is accumulating fatigue without producing adaptation. Step back, recover, and reassess readiness before re-entering the ladder. The level steps back by one full tier — the system acknowledges the athlete needs to rebuild the foundation before climbing again. The message: this is not working right now. That is information, not verdict. We reset, we recover, we return.

**Successful session at any point resets the counter.** The athlete should not carry the weight of past misses indefinitely. A completed on-target session clears the escalation history and the ladder resumes normally. Progress is not permanently marked by a rough patch.

#### Cause Modifies the Response

The cause category from the session interruption protocol modifies the escalation at every tier:

**Environmental or mechanical** — logistical failure, not an athlete signal. Note only, reschedule, no escalation. These misses do not count toward the consecutive miss counter.

**Nutritional** — correctable and instructive. Note, education conversation, reschedule. Counts toward the miss counter only if the nutritional issue is persistent rather than a one-off preparation miss.

**Physical** — welfare flag regardless of miss count. The ladder pauses immediately on a physical stop, not after three. Physical signals take priority over all progression considerations.

**Mental** — escalate one tier faster than the standard protocol. A mental miss carries more signal than a logistical one. Two consecutive mental misses trigger the third-tier response. This is not a punitive response — it is an acknowledgement that persistent mental barriers to training are a welfare signal that warrants earlier and more careful attention than a mechanical failure.

#### The Step Back

When a step back is warranted — third consecutive below-target session, or a physical stop — the level decreases by the same increment as the most recent advance. The system undoes one step. It does not catastrophise by resetting to the beginning or making a large demotion. The athlete is close to where they were; they just need to rebuild the rung below before climbing again.

The step back is always framed as a return to a level the athlete has already demonstrated they can complete. It is not a punishment. It is the system finding the right floor to build from.

### 4.4 Cross-System Interactions and the Weight Matrix

The energy systems are not switches — they are sliders. Every session touches multiple systems simultaneously; what changes with intensity is which system is dominant and which are receiving a secondary signal. A VO2max session produces a small aerobic base signal alongside its primary ceiling stimulus. A four-hour Z2 ride produces an almost pure aerobic signal with negligible touch on any higher system.

The weight matrix formalises this. Every session type carries a set of weights — one per trainable system — that determine what fraction of the primary advancement signal flows to each secondary system. The primary system always receives a weight of 1.0. Secondary systems receive a fractional weight based on how much they are physiologically loaded by that session type.

#### Advancement Signal Flow

When a session outcome produces an advancement increment — say +0.1 for an on-target session — that increment is multiplied by each system's weight to produce the secondary signal:

- Primary system (weight 1.0): +0.1
- Secondary system (weight 0.3): +0.03
- Tertiary system (weight 0.1): +0.01

These fractional increments accumulate across a training block. A block of consistent threshold work produces not just threshold advancement but a small but real upward signal to VO2max and aerobic base. This is physiologically accurate — it is what actually happens — and it means the signature profile evolves as a whole rather than in isolated system silos.

**A floor of 0.05 applies.** Weights below this threshold round to zero — the signal is physiological noise too small to meaningfully advance any ladder.

#### The Weight Matrix by Primary Session Zone

(Sweet spot appears below as a primary session intensity even though section 1.6 defines it as an intensity range rather than a formal zone. It earns a row because sessions are routinely prescribed with sweet spot as their primary target; the matrix is keyed by what a session primarily trains, and sweet spot sessions are common enough to need their own weights.)

**Active Recovery (Z1)**

| System | Weight |
|---|---|
| Aerobic base | 0.2 |
| All others | 0.0 |

Minimal signal. Small aerobic contribution from circulation and movement. Not sufficient to meaningfully advance any ladder — recovery sessions are restorative, not adaptive.

**Endurance (Z2)**

| System | Weight |
|---|---|
| Aerobic base | 1.0 |
| All others | 0.0 |

Pure aerobic signal. The glycolytic system is not meaningfully stressed. No upward signal to higher systems. Duration is the progression axis here — more hours at Z2 is the adaptation driver.

**Tempo (Z3)**

| System | Weight |
|---|---|
| Aerobic base | 0.4 |
| Threshold | 0.2 |
| All others | 0.0 |

Aerobic still dominant but glycolysis is now contributing meaningfully. A small upward signal to threshold reflects that the athlete is working at or above LT1, in the band between the two lactate thresholds.

**Sweet Spot**

| System | Weight |
|---|---|
| Aerobic base | 0.3 |
| Threshold | 0.5 |
| VO2max | 0.0 |
| All others | 0.0 |

Threshold is the primary beneficiary — this is the zone's purpose. Aerobic base still receives a meaningful signal because of the sustained duration at high aerobic demand. VO2max is not touched — the intensity does not approach the ceiling.

**Threshold (Z4)**

| System | Weight |
|---|---|
| Aerobic base | 0.2 |
| Threshold | 1.0 |
| VO2max | 0.15 |
| All others | 0.0 |

The multi-system signal established in section 2.2. Threshold work nudges the VO2max needle as a secondary effect because the aerobic system is working at a high fraction of its ceiling. The aerobic base signal reflects the sustained cardiovascular demand.

**VO2max (Z5)**

| System | Weight |
|---|---|
| Aerobic base | 0.1 |
| Threshold | 0.3 |
| VO2max | 1.0 |
| Anaerobic | 0.1 |
| Neuromuscular | 0.0 |

Threshold receives a significant secondary signal — VO2max intervals pass through and sit above the threshold zone, and recovery between efforts keeps the system elevated near Z4. This asymmetry with threshold's VO2max weight (0.15 vs 0.3) is intentional: VO2max sessions spend more cumulative time in and around the threshold zone than threshold sessions spend near the VO2max ceiling.

**Anaerobic (Z6)**

| System | Weight |
|---|---|
| Aerobic base | 0.0 |
| Threshold | 0.1 |
| VO2max | 0.2 |
| Anaerobic | 1.0 |
| Neuromuscular | 0.15 |

VO2max receives a meaningful secondary signal because the cardiovascular system is working near its ceiling even though the primary stimulus is glycolytic. The neuromuscular signal reflects the high-force explosive nature of anaerobic efforts.

**Neuromuscular (Z7)**

| System | Weight |
|---|---|
| Aerobic base | 0.0 |
| Threshold | 0.0 |
| VO2max | 0.0 |
| Anaerobic | 0.1 |
| Neuromuscular | 1.0 |

Almost entirely isolated. The small anaerobic signal reflects the explosive phosphocreatine and fast-twitch recruitment. All other systems are below the noise floor.

#### Pattern Modifiers

Structural patterns shift the base weights when the session design meaningfully changes the physiological loading relative to the zone alone:

**Over-Unders** (threshold or sweet spot) — the over phases repeatedly push into Z5. VO2max weight increases by +0.1 above the base zone weight. A threshold Over-Under session carries a VO2max weight of 0.25 rather than 0.15.

**On-Offs** (VO2max) — the short maximal efforts have a stronger glycolytic component than traditional long intervals. Anaerobic weight increases from 0.1 to 0.2.

**Hard Starts** — the opening surge above the target zone adds a signal from the zone above. Weight of the zone immediately above the primary increases by +0.1.

**Attacks** — the Phase 1 explosive effort is genuinely maximal. Neuromuscular weight increases to 0.3 regardless of the primary zone classification.

**Traditional VO2max (5x5)** — longer sustained efforts at ceiling. Threshold secondary signal increases to 0.4 because the extended duration spends more time in the threshold-adjacent range during and between efforts.

**Float Sets** — elevated recovery keeps the cardiovascular system above threshold between efforts. Threshold secondary signal increases to 0.35.

#### Interpreting the Matrix

The weight matrix is not a precise physiological measurement — it is a structured approximation grounded in the energy system model from section 2.1. The exact weights are calibrated estimates that should be treated as a starting point subject to refinement as the system accumulates athlete data.

What the matrix provides is directionality and proportionality — the right systems move in the right direction by roughly the right relative amounts. A VO2max block advances the VO2max ladder fastest, advances threshold meaningfully, and barely touches aerobic base. A Z2 block advances aerobic base and nothing else. The signature profile evolves in a way that reflects what the athlete has actually been training.

### 4.5 Phase Transitions

A training plan moves through phases — base, build, peak, taper, recovery — and each phase has a dominant system that receives the primary training focus. Phase transitions are the moments when that dominant system shifts. The ladder system needs a principled way to handle these transitions without losing the progression history the athlete has built or artificially resetting work that doesn't need resetting.

#### The Core Principle

Levels are never deleted. A phase transition does not reset the signature profile — it changes which ladders are active and which are dormant. The threshold level an athlete built through a build phase is preserved when base phase returns. When threshold resumes its role as dominant system in the next build phase, the ladder picks up where it left off, adjusted only for any FTP recalibration that occurred in the interim.

This is the training biography principle from section 4.2 in action. The signature profile is a cumulative record. Phase transitions are chapters, not restarts.

#### Active and Dormant Ladders

At any point in the plan, ladders exist in one of three states:

**Active** — the dominant system for the current phase. Receives full prescription focus, full advancement logic, full session feedback. This is the ladder being actively climbed.

**Supporting** — secondary systems that receive sessions in the current phase but at reduced frequency and volume. Typically one level below their peak from the previous phase to reflect reduced stimulus. Advancement logic applies but at a slower rate — fewer sessions means slower progression.

**Dormant** — systems not being trained in the current phase. Level is preserved exactly as left. No advancement, no regression from disuse in the short term. Over extended periods of complete disuse a small decay may apply — see below.

#### Phase Dominant Systems

| Phase | Dominant system | Supporting systems | Dormant |
|---|---|---|---|
| Base | Aerobic base (Z2) | Tempo, sweet spot | Threshold, VO2max, anaerobic, neuromuscular |
| Build | Threshold | Sweet spot, aerobic base | VO2max, anaerobic, neuromuscular |
| Peak | VO2max | Threshold, anaerobic | Aerobic base, neuromuscular |
| Taper | None — load reduction | Light threshold or sweet spot | All others |
| Recovery | Active recovery only | None | All |

Neuromuscular work — the With Bursts pattern — sits outside this structure. Short neuromuscular activation efforts embedded in endurance rides are appropriate in any phase and do not constitute a neuromuscular ladder session. They maintain the system without advancing it. Dedicated neuromuscular ladder sessions are a peak phase tool.

#### The Handoff at a Phase Boundary

When a phase transition occurs and a ladder is mid-climb, the handoff logic applies:

**Dominant system transitioning to supporting** — the level is preserved. The ladder drops to supporting status and receives fewer sessions. The next session in that system when it resumes as dominant picks up at the preserved level. No step back is applied for the transition itself — the athlete earned that level and retains it.

**Supporting system transitioning to dominant** — the ladder resumes at the preserved level. If significant time has passed since the last session in that system a single consolidation session at the current level is prescribed before advancement resumes. This reestablishes the baseline before climbing further.

**Dominant system transitioning to dormant** — level preserved exactly. No action until the system becomes active or supporting again.

**FTP recalibration at a phase boundary** — if a phase transition coincides with an FTP test and recalibration, the recalibration is applied first and then the phase transition logic follows. The new phase begins with recalibrated levels.

#### Level Decay for Extended Dormancy

Fitness is not permanent. A system that receives no stimulus for an extended period will lose some of the adaptation it built. The decay is not applied mechanically on a timer — it is triggered by the gap between the last session in a system and the current date at the point the system is reactivated.

A gap of up to six weeks: no decay applied. The athlete retakes the ladder at the preserved level with a consolidation session.

A gap of six to twelve weeks: small decay applied — typically 0.2 to 0.5 level reduction depending on the system. Higher intensity systems decay faster than aerobic base, which is the most durable adaptation.

A gap beyond twelve weeks: meaningful decay applied. The seeding logic from section 4.2 may be re-run against the current power profile rather than relying on the preserved level, as the profile itself may have shifted.

Aerobic base decays slowest — the mitochondrial and cardiac adaptations built over years are the most durable. Neuromuscular and anaerobic sharpness decay fastest — they require frequent stimulus to maintain. Threshold and VO2max sit between these extremes.

#### Taper and Recovery Phase Handling

Taper and recovery phases are protected from advancement logic. No ladder advances during a taper or recovery phase regardless of session outcome. The athlete may complete sessions that would ordinarily trigger advancement — this is expected and positive, reflecting the fitness the block has built — but the level holds. The taper is not for building; it is for expressing what has already been built.

Sessions completed during taper that would have triggered advancement are noted. When the next build phase begins the first session starts from the preserved level with the accumulated advancement signal carried forward as a positive indicator for early rung progression.

Recovery phases work similarly — levels hold, sessions are restorative only, and the ladder resumes when recovery is complete.

---

## Session Modifiers

Some training variables operate outside the core intensity-duration relationship but meaningfully change the physiological stimulus a session produces. These are session modifiers — prescribable variables that adjust the weight matrix, the expected discomfort profile, and the recovery requirement for a session without changing its zone classification.

Modifiers are not fully specified in this document. The principle is established here; implementation detail is deferred to the system layer where context determines appropriateness.

### Cadence

At a given power output, cadence determines how that power is produced and therefore which system is predominantly stressed to generate it. The body does not know speed or Strava segments — it knows pressure and duration. Cadence changes the distribution of that pressure.

**Low cadence, high torque (approximately 50–65rpm).** The same watts at low cadence requires significantly higher peak force per pedal stroke. Cardiovascular demand is lower than expected for the power output — heart rate will often sit below the normal zone range — but muscular and neuromuscular demand increases substantially. Fast-twitch fibre recruitment rises, muscular endurance is stressed, and the session load shifts from the cardiovascular system toward the musculoskeletal system. A threshold session at 55rpm is a different physiological prescription from a threshold session at 90rpm despite identical power and duration.

**High cadence, low torque (approximately 100–110rpm).** The inverse shift. Peak force per stroke drops and demand moves back toward the cardiovascular and neuromuscular coordination systems. Used to develop pedalling efficiency, cardiovascular output, and as a recovery tool when the legs carry fatigue but cardiovascular capacity remains.

**Cadence as a progression axis.** Cadence can be a prescribable rung variable independent of intensity and duration. Progressively reducing cadence across a block at fixed power and duration is a legitimate extensive progression of the muscular endurance pathway. This is an additional ladder axis not yet fully specified in the level system.

**Weight matrix implication.** A low cadence session modifier increases the neuromuscular and musculoskeletal weights and reduces the cardiovascular weight relative to the base zone matrix. The exact adjustment is system-layer implementation detail.

### Heat Training

Training in elevated ambient temperature adds a thermoregulatory stress on top of the base physiological stimulus. The same session in heat costs more than in temperate conditions — heart rate is elevated, perceived effort is higher, plasma volume is stressed, and recovery requirement is longer. The power file looks identical; the physiological cost is not.

**The primary adaptation mechanism** is plasma volume expansion — an increase in blood plasma volume that improves cardiovascular efficiency, oxygen delivery, and thermoregulation. This adaptation develops within days of consistent heat exposure and produces performance benefits even in cool conditions, making heat training a legitimate performance tool for athletes who cannot access altitude.

**As a session modifier** heat amplifies the cardiovascular weight of any session and adds a thermoregulatory stress component that sits outside the normal weight matrix. A below-target session in prescribed heat conditions should not trigger the standard escalation protocol without first accounting for the environmental cost — the athlete may have executed the session correctly under conditions that made the targets genuinely harder to reach.

**Welfare flag.** Heat training carries a genuine risk profile that temperate training does not. Inadequate hydration, excessive progression of heat exposure, or illness superimposed on heat stress can escalate from performance impairment to medical risk quickly. Any heat training prescription should carry an explicit welfare awareness flag. The physical interruption protocol from section 3.3 applies with heightened sensitivity in heat conditions.

**Implementation status.** Heat training is a known modifier that the system should be aware of and able to account for when present. Full implementation — specific weight adjustments, RPE correction factors, recovery timeline modifications — is deferred. The principle is established here.

---

## Skill Sessions

Cycling is not purely a physiological contest. Every discipline demands a specific technical skill set that is as trainable as threshold power and as performance-relevant as VO2max. The strongest physiological engine does not automatically win a criterium, a cyclocross race, or a technical descent. Skill deficits cost time and create risk that no amount of interval work can recover.

Skill development operates through a different adaptation mechanism than physiological training — neuromuscular pattern development, movement efficiency, and confidence under pressure built through deliberate practice rather than physiological load. Skill sessions are a legitimate and important category of training that sits alongside the physiological framework rather than within it.

**Never be afraid to programme a skill session.** An active recovery ride conducted on a field learning to corner for the next cyclocross race is time genuinely well spent. The physiological cost is negligible. The performance return is real.

### What Skill Sessions Are

A skill session targets a specific technical competency relevant to the athlete's discipline and goals. The physical load is incidental — typically Z1 to Z2, never the point. The primary adaptation target is movement pattern, technique, and the confidence to execute under race conditions.

Examples by discipline:

**Road** — descending technique, cornering at speed, bunch riding and positioning, sprint lead-out timing, effective use of drafting.

**Criterium** — tight cornering, accelerating out of corners, repeated short efforts in traffic, positioning in a fast bunch.

**Cyclocross** — dismount and remount, running with the bike, technical off-camber cornering, mud and sand riding, barrier technique.

**Track** — standing start, flying effort, bunch race tactics, derny and motor pacing technique.

**Gravel and off-road** — technical descent, loose surface cornering, line choice, bike handling in varied terrain.

### Skill Sessions and the Physiological Framework

Skill sessions do not advance any physiological ladder. They carry negligible weight matrix values across all energy systems — the physiological load is too low and too incidental to meaningfully move any level. The system should not attempt to extract a training stress signal from a skill session beyond its recovery-level physiological cost.

What skill sessions do affect:

**Recovery.** A skill session conducted at Z1–Z2 is simultaneously restorative and productive. Active recovery riding does not have to be mindless laps — it can be purposeful technical work that develops a real performance quality without adding physiological cost. This is a legitimate and underused prescription tool, particularly in recovery weeks and taper phases where the athlete has time and energy that cannot productively be directed at physiological load.

**Confidence.** Skill competence directly affects race performance in ways that do not appear on a power file. An athlete who corners confidently carries more speed through every bend. One who descends well recovers position without burning matches. These are performance gains the physiological framework cannot measure but the coach knows are real.

**Race readiness.** In the weeks before a target event, discipline-specific skill work is as important as physiological peaking. An athlete arriving at their target cyclocross race having practised dismounts in the week before is better prepared than one who did an extra threshold session instead.

### Skill Progression

Skill development has its own progression logic that is distinct from the physiological ladder system. It is not defined in detail here — skill progression is discipline-specific, highly individual, and requires a different kind of feedback than power and RPE. What the framework establishes is that:

Skill sessions are a first-class session type. They are prescribed deliberately, not filled in as afterthoughts. They have a specific technical target, a defined environment, and a purpose the athlete understands.

Skill sessions belong in the plan at any phase. Recovery weeks, base phase, pre-competition blocks — there is no phase in which technical development is inappropriate. The prescription system should be willing to suggest a skill session whenever the athlete has a technical limiter relevant to their target event and the physiological load permits it.

*"The bike does not know your FTP. Programme the skills."*

---

## Race Practice

*"The best predictor of performance is performance itself."* — Andrew Coggan

Coggan's principle applies not only to understanding an athlete's current capability but to how they are prepared for their target event. Sometimes the most appropriate prescription is not an interval session — it is a race.

A competitive event in a training block is not a distraction from the plan. Used correctly it is the plan — a performance expression that delivers race-specific stimulus, tests equipment and fuelling choices under pressure, and builds the confidence and tactical experience that no training session can fully replicate. Never be afraid to plan a practice race in place of a session.

### Two Criteria for Race Substitution

Not every available race is an appropriate substitution. Two questions determine whether a race belongs in the plan:

**Does the event match the goal?** A race that closely resembles the target event in character — discipline, duration, intensity profile, terrain — serves double duty. It is simultaneously a training stimulus and a rehearsal. Equipment choices can be tested. Fuelling strategy can be practised under race conditions. Warm-up routines can be refined. Tactical and positional habits can be developed. The athlete arrives at their target event having already rehearsed it rather than experiencing it for the first time.

**Does the event match the planned stimulus?** The physiological cost and recovery requirement of the race should be broadly comparable to the session it is replacing. A local criterium that maps to a threshold session in duration and intensity profile is a reasonable and often superior substitution — it delivers the intended stimulus in a context that also develops race-specific skills and confidence. A three-day stage race is not a substitute for a single threshold session. The accumulated load, the multi-system stimulus, and the recovery requirement are categorically different. Replacing a planned session with a race that vastly exceeds its physiological cost is not creative training — it is an unplanned overload that will compromise subsequent sessions and potentially the entire block.

### Race Practice in the Prescription

When a race is prescribed in place of a session the system should:

Record it as a race substitution with the matched session type noted. The physiological load is assessed against the planned session after the fact and the ladder responds accordingly — a race that delivered the equivalent of an on-target threshold session advances the threshold ladder as such.

Flag any significant deviation from the planned stimulus. A race that substantially exceeded the planned load is noted and the recovery requirement adjusted. The following sessions are assessed in the context of the additional load carried from the race.

Treat the race as an opportunity for the full athlete picture — not just power data but fuelling execution, equipment performance, tactical decision-making, and subjective race feel. These are inputs the system cannot fully quantify but the coach should note.

*"Race the races that make you better at the races that matter."*

---

## Worked Example: End-to-End Trace

This is a single concrete trace through the system, from profile to prescription to level change. Its purpose is to make the logic above unambiguous by showing it executed once. The numbers are illustrative but internally consistent with the rules in Layers 2 and 4.

**The athlete.** Experienced amateur. FTP 280W, established three weeks ago. Power curve reads as Coggan "Good" at long durations and "Moderate" at 5 minutes — a diesel phenotype, strong sustained power, weaker top end. Fresh curve holds well; durability not yet a limiter. Target event in nine weeks: a hilly road race with repeated 3–5 minute selections — a VO2max demand the athlete's profile currently undersupplies. This is a limiter, not an edge: the prescription is in limiter-removal mode.

**Seeding.** Threshold seeds from FTP against the Coggan bands — "Good" maps to roughly level 6, capped into the lower half of the band, seeded at 5.5. VO2max seeds from 5-minute power — "Moderate" maps to roughly level 4–5, seeded at 4.2. The gap between threshold (5.5) and VO2max (4.2) confirms the limiter the race demands.

**Phase and mode.** Nine weeks out, the athlete is in a build phase shifting toward peak. VO2max is the dominant system to develop. The VO2max ladder is active at level 4.2.

**Today's prescription (single-session mode).** The daily Go/No-Go layer returns Go — subjective scores good, HRV and resting HR stable, no accumulated red flags. At level 4.2 the athlete is in the foundational-development band and traditional 5×5 is not yet appropriate. The ladder routes through the micro-burst bridge (section 2.2): an on-offs session at Z5. Prescription: warm-up, settle, then 3 sets of 9 minutes of 30s on / 15s off, work intervals targeting Z5 (≈105–115% FTP, so 294–322W), 5 minutes easy between sets, cool-down. Expected work RPE for Z5: 8–9.

**The result.** The athlete completes all three sets. File analysis (section 2.3 scoring):
- All work intervals average within the 294–322W band. No interval drifts below the lower edge for a sustained stretch. → all intervals pass.
- Power-to-HR decoupling across the work portion: 3%. Below the ~5% flag. → clean.
- Inter-set trend: power holds across all three sets. → no progression fade.
- Reported work RPE: 7.5.

**Scoring the outcome (section 4.3).** Targets met, so the RPE comparison applies. Expected Z5 band is 8–9; reported 7.5 sits 0.5 below the lower edge. That falls in the "Below (Above target)" tier: **+0.2**. The athlete had a little reserve — consistent with a rung seeded conservatively.

**The level change and matrix flow (section 4.4).** Primary increment +0.2 flows to secondary systems via the VO2max (Z5) weights, with the On-Offs pattern modifier raising the anaerobic weight from 0.1 to 0.2:

| System | Weight | Signal | Result |
|---|---|---|---|
| VO2max (primary) | 1.0 | +0.2 × 1.0 | **+0.20** → 4.2 to 4.4 |
| Threshold | 0.3 | +0.2 × 0.3 | +0.06 |
| Aerobic base | 0.1 | +0.2 × 0.1 | +0.02 |
| Anaerobic (modified) | 0.2 | +0.2 × 0.2 | +0.04 |
| Neuromuscular | 0.0 | — | 0 |

All contributing weights sit at or above the 0.05 noise floor, so all carry through. The VO2max ladder advances to 4.4; threshold, aerobic base, and anaerobic accrue small secondary gains that accumulate across the block. The signature profile updates as a whole.

**What the system does next.** With a Go signal and a clean +0.2, the next VO2max session steps to level 4.4 — slightly longer work intervals or a marginally tighter work:rest ratio, per the micro-burst progression toward 40/20 and eventually traditional intervals. The block continues raising the roof so the athlete's threshold has room to follow, and the durability and skill layers are slotted in around the quality work as the readiness signal permits. Had the RPE come back at 9.5 instead, the same completed session would have scored at-the-limit (+0.0) and the system would have held 4.2 to consolidate rather than advanced.

---

## A Note on Principles Over Rules

The numbers, weights, thresholds, and decision rules in this document are grounded estimates — informed by physiology, coaching experience, and reasoned approximation. They are not empirical constants derived from a large controlled dataset. They are the best available starting point.

Coaching is irreducibly individual. Two athletes with identical power profiles, identical FTP, and identical training histories will respond differently to the same prescription. The system will encounter situations the rules do not cleanly resolve — an athlete whose response pattern fits none of the standard categories, a week where every signal points in a different direction, a session outcome that defies simple classification.

In these cases the system should reason from the principles that generated the rules rather than defaulting to the nearest approximation. The weights in the matrix are informed estimates of physiological reality — when the system's output conflicts with the observable reality of a specific athlete, the observable reality wins. The goal is always the right outcome for this athlete today, not strict adherence to a framework.

The framework exists to make good decisions consistently and transparently. It is a thinking tool, not a replacement for thinking.

*"It depends"* is not a failure of the system. It is the honest answer to most coaching questions, and the system's job is to read what it depends on as clearly as possible and respond accordingly.

---

## Changelog

- v0.1 — skeleton drafted.
- v0.2 — Section 2.1 written: energy system physiology locked in as zone foundation, zone model table with FTP expression ranges, boundary zone notes.
- v0.3 — Section 1.6 written: full structural pattern menu defined and formalised. Sweet spot clarified as intensity range not zone. Long Suprathreshold parked for later.
- v0.4 — Section 1.1 written: effort block defined. Dual intensity expression modes (absolute and zone-relative) locked in. Validity criteria established.
- v0.5 — Section 1.4 written: TSS as load metric, its fundamental limitation formalised ("not all TSS is created equal"), fatigue type dimensions defined, stimulus tagging introduced as the system's response to the limitation.
- v0.6 — Sections 1.2, 1.3, 1.5 written: recovery intervals (character, duration ratios, corruption), session shape (warm-up/settle/main set/cool-down by session type), intensity expression hierarchy (absolute watts, % FTP, zone-relative, RPE).
- v0.7 — Section 1.7 written: extensive vs intensive progression defined, power curve as diagnostic, phenotype implications, progression mode as explicit ladder property.
- v0.8 — Section 2.2 written: intensity domain vs stimulus type, sweet spot vs threshold distinction, VO2max traditional vs micro-burst formats including confidence bridge and bridging ladder function.
- v0.9 — Preamble written: Coggan framing, interval training premise, two modes of curve modification, limiter removal vs blade sharpening, individual response variability.
- v0.10 — Section 2.3 written: stimulus specificity, failure modes (drift, hero watts, recovery signals, sudden cessation), interval-level reading signals, grey zone problem defined and guarded against.
- v0.11 — Section 2.4 written: one session one stimulus principle, weekly sequencing, fatigue resistance exception, cross-system fatigue profiles, overtraining vs under-recovery distinction, training age and mental success principle.
- v0.12 — Sections 3.1 and 3.2 written: acute responses across four fatigue dimensions with recovery timeline table, chronic adaptations by system with timescales, adaptation hierarchy formalised.
- v0.13 — Section 3.2 extended: house model added — foundation/walls/roof/raising the roof as sequencing logic, thin threshold returns as VO2max block signal, genetic ceiling and fractional utilisation nuance.
- v0.14 — Section 3.3 written: three-outcome framework (completed/degraded/stopped), completion spectrum defined, expected discomfort profiles by stimulus type, mental framing principle and duration decomposition, session interruption protocol with five cause categories.
- v0.15 — Section 3.3 terminology pass: failure language removed throughout. Sessions are now on target, below target, or interrupted. Reframed as result or lesson. Edison principle and constructive language applied consistently.
- v0.16 — Section 3.4 written: supercompensation model, adaptation-in-recovery principle, timing errors, load errors, productive overreach vs overtraining, maladaptation signals, prescription response.
- v0.17 — Section 3.4 extended: heart rate diagnostics added (elevated HR at submaximal effort, suppressed HR at maximal effort, resting metrics), habituation stagnation defined and distinguished from overreach stagnation, two-question diagnostic framework.
- v0.18 — Sections 4.1 and 4.2 written: 1.0–10.0 decimal level scale defined, natural level bands, feedback resolution mapped to decimal increments, signature profile concept formalised, seeding logic with deliberate cap, FTP recalibration nudge mechanic, phase transition level preservation.
- v0.19 — Section 4.2 extended: Coggan power profile as seeding anchor, system-specific seeding from characteristic durations, w/kg for seeding only then converted to FTP-relative, body weight and body image position stated explicitly.
- v0.20 — Section 4.3 written: tiered advancement (+0.1/+0.2/+0.3/hold/escalate), below-target escalation protocol (one miss/two miss/three miss), cause modifies response, physical bypass, mental escalation, step back mechanic, counter reset on success.
- v0.21 — Section 4.4 written: slider model formalised, weight matrix defined for all zones with 0.05 noise floor, pattern modifiers for Over-Unders/On-Offs/Hard Starts/Attacks/Traditional/Float Sets, matrix interpretation note.
- v0.22 — Section 4.5 written: active/dormant/supporting ladder states, phase dominant system table, handoff logic at phase boundaries, level decay for extended dormancy by system type, taper and recovery phase protection.
- v0.23 — Closing principles statement added: principles over rules, "it depends" as honest coaching answer, framework as thinking tool not replacement for thinking.
- v0.24 — Session modifiers section added: cadence (low torque/high torque shift, cadence as progression axis, weight matrix implication) and heat training (plasma volume mechanism, cardiovascular amplifier, welfare flag, implementation deferred).
- v0.25 — Skill sessions section added: first-class session type alongside physiological framework, discipline-specific examples, recovery and confidence and race readiness dimensions, skill progression as parallel track.
- v0.26 — Race practice section added: Coggan principle applied to race substitution, two criteria (event match and stimulus match), stage race anti-pattern named explicitly, prescription handling for race substitutions.
- v0.27 — Document structure pass: Layer 1 and Layer 2 headings restored, sections 1.4 and 1.5 reordered into correct sequence, numbered subsection headings cleaned throughout (1.4.x and 2.1.x), heading hierarchy consistent throughout.
- v0.28 — Science and grounding review pass (external critique folded in):
  - Scope and System Context section added: data sources (power profile, intervals.icu, Go/No-Go layer), two modes of use, explicit out-of-scope exclusions (female physiology, off-bike strength), ontology-vs-protocol filename collision flagged.
  - LT1 placement corrected: moved from the Z3/Z4 boundary to the Z2/Z3 boundary; Z3 and Z4 zone rows and the boundary-zones paragraph reconciled accordingly. LT2 placed near FTP at the top of Z4.
  - Supercompensation reframed as a teaching metaphor; fitness-fatigue (impulse-response, PMC) named as the model the system actually reasons on; Go/No-Go layer named as the readiness proxy.
  - House model softened: VO2max ceiling no longer stated as an absolute wall; trainable fractional utilisation reconciled. Thin-threshold-returns signal given alternative diagnoses before defaulting to a VO2max block.
  - CNS fatigue mechanism hedged; 48-72h timeline marked as a heuristic.
  - Expected work-RPE-by-zone table added to section 3.3 as the reference for the advancement tiers.
  - Tier thresholds defined numerically in section 4.3 (RPE gap to band edge); RPE-vs-power disagreement rule added.
  - Session outcome scoring spec added to section 2.3 ("Scoring a Session from the File") — per-interval pass rules, decoupling flag, time-in-band for sustained/endurance, HR fallback.
  - Durability / fatigued power curve added as a second diagnostic dimension (section 1.7) and to the signature profile (section 4.2).
  - Worked end-to-end example appended.
  - Sweet spot's presence in the zone-keyed weight matrix clarified.
