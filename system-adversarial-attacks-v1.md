# SYSTEM Training — Adversarial Attack Suite V1

**Status:** Pre-implementation architecture gate  
**Purpose:** Break the SYSTEM training architecture before production code is written.

---

## 0. Purpose

This is the canonical adversarial test suite for SYSTEM's training data model and progression pipeline.

The objective is to determine whether an athlete can manufacture progression from fiction, inflate credit, poison baselines, manipulate prescriptions, replay records, exploit corrections, manipulate time, spoof evidence, farm PRs/quests/achievements, manipulate rank, or force a manual/camera architecture split.

**Core principle:**

> History may contain ugly or untrusted claims. Progression is not required to believe them.

---

# 1. Canonical Pipeline

**Input → Write Boundary → Immutable Training Record → Evaluation → Credit Ledger → Projections**

### Write Boundary
Authentication, authorization, schema validation, canonical unit conversion, idempotency, rate limits, lifecycle transitions, and rejection of client-supplied trust/progression fields.

### Training Record
Preserves the original claim. Raw facts are append-only after acceptance.

### Evaluator
Derives claim state, effective role, plausibility, evidence interpretation, conflict resolution, credit inputs, baseline inputs, and PR/rank eligibility inputs. It does not directly mutate XP balances.

### Credit Ledger
Handles grants, compensations (V1 reversal/clawback is the `COMPENSATION` kind), policy pinning, economic idempotency, caps, concavity, and lineage.

### Progression
Handles Level, Quest predicates, Achievement predicates, Rank predicates, and derived capability projections.

---

# 2. Required State Separation

Keep these concepts separate:

- `claim_state`
- `credit_tier`
- `rank_eligibility`
- evidence/provenance
- progression ledger
- history

A coherent manual claim may be valid history while receiving limited/zero progression credit and no rank eligibility.

---

# 3. Attack Suite

## A1 — Manual Fiction Inflation

**Attack:** Repeatedly self-report implausible but syntactically valid work, such as 5×5 @ 140 kg squat every day without actually doing it.

**Failure:** Manual VALID claims continuously generate meaningful progression.

**Required behavior:** Preserve claims in history; cap manual credit globally and per family; apply diminishing returns; exclude manual-only claims from rank by default; prevent the manual lane from controlling baselines/prescriptions.

**Invariant:** Manual history ≠ verified capability ≠ rank eligibility.

**Layer:** Evaluator + Credit Ledger + Progression.

---

## A2 — History Smear Through Rejected or Voided Records

**Attack:** Flood history with rejected/voided garbage and cause a baseline, PR, volume graph, or capability projection to aggregate it.

**Failure:** Sacred history contaminates derived state.

**Required behavior:** Baseline/PR/capability projections consume only explicitly eligible records. Rejected and voided records remain auditable but do not feed official projections.

**Invariant:** Raw history is complete; derived capability is filtered.

**Layer:** Projection + Evaluator.

---

## A3 — Client UUID Replay / Duplicate Retry

**Attack:** Submit the same payload repeatedly with the same `client_event_id`.

**Failure:** Network retries create multiple records or credits.

**Required behavior:** `client_event_id` is idempotent per athlete. Duplicate submission returns/references the original and creates no second economic grant.

**Invariant:** Retrying an accepted event is economically equivalent to not retrying it.

**Layer:** API + Ledger.

---

## A4 — Semantic Duplicate With New UUID

**Attack:** Submit materially identical performance repeatedly while generating a new UUID every time.

**Failure:** UUID idempotency is bypassed.

**Required behavior:** Secondary semantic duplicate detection. Default duplicate action: preserve history, zero progression credit, suspicion/quarantine as appropriate, and no crisp duplicate oracle.

**Invariant:** Changing an idempotency UUID does not make materially identical claims economically independent.

**Layer:** Write Boundary + Evaluator.

---

## A5 — Same-Workout Identical Sets False Positive

**Attack:** A legitimate prescribed 5×5 contains five identical sets; naive semantic dedupe marks them duplicates.

**Failure:** Normal workouts are damaged by anti-fraud logic.

**Required behavior:** Same-workout sets with distinct sequence indices are legitimate. Semantic dedupe targets cross-workout/cross-submit clones.

**Invariant:** Distinct sequence positions in one accepted workout are not automatically duplicates.

**Layer:** Evaluator.

---

## A6 — Edit After Credit

**Attack:** Log 3×8 @ 60 kg, receive credit, then change it to 3×8 @ 120 kg.

**Failure:** Mutable facts after mint.

**Required behavior:** Accepted actual performance is frozen. Corrections are append-only events and may trigger re-evaluation/compensation.

**Invariant:** No silent mutation of an economically evaluated raw fact.

**Layer:** Write Boundary + Ledger.

---

## A7 — Correction Oscillation Farming

**Attack:** Repeatedly change high → low → high to probe evaluator behavior.

**Failure:** Corrections become an economic faucet.

**Required behavior:** Time-boxed/rate-limited corrections, event sourcing, normally non-crediting self-serve corrections, explicit compensation lineage, no net-positive correction loop.

