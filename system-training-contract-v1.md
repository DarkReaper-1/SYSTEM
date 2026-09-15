# SYSTEM — TRAINING CONTRACT V1
Version: 1.0
Status: Implementation Contract
Scope: Workout, Set, Prescription, Performance, Evidence, Evaluation, Correction, and Credit interfaces.

> This contract turns the SYSTEM Master Spec into concrete training-domain invariants.
> It is the contract Cursor implementation and tests must satisfy.

---

# 1. CONTRACT PRINCIPLES

1. A workout is a bounded session envelope.
2. A set is the atomic performance claim.
3. Actual performance is typed.
4. Prescription is immutable intent; actual is immutable accepted fact.
5. Evidence attaches to canonical records.
6. Evaluation is server-derived.
7. Credit is append-only and idempotent.
8. Rank/Level/Quest/Achievement are downstream consumers, not training-record fields writable by clients.
9. Corrections are events.
10. No client input can directly mint progression.

---

# 2. IDENTITIES

## Athlete

```text
athlete_id: UUID
account_id: UUID
```

V1:

`account_id → exactly one athlete_id`

No client operation may select another athlete without authorization.

---

# 3. WORKOUT CONTRACT

Conceptual shape:

```text
Workout {
  workout_id
  athlete_id
  status
  intent_type
  prescription_snapshot_id?
  started_at
  ended_at?
  received_at
  timezone_offset
  session_clock
  created_at
  accepted_at?
  schema_version
  device_session_id?
}
```

### `intent_type`

```text
FREE
PRESCRIBED
```

### `status`

```text
DRAFT
OPEN
SUBMITTED
EVALUATING
CLOSED
ABANDONED
VOIDED
```

### Rules

- DRAFT may mutate freely.
- OPEN accepts training records under session policy.
- SUBMITTED freezes accepted actual facts.
- CLOSED is immutable except correction/void events.
- ABANDONED blocks new normal writes.
- VOIDED remains queryable.
- Empty submit → ABANDONED.
- Abandoned records default to `HISTORY_ONLY`.
- Recovery, if implemented, is explicit and time-bounded.
- `started_at` / `ended_at` are CLAIMED_CLOCK bout/session bounds (client-claimed).
- `received_at` is server receipt of the workout write (OPEN or accept, as applicable).
- `timezone_offset` is captured at OPEN for display reconstruction. It does not bucket economy weeks.
- `session_clock` is a clock identity selected at OPEN. It is not a duration. It cannot be changed after the first accepted set.

```text
session_clock = {
  selected_at_received,
  timezone_offset
}
```

Intra-session elapsed time is derived from claimed `started_at` / `ended_at` relative to this selection. Economy does not use `session_clock` to bucket weeks.

---

# 4. WORKOUT BOUNDS

CreditPolicy must define:

```text
max_open_duration
max_idle_gap
max_offline_delay
max_recovery_window
```

Optional modality-specific bounds are policy configuration.

No workout may be used as an indefinite economic staging container.

Economy windows are bounded from the first accepted training event.

---

# 5. SET CONTRACT

Conceptual shape:

```text
TrainingSet {
  set_id
  workout_id
  athlete_id
  exercise_id
  sequence_index
  declared_role
  outcome
  started_at
  ended_at?
  actual
  source
  server_provenance
  client_event_id
  schema_version
  prescription_ref?
  prescription_snapshot?
  side
  context
  evidence_ids[]
  to_failure
  amrap
  terminated_reason?
  created_at
  received_at
  accepted_at
  timezone_offset
}
```

### Required

All fields above marked conceptually required by the Master Spec must be present.

`evidence_ids` must exist even when empty.

`started_at` / `ended_at` are CLAIMED_CLOCK bout bounds. They are this record’s claimed time. V1 does not store a separate `claimed_at` column; `claimed_at` in §46 refers to these claimed bounds.

`received_at` is server receipt. `timezone_offset` is copied from the parent workout at accept.

### `source` and `server_provenance`

Both use:

```text
MANUAL
CAMERA
WEARABLE
IMPORTED
```

