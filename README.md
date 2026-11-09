# Prompt Engine Architecture

This document outlines the architecture and logic flow for the Health Services Prompt Engine. The system is designed to separate **Strategy** (Marketing/Clinical definitions) from **Execution** (Delivery, Consent, Frequency Capping).

## 1. High-Level Logic Flow

1.  **Definition**: Marketers/Admins define *Segments* (Who) and *Nudge Definitions* (What).
2.  **Generation & Triggers**:
    *   **Segment-Based (Pull)**: A scheduled job runs, compiles the Segment definition tree into SQL, finds matching users (e.g., "All Diabetics"), and creates a `nudge_enrollment` for each person not already enrolled.
    *   **Event-Based (Push)**: External clinical systems insert signals (e.g., "Doctor Recommendation") into `clinical_recommendation`. The engine reacts immediately and creates a `nudge_enrollment` linked to the signal.
    *   For every new enrollment, the engine assigns the experiment group (once) and queues **Step 1** in `prompt_queue`.
3.  **Filtration (The Gatekeeper)**: Before sending, the system checks the **Goal**, **Consent** (Opt-in) and **Frequency Caps** (Anti-spam).
4.  **Delivery**: The message is sent via the appropriate channel provider. Sending step N queues step N+1.
5.  **Feedback Loop**:
    *   **Interaction**: User opens/clicks (logged in `interaction_log`).
    *   **Action**: The system checks if the goal is met using the `goal_segment_id`, counting only evidence after `enrolled_at`. If met, the enrollment is marked `completed` and the remaining workflow stops.

### 1.1. Two Levels: Enrollment vs. Message

| Level | Table | One row per | Holds |
|:---|:---|:---|:---|
| **Journey** | `nudge_enrollment` | Person × Nudge run | Trigger source (campaign / signal), experiment group, goal status, enrollment window |
| **Message** | `prompt_queue` | Step attempt | Channel, schedule, delivery status, resolved `context_data` snapshot |

The enrollment is the unit for **dedup** ("is this person already in this nudge?"), **cancellation** ("stop the remaining steps") and **analytics** ("did the nudged person reach the goal?"). The queue is a pure delivery log.

---

## 2. Targeting & Triggers

The system supports two main ways to target users: **Segmentation** (Criteria-based) and **Events** (Signal-based).

### 2.0. Segmentation Logic (The "Pull" Model)

A segment's rules are stored as one **logic tree** in `segment.definition` (jsonb). The Query Builder walks the tree recursively and compiles it into SQL. There are no labels and no expression string to parse: the structure of the tree *is* the logic.

**Node Types:**

| Node | Shape | Compiles to |
|:---|:---|:---|
| **and** | `{"id": "...", "and": [node, node, ...]}` | `(child AND child ...)` |
| **or** | `{"id": "...", "or": [node, node, ...]}` | `(child OR child ...)` |
| **not** | `{"id": "...", "not": node}` | `NOT (child)` |
| **table leaf** | `{"id": "...", "table": "lab_result", "where": [...], "time": {...}}` | `EXISTS (SELECT 1 FROM lab_result_v x WHERE x.person_id = p.id AND <conditions> AND <time filter>)` |
| **segment leaf** | `{"id": "...", "segment": 50}` | The compiled definition of segment 50 (nesting) |

**Rules:**
*   **Conditions in one leaf are ANDed on the same row.** `"where": [loinc_code = 4548-4, value_numeric < 7]` means *one* lab result that is an HbA1c **and** has a value under 7.
*   **OR** is expressed with an `or` node, or with the `IN` operator inside one condition (`icd_code IN ('E10','E11')`).
*   **`master_person` leaves** compile to plain conditions on `p` (no `EXISTS`), since it is the base table.
*   **Every node has an `id`** (unique within the segment), so analytics can report per-node counts that stay stable when the tree is edited.
*   **Values keep their JSON type** (`9` is a number, `"E11%"` a string), which avoids casting from varchar.

**Time Filters (`time` on a table leaf):**
All external tables are queried through views that expose a standard `event_date` column (see the convention at the top of `prompt_engine.dbml`).

| `time.scope` | Filter | Allowed in |
|:---|:---|:---|
| `any_time` (default, or omit `time`) | none | All segments |
| `within_last` + `window` | `x.event_date >= CURRENT_DATE - window` | All segments |
| `since_enrollment` | `x.event_date BETWEEN e.enrolled_at::date AND e.expires_at::date` | Goal segments only (see 2.3) |

