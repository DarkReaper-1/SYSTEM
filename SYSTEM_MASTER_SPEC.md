# SYSTEM — MASTER SPECIFICATION
Version: 1.0
Status: Implementation Gate
Purpose: Canonical product, domain, data, evaluation, economy, and progression contract for SYSTEM.

> **Core thesis:** Your body is the save file.
>
> SYSTEM records training as immutable claims about real training, evaluates those claims, awards bounded credit through an append-only ledger, and derives progression from credited stimulus—not from logging activity itself.

---

## 0. Non-Negotiable Architecture

The canonical pipeline is:

**Input → Write Boundary → Immutable Training Record → Evaluation → Credit Ledger → Projections**

The layers are separate:

- **History** records what the athlete claimed and what the system observed.
- **Evaluation** judges claim coherence, plausibility, evidence, and conflicts.
- **Credit** determines how much progression value the record can mint.
- **Progression** derives Level, Quest success, Achievements, PRs, and Rank under separate rules.

No client may directly write trust, evaluation, credit, XP, rank, PR, baseline, or progression state.

### 0.1 Sacred facts

After a training record is accepted:

- raw actual performance is append-only;
- original claimed timestamps are preserved;
- prescription snapshots are immutable;
- attached evidence is preserved;
- corrections are events, never silent mutations;
- awarded credit is ledgered;
- voiding does not delete the claim.

---

# 1. DOMAIN MODEL

## 1.1 Athlete

V1 invariant:

**One human ↔ one account ↔ one athlete identity.**

The core training model does not support shared-body profiles, partner logging, coach-owned athletes, or multiple humans on one save file.

---

## 1.2 Workout

A workout is a **bounded training session**:

- one athlete;
- one start/end window;
- one intent container;
- ordered training records;
- optionally intervals/segments;
- optionally a prescription.

A workout is not:

- an XP event;
- a checkbox;
- a calendar completion;
- a collection of arbitrary floats.

### Lifecycle

Recommended states:

`DRAFT → OPEN → SUBMITTED → EVALUATING → CLOSED`

Exceptional:

`ABANDONED`
`VOIDED`

### Rules

- DRAFT is editable and non-historical.
- ACCEPT/SUBMIT freezes actual facts.
- OPEN cannot remain an unlimited economic staging area.
- Server policy defines maximum open duration and idle gap.
- Modality-specific policy may permit longer sessions.
- Economy is bounded from the first accepted set; an indefinitely open workout cannot bypass rolling caps.
- Empty submit becomes ABANDONED, with no credit and no quest success.
- ABANDONED records remain history, but default to `HISTORY_ONLY` credit unless recovered through an explicit valid submission path.
- Recovery must be time-bounded.

---

## 1.3 Set

A set is the **atomic performance claim**:

> one continuous bout of one exercise under one load/condition producing one primary result.

Required:

- stable `set_id`;
- `workout_id`;
- `athlete_id`;
- canonical `exercise_id`;
- sequence index;
- declared set role;
- outcome;
- timing;
- typed actual performance;
- source (client-submitted origin claim; untrusted);
- server provenance (server-derived; not client-writable);
- client event id;
- schema version.

Expected:

- prescription snapshot/ref;
- side;
- equipment/context;
- evidence array;
- `to_failure`;
- `amrap`;
- assisted flag.

A set is not a muscle group, mood, photo dump, or exercise-block total.

---

# 2. TYPED PERFORMANCE

Never use a generic `weight` + `reps` model.

Performance is discriminated by exercise family.

Supported V1 types:

### REPS_LOAD
- reps
- structured load

### REPS_BODYWEIGHT
- reps
- bodyweight snapshot/context

### REPS_ASSISTED
- reps
- structured assistance/resistance

### DURATION
- seconds only
- no load field in V1

If load-bearing duration becomes necessary later, introduce it as a versioned contract change. Do not extend this V1 `DURATION` shape.

### DISTANCE_TIME
- distance
- duration

### DISTANCE_ONLY
- distance

### TIME_ONLY
- duration

### ISOMETRIC_HOLD
- duration
- optional load