`source` is the client-submitted origin claim. It is untrusted. It is analogous to Evidence `submitted_source`.

`server_provenance` is server-derived. The client cannot write it. Import quarantine and official-projection filters key off `server_provenance`, not `source`.

Do not invent additional origin tokens (`APP`, `WATCH`, `WEB`). Do not treat client `source` as provenance.

---

# 6. SET ROLE

```text
WARMUP
WORK
BACKOFF
TEST
OTHER
```

Declared role is an input.

Evaluator may derive an authoritative detected role.

The resulting:

```text
effective_role
```

is the role used by:

- credit;
- baseline;
- PR.

Display may preserve both values.

---

# 7. SET OUTCOME

```text
COMPLETED
FAILED
TERMINATED
```

Failure is never represented by omission.

A failed set may contain:

- completed reps;
- completed duration;
- partial distance;
- termination reason.

V1 credit policy:

```text
FAILED = ZERO progression credit
```

---

# 8. PERFORMANCE UNION

Use one discriminated union.

```text
Performance =
  RepsLoad
| RepsBodyweight
| RepsAssisted
| Duration
| DistanceTime
| DistanceOnly
| TimeOnly
| IsometricHold
| CardioSegment
```

No generic nullable float bag.

---

# 9. REPS_LOAD

```text
{
  type: "REPS_LOAD",
  reps: integer >= 0,
  load: Load
}
```

Load is structured.

---

# 10. REPS_BODYWEIGHT

```text
{
  type: "REPS_BODYWEIGHT",
  reps: integer >= 0,
  bodyweight_snapshot_id: UUID
}
```

No magic `null` load.

---

# 11. REPS_ASSISTED

```text
{
  type: "REPS_ASSISTED",
  reps: integer >= 0,
  assist: {
    magnitude,
    unit,
    kind
  }
}
```

Assistance/resistance direction is explicit.

Assisted work cannot silently become strict bodyweight.

---

# 12. DURATION

```text
{
  type: "DURATION",
  seconds: number > 0
}
```

V1 `DURATION` contains `seconds` only. It has no `load` field.

Pure timed exercises cannot receive arbitrary reps.

If load-bearing duration becomes necessary later, it must be introduced as a versioned contract change. Do not extend this V1 `DURATION` shape.

---

# 13. DISTANCE_TIME

```text
{
  type: "DISTANCE_TIME",
  distance: Measurement,
  duration_seconds: number > 0
}
```

Pace is derived.

---

# 14. DISTANCE_ONLY / TIME_ONLY

```text
DISTANCE_ONLY {
  distance: Measurement
}

TIME_ONLY {
  duration_seconds: number > 0
}
```

Use only when the exercise definition permits missing counterpart data.

---

# 15. ISOMETRIC_HOLD

```text
{
  type: "ISOMETRIC_HOLD",
  duration_seconds: number > 0,
  load?: Load
}
```

Reps are illegal unless the exercise definition explicitly declares a rep-based variation.

---

# 16. CARDIO_SEGMENT

```text
{
  type: "CARDIO_SEGMENT",
  sport: SportCode,
  distance?: Measurement,
  duration_seconds: number > 0,
  elevation_gain?: Measurement,
  surface?: SurfaceCode,
  grade?: number
}
```

A run/walk is not stored as:

`reps=0, weight=0`.

Intervals are represented as ordered segments.

---

# 17. MEASUREMENT

All measurements contain:

```text
{
  magnitude: decimal,
  unit: UnitCode
}
```

Input units are canonicalized once at the server write boundary.

Stored evaluator values use canonical units.

The client’s submitted/native unit is not persisted as part of the training record in V1; the write boundary canonicalizes the submitted measurement exactly once, and only the canonical magnitude and canonical unit are stored for evaluation and economic purposes.

Any preservation of the original submitted unit for audit purposes is out of scope for V1 and requires a future versioned contract change.

Client display conversion is presentation-only.

---

# 18. LOAD

```text
{
  magnitude: decimal,
  unit: UnitCode,
  kind: LoadKind
}
```

`LoadKind`:

```text
EXTERNAL
BODYWEIGHT
ASSISTED
ADDED_TO_BODYWEIGHT
PERCENT_1RM
```

`PERCENT_1RM` is prescription-only.

---

# 19. LATERALITY

```text
NONE
LEFT
RIGHT
ALTERNATING
```

Exercise catalog declares expected laterality.

Unilateral PR eligibility requires LEFT/RIGHT records.

ALTERNATING records may be valid history but cannot create unilateral PRs without side-specific evidence.

---

# 20. CONTEXT

Context is a controlled structure:

```text
{
  equipment?
  modifiers[]
}
```

Only catalog-approved modifiers can split baseline/PR meaning.

Display-only modifiers cannot create new progression contexts.

---

# 21. EXERCISE CATALOG CONTRACT

Each canonical exercise contains at minimum:

```text
exercise_id
name
family
load_model
laterality
allowed_performance_types[]
success_metric
standard_constraints
allowed_context_splits[]
pr_classes[]
version
```

Exercise IDs are stable.

Catalog aliases map to canonical IDs.

Client cannot invent a canonical exercise by sending an arbitrary label.

### V1 catalog scope

- Authorship: system-authored canonical IDs only. Client cannot create canonical IDs.
- Aliases map to canonical IDs.
- Families: the five V1 families in §22 only.
- Closed modifiers: only catalog-approved codes may split baseline/PR context. Display-only modifiers cannot.
- Initial canonical ID list: **not yet enumerated**. Attack-fixture names (squat, plank, run, burpees, pull-up) are not IDs. Schema seed is forbidden until this list is written here.

Do not seed a database from fixture prose.

---

# 22. FAMILY CONTRACT

V1:

```text
ABSOLUTE_STRENGTH
REPS_AT_LOAD
ISOMETRIC_DURATION
LOCOMOTION
SKILL_VARIATION
```

Family constraints are authoritative for evaluation.

A client cannot relabel an exercise family.

---

# 23. PRESCRIPTION CONTRACT

Conceptual:

```text
PrescriptionSnapshot {
  prescription_snapshot_id
  source_type
  source_id?
  version
  created_at
  targets[]
  success_criteria[]
}
```

Once a workout begins, the snapshot is immutable.

Targets may include:

```text
exercise_id
set_index
mode
target_reps?
target_load?
target_duration?
target_distance?
time_cap?
rpe_cap?
rest_guidance?
```

---

# 24. AMRAP CONTRACT

```text
mode = AMRAP
```

may define:

- load;
- time cap;
- target;
- success criteria.

Actual remains normal performance data.

AMRAP is not a special set table.

---

# 25. EVIDENCE CONTRACT

Conceptual:

```text
Evidence {
  evidence_id
  subject_type
  subject_id
  submitted_source
  server_provenance
  produced_at
  producer_version?
  payload_ref?
  content_hash?
  perceptual_fingerprint?
  observations[]
  review_state
  created_at
}
```

`submitted_source` is not trusted as final evidence class.

---

# 26. EVIDENCE PROVENANCE

Server-derived provenance:

```text
MANUAL
CAMERA
WEARABLE
IMPORTED
```

Client cannot send:

```text
credit_tier
claim_state
rank_eligible
verified
trust_weight
official
```

and have it accepted as authoritative.

---

# 27. MANUAL EVIDENCE

Manual entry becomes:

```text
MANUAL_ASSERTION
```

It is valid history when coherent.

It is not automatically verified truth.

Manual credit is bounded.

Manual evidence does not establish rank eligibility in V1.

---

# 28. CAMERA EVIDENCE

Camera evidence must pass:

1. payload validation;
2. observation extraction;
3. confidence/completeness requirements;
4. exercise/context match;
5. temporal alignment;
6. assertion agreement policy.

Media existence alone does not create verified credit.

---

# 29. TEMPORAL ALIGNMENT

An observation is eligible only when:

```text
observation_window
```

aligns with:

```text
set.started_at … set.ended_at
```

within configured clock-skew tolerance.

Out-of-window media is invalid for that set's trust upgrade.

---

# 30. MANUAL + CAMERA CONFLICT