**Invariant:** Correction cannot become a progression faucet.

**Layer:** API + Credit Ledger.

---

## A8 — PENDING Rewrite Channel

**Attack:** Accept a low claim, then alter it to a high claim while evaluation is pending.

**Failure:** PENDING becomes an economically mutable staging area.

**Required behavior:** `accept` freezes actuals. Editing is allowed only in pre-accept draft state. Post-accept changes are correction events.

**Invariant:** PENDING is not a hidden draft state.

**Layer:** Schema + Write Boundary.

---

## A9 — Warm-Up XP Farming

**Attack:** Log hundreds of empty-bar or very-light sets as warmups.

**Failure:** Warmups generate progression.

**Required behavior:** Warmups remain historical and do not receive normal progression credit. No set-count bonus.

**Invariant:** Warm-up volume cannot become an XP faucet.

**Layer:** Evaluator + Credit Policy.

---

## A10 — Warm-Up Laundering

**Attack:** Log light sets as WORK, receive credit, then relabel them.

**Failure:** Role is a soft economic label.

**Required behavior:** Derive `effective_role`; evaluator may infer warmup from load relative to credible working bands. Credit, PR, and baseline use effective role.

**Invariant:** Changing the role label alone cannot manufacture working-set credit.

**Layer:** Evaluator.

---

## A11 — Prescription Sandbagging

**Attack:** Choose an easy target, such as 3×5 @ 40 kg when credible capacity is 100 kg, then claim full progression for completing it.

**Failure:** Compliance with a weak target becomes progress.

**Required behavior:** System/quest prescriptions are authoritative. User-edited easier targets become free/reduced-credit intent. Prescription authority uses official/conservative baselines.

**Invariant:** Athletes cannot manufacture difficulty by choosing weaker targets.

**Layer:** Evaluator + Progression.

---

## A12 — Prescription Overstatement

**Attack:** Create an absurd prescription and claim large partial credit for ordinary work.

**Failure:** Fantasy denominators generate economic reward.

**Required behavior:** Credit is bounded by productive capacity: `min(prescribed demand, productive cap)`, with constrained cap inputs.

**Invariant:** Impossible prescriptions cannot create additional economic value.

**Layer:** Evaluator + Credit Policy.

---

## A13 — Baseline Crash

**Attack:** Deliberately log weak work until normal work appears extraordinarily intense.

**Failure:** Baseline is attacker-controlled.

**Required behavior:** Baselines favor credited working sets, robust statistics, evidence weighting, slow downward movement, and conservative floors.

**Invariant:** An athlete cannot cheaply manufacture an extreme relative-intensity multiplier by tanking baseline history.

**Layer:** Evaluator.

---

## A14 — Productive-Cap Manipulation

**Attack:** Manipulate variables feeding `productive_cap()`.

**Failure:** One opaque function becomes the new god-variable exploit.

**Required behavior:** Cap inputs are explicit and testable. Bound upward/downward movement and use evidence-weighted top sets, minimum samples, and snapshot isolation.

**Invariant:** Productive capacity cannot change arbitrarily because of small attacker-controlled inputs.

**Layer:** Evaluator.

---

## A15 — Client Trust-Field Spoof

**Attack:** Send `claim_state=VALID`, `credit_tier=VERIFIED`, `rank_eligibility=true`, `evidence_class=CAMERA`, or XP directly.

**Failure:** Client controls progression semantics.

**Required behavior:** All trust/progression fields are server-derived.

**Invariant:** No client request can directly grant trust, XP, or rank eligibility.

**Layer:** API.

---

## A16 — Dual Clock Confusion

**Attack:** Use claimed timestamps for a perfect training narrative while economy uses server receipt time.

**Failure:** Consumers treat both clocks as equivalent.

**Required behavior:** Every projection declares its clock: `DISPLAY_CLAIMED` or `ECONOMY_RECEIVED`. Rank/credit/caps use economy time. Review uses received time as primary.

**Invariant:** No progression consumer silently substitutes claimed time for economy time.

**Layer:** Schema + Projection.

---

## A17 — Timestamp Compression

**Attack:** Claim 30 heavy working sets occurred in four minutes.

**Failure:** Workout-level timing cannot detect impossible work rate.

**Required behavior:** Set-level timestamps or ordered timing. Impossible rates become partial/rejected for progression while raw claims remain.

**Invariant:** Impossible work cannot be packed into tiny elapsed time to bypass limits.

**Layer:** Evaluator.

---

## A18 — Timestamp Expansion / Timezone Hopping

**Attack:** Manipulate local clocks/timezones to make one training period appear across multiple days.

**Failure:** Daily caps use client-local dates.

**Required behavior:** Economy uses server UTC epoch windows. Claimed timestamp/offset are retained for display.

**Invariant:** Timezone changes cannot reset economic limits.

**Layer:** API + Credit Policy.

---

## A19 — Future-Dated Claim

**Attack:** Submit a workout with a future timestamp.

**Failure:** Future work reserves/manipulates future caps.

**Required behavior:** Future claims receive no progression credit and may be rejected. `received_at` controls economy.