`master_person` has no `event_date`, so its leaves cannot have a time filter.

**Example: Segment 300 "Diabetes follow-up"**

*Poorly controlled diabetics, **or** anyone without a flu shot in the last year.*

```json
{
  "id": "root",
  "or": [
    { "id": "diabetic_uncontrolled",
      "and": [
        { "id": "A", "table": "clinical_diagnosis",
          "where": [ {"field": "icd_code", "op": "LIKE", "value": "E11%"},
                     {"field": "status",   "op": "=",    "value": "active"} ] },
        { "id": "B", "table": "lab_result",
          "where": [ {"field": "loinc_code",    "op": "=", "value": "4548-4"},
                     {"field": "value_numeric", "op": ">", "value": 9} ],
          "time": {"scope": "within_last", "window": "6 months"} }
      ] },
    { "id": "not_C",
      "not":
        { "id": "C", "table": "vaccination_record",
          "where": [ {"field": "vaccine_code", "op": "=", "value": "141"},
                     {"field": "status",       "op": "=", "value": "completed"} ],
          "time": {"scope": "within_last", "window": "12 months"} } }
  ]
}
```

**Generated SQL:**
```sql
SELECT p.id
FROM master_person p
WHERE
  (
    EXISTS (SELECT 1 FROM clinical_diagnosis_v x            -- A
            WHERE x.person_id = p.id
              AND x.icd_code LIKE 'E11%'
              AND x.status = 'active')
    AND
    EXISTS (SELECT 1 FROM lab_result_v x                    -- B
            WHERE x.person_id = p.id
              AND x.loinc_code = '4548-4'
              AND x.value_numeric > 9
              AND x.event_date >= CURRENT_DATE - INTERVAL '6 months')
  )
  OR
  NOT EXISTS (SELECT 1 FROM vaccination_record_v x          -- NOT C
              WHERE x.person_id = p.id
                AND x.vaccine_code = '141'
                AND x.status = 'completed'
                AND x.event_date >= CURRENT_DATE - INTERVAL '12 months');
```

Note that `NOT C` negates the *whole row*: "no completed flu shot in the last 12 months", not "has some vaccination that isn't a flu shot".

**Validation on Save:**
1.  The tree matches the definition JSON Schema (node shapes, operators).
2.  Every `table` and `field` is on the allowlist (see TODO: Dynamic SQL safety).
3.  `since_enrollment` appears only in `type='goal'` segments, including through nested segments.
4.  `segment_dependency` is rewritten from the `{"segment": N}` leaves; a save that would create a cycle is rejected.

**Result**: A list of `person_id`s. Each one that has no `active` / `control_group` enrollment for this nudge gets a new `nudge_enrollment`.

### 2.1. Multi-Table & Unified Segmentation

The system supports advanced targeting beyond simple `master_person` attributes.

**Multi-Table Support**
Each table leaf names its `table`. The Query Builder turns it into an `EXISTS` sub-query against that table's view, correlated on `person_id`.
*   *Example*: `{"table": "lab_result", "where": [{"field": "loinc_code", "op": "=", "value": "4548-4"}]}` (has an HbA1c test).

**External Data Integration**
The engine connects to read-only external tables for rich segmentation:
*   **Clinical**: `clinical_diagnosis` (ICD-10), `lab_result` (LOINC), `medication_prescription` (RxNorm/ATC), `vaccination_record` (CVX).
*   **Claims**: `insurance_claim` and `claim_line_item` (CPT/HCPCS) for billing data.
*   **Encounters**: `hospital_encounter` (Admissions, Surgeries) and `appointment_history`.
*   **Risk & Social**: `risk_score` (ML models) and `social_determinants`.

Each is read through its `<table>_v` view, which adds `event_date`. The view is also the single place to adapt when a source system changes its columns.

**Unified Segmentation (Ad-Hoc Filters)**
Instead of separate "Campaign Criteria", the system uses a **Unified Segmentation Model**.
*   Every Campaign points to a single `Segment`.
*   **Library Segments**: Reusable Personas (e.g., "Seniors").
*   **Ad-Hoc Segments**: If a campaign needs specific filtering, a new `Segment` is created with `type='campaign_ad_hoc'`.
    *   Its definition is: `{"and": [{"segment": <base library segment>}, <ad-hoc rules>]}`.
    *   This keeps the architecture clean: The Query Builder only ever resolves a Segment.