Both remain attached to one set.

History can display:

```text
Claim: 10 reps
Observation: 6 reps
```

Evaluator decides credit.

If disagreement exceeds policy margin:

- verified status is denied;
- rank eligibility is denied;
- progression is reduced/zeroed according to policy.

No second set is created.

---

# 31. MEDIA REUSE

Use:

- exact content hash;
- perceptual fingerprint where available;
- temporal overlap;
- reuse count.

Suspicious reuse:

- remains in history;
- cannot create duplicate verified credit;
- receives suspicion/invalid evidence treatment.

---

# 32. CLAIM STATE

Server-derived:

```text
PENDING
VALID
PARTIAL
REJECTED
```

Meaning:

- PENDING = awaiting evaluation;
- VALID = coherent/eligible under claim policy;
- PARTIAL = some constraints/credit conditions failed;
- REJECTED = not credible/eligible for progression.

Raw history is retained.

---

# 33. CREDIT TIER

Server-derived.

V1 examples:

```text
VERIFIED
MANUAL_LIMITED
HISTORY_ONLY
ZERO
```

Credit tier is not a user-editable property.

---

# 34. RANK ELIGIBILITY

Server-derived.

It is never stored as a client-authoritative flag.

Rank eligibility requires downstream predicates.

Training records only provide evidence inputs.

---

# 35. ACCEPTANCE

Before acceptance:

```text
DRAFT
```

may be edited.

At acceptance:

- actual facts freeze;
- server timestamps are recorded;
- idempotency is committed;
- policy/baseline snapshot is pinned as required.

After acceptance, changes occur only through correction events.

---

# 36. CORRECTION EVENT

Conceptual:

```text
CorrectionEvent {
  correction_id
  target_record_id
  old_value
  new_value
  reason
  actor
  created_at
}
```

Raw original remains intact.

Corrections are rate-limited and time-boxed for self-service.

Correction loops cannot produce net-positive economic value.

---

# 37. EVALUATION INSTANCE

Conceptual:

```text
Evaluation {
  evaluation_id
  target_record_id
  evaluator_version
  policy_id
  baseline_snapshot_id
  result
  derived_observations
  created_at
}
```

Multiple evaluations may exist.

They are immutable.

---

# 38. CURRENT EVALUATION POINTER

A current pointer is server-owned.

Pointer changes create:

```text
PointerChange {
  pointer_type
  target_id
  previous_id?
  new_id
  actor
  reason
  created_at
}
```

The pointer history is append-only.

---

# 39. BASELINE SNAPSHOT

Evaluation receives a pinned baseline snapshot:

```text
BaselineSnapshot {
  snapshot_id
  athlete_id
  exercise/context key
  version
  values
  created_at
}
```

Evaluation must not read a mutable live baseline halfway through processing.

---

# 40. CREDIT LEDGER

Conceptual:

```text
CreditLedgerEntry {
  ledger_entry_id
  athlete_id
  training_record_id
  policy_id
  entry_kind
  amount
  lineage_id
  created_at
}
```

Entry kinds include:

```text
GRANT
COMPENSATION
```

V1 reversal/clawback is `COMPENSATION`. Do not add a separate `CLAWBACK` `entry_kind` unless this contract is explicitly versioned.

Other kinds may be added only by explicit policy.

---

# 41. ECONOMIC UNIQUENESS

The primary economic identity is:

```text
training_record_id × credit_policy_id
```

Evaluator version is metadata.

A new evaluator version does not create a fresh independent grant chain.

Re-evaluation must compensate prior economic state before applying the replacement state.

All operations are serialized per training record.

---

# 42. POLICY PINNING

Each credit result stores:

```text
policy_id
```

The applicable policy is pinned at acceptance/economic admission.

Later policy deployments cannot retroactively sweeten queued submissions.

Policy migrations require explicit migration logic and compensating ledger entries.

---

# 43. IDEMPOTENCY

Every client write contains:

```text
client_event_id
```

Unique within the athlete/write namespace.

Retrying the same event returns the original record.

No second credit is created.

---

# 44. SEMANTIC DEDUPE