**Invariant:** Client timestamps cannot mint future progression.

**Layer:** Write Boundary.

---

## A20 — Backdated Volume Injection

**Attack:** Submit a huge “last month's” training block after learning the rules.

**Failure:** Historical timestamp controls current economic treatment.

**Required behavior:** Bound backdating; use server receipt time for economy; preserve claimed time as metadata; constrain delayed batches.

**Invariant:** Backdating cannot manufacture economic capacity.

**Layer:** API + Ledger.

---

## A21 — Offline Stash and Reorder

**Attack:** Accumulate offline records and submit easy baseline-setters before harder claims.

**Failure:** Evaluation assumes submission order is actual chronology.

**Required behavior:** Economy uses `received_at`; baseline snapshots are pinned; large offline batches face rolling limits.

**Invariant:** Offline submission order cannot strategically rewrite the economic timeline.

**Layer:** API + Evaluator.

---

## A22 — Evaluation Race

**Attack:** Trigger concurrent evaluations for the same record.

**Failure:** Both workers grant credit.

**Required behavior:** Per-record single-flight evaluation and serialized ledger writes.

**Invariant:** Concurrent evaluation cannot create concurrent economic grants.

**Layer:** Evaluator + Ledger.

---

## A23 — Re-Evaluation Money Printer

**Attack:** Attach no-op evidence, trigger re-evaluation, or bump evaluator version.

**Failure:** Each evaluator version creates another grant.

**Required behavior:** Economic uniqueness is anchored to the training claim and credit policy, not evaluator version. Re-evaluation creates compensating entries where necessary.

**Invariant:** Re-evaluation cannot create net-new economic value merely because evaluator version changed.

**Layer:** Credit Ledger.

---

## A24 — Correction / Re-Evaluation Ordering Attack

**Attack:** Exploit a race where an old grant remains while a corrected evaluation grants a new amount.

**Failure:** Both values remain economically active.

**Required behavior:** Explicit lineage and serialized economic transitions. Projection rebuild must equal the live result.

**Invariant:** Every correction/re-evaluation transition has one deterministic net economic result.

**Layer:** Ledger.

---

## A25 — Policy Version Hoarding

**Attack:** Queue offline submissions until a generous formula deploys.

**Failure:** Deployment timing becomes an economic lottery.

**Required behavior:** Credit policy is pinned at acceptance. Formula changes are prospective.

**Invariant:** Formula deployment timing cannot be exploited.

**Layer:** API + Ledger.

---

## A26 — Manual Baseline Prescription Poisoning

**Attack:** Poison manual baseline so future system prescriptions become easier.

**Failure:** Manual history controls official prescription authority.

**Required behavior:** Use official/evidence-backed lane when present; otherwise conservative standards floors. Never blindly trust attacker-shaped manual percentiles.

**Invariant:** Manual-only history cannot indefinitely lower official prescription difficulty.

**Layer:** Evaluator + Progression.

---

## A27 — Easy Credited Stimulus Quest Farming

**Attack:** Perform endless easy walks, mobility holds, or low-intensity work and use credit totals to complete quests.

**Failure:** Quest becomes a habit checklist.

**Required behavior:** Quest success requires meaningful stimulus predicates, intensity bands, and where appropriate family diversity and authoritative prescription compliance.

**Invariant:** Low-effort ledger activity alone cannot complete meaningful quests.

**Layer:** Progression.

---

## A28 — Session Completion Bonus Farming

**Attack:** Log a short walk and receive a generic completed-workout bonus.

**Failure:** Opening the logger becomes the game.

**Required behavior:** No session-completion XP. Credit derives from eligible set/segment stimulus.

**Invariant:** Workout completion itself has no generic economic reward.

**Layer:** Credit Policy.

---

## A29 — Micro-Set Packing

**Attack:** Submit 200 single-rep sets to exploit per-set minimum credit.

**Failure:** Set count becomes an XP faucet.

**Required behavior:** No per-set credit floor. Group stimulus with strong concavity.

**Invariant:** Fragmenting identical work cannot increase net credit.

**Layer:** Evaluator + Ledger.

---

## A30 — Cardio Physics Fantasy

**Attack:** Claim 10 km walking in 12 minutes.

**Failure:** Cardio is treated as generic numbers.

**Required behavior:** Typed locomotion/cardio segments; pace derived; impossible sport-specific performance rejected/constrained for progression.

**Invariant:** Impossible locomotion cannot mint credit.

**Layer:** Schema + Evaluator.

---

## A31 — AMRAP Rate Fraud

**Attack:** Claim 120 burpees in 60 seconds.

**Failure:** AMRAP accepts arbitrary rep counts.

**Required behavior:** Family-specific rate plausibility. Absurd rates become partial/rejected for progression.

**Invariant:** AMRAP cannot bypass performance-rate plausibility.

**Layer:** Evaluator.

---

## A32 — Duration Infinity

**Attack:** Claim a three-hour plank.

**Failure:** Duration scales linearly forever.

**Required behavior:** Concave duration credit and hard family caps. Excess may remain historical without progression credit.

**Invariant:** Unbounded duration cannot produce unbounded progression.