### CARDIO_SEGMENT
- sport
- distance
- duration
- optional elevation/surface/grade

Pace is derived from distance and duration.

The write boundary receives client/native units, canonicalizes the measurement exactly once, and stores only the canonical magnitude and canonical unit. Pre-canonical/native units are not persisted in V1. Evaluators never reconvert units. Display conversion is presentation-only.

---

# 3. LOAD MODEL

Load is structured:

```text
{
  magnitude,
  unit,
  kind
}
```

`kind`:

- `EXTERNAL`
- `BODYWEIGHT`
- `ASSISTED`
- `ADDED_TO_BODYWEIGHT`
- `PERCENT_1RM` — prescription aid only

Percent 1RM is not an actual scored load.

Never encode `BW+25`, `60kg assisted`, or equivalent meaning as a string.

Bodyweight references a versioned athlete mass snapshot.

---

# 4. SET ROLES AND OUTCOMES

## Roles

- `WARMUP`
- `WORK`
- `BACKOFF`
- `TEST`
- `OTHER`

Warmups are real history.

Warmups normally:

- receive no normal progression credit;
- receive no normal PR;
- affect plausibility and context.

Explicit test protocols may be exceptions.

### Effective role

Declared role is not sufficient for economy.

Evaluator derives:

`effective_role = server_detected_role OR declared_role`

where server detection is authoritative when confidently triggered.

Credit, PR, and baseline projections use `effective_role`.

Display may show both declared and detected roles.

---

## Outcomes

- `COMPLETED`
- `FAILED`
- `TERMINATED`

Failed/incomplete work is retained.

V1:

**FAILED → ZERO progression credit.**

Failure may still matter for history and future analysis.

---

# 5. LATERALITY

Exercise definition declares laterality:

- bilateral → `NONE`
- unilateral → `LEFT` / `RIGHT`
- alternating → allowed for history but not eligible for unilateral PRs without per-side records.

V1 standard:

**Per-side sets are preferred for unilateral exercises.**

Never auto-double unilateral load into a bilateral PR.

---

# 6. EXERCISE CATALOG

Exercises use stable canonical IDs.

Each exercise declares:

- primary family;
- load model;
- laterality;
- success metric;
- standard constraints;
- allowed performance types;
- PR classes;
- allowed context modifiers.

Major mechanical changes are different exercise IDs.

Minor context is a closed modifier set.

V1 context modifiers are explicitly catalog-controlled. Arbitrary user-created modifiers cannot create new baseline/PR contexts.

Aliases map to canonical exercises.

V1 authorship: system-authored canonical IDs only. The client cannot create canonical IDs. The initial canonical ID list must be enumerated in the Training Contract before any schema seed. Attack-fixture movement names are not IDs.

The evaluator uses canonical exercise/family constraints, not client-selected labels.

---

# 7. EXERCISE FAMILIES

V1 families:

- `ABSOLUTE_STRENGTH`
- `REPS_AT_LOAD`
- `ISOMETRIC_DURATION`
- `LOCOMOTION`
- `SKILL_VARIATION`

Future families may include:

- `REPEAT_EFFORT`
- `DENSITY`
- `UNILATERAL_SYMMETRY`

Progression is family-defined.

There is no global “more weight = improvement” rule.

---

# 8. PRESCRIPTION

A prescription is a **versioned prior intent contract**.

It is not actual performance and is not a note.

It can contain:

- exercise targets;
- set schemes;
- target load/reps/duration/distance;
- mode;
- AMRAP constraints;
- RPE caps;
- rest guidance;
- success criteria.

At session start, the prescription is snapshotted and immutable.

### Prescribed vs actual

Always separate:

`prescribed`
from
`actual`

Evaluation compares them.

Missing prescribed work is never invented.

Extra work does not create unbounded quest credit.

---

# 9. FREE VS PRESCRIBED

### Prescribed

Linked to a system/quest/plan template.

Has explicit compliance targets.

### Free

Open training without a compliance contract.

Free training is still evaluated for integrity and can produce bounded credit.

Free does not mean lawless.

---

# 10. PRESCRIPTION AUTHORITY