### 2.2. Segment Composition & Exclusion

The engine supports complex set operations including **Nesting** (Union/Intersection of Segments) and **Exclusion** (Negation).

**Segment Composition (Nesting)**
A `segment` leaf refers to another existing Segment instead of raw conditions.
*   *Example*: Define a "Diabetes" segment (id 40). Then define "High Risk Diabetes" as:
    ```json
    {"and": [ {"segment": 40},
              {"table": "lab_result",
               "where": [{"field": "loinc_code", "op": "=", "value": "4548-4"},
                         {"field": "value_numeric", "op": ">", "value": 9}]} ]}
    ```
*   This promotes reusability and consistency. `segment_dependency` records that "High Risk Diabetes" uses segment 40, so segment 40 cannot be deleted while it is in use.

**Exclusion (The `not` Node)**
*   *Example*: Target users who have Asthma but are NOT in the "Seniors" segment (id 50):
    ```json
    {"and": [ {"table": "master_person", "where": [{"field": "has_asthma", "op": "=", "value": true}]},
              {"not": {"segment": 50}} ]}
    ```

### 2.3. Goal Segments (Defining Success)

Just like "Personas", "Goals" are defined using the standard **Segment** structure (with `type='goal'`).
*   **Unified Logic**: The system checks: *"Has the user entered the Goal Segment since they were enrolled?"*
    *   If **YES** -> The Nudge is successful. Stop Reminders.
    *   If **NO** -> Send Nudge / Continue Reminders.

**Example: Goal "HbA1c Under Control"**
```json
{ "id": "goal", "table": "lab_result",
  "where": [ {"field": "loinc_code",    "op": "=", "value": "4548-4"},
             {"field": "value_numeric", "op": "<", "value": 7.0} ],
  "time": {"scope": "since_enrollment"} }
```

**Structure:**
*   **Nudge Definition**: Points to `goal_segment_id`.
*   **Completion Window**: `goal_completion_window` on `nudge_definition` (e.g., "User has 14 days to complete this"). At enrollment, `nudge_enrollment.expires_at = enrolled_at + goal_completion_window`.

**Time-Bound Evaluation:**
Goals are evaluated **per enrollment**, not per person. The compiled goal starts from `nudge_enrollment` so each person is checked against their *own* window:

```sql
SELECT e.id
FROM nudge_enrollment e
JOIN master_person p ON p.id = e.person_id
WHERE e.nudge_definition_id = :nudge_id
  AND e.status IN ('active', 'control_group')
  AND EXISTS (SELECT 1 FROM lab_result_v x
              WHERE x.person_id = e.person_id
                AND x.loinc_code = '4548-4'
                AND x.value_numeric < 7.0
                AND x.event_date BETWEEN e.enrolled_at::date AND e.expires_at::date);
```

*   Only evidence **dated inside the window** counts. An HbA1c test from three years ago no longer marks a new enrollment as complete, and evidence after the window does not count either.
*   Because the time filter sits in the same `EXISTS` as the conditions, the *same* lab result must be an HbA1c, be under 7, **and** fall inside the window.
*   `since_enrollment` compares dates (`::date`), so a test done on the enrollment day itself counts.
*   **People who already meet the goal at enrollment** should be excluded by the *target* segment, e.g. `Diabetics AND NOT (HbA1c < 7 within_last 3 months)`. This is a modelling convention, not an engine rule.

**Outcome:**
*   Goal met → `nudge_enrollment.status = 'completed'`, `goal_met_at` is set, and any not-yet-sent step is marked `cancelled_completed`.
*   Window passes without the goal → `nudge_enrollment.status = 'expired'`.

The goal is also evaluated for **control-group** enrollments, which is how the "natural action rate" is measured.

### 2.4. Experimentation (A/B Testing)

The engine supports randomized controlled trials (RCTs) to measure effectiveness.

**Structure:**
*   **Experiment**: Linked to a `nudge_definition`. Defines the test period.
*   **Groups**:
    *   **Control (Holdout)**: The enrollment gets `status='control_group'`. **No `prompt_queue` rows are created**, but the goal is still tracked to compare "Natural Action Rate" vs "Nudged Action Rate".
    *   **Variants**: Users receive specific `message_template` versions (e.g., "Subject Line A" vs "Subject Line B").