Secondary duplicate detection compares:

- athlete;
- approximate time window;
- exercise;
- performance signature;
- context;
- sequence/session relationship.

Default suspicious duplicate treatment:

```text
history retained
credit = ZERO
suspicion += 1
```

Do not reveal exact similarity thresholds to clients.

Sets with distinct sequence indices inside the same workout are exempt from cross-workout clone logic.

---

# 45. RATE LIMITS

Limits are independent of batch size.

At minimum policy supports:

```text
hourly credited stimulus
daily credited stimulus
weekly credited stimulus
manual weekly global
manual weekly per-family
```

These are keyed to immutable athlete identity and server economy time.

### V1 numeric cells

The following keys are required. **Values are not locked.** Inventing figures in schema is forbidden. Schema migration remains blocked until this table is filled by an explicit contract version.

```text
max_open_duration
max_idle_gap
max_offline_delay
max_recovery_window
hourly credited stimulus
daily credited stimulus
weekly credited stimulus
manual weekly global
manual weekly per-family
```

Also still unquantified, already named elsewhere:

```text
self-serve correction time box
self-serve correction rate
semantic-duplicate time window
device-clock skew tolerance
observation-agreement margin
future-claim rejection rule
backdate bound
baseline_upward_change_limit
baseline_downward_change_limit
baseline_minimum_sample
baseline_floor
```

---

# 46. TIME CONTRACT

Store:

```text
started_at / ended_at   // CLAIMED_CLOCK; this is claimed_at for the record
received_at             // RECEIVED_CLOCK
session_clock           // selected clock identity; not a duration
timezone_offset         // display reconstruction only
```

`claimed_at` is not a third timestamp. For a workout or set, claimed time is `started_at` (and `ended_at` when present).

`session_clock` is selected at OPEN:

```text
session_clock = {
  selected_at_received,
  timezone_offset
}
```

It cannot change after the first accepted set. Intra-session elapsed time is derived from claimed `started_at` / `ended_at`. Economy does not bucket weeks with `session_clock`.

Economy week:

**UTC server epoch week.**

Client timezone cannot reset weekly economy.

Future claims do not mint.

Backdating is bounded.

Economy, caps, and rate limits use `received_at` / UTC epoch week, never `timezone_offset` or `started_at`.

---

# 47. OPEN / OFFLINE CONTRACT

Policy defines:

```text
max_open_duration
max_idle_gap
max_offline_delay
```

Offline packets are accepted only within the permitted delay policy.

Large batches are not the sole trigger for stricter treatment.

Rolling rates always apply.

---

# 48. ABANDONED CONTRACT

If a workout is automatically abandoned:

- raw accepted sets remain;
- new writes are blocked;
- default progression credit = `HISTORY_ONLY`;
- quests do not complete from abandoned state;
- bounded recovery may restore a valid submit state.

Network failure recovery must not require the athlete to recreate all history.

---

# 49. PENDING CONTRACT

PENDING cannot be consumed as success by any downstream projection.

Forbidden consumers:

- Quest;
- Achievement;
- streak;
- Rank;
- PR;
- baseline;
- capability;
- progression.

Allowed:

- awaiting evaluation UI;
- operational queues.

---

# 50. IMPORT CONTRACT

Imports are permanently quarantined in V1.

They may appear only as:

```text
Imported Archive
```

They cannot feed official:

- baseline;
- PR;
- rank;
- capability;
- progression.

No automatic promotion path exists in V1.

---

# 51. BASELINE INPUT CONTRACT

Official baseline projections read only eligible records:

```text
claim_state ∈ {VALID, PARTIAL}
AND void = false
AND effective_role eligible
AND provenance != IMPORT
AND credit_tier eligible_by_policy
```

Manual-only data cannot arbitrarily lower official prescription authority.

Baseline statistics must be robust and slow-moving.

---

# 52. PR INPUT CONTRACT

PR projections read eligible history only.

PR key includes all context dimensions that materially change meaning.

Unilateral records remain side-aware.

Alternating records cannot silently create bilateral PRs.

PRs are not XP sources.

---

# 53. QUEST INPUT CONTRACT