User-editable prescriptions that make work materially easier are not treated as authoritative system targets.

System/quest prescriptions should derive from:

1. official evidence-weighted baseline, when available;
2. conservative onboarding/standard floors;
3. slow-moving top credited working-set statistics.

Manual-only baselines cannot make system prescriptions arbitrarily easier.

Baseline downward movement is bounded.

---

# 11. AMRAP

AMRAP is a prescription mode.

Prescription may specify:

- target load;
- time cap;
- rep/round target;
- success rule.

Actual records:

- reps or rounds+reps;
- time result;
- termination reason;
- failure/time-expired status.

Family-specific rep-rate plausibility applies.

V1 does not award special AMRAP bonuses merely for choosing AMRAP.

---

# 12. CARDIO

Cardio uses segments, not fake strength sets.

A segment can contain:

- sport;
- distance;
- duration;
- surface;
- treadmill grade;
- elevation.

Intervals are multiple segments with roles.

Pace is derived.

Impossible pace/distance combinations are retained as claims but receive zero progression credit when rejected.

---

# 13. EVIDENCE

Evidence attaches to workouts/sets.

One canonical training record can have many evidence items.

Evidence object includes conceptually:

- evidence id;
- subject;
- class;
- produced_at;
- producer version;
- payload reference;
- hash/fingerprint;
- observations;
- review state;
- trust inputs.

Manual entry is explicit weak evidence:

`MANUAL_ASSERTION`

Camera does not create another workout/set graph.

### Trust upgrade

Media presence alone never upgrades trust.

Evidence becomes higher-trust only after successful observation extraction and policy checks.

For `VERIFIED`:

- observation must meet confidence/completeness requirements;
- timestamps must align to the claimed set window;
- exercise/context must match;
- observation must agree with the assertion within policy margin.

Large conflicts result in reduced/zero progression and no rank eligibility.

Evidence extractor version is pinned.

---

# 14. MEDIA REUSE

Exact content hash is necessary but insufficient.

Evidence reuse detection uses:

- content hash;
- perceptual fingerprint where available;
- temporal alignment;
- reuse count/window rules.

A legitimate multi-set recording may support several sets only within explicit policy.

Re-encoding must not trivially create a new trusted identity.

---

# 15. PROVENANCE

Provenance is server-derived.

A training set also carries a client-submitted `source` claim. That claim is untrusted.

Both `source` and `server_provenance` use:

- `MANUAL`
- `CAMERA`
- `WEARABLE`
- `IMPORTED`

The client cannot write `server_provenance`. Import quarantine and official-projection filters key off `server_provenance`, not `source`.

Import is permanently quarantined from official progression in V1.

Imported records may display in a clearly labeled archive lane.

Imports do not feed:

- official baselines;
- official PRs;
- rank eligibility;
- capability projections.

---

# 16. CLAIM STATE ≠ CREDIT ≠ RANK

These are distinct.

### Claim state

Describes the evaluation of the claim:

- `PENDING`
- `VALID`
- `PARTIAL`
- `REJECTED`

### Credit tier

Describes economic treatment, for example:

- `VERIFIED`
- `MANUAL_LIMITED`
- `HISTORY_ONLY`
- `ZERO`

### Rank eligibility

A separate server-derived predicate.

The client cannot write any of these.

Manual validity does not imply verified truth.

---

# 17. MANUAL PROGRESSION

Manual claims may contribute bounded progression, but never become an unlimited XP faucet.

V1 requirements:

- global weekly manual credit ceiling;
- per-family weekly manual sub-ceiling;
- asymptotic diminishing returns within the manual lane;
- manual progression is visually segregated from verified progression;
- manual progression does not establish rank eligibility;
- manual-only records cannot become official capability evidence.

Exact numeric caps belong to the CreditPolicy configuration, not the data model. Until the Training Contract V1 numeric cells are filled, schema migration must not proceed.

---

# 18. CREDIT POLICY

No session-completion XP.

No checkbox XP.

No per-set minimum XP floor.

No PR XP.

No achievement XP.

No arbitrary “hard workout” bonus.

Credit is based on productive stimulus.