**Layer:** Evaluator + Credit Policy.

---

## A33 — Evidence Class Spoof

**Attack:** Upload an empty/unrelated video while claiming CAMERA.

**Failure:** Media presence automatically upgrades trust.

**Required behavior:** Evidence class is server-derived. Media alone is insufficient. Trust upgrade requires successful observation extraction.

**Invariant:** Attachment presence is not evidence quality.

**Layer:** Evidence Pipeline + Evaluator.

---

## A34 — Borderline Observation Gaming

**Attack:** Submit adversarial clips designed to barely pass a confidence threshold.

**Failure:** One confidence number becomes a trust switch.

**Required behavior:** Use multiple signals where available: detected performance, timing, pose/camera checks, completeness, assertion agreement. Borderline/conflicting observations receive reduced trust or no rank eligibility.

**Invariant:** A single confidence scalar cannot independently manufacture verified capability.

**Layer:** Evaluator.

---

## A35 — Camera / Manual Conflict

**Attack:** Manual claim says 10 reps; camera observes 6.

**Failure:** Manual automatically wins.

**Required behavior:** Display both. Credit/PR/rank use higher-trust observation under explicit conflict rules. Large disagreement prevents rank eligibility and may reduce credit.

**Invariant:** Manual assertion cannot override materially stronger contradictory evidence.

**Layer:** Evaluator.

---

## A36 — Wrong Video / Wrong Set

**Attack:** Attach yesterday's strong video to today's weak set.

**Failure:** Evidence is trusted solely because it references a valid set.

**Required behavior:** Observation timestamps must align with the set session-clock window within policy tolerance.

**Invariant:** Evidence must be temporally attributable to the claim it supports.

**Layer:** Evidence Pipeline + Evaluator.

---

## A37 — Same Media Reuse

**Attack:** Attach one video to 40 sets.

**Failure:** One artifact multiplies into many verified claims.

**Required behavior:** Exact content hash plus perceptual fingerprint where feasible, temporal alignment, and reuse limits.

**Invariant:** Evidence cannot be multiplied indefinitely across unrelated claims.

**Layer:** Evidence Pipeline.

---

## A38 — Camera Fork

**Attack:** Implement camera training as a separate `camera_workout` model.

**Failure:** Manual and camera sessions develop different IDs, PRs, histories, and progression.

**Required behavior:** One canonical workout/set identity model. Camera attaches evidence/observations only.

**Invariant:** Camera enriches the training record; it never creates a second training universe.

**Layer:** Schema.

---

## A39 — Import Laundry

**Attack:** Import 12 weeks of elite training from a forged CSV.

**Failure:** Imported history silently becomes official capability.

**Required behavior:** Import is quarantined from official baseline, PR, rank, capability projections, and progression economics in V1.

**Invariant:** Imported history cannot silently become official capability.

**Layer:** Evaluator + Projections.

---

## A40 — Import Social-Proof Leakage

**Attack:** Use imported elite volume graphs/profile displays to create the appearance of established capability.

**Failure:** Economic quarantine exists but social/capability projections still imply verification.

**Required behavior:** Imported data appears only in explicitly labeled archive/import views and does not feed capability projections.

**Invariant:** Import quarantine applies to capability displays, not only XP.

**Layer:** Projection.

---

## A41 — PR Faucet

**Attack:** Create a novelty exercise every day and earn progression from each PR.

**Failure:** PR surface is effectively infinite.

**Required behavior:** PRs award no XP. PRs are derived capability facts with context and evidence rules.

**Invariant:** Novelty cannot become progression currency.

**Layer:** Progression.

---

## A42 — Variation Shopping

**Attack:** Choose a nearby exercise label to manufacture an easy PR, such as treating incline push-ups as push-ups.

**Failure:** Exercise identity is ambiguous.

**Required behavior:** Canonical exercise IDs, strict variation graph, separate IDs for major mechanical changes, closed progression-relevant context modifiers.

**Invariant:** Changing labels cannot change the meaning of a performance claim.

**Layer:** Catalog + Schema.

---

## A43 — Context Grain Explosion

**Attack:** Create endless context modifiers so every session gets a fresh baseline/PR lane.

**Failure:** Every new context becomes a soft reset.

**Required behavior:** Only a closed set of progression-relevant modifiers can split context. Everything else is display metadata.

**Invariant:** Athletes cannot manufacture new baselines by inventing arbitrary context dimensions.

**Layer:** Exercise Catalog.

---

## A44 — Unilateral Double Count

**Attack:** Perform 50 kg split squats per side and claim a 100 kg bilateral PR.

**Failure:** Laterality is ignored.

**Required behavior:** Exercise definition declares laterality. Unilateral PRs are side-aware. No automatic doubling/summing.

**Invariant:** Unilateral load cannot become false bilateral performance.

**Layer:** Schema + Evaluator.

---

## A45 — Alternating Compression

**Attack:** Log 20 alternating unilateral reps as one ambiguous set.

**Failure:** One side receives a PR or asymmetry data is corrupted.

**Required behavior:** Alternating sets are not eligible for unilateral PRs. Per-side sets are required for official unilateral PR eligibility.