Quest completion reads credited stimulus and explicit success predicates.

It cannot use:

- workout existence;
- pending status;
- set count;
- achievement count;
- PR count.

Quest targets must include a meaningful stimulus floor.

Recurring quests may require family diversity/intensity bands.

---

# 54. ACHIEVEMENT INPUT CONTRACT

Achievements are predicates only.

V1:

```text
achievement → 0 XP
achievement → 0 rank points
```

Achievements can unlock titles/badges/narrative state.

They cannot become a hidden mint.

---

# 55. RANK INPUT CONTRACT

Rank does not read Level as a predicate in V1.

Rank reads:

- pinned standards;
- consistency;
- credited sessions/families;
- evidence coverage;
- recency;
- valid current state.

Manual-only evidence is not sufficient for rank.

A single verified session cannot erase a long absence of evidence.

---

# 56. LEVEL INPUT CONTRACT

Level reads the credit ledger.

No writable XP balance exists.

Ledger rebuild must produce deterministic Level.

If credit is clawed back, Level reflects the net ledger state.

---

# 57. BODYWEIGHT CONTRACT

Bodyweight is a versioned snapshot:

```text
BodyMassSnapshot {
  snapshot_id
  athlete_id
  mass
  unit_canonical
  recorded_at
  received_at
}
```

Suspicious rapid movement may suspend relative metrics.

Absolute external-load credit can continue when otherwise valid.

---

# 58. UNIT CONTRACT

Canonicalize exactly once.

Example:

```text
client: 100 lb
      ↓
write boundary
      ↓
canonical stored value
      ↓
evaluator
```

Evaluator never performs a second hidden conversion.

Only the canonical magnitude and canonical unit are stored. The client’s submitted/native unit is not persisted in V1.

Unit display conversion occurs only at presentation.

---

# 59. EVALUATOR INPUT SNAPSHOT

An evaluation must be deterministic over:

```text
immutable record
+ correction projection
+ prescription snapshot
+ exercise version
+ evidence observations
+ baseline snapshot
+ policy id
+ committed prior state required by policy
```

No mutable live baseline read.

---

# 60. EVALUATOR OUTPUT

Conceptual:

```text
EvaluationResult {
  claim_state
  effective_role
  credit_inputs
  plausibility_flags[]
  evidence_resolution
  rank_evidence_eligibility
  baseline_eligibility
  pr_eligibility
  quest_eligibility
}
```

The evaluator does not directly mutate Level or XP balance.

---

# 61. CREDIT WRITER

The credit writer consumes evaluator output.

It:

1. verifies policy id;
2. verifies economic uniqueness;
3. serializes against concurrent writes;
4. emits grant/compensation entries;
5. commits atomically;
6. triggers downstream projection refresh.

It does not rewrite history.

---

# 62. RE-EVALUATION

Re-evaluation is allowed for:

- new valid evidence;
- correction;
- evaluator bug fix;
- explicit policy migration.

Re-evaluation must:

- preserve previous evaluation;
- preserve ledger lineage;
- compensate old economic result when necessary;
- apply replacement result;
- never net-positive merely because evaluator version changed.

---

# 63. SINGLE-FLIGHT

All economic evaluation triggers for a record are single-flight.

Concurrent:

- initial evaluation;
- evidence attach;
- correction;
- migration;

must serialize per target record.

Retries must be safe.

---

# 64. VOID

Void is an append-only state transition.

Void does not erase:

- original set;
- original workout;
- correction history;
- evidence;
- evaluation;
- ledger lineage.

Void removes the record from eligible projections according to policy.

---

# 65. PROJECTION RULE

Every projection must declare:

```text
source eligibility
clock
provenance filters
claim-state filters
credit-tier filters
void behavior
```

No generic “workout total” API may silently become the source for progression.

---

# 66. TEST CONTRACT

Minimum automated tests:

### Data integrity
- typed performance validation;
- canonical unit conversion;
- canonical exercise enforcement;
- required evidence array;
- laterality validation.

### Lifecycle
- draft editing;
- accept freeze;
- abandoned;
- void;
- correction events.