Credit uses concave curves and rolling caps.

Tiny micro-sets collapse into grouped stimulus packets.

Examples:

- 200 × 1-rep sets do not create 200 economic floors;
- 100 empty-bar warmups do not mint normal XP;
- multi-session splitting cannot manufacture session bonuses.

Credit policy is pinned at `accepted_at`.

Queued/offline submissions do not receive retroactive benefit from later policy changes.

---

# 19. TIME MODEL

Store multiple clocks explicitly.

### CLAIMED_CLOCK
Athlete/device-reported event time.

### RECEIVED_CLOCK
Server receipt time.

### SESSION_CLOCK
Clock identity selected at session start (`OPEN`) and bound to the session. Not a duration.

V1 stores:

```text
session_clock = {
  selected_at_received,
  timezone_offset
}
```

Immutable after the first accepted set. Intra-session elapsed time is derived from claimed `started_at` / `ended_at`. Economy does not bucket weeks with `session_clock`.

### ECONOMY_CLOCK
V1: server epoch weeks in UTC.

Rules:

- credit/caps/rank use economy clock;
- review/admin consumers show received clock first and claimed clock second;
- claimed time for a workout or set is `started_at` / `ended_at` (there is no separate `claimed_at` column);
- display projections explicitly declare their clock;
- client timezone cannot reset economy caps;
- future claims receive no progression credit;
- backdating is bounded;
- server `received_at` controls economy admission.

---

# 20. OPEN SESSION BOUNDS

CreditPolicy contains:

- `max_open_duration`;
- `max_idle_gap`;
- `max_offline_delay`;
- modality-specific bounds where justified.

An OPEN workout cannot remain an economic staging area indefinitely.

On automatic abandonment:

- history remains;
- new writes are blocked;
- default credit becomes `HISTORY_ONLY`;
- a bounded recovery path may restore the session to valid submission.

Economy remains bounded from the original accepted activity.

---

# 21. IDEMPOTENCY AND DEDUPE

### Primary

`client_event_id` is unique per athlete and write operation.

A retry returns the original record rather than creating a second claim.

### Secondary

Semantic duplicate detection uses a server-side fingerprint based on an approximate time window and set signature.

Default duplicate action:

- retain history;
- assign zero progression credit;
- add suspicion;
- do not expose a crisp threshold oracle to the client.

Sets within the same workout with distinct sequence indices are not treated as cross-workout semantic duplicates.

Cross-workout splitting is controlled by session proximity/rate rules.

---

# 22. ACCEPTANCE AND CORRECTIONS

DRAFT may be freely edited.

Once accepted:

**actual facts are frozen.**

Corrections are append-only events.

A correction contains:

- target record;
- old value;
- new value;
- reason;
- actor;
- timestamp.

Corrections:

- are rate-limited;
- are time-boxed for self-service;
- do not silently mutate raw facts;
- normally do not create net-positive progression;
- trigger controlled re-evaluation;
- can cause compensating ledger entries (`COMPENSATION`). V1 has no separate `CLAWBACK` `entry_kind`.

No correction loop may produce net-positive credit through repeated A→B→A changes.

---

# 23. EVALUATION

Evaluation is a pure decision over:

- immutable training facts;
- correction-event projection;
- prescription snapshot;
- evidence;
- exercise definition;
- baseline snapshot;
- pinned policy;
- prior committed state.

It must not depend on a live mutable baseline during the evaluation.

### Snapshot isolation

At session start, baseline snapshot version is pinned.

Baseline updates happen after credit commits.

Two concurrent sessions cannot both read an unstable halfway-updated baseline.

---

# 24. PLAUSIBILITY

Severity ladder:

1. accept with soft cap/reduced credit;
2. require confirmation;
3. reject progression credit;
4. hard reject malformed/nonsensical writes.

Never silently “fix” user facts.

Examples:

- negative load → hard reject;
- future timestamp → no progression;
- impossible running pace → rejected credit;
- impossible AMRAP rate → partial/rejected;
- multi-hour plank → capped/rejected progression.

Raw claims remain auditable unless legal retention rules require deletion.

---

# 25. BASELINES