**Invariant:** Ambiguous alternating data cannot produce side-specific official records.

**Layer:** Evaluator.

---

## A46 — Assistance Fraud

**Attack:** Perform heavily band-assisted pull-ups and log them as strict bodyweight pull-ups.

**Failure:** Assistance disappears through labeling.

**Required behavior:** Structured load kinds include `EXTERNAL`, `BODYWEIGHT`, `ASSISTED`, `ADDED_TO_BODYWEIGHT`, and `PERCENT_1RM` for prescription aid only. Assisted work has separate PR semantics.

**Invariant:** Assistance cannot disappear through labeling.

**Layer:** Schema + Evaluator.

---

## A47 — Bodyweight Snapshot Fraud

**Attack:** Claim artificial body mass to manipulate relative strength.

**Failure:** Body mass is a freely mutable scalar.

**Required behavior:** Versioned mass snapshots, suspicious-delta handling, and relative-metric suspension where appropriate. Absolute external-load credit should not be unnecessarily destroyed.

**Invariant:** Mass manipulation cannot cheaply manufacture relative-strength capability.

**Layer:** Evaluator + Progression.

---

## A48 — Bodyweight Cut False Positive

**Attack:** An athlete legitimately loses significant weight.

**Failure:** Anti-fraud logic automatically destroys otherwise valid training credit.

**Required behavior:** Constrain relative metrics without unnecessarily rejecting unrelated training.

**Invariant:** Fraud resistance must not require treating legitimate bodyweight change as invalid training.

**Layer:** Evaluator.

---

## A49 — Exercise Family Bypass

**Attack:** Enter cardio as strength or holds as reps to obtain softer plausibility rules.

**Failure:** Client-selected labels control validation.

**Required behavior:** Canonical exercise ID determines family and hard constraints. Incompatible payloads receive no progression credit or are rejected.

**Invariant:** Changing the family label cannot bypass family-specific physics.

**Layer:** Schema + Evaluator.

---

## A50 — Failed-Set Faucet

**Attack:** Spam difficult failed attempts if failure awards even small progression credit.

**Failure:** Failure becomes a safe economic faucet.

**Required behavior:** V1 policy: `FAILED → ZERO progression credit`. Failures remain history.

**Invariant:** Failure cannot be spammed for XP.

**Layer:** Credit Policy.

---

## A51 — Skill-Attempt Faucet

**Attack:** Spam skill attempts or failed singles if each receives a small grant.

**Failure:** Attempt count becomes progression currency.

**Required behavior:** V1: `SKILL_VARIATION → HISTORY_ONLY` unless a future explicit verified policy enables progression.

**Invariant:** Unverified skill attempts cannot mint progression.

**Layer:** Credit Policy.

---

## A52 — Achievement Backdoor Mint

**Attack:** Quietly grant XP for first logs, streaks, variety, or achievement count after PR XP is removed.

**Failure:** Economy simply moves from PRs to achievements.

**Required behavior:** Achievements never grant XP and rank does not count achievement quantity.

**Invariant:** Achievements cannot become an alternate XP faucet.

**Layer:** Progression.

---

## A53 — Quest / Achievement / Level / Rank Bleed

**Attack:** One checkbox completion grants quest success, achievement, XP, and rank progress.

**Failure:** All progression systems become copies of one logging economy.

**Required behavior:** Quest = contract success; Achievement = rare historical predicate; Level = credited stimulus; Rank = capability/evidence/consistency predicates.

**Invariant:** No generic completion signal controls all progression layers.

**Layer:** Progression.

---

## A54 — Checklist Rank Farming

**Attack:** Train only the smallest pinned standards in the easiest allowed context.

**Failure:** Rank becomes a tiny checklist.

**Required behavior:** Rank requires standards/capability plus consistency, evidence coverage, and recency. V1 rank must not use Level.

**Invariant:** Passing a small checklist is insufficient without sustained credible training.

**Layer:** Progression.

---

## A55 — Borrowed Glory

**Attack:** Maintain months of manual fiction, then perform one strong verified session and use it to legitimize the whole history.

**Failure:** One verified day authenticates everything.

**Required behavior:** Rank uses verified evidence coverage across a defined window.

**Invariant:** Verification requires coverage, not merely existence.

**Layer:** Progression.

---

## A56 — Rank Stickiness After Clawback

**Attack:** Gain rank, then have underlying credit removed through correction/re-evaluation.

**Failure:** Rank remains permanently.

**Required behavior:** Rank is recomputable from current predicates or has explicit revocation events.

**Invariant:** Invalidated capability cannot remain official solely because it was once awarded.

**Layer:** Progression.

---

## A57 — Level-as-Rank Loophole

**Attack:** Use high Level as an automatic path to Rank.

**Failure:** Manual inflation eventually becomes rank inflation.

**Required behavior:** V1 rank predicates must not include Level.

**Invariant:** Level and Rank are separate systems.

**Layer:** Progression.

---

## A58 — PENDING Side Effects

**Attack:** Submit questionable sets and cause PENDING records to feed UI streaks, quests, achievements, PRs, or baselines.