### Economy
- duplicate retry;
- semantic duplicate;
- concurrent evaluation;
- correction oscillation;
- policy-version re-evaluation;
- cap enforcement;
- manual global + family ceiling;
- no session XP;
- no achievement XP;
- no PR XP.

### Time
- future;
- backdate;
- timezone hopping;
- open-session expiry;
- offline delay;
- rolling cap.

### Evidence
- manual;
- camera;
- black media;
- wrong media;
- temporal mismatch;
- manual/camera disagreement;
- media reuse;
- extractor version pinning.

### Progression
- pending isolation;
- import quarantine;
- baseline poisoning;
- prescription sandbagging;
- rank without Level;
- rank evidence coverage;
- rank revocation;
- ledger rebuild.

---

# 67. ATTACK FIXTURES

The implementation test suite must include fixtures for:

```text
100 warmup sets
10x10 fantasy load
duplicate retry
new-UUID semantic duplicate
60→160 correction
correction A→B→A loop
baseline crash week
easy prescription farming
insane prescription
30 sets in 4 minutes
timezone day split
future timestamp
42.2 km impossible run
120 burpees / 60 sec
3-hour plank
offline reorder
two evaluator workers
new evaluator version
fake CAMERA source
black video
yesterday's video on today's set
same media on many sets
forged import
alternating unilateral compression
assisted pull-up labeled BW
bodyweight oscillation
junk variation PR
achievement first-log faucet
train-3-days checkbox quest
pinned-standard checklist rank
pending streak
abandoned workout mint
pointer rollback
GRANT-only projection
kg/lb double conversion
two humans / one account
200 micro-sets
```

Each fixture must assert:

1. raw claim is preserved where applicable;
2. progression outcome is bounded;
3. no duplicate net grant occurs;
4. downstream projections respect eligibility;
5. audit lineage remains reconstructable.

---

# 68. CLIENT CONTRACT

Client may submit:

- training intent;
- actual performance;
- claimed timestamps;
- declared role;
- evidence payloads;
- client event ID;
- presentation preferences.

Client may not submit authoritative:

- XP;
- Level;
- Rank;
- credit tier;
- claim state;
- evidence trust;
- provenance upgrade;
- official PR;
- official baseline;
- evaluation result.

---

# 69. SERVER CONTRACT

Server owns:

- athlete authorization;
- canonical IDs;
- canonical units;
- received timestamps;
- acceptance;
- lifecycle transitions;
- evidence provenance;
- evaluation;
- effective role;
- plausibility;
- credit;
- ledger;
- baselines;
- PR state;
- quest eligibility;
- rank eligibility;
- pointer changes.

---

# 70. IMPLEMENTATION RULE

If a proposed database/API shape makes any of the following possible, reject it:

- nullable generic `weight`/`reps` as the entire performance model;
- client-written XP;
- client-written rank;
- mutable accepted actuals;
- separate camera workout records;
- evaluation-version-only economic uniqueness;
- live baseline reads during evaluation;
- session-completion XP;
- achievement XP;
- PR XP;
- import-as-official;
- pending-as-success;
- silent corrections;
- silent pointer changes;
- unit conversion in multiple layers.

---

# 71. V1 LOCK

This contract is an implementation gate.

Before schema migration:

- Master Spec and Training Contract must agree;
- adversarial test fixtures must be enumerated;
- CreditPolicy numeric values must be explicit (**cells listed; values not yet locked**);
- ExerciseCatalog initial scope must be explicit (**authorship and families locked; canonical ID list not yet enumerated**);
- all client-writable fields must be enumerated;
- all server-derived fields must be enumerated.

Locked before schema and already written:

- V1 `DURATION` has no `load` field;
- pre-canonical/native units are not persisted;
- V1 ledger reversal kind is `COMPENSATION` only;
- claimed time is `started_at` / `ended_at`; `session_clock` is a selected identity, not a duration;
- `TrainingSet.source` is untrusted client origin; `server_provenance` is server-derived; tokens `MANUAL | CAMERA | WEARABLE | IMPORTED`.

Only then should database schema and API implementation begin.