Baseline is a rolling reference, not a trophy.

It is keyed by:

`athlete × exercise × relevant context`

Official baseline sources:

- credited working sets;
- evidence-weighted history;
- robust statistics;
- top credible working capacity;
- slow movement.

Baseline must not be easily crashed.

Invariants:

- baseline cannot spike beyond policy bounds per week;
- baseline cannot collapse beyond policy bounds per week;
- manual-only history cannot lower official prescription authority;
- imported data cannot seed official baseline;
- baseline uses effective role, not declared role alone.

---

# 26. PRs

PRs are derived facts.

They answer:

> What is the best credible mark?

PR keys include:

- exercise;
- PR class;
- relevant context;
- side when unilateral.

PR states may include:

- provisional;
- official.

PRs do not mint meaningful XP.

Official PRs require the applicable evidence/trust gate.

PR invalidation occurs through lineage/evaluation changes, not deletion.

Promotion of provisional→official is idempotent and ledgered where needed.

---

# 27. QUESTS

Quest = contract success.

Quest completion cannot be based on:

- logging a workout;
- pending records;
- session existence;
- set count alone.

Quest predicates require credited productive stimulus and appropriate intensity/coverage conditions.

For recurring or multi-family quests, success may require family diversity.

Prescription success is not the same as “athlete did anything.”

Substitutions require explicit policy mapping.

Overperformance does not produce unbounded quest credit.

---

# 28. ACHIEVEMENTS

Achievements are historical predicates/titles.

**V1 achievements never grant XP.**

Achievement count does not grant rank progress.

Achievements must not become a hidden economic faucet through:

- first-log bonuses;
- streak bonuses;
- variety bonuses;
- PR bonuses;
- completion bonuses.

They can recognize behavior without becoming the progression ledger.

---

# 29. LEVEL

Level is derived from the credit ledger.

There is no writable XP balance.

Level changes only from committed ledger entries.

Ledger rebuild must reproduce the same Level.

Manual and verified progression may be displayed as distinct components, but the exact Level policy must enforce the manual ceiling and not allow manual fiction to dominate verified progression.

---

# 30. RANK

Rank is accepted status, not an automatic function of Level.

V1 rank predicates **must not include Level**.

Rank requires:

1. capability against pinned standards;
2. consistency over a defined window;
3. evidence coverage;
4. recency;
5. anti-sandbox rules.

Rank evidence uses a coverage ratio over a window, not one cherry-picked verified session.

Manual-only evidence is not default rank evidence.

Rank is recomputable.

If supporting facts are invalidated, rank can be revoked through explicit server-derived state or compensating rank events.

No sticky rank based only on historical Level.

---

# 31. PINNED STANDARDS

Rank standards are curated, stable requirements.

They are not an open PR board.

Standards include:

- canonical exercise;
- required context;
- required metric;
- threshold;
- laterality;
- evidence requirement;
- recency window.

The standard set must be broad enough to resist checklist farming while remaining comprehensible.

Rank also requires consistency and coverage.

---

# 32. PENDING CONSUMER INVARIANT

`PENDING` is never success.

PENDING cannot drive:

- Quest success;
- Achievement completion;
- streaks;
- Rank;
- PRs;
- Baselines;
- capability projections;
- progression rewards.

It may drive only explicitly labeled “awaiting evaluation” UI.

---

# 33. IMPORT LANE

Imported history is useful as an archive but not official capability.

It is:

- labeled;
- provenance-preserving;
- excluded from official baselines;
- excluded from official PRs;
- excluded from rank;
- excluded from capability projections.

V1 does not provide automatic import promotion.

---

# 34. POINTER INTEGRITY

Mutable “current” pointers are server-owned.

Examples:

- current evaluation;
- official PR pointer;
- baseline head.

Every pointer move produces an append-only `PointerChange` event:

- target;
- previous pointer;
- new pointer;
- actor/system;
- reason;
- time;
- event/version.

Current state is reconstructable from the log.

---

# 35. LEDGER INTEGRITY

Economic uniqueness is based on the economic identity of the claim, not merely evaluation version.

A set/work unit can have one net grant chain per:

`training_record × credit_policy`

Evaluation version is metadata, not the primary uniqueness key.

Re-evaluation:

- never silently reprints;
- compensates prior credit via `COMPENSATION` entries (V1 has no separate `CLAWBACK` `entry_kind`);
- applies the new result;
- preserves lineage;
- is serialized per record.

Policy changes are explicit migrations, not accidental re-evaluations.

---

# 36. SINGLE-FLIGHT EVALUATION

Per-set economic evaluation is serialized.

Evidence attachment, correction, first evaluation, and migration triggers all enter the same single-flight path.

State transitions are guarded.

Retries are safe.

No concurrent path may produce two net grants for the same economic claim/policy.

---

# 37. PROJECTION RULES

Every read model declares its source and clock.

Projection consumers must explicitly exclude:

- voided records where appropriate;
- rejected records from capability;
- pending records from success;
- imports from official capability;
- history-only records from economic calculations.

Workout-level aggregates are UX summaries only.

Credit, rank, PR, and baseline decisions operate on eligible set/segment records.

---

# 38. ECONOMIC CLOCK AND RATE LIMITS

Rolling limits exist independently of batch size.

They apply to:

- hour;
- day;
- week;
- manual lane;
- per family where required.

Offline batches cannot bypass limits by staying below a “large batch” threshold.

Economy is based on server receipt/acceptance order.

---

# 39. BODYWEIGHT CONTEXT

Bodyweight is versioned.

Rapid suspicious changes:

- flag relative metrics;
- may temporarily suspend relative PR updates;
- do not automatically invalidate absolute external-load credit.

Rank prefers standards that are robust to bodyweight manipulation where possible.

---

# 40. ANTI-EXPLOIT INVARIANTS

At minimum, V1 must test:

1. warmup spam;
2. impossible loads;
3. duplicate retry;
4. semantic duplicate;
5. edit-after-credit;
6. correction oscillation;
7. baseline poisoning;
8. prescription sandbagging;
9. timestamp compression;
10. timezone hopping;
11. future/backdated claims;
12. impossible cardio;
13. AMRAP rate fraud;
14. duration inflation;
15. offline reorder;
16. evaluation race;
17. policy-version regrant;
18. evidence spoof;
19. camera/manual conflict;
20. media reuse;
21. import laundering;
22. unilateral double count;
23. assistance fraud;
24. bodyweight manipulation;
25. variation shopping;
26. achievement faucet;
27. quest checklist farming;
28. rank checklist farming;
29. pending side effects;
30. abandoned-session minting;
31. pointer rollback;
32. projection GRANT-only bugs;
33. unit double conversion;
34. multi-athlete account sharing;
35. workout splitting;
36. micro-set packing.

---

# 41. V1 NON-GOALS

Do not add to the core training schema yet:

- social graph;
- dungeons/circuits as foundational entities;
- tempo grammar;
- ROM/form AI;
- complex coaching graphs;
- multiple workout systems;
- camera-specific workout tables;
- automatic coach accounts;
- arbitrary context dimensions.

---

# 42. IMPLEMENTATION GATE

Do not begin feature screens against the training database until these are locked:

- typed performance;
- canonical exercise IDs;
- workout/set lifecycle;
- prescription/actual split;
- evidence model;
- claim/credit/rank separation;
- immutable acceptance;
- correction events;
- evaluation snapshots;
- idempotent ledger;
- policy pinning;
- baseline rules;
- manual credit ceilings;
- rank predicates;
- pointer audit;
- pending consumer rules;
- import quarantine;
- time model;
- rate limits;
- semantic dedupe.

If an implementation violates this document, the implementation is wrong until the specification is intentionally changed.

---

# 43. DOCUMENT AUTHORITY

This file is the master architectural contract.

The Training Contract defines the concrete storage/API/evaluation contract for training records.

The Adversarial Attacks document is the red-team gate and may identify additional tests, but it does not silently override this specification.

When documents conflict:

1. intentional, versioned Master Spec change wins;
2. Training Contract must be updated to match;
3. tests must be updated;
4. implementation follows the new contract.

No silent semantic drift.