**Logic:**
1.  At enrollment, the engine checks if an active Experiment exists for the Nudge.
2.  It randomly assigns the user to a Group based on `allocation_percent`.
3.  The assigned `experiment_group_id` is stamped **once** on the `nudge_enrollment`. Every step of the workflow uses that same group, so a person never sees Variant A's email followed by Variant B's SMS.

### 2.5. Event-Driven Triggers (Clinical Signals)

For scenarios requiring immediate action based on a specific event (e.g., "Doctor just recommended a cardio review"), we use **Event Triggers**.

**The Signal Table: `clinical_recommendation`**
External systems (NLP, EHR Integration) insert rows here.
*   **Fields**: `person_id`, `recommendation_type` (e.g., 'cardio_review'), `source_doctor_name`, `visit_date`.

**Execution:**
1.  **Ingestion**: A new signal arrives in `clinical_recommendation`.
2.  **Matching**: The engine finds active Nudge Definitions where `trigger_type='event_trigger'` and `trigger_event_type` matches the signal.
3.  **Creation**: A `nudge_enrollment` is created immediately, linked to this specific `clinical_recommendation_id`, and Step 1 is queued. Two signals of the same type for the same person produce two separate enrollments (unless one is still active, see dedup in Section 3).

### 2.6. Data Context & Dynamic Variables

To personalize messages with specific data (e.g., "Dr. House recommended..."), we use a **Source + Variable** model. This ensures data consistency (getting the Date and Doctor from the *same* visit).

**1. Data Source (`nudge_data_source`)**
Defines *which* record to pull data from.
*   **Lookup Type** (from `source_lookup_type`): `trigger_event`, `master_person`, `latest_history`, `upcoming_event`.
*   **Example**: Alias `source_event` -> Linked to the triggering `clinical_recommendation`.

**2. Template Variables (`template_variable`)**
Maps a placeholder to a specific column in the Source.
*   `{{doc_name}}` -> Source: `source_event`, Column: `source_doctor_name`
*   `{{visit_date}}` -> Source: `source_event`, Column: `visit_date`

#### 2.6.1. Resolution Algorithm (How the engine builds `context_data`)

At send time (or at queue-insert time if you prefer pre-materialization), the engine resolves placeholders by building a dictionary `context_data`.

**Inputs**
1.  The `prompt_queue` row (contains `enrollment_id`, `person_id`, `nudge_step_id`) and its parent `nudge_enrollment` row (contains `nudge_definition_id`, `clinical_recommendation_id`, `experiment_group_id`)
2.  The `nudge_definition` row (mainly for IDs; trigger logic already happened earlier)
3.  All `nudge_data_source` rows for this `nudge_definition_id`
4.  All `template_variable` rows for this `nudge_definition_id`

**Output**
- A JSON-like map: `context_data[variable.name] = resolved_value`

**Step A — Preload sources**
1.  Load all sources `S = SELECT * FROM nudge_data_source WHERE nudge_definition_id = nudge_enrollment.nudge_definition_id`.
2.  For each source `s in S`, resolve it into exactly **one** record (or `NULL` if not found): `resolved_source_records[s.alias]`.

**Step B — Resolve variables**
1.  Load all variables `V = SELECT * FROM template_variable WHERE nudge_definition_id = nudge_enrollment.nudge_definition_id`.
2.  For each variable `v in V`:
    *   Find its source: `s = resolved_source_records[alias_of(v.nudge_data_source_id)]`.
    *   Read the column: `value = s[v.column_name]`.
    *   Store: `context_data[v.name] = value`.

#### 2.6.2. Source Resolution Rules (per `lookup_type`)

**Case 1: `lookup_type = master_person` (marketing personalization like `{{first_name}}`)**
1.  Use `prompt_queue.person_id`.
2.  Fetch one row:
    *   `SELECT * FROM master_person WHERE id = prompt_queue.person_id`.
3.  Store it as `resolved_source_records[s.alias]`.

**Case 2: `lookup_type = trigger_event` (event-driven personalization like `{{visit_date}}`)**
1.  Require `nudge_enrollment.clinical_recommendation_id`.
2.  Fetch one row:
    *   `SELECT * FROM clinical_recommendation WHERE id = nudge_enrollment.clinical_recommendation_id`.
3.  Store it as `resolved_source_records[s.alias]`.