**Failure:** Pending claims become habit-tracker progress.

**Required behavior:** PENDING cannot feed quests, achievements, streaks, rank, PRs, baselines, or capability projections. It only supports “awaiting judgment” UX.

**Invariant:** PENDING is not success.

**Layer:** Progression + Projections.

---

## A59 — ABANDONED-as-Submit

**Attack:** Let a workout auto-abandon after accepted sets and rely on those sets to mint credit.

**Failure:** Abandon becomes an alternative submit button.

**Required behavior:** Sets in abandoned workouts default to history-only unless an explicit short-window recovery/submit policy applies.

**Invariant:** ABANDONED cannot silently become a progression submission path.

**Layer:** Lifecycle + Evaluator.

---

## A60 — Indefinite OPEN Staging

**Attack:** Keep a workout OPEN for days and stream sets as baselines/policies/economic windows change.

**Failure:** OPEN becomes an economic staging area.

**Required behavior:** Server enforces maximum open duration and idle gap. Economy is bounded from the accepted session start. Auto-abandon freezes further acceptance except through defined recovery.

**Invariant:** An OPEN workout has bounded economic meaning.

**Layer:** API + Lifecycle + Credit Policy.

---

## A61 — Open-Session Modality Bypass

**Attack:** Label an exploitative long session as hiking/conditioning to obtain longer bounds.

**Failure:** Modality-specific bounds become self-labeling loopholes.

**Required behavior:** Bounds are derived from canonical exercise/session type, not freely chosen narrative labels.

**Invariant:** Session-duration policy cannot be bypassed through self-labeling.

**Layer:** Catalog + Lifecycle.

---

## A62 — Multi-Account / Shared-Body Minting

**Attack:** Two people use one athlete account.

**Failure:** One save file represents multiple bodies.

**Required behavior:** V1 invariant: one account = one athlete. Coach/partner multi-body models remain outside the core.

**Invariant:** A training record belongs to one athlete identity.

**Layer:** API + Policy.

---

## A63 — Unit Conversion Double-Mint

**Attack:** Enter 100 kg, convert to pounds on the client, convert again at the server, or reconvert during evaluation.

**Failure:** Physical magnitude changes.

**Required behavior:** Canonicalize once at the write boundary. Evaluator consumes canonical units. Display conversion occurs only at presentation.

**Invariant:** Unit selection cannot change the physical magnitude of a stored claim.

**Layer:** API + Schema.

---

## A64 — Projection Grant-Only Bug

**Attack:** Projection code sums only GRANT rows and ignores compensations.

**Failure:** Level remains inflated after re-evaluation.

**Required behavior:** Projection rebuild consumes the complete ledger. Grant/compensation lineage is preserved.

**Invariant:** Incremental projection and full ledger rebuild produce identical progression state.

**Layer:** Credit Ledger + Progression.

---

# 4. Cross-Cutting Invariants

## I1 — Raw facts are append-only after acceptance

No silent mutation of actual performance, reported timestamps, prescription snapshots, or already-attached evidence.

## I2 — Client cannot write progression state

Client cannot directly write XP, Level, Rank, claim state, credit tier, rank eligibility, evidence trust class, official PR state, or baseline heads.

## I3 — One canonical training graph

There is one Workout ID space, Set ID space, Exercise ID space, and Evidence model. Camera does not create another workout graph.

## I4 — Credit is ledgered

No mutable `player.xp += N` balance is the source of truth.

## I5 — Economic idempotency is independent of evaluator version

Evaluator version changes do not automatically create new economic value.

## I6 — Policy is pinned at acceptance

Every economic grant identifies the applicable credit policy.

## I7 — PENDING is economically inert

Pending is neither success nor failure.

## I8 — Rejected claims are retained

Rejected claims are not deleted and receive no progression credit.

## I9 — Voiding does not erase history

Void is a state/event, not hard deletion.

## I10 — Baselines are derived

Baseline heads are derived projections, not raw facts.

## I11 — Pointer changes are append-only

Changes to current evaluation, baseline head, official PR head, or equivalent pointers produce auditable pointer-change events.

## I12 — Evaluation uses a pinned baseline snapshot

Baseline updates occur asynchronously after the relevant credit transaction.

## I13 — Evidence must be attributable

Evidence must be provenance-labeled, versioned, attached to a canonical subject, and temporally attributable.

## I14 — Media presence is not verification

An attachment alone never upgrades trust.

## I15 — Manual credit is bounded

Both global weekly and per-family weekly manual ceilings exist, with diminishing returns.

## I16 — Failed work does not mint in V1

`FAILED → ZERO`.

## I17 — Skill attempts do not mint in V1

`SKILL_VARIATION → HISTORY_ONLY` unless a future verified policy explicitly changes this.

## I18 — Achievements do not mint

Achievements never award XP.

## I19 — PRs do not mint

PRs are derived capability facts, not economic faucets.

## I20 — Rank is recomputable

Rank cannot be an irreversible cosmetic award.

## I21 — Rank does not use Level in V1

Level and Rank remain intentionally separate.

## I22 — Imported data is quarantined