**Case 3: `lookup_type = latest_history` (pull the most recent past record)**
1.  Use `prompt_queue.person_id`.
2.  Query the table specified by `s.table_name`.
3.  Apply filters from `s.filter_criteria_json`.
4.  Sort by `s.date_column DESC`.
5.  Take `LIMIT 1`.

**Case 4: `lookup_type = upcoming_event` (pull the next future record)**
1.  Use `prompt_queue.person_id`.
2.  Query the table specified by `s.table_name`.
3.  Apply filters from `s.filter_criteria_json`.
4.  Filter by `s.date_column >= NOW()`.
5.  Sort by `s.date_column ASC`.
6.  Take `LIMIT 1`.

#### 2.6.3. Failure Handling & Safety Rules

*   If a source cannot be resolved (no matching record), the engine should:
    *   Set dependent variables to `NULL`, or
    *   Fail the prompt with a clear reason (recommended for `trigger_event`, because the message would be misleading).
*   If a variable refers to a column that does not exist in the resolved source, treat it as a configuration error (fail fast).
*   For `trigger_event`, if `nudge_enrollment.clinical_recommendation_id` is `NULL`, treat it as a configuration / orchestration error.

**Resulting Message:**
"During your visit on **2023-10-10**, **Dr. House** recommended a review."

---

## 3. Enrollment & Prompt Queue Lifecycle

### 3.1. Enrollment (`nudge_enrollment`)

One row per person per run through a nudge. Created by the segment job or by an event trigger.

**Dedup Rule:** A person can have at most one `active` (or `control_group`) enrollment per nudge at a time. This is enforced by a unique partial index on `(person_id, nudge_definition_id) WHERE status IN ('active','control_group')`. The daily segment job can therefore safely re-run: people already enrolled are skipped. After an enrollment ends (`completed`, `exited`, `expired`), the person can be enrolled again.

**Status Transitions:**

*   **ACTIVE**: Steps are being sent, or all steps are sent and the person is still inside the `goal_completion_window`.
*   **CONTROL_GROUP**: Holdout. No messages, goal still tracked until `expires_at`. It then moves to `COMPLETED` or `EXPIRED` like any other enrollment; the holdout membership is preserved via `experiment_group_id` (`is_control = true`).
*   **COMPLETED**: Person entered the Goal Segment after `enrolled_at`. Remaining steps are cancelled.
*   **EXITED**: Stopped for a non-goal reason (campaign deactivated, person left the target segment, manual stop).
*   **EXPIRED**: `expires_at` passed without the goal being met.

An enrollment stays `active` after the last step is sent, so a goal reached a few days after the final reminder is still credited to the nudge.

### 3.2. Messages (`prompt_queue`)

One row per step attempt. Rows are created **one step at a time** (see Section 4), so at most one unsent row exists per enrollment.

**Status Transitions:**

*   **PENDING**: Newly created.
*   **SCHEDULED**: If `scheduled_for` is in the future, it waits here.
*   **QUEUED**: Ready for processing. The "Dispatcher" picks these up.
*   **PROCESSING CHECKS** (in this order):
    1.  **Enrollment Check**: If the parent enrollment is no longer `active` -> `CANCELLED_EXITED`. If `expires_at` (message validity) has passed -> `EXPIRED`.
    2.  **Goal Check**: Evaluate the **Goal Segment** (since `enrolled_at`). If user matches -> `CANCELLED_COMPLETED`, and the enrollment becomes `completed`.
    3.  **Consent Check**: Query `channel_consent`. If false -> `SKIPPED_CONSENT`.
    4.  **Frequency Check**: Query `frequency_cap_policy` vs recent history. If limit exceeded -> `SKIPPED_FREQUENCY`.
*   **SENT**: Successfully dispatched to provider (SendGrid, Twilio, etc.).
*   **FAILED**: Provider returned an error.

The goal check runs first so that completions are recorded even for people who have opted out of the step's channel, and so that someone who already reached the goal is not logged as a frequency skip.

---

## 4. Nudge Workflows (Reminders & Sequences)

Nudges are not just single messages; they are **Sequences of Steps**.

**Structure:**
*   **Nudge Definition**: The container (e.g., "Flu Shot Campaign"). Links to the **Goal Segment**.
*   **Nudge Steps**: Ordered sequence of interactions.
    *   **Step 1** (Order=1, Delay=0): Initial Email.
    *   **Step 2** (Order=2, Delay=3 days): Follow-up SMS.
    *   **Step 3** (Order=3, Delay=2 days): Final Push Notification.