Import cannot become official capability without later corroboration/approval.

## I23 — Economy uses server time

Client timestamps are claim metadata. Server receipt time controls economic windows.

## I24 — Canonical units are stored once

Evaluator does not perform ambiguous multi-layer conversions.

## I25 — Exercise family determines validation grammar

Client labeling cannot bypass family-specific constraints.

## I26 — Unilateral data is side-aware

Unilateral work cannot automatically become bilateral performance.

## I27 — No session-completion XP

Workout completion itself is not an economic event.

## I28 — No set-count XP floor

Fragmentation cannot increase credit.

## I29 — Workout aggregates are UX only

Credit and rank decisions use set/segment facts.

## I30 — One athlete per account in V1

Shared-body/coaching identity models are outside the core training model.

---

# 5. Required Lifecycle

Conceptual training-record flow:

```text
DRAFT
  |
  | accept
  v
ACCEPTED / PENDING EVALUATION
  |
  +----> VALID
  |
  +----> PARTIAL
  |
  +----> REJECTED
  |
  +----> HISTORY_ONLY
```

Conceptual workout flow:

```text
OPEN
  |
  +----> SUBMITTED
  |
  +----> ABANDONED
  |
  +----> VOIDED
```

`DRAFT` is the freely mutable state.

`accept` freezes actuals.

`PENDING` is not a draft.

`ABANDONED` is not a hidden submit button.

---

# 6. Required Economic Rules

### Manual
Manual progression is bounded by global and per-family weekly ceilings and diminishing returns. Manual-only evidence does not establish rank.

### Verified
Verified progression requires qualifying evidence/observation rules. Media presence alone is insufficient.

### Failed
`FAILED → ZERO`.

### Skill
`SKILL_VARIATION → HISTORY_ONLY` in V1 unless explicitly verified under a future policy.

### Import
`IMPORT → QUARANTINED` for official progression/capability/rank in V1.

### Achievement
`ACHIEVEMENT → NO XP`.

### PR
`PR → NO XP`.

### Session
`SESSION_COMPLETION → NO XP`.

---

# 7. Required Ledger Properties

The ledger must support:

- grants,
- compensations (V1 reversal/clawback kind is `COMPENSATION`; there is no separate `CLAWBACK` `entry_kind`),
- lineage,
- policy ID,
- training record ID,
- actor/system source,
- evaluator metadata,
- timestamps,
- reason.

Economic uniqueness is anchored to the underlying training claim and credit policy.

Evaluator version is metadata, not the sole economic uniqueness grain.

Every economic transition must be reproducible.

---

# 8. Required Correction Model

After acceptance:

```text
Original Record
      |
      +--> Correction Event
      |
      +--> New Evaluation
      |
      +--> Compensation / New Grant if policy permits
```

Never silently mutate an evaluated field.

Corrections preserve:

- old value,
- new value,
- reason,
- actor,
- time,
- target,
- evaluation lineage.

Self-serve corrections are time-boxed and rate-limited and normally do not create net-positive progression.

---

# 9. Required Projection Rules

Every projection declares:

### State filter
Explicitly define whether it consumes VALID/PARTIAL and excludes PENDING/REJECTED/VOID.

### Provenance filter
Explicitly define treatment of MANUAL/CAMERA/WEARABLE/IMPORT.

### Clock
Explicitly choose `DISPLAY_CLAIMED` or `ECONOMY_RECEIVED`.

### Credit eligibility
Projections do not invent their own definition of credited stimulus.

---

# 10. Required Rank Model

V1 Rank is based on:

**Capability standards × Consistency × Evidence Coverage × Recency**

Rank does not depend on:

- Level,
- achievement count,
- PR count,
- session count alone,
- imported history,
- manual-only capability claims.

A single verified session cannot authenticate months of unverified history.

Rank evidence is measured over a defined window.

Rank is recomputable after corrections/clawbacks.

---

# 11. Required Baseline Model

A baseline is a condition-keyed evaluation reference, not a trophy.

Baseline inputs favor:

- credited work,
- official/evidence-backed records,
- recent credible top sets,
- robust statistics.

Baseline movement is bounded.

Required policy controls:

```text
baseline_upward_change_limit
baseline_downward_change_limit
baseline_minimum_sample
baseline_floor
```

Baseline snapshots are pinned at evaluation start. Baseline heads update after credit commits.

---

# 12. Required Exercise Model

Exercise definitions provide:

- canonical exercise ID,
- family,
- load model,
- laterality,
- allowed performance types,
- progression-relevant context,
- PR keys,
- success/standard constraints.

Performance families include, as appropriate:

- `REPS_LOAD`
- `REPS_BODYWEIGHT`
- `REPS_ASSISTED`
- `DURATION`
- `DISTANCE_TIME`
- `DISTANCE_ONLY`
- `TIME_ONLY`
- `ISOMETRIC_HOLD`
- `CARDIO_SEGMENT`

Structured load:

```text
{
  magnitude,
  unit,
  kind
}
```

Pace is derived.

Bodyweight references a versioned mass snapshot.

---

# 13. Required Evidence Model