**Execution Logic (Lazy, One Step at a Time):**
1.  **Enrollment**: User enters the Target Segment (or a signal arrives) -> System creates a `nudge_enrollment` and queues **Step 1** (`current_step_order = 1`).
2.  **Step Outcome**: When Step N reaches a final outcome (`SENT`, `FAILED`, `SKIPPED_CONSENT`, `SKIPPED_FREQUENCY`, `EXPIRED`), the System looks up Step N+1.
    *   A skipped step does **not** end the journey: the next step may use a different channel the person *has* consented to.
3.  **Scheduling**: System inserts Step N+1 with `scheduled_for = (Step N sent_at, or the time it was skipped) + Step N+1 delay_interval`, and `expires_at = LEAST(scheduled_for + Step N+1 message_ttl, enrollment expires_at)`, so no message is ever sent after the enrollment has expired.
    *   Because the next row is created only when the previous one finishes, delays are always measured from what actually happened, not from a guess made at enrollment time.
4.  **Goal Check (The Circuit Breaker)**:
    *   Before sending *any* step, the system checks the **Goal Segment** (since `enrolled_at`).
    *   **If User in Goal Segment** -> enrollment `completed`, step `CANCELLED_COMPLETED`. No further steps are created (Success!).
    *   **If NOT in Goal Segment** -> continue with consent / frequency checks and **SEND** the step.
5.  **No More Steps**: After the last step, the enrollment stays `active` until the goal is met (`completed`) or `expires_at` passes (`expired`). A periodic job evaluates goals for active and control-group enrollments that have no pending message.

**Timing Fields:**

| Field | Scope | Meaning |
|:---|:---|:---|
| `nudge_step.delay_interval` | Step | Wait after the previous step finished |
| `nudge_step.message_ttl` | Message | How long a queued step stays sendable (`prompt_queue.expires_at`, capped at the enrollment's `expires_at`) |
| `nudge_definition.goal_completion_window` | Enrollment | How long the person has to reach the goal (`nudge_enrollment.expires_at`) |

```
enrolled_at ─┬─ step1 ──delay──> step2 ──delay──> step3 ───────────────┬─ expires_at
             │   └ttl┘            └ttl┘            └ttl┘                │
             └─────────────────── goal_completion_window ───────────────┘
                     goal evidence counts only inside this span
```

**Configuration Rule:** When a nudge is saved, `SUM(nudge_step.delay_interval) < goal_completion_window`. Otherwise the last steps could never be sent.

---

## 5. Frequency Capping Algorithm

To prevent user fatigue, we implement a "Token Bucket" or "Sliding Window" check.

**Inputs:**
*   `person_id`
*   `channel` (e.g., Email)
*   `category` (e.g., Marketing)

**Check:**
1.  Look up policy: `Max 2 Marketing Emails per 7 days`.
2.  Count `SENT` messages in `prompt_queue` for this person/channel in the last 7 days, joining `nudge_enrollment` -> `nudge_definition` to filter by category.
3.  `Count < Max` ? **ALLOW** : **BLOCK**.

*Note: Service/Transactional messages usually have a NULL policy or high limit, effectively bypassing this.*

---

## TODO

- [x] **Time-bound goal checks**: Goal segments use `"time": {"scope": "since_enrollment"}` and are evaluated per enrollment (`event_date BETWEEN enrolled_at AND expires_at`). See 2.0 and 2.3.
- [ ] **Dynamic SQL safety**: Add an allowlist / metadata registry for the `table` and `field` names used in `segment.definition` and `nudge_data_source`, to prevent SQL injection and to validate value types.
- [x] **Timing fields**: `delay_interval` (step), `message_ttl` (message, moved from `nudge_definition.validity_duration`), `goal_completion_window` (enrollment). See Section 4.
- [x] **Check order**: Goal check runs before consent and frequency. See 3.2.
- [ ] **Late-arriving data (`goal_data_lag`)**: Claims can arrive 30-90 days late with an earlier service date. Keep re-checking expired enrollments for a configurable lag and flip them to `completed` if evidence dated inside the window shows up.
- [ ] **Event-trigger deadline**: Decide whether event-triggered enrollments anchor `expires_at` to `clinical_recommendation.recommended_action_date + goal_completion_window` instead of `enrolled_at + goal_completion_window`.
- [ ] **Segment versioning**: Snapshot `segment.definition` on each enrollment (`segment_version`), so analytics know which rules applied.
- [ ] **Segment analytics**: `segment_snapshot` (segment, node id, computed_at, person count) written by the daily segment job.

### Known Limitations

**Top priorities:** #8 Re-enrollment loop, #1 Person lifecycle, #9 Nudge arbitration, #2 Consent per category, #14 Content versioning.

#### Safety & Compliance
- [ ] **1. Person lifecycle**: `master_person` has no deceased / inactive / merged status. Add `status` / `deceased_date` and a global suppression check before the goal check.
- [ ] **2. Consent per category**: `channel_consent` is per channel only and overwrites history. Make it channel × category, append-only for audit.
- [ ] **3. Missing consent default**: Define what "no consent row" means (opt-in vs opt-out), possibly different for service vs marketing.
- [ ] **4. Sensitive content**: Templates can put a diagnosis in an SMS or a shared-inbox email. Add a sensitivity flag on `nudge_definition` restricting variables / channels.
- [ ] **5. Quiet hours & time zone**: No person time zone or send windows per channel.
- [ ] **6. Minors & proxies**: No guardian / caregiver concept; a child's nudge goes to the child's contact.
- [ ] **7. Data retention**: `context_data` (health data) in `prompt_queue` and `interaction_log` is kept indefinitely. Define a retention policy.

#### Journey Logic
- [ ] **8. Re-enrollment loop**: An `expired` person is re-enrolled by the next segment run, forever. Add a `reenrollment_cooldown` and/or max enrollments per person per nudge.
- [ ] **9. Nudge arbitration**: `default_priority` is unused; frequency caps are first-come-first-served, so a low-priority marketing message can block a critical one. Dispatch by priority; let service / critical bypass marketing caps.
- [ ] **10. Shared goal attribution**: Two nudges with the same goal both record `completed`. Define an attribution rule.
- [ ] **11. Exit criteria**: `exited` ("left the target segment") has no field or job defining when it is checked.
- [ ] **12. Event triggers too narrow**: Only `clinical_recommendation` can trigger; no eligibility filter (minors, deceased); no idempotency key from the source system; `processed boolean` can't record partial failures when several nudges match.
- [ ] **13. Event nudges can't be switched off**: Only `campaign` has `is_active` / dates; `nudge_definition` has no status.

#### Content & Versioning
- [ ] **14. Content versioning**: Editing steps / templates changes in-flight journeys. `prompt_queue` doesn't store `message_template_id` or the rendered content, so what was sent can't be proven.
- [ ] **15. Language / locale**: No locale on `message_template`, no preferred language on the person, no locale-aware date formatting.
- [ ] **16. Contact points**: No device-token table for push / in-app; no verified / bounced / invalid status on email and phone.

#### Segmentation & Data
- [ ] **17. Lab units**: Conditions on `value_numeric` ignore `unit` (HbA1c 7 % ≈ 53 mmol/mol). Require or normalize units.
- [ ] **18. Value sets**: Clinical definitions use code lists with hundreds of codes. Add a `value_set` table and an `in_value_set` operator.
- [ ] **19. Derived fields**: Examples use `age`, but only `date_of_birth` exists. The query builder needs computed fields.
- [ ] **20. No PKs on external tables**: `lab_result`, `vaccination_record`, etc. have no id, so the record that satisfied a goal can't be referenced.
- [ ] **21. Data freshness**: Nothing checks that external data was refreshed before the segment job runs.

#### Operations & Scale
- [ ] **22. Concurrency**: No claim / lock on `prompt_queue` (double sends) and frequency checks can race. Use `FOR UPDATE SKIP LOCKED` and a per-person lock.
- [ ] **23. Delivery tracking**: No provider message id, error reason, retry count, or delivered / bounced receipts; `interaction_log.event_type` is free text.
- [ ] **24. Segment computation cost**: Daily full recompute with nested segments inlined won't scale. Plan for materialized / incremental segments.
- [ ] **25. Job logs & alerting**: No record of "segment job ran, enrolled N, failed M".

#### Experimentation
- [ ] **26. Assignment consistency**: Random per enrollment, so the same person can be control once and variant the next time. Use hash-based assignment on `person_id`.
- [ ] **27. Allocation validation**: `allocation_percent` isn't checked to sum to 100.
- [ ] **28. Global holdout**: No way to measure the whole program's effect, only per nudge.