Evidence conceptually contains:

- evidence ID,
- subject,
- evidence class,
- produced time,
- producer/version,
- payload reference,
- content hash,
- observation data,
- confidence,
- processing state,
- review state.

Manual entry is `MANUAL_ASSERTION`.

Camera does not automatically become trusted.

Future camera functionality attaches to canonical sets.

---

# 14. Pre-Code Decision Checklist

The architecture must explicitly lock:

1. Persistence stack.
2. Exercise catalog authorship and merge rules.
3. TIME_ONLY/DURATION/ISOMETRIC_HOLD examples.
4. Empty-snapshot manual policy.
5. Global + per-family manual ceilings.
6. Economy week = server UTC epoch week.
7. Maximum open/session duration.
8. Maximum idle gap.
9. Offline submission age bound.
10. FAILED → ZERO.
11. SKIPPED policy.
12. Self-serve correction window/quota.
13. Semantic duplicate default.
14. Import quarantine.
15. Rank excludes Level in V1.
16. One athlete per account.
17. SKILL_VARIATION = HISTORY_ONLY unless verified.
18. Empty submit → ABANDONED/no credit/no quest.
19. Spec/document home.
20. Pointer changes are append-only.
21. Observation/assertion agreement requirement.
22. Closed ExerciseContext modifier set.
23. ABANDONED default = HISTORY_ONLY.
24. Per-set evaluation single-flight serialization.

---

# 15. Test Matrix

Every attack should eventually become an automated test.

### API tests

- authorization,
- client trust-field injection,
- duplicate client event,
- semantic duplicate,
- future timestamp,
- backdate,
- unit conversion,
- correction limits,
- lifecycle violations.

### Evaluator tests

- warmup detection,
- implausible load,
- implausible rep rate,
- duration limits,
- cardio pace,
- baseline poisoning,
- prescription sandbagging,
- evidence conflicts,
- evidence timing,
- laterality,
- assistance,
- exercise-family mismatch.

### Ledger tests

- duplicate grant,
- concurrent evaluation,
- re-evaluation,
- correction compensation,
- policy changes,
- projection rebuild,
- grant/compensation lineage.

### Progression tests

- manual global ceiling,
- manual family ceiling,
- no session XP,
- no achievement XP,
- no PR XP,
- quest stimulus floor,
- rank evidence coverage,
- rank recomputation,
- Level/Rank separation.

### Projection tests

- PENDING excluded,
- REJECTED excluded,
- VOID excluded,
- IMPORT quarantined,
- claimed vs received clock,
- baseline filtering,
- PR filtering.

---

# 16. Red-Team Acceptance Criteria

The architecture is not ready for implementation until these are true:

### Truth
The system can preserve a false claim without treating it as verified truth.

### Credit
No retry, correction, evaluator version, PR, achievement, or session completion can unexpectedly mint progression twice.

### Time
Client clocks and timezone changes cannot manufacture economic windows.

### Baseline
An attacker cannot cheaply reshape their baseline to make normal work look extraordinary.

### Prescription
An athlete cannot manufacture easier official prescriptions through self-selected targets or poisoned baselines.

### Evidence
Media presence cannot manufacture verified status.

### Camera
Camera evidence attaches to canonical training records rather than creating a parallel graph.

### Rank
Rank cannot be obtained merely by Level, PR count, achievement count, session count, imported history, or one verified day.

### History
Rejected, failed, voided, imported, and corrected claims remain auditable without contaminating official capability.

### Ledger
Re-evaluation and concurrency cannot create net-new progression without an explicit policy reason.

### Identity
One V1 athlete identity corresponds to one account.

---

# 17. Implementation Gate

Do not proceed directly from this document to feature development.

The next implementation step is a schema/domain review.

Cursor should:

1. Read this adversarial suite.
2. Produce the domain model and Prisma schema proposal.
3. Map every invariant to database-level or application-level enforcement.
4. Identify invariants that cannot be guaranteed by the database.
5. Produce state-transition diagrams.
6. Produce ledger uniqueness constraints.
7. Produce evaluator input/output contracts.
8. Produce projection filtering contracts.
9. Write tests for the highest-severity attacks.
10. Stop before implementing UI or XP formulas.

Do not introduce:

- generic nullable `weight`/`reps` fields as the core performance model,
- mutable XP balances,
- client-writable trust fields,
- separate camera workouts,
- hard-delete training records,
- session-completion XP,
- PR XP,
- achievement XP,
- Level → Rank coupling,
- evaluator-version-based economic grants.

---

# 18. Final Adversarial Verdict

SYSTEM is ready to move toward implementation only when the above invariants are explicit and enforceable.

The goal is not to prevent athletes from lying. That is impossible.

The goal is:

> **A lie can exist in the save file without becoming capability.**

> **A correction can change interpretation without rewriting what was originally claimed.**

> **A new evaluator can change judgment without silently printing money.**

> **A camera can increase evidence quality without requiring a second training database.**

> **Rank represents credible capability, not logging proficiency.**

This document is the adversarial gate.

**No feature work should override these invariants without an explicit architecture change and a new attack review.**
