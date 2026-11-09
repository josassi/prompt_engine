# Prompt Engine Architecture

This document outlines the architecture and logic flow for the Health Services Prompt Engine. The system is designed to separate **Strategy** (Marketing/Clinical definitions) from **Execution** (Delivery, Consent, Frequency Capping).

## 1. High-Level Logic Flow

1.  **Definition**: Marketers/Admins define *Segments* (Who) and *Nudge Definitions* (What).
2.  **Generation & Triggers**:
    *   **Segment-Based (Pull)**: A scheduled job runs, translates Segment Criteria into SQL, finds matching users (e.g., "All Diabetics"), and creates a `nudge_enrollment` for each person not already enrolled.
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

The `segment_criteria` table stores the rules. The `segment` table now stores a `logic_expression` (e.g., `(A OR B) AND C`) to combine them.

**Table Structure Example:**
| segment_id | label | field_name | operator | value | child_segment_id |
|:---|:---|:---|:---|:---|:---|
| 101 | A | age | > | 45 | null |
| 101 | B | has_asthma | = | true | null |
| 101 | C | null | null | null | 50 (Seniors) |

**Logic Expression:** `(A AND B) OR NOT C`

**Generated Query Logic:**
1.  Start with `SELECT id FROM master_person WHERE ...`
2.  Iterate through criteria for `segment_id = 101`.
3.  Construct the query (conceptually):
    ```sql
    SELECT id, email, first_name 
    FROM master_person 
    WHERE 
      (
        (age > 45 AND has_asthma = true) -- (A AND B)
        OR 
        NOT (id IN (SELECT id FROM segment_50_view)) -- NOT C (Exclusion of nested segment)
      )
    ```
4.  **Result**: A list of `person_id`s. Each one that has no `active` / `control_group` enrollment for this nudge gets a new `nudge_enrollment`.

### 2.1. Multi-Table & Unified Segmentation

The system supports advanced targeting beyond simple `master_person` attributes.

**Multi-Table Support (`table_name`)**
Criteria can now specify a `table_name`. The Query Builder dynamically joins these tables to `master_person`.
*   *Example*: `table_name='lab_result'`, `field='loinc_code'`, `value='4548-4'` (HbA1c).

**External Data Integration**
The engine connects to read-only external tables for rich segmentation:
*   **Clinical**: `clinical_diagnosis` (ICD-10), `lab_result` (LOINC), `medication_prescription` (RxNorm/ATC), `vaccination_record` (CVX).
*   **Claims**: `insurance_claim` and `claim_line_item` (CPT/HCPCS) for billing data.
*   **Encounters**: `hospital_encounter` (Admissions, Surgeries) and `appointment_history`.
*   **Risk & Social**: `risk_score` (ML models) and `social_determinants`.

**Unified Segmentation (Ad-Hoc Filters)**
Instead of separate "Campaign Criteria", the system uses a **Unified Segmentation Model**.
*   Every Campaign points to a single `Segment`.
*   **Library Segments**: Reusable Personas (e.g., "Seniors").
*   **Ad-Hoc Segments**: If a campaign needs specific filtering, a new `Segment` is created with `type='campaign_ad_hoc'`.
    *   This Ad-Hoc segment is defined as: `(Base Library Segment) AND (Ad-Hoc Rules)`.
    *   This keeps the architecture clean: The Query Builder only ever resolves a Segment.

### 2.2. Segment Composition & Exclusion

The engine supports complex set operations including **Nesting** (Union/Intersection of Segments) and **Exclusion** (Negation).

**Segment Composition (Nesting)**
A criterion can now refer to another existing Segment via `child_segment_id` instead of a raw SQL condition.
*   *Example*: Define a "Diabetes" segment. Then define a "High Risk Diabetes" segment as: `("Diabetes" Segment) AND (HbA1c > 9)`.
*   This promotes reusability and consistency.

**Exclusion (The 'NOT' Operator)**
The `logic_expression` supports the `NOT` operator for exclusion.
*   *Example*: Target users who have Asthma but are NOT in the "Seniors" segment.
*   *Logic*: `A AND (NOT B)`
    *   Criteria A: `has_asthma = true`
    *   Criteria B: `child_segment_id = [ID of Seniors Segment]`

### 2.3. Goal Segments (Defining Success)

Just like "Personas", "Goals" are defined using the standard **Segment** structure (with `type='goal'`).
*   **Unified Logic**: The system checks: *"Is the user a member of the Goal Segment?"*
    *   If **YES** -> The Nudge is successful. Stop Reminders.
    *   If **NO** -> Send Nudge / Continue Reminders.

**Example: Goal "Diabetic Screening Complete"**
This is just a Segment defined as: `(Table: lab_result, Code: 4548-4)` AND `(Value < 7.0)`.

**Structure:**
*   **Nudge Definition**: Points to `goal_segment_id`.
*   **Completion Window**: `goal_completion_window` on `nudge_definition` (e.g., "User has 14 days to complete this"). At enrollment, `nudge_enrollment.expires_at = enrolled_at + goal_completion_window`.

**Time-Bound Evaluation:**
The goal is checked **relative to the enrollment**: only evidence dated after `nudge_enrollment.enrolled_at` counts. Otherwise an HbA1c test from three years ago would mark a brand-new enrollment as complete. (The query builder needs to know each criteria table's date column, e.g. `lab_result.test_date`, to apply this filter.)

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
3.  **Scheduling**: System inserts Step N+1 with `scheduled_for = (Step N sent_at, or the time it was skipped) + Step N+1 delay_interval`, and `expires_at = scheduled_for + validity_duration`.
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
| `nudge_definition.validity_duration` | Message | How long a queued step stays sendable (`prompt_queue.expires_at`) |
| `nudge_definition.goal_completion_window` | Enrollment | How long the person has to reach the goal (`nudge_enrollment.expires_at`) |

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

- [ ] **Time-bound goal checks**: Only count goal evidence after `enrolled_at`, within `goal_completion_window`. Otherwise an old HbA1c test counts as success.
- [ ] **Dynamic SQL safety**: Add an allowlist / metadata registry for `table_name` and `field_name` (`segment_criteria`, `nudge_data_source`) to prevent SQL injection and to cast `value` (stored as varchar) to the right type.
- [ ] **Timing fields**: Give `validity_duration`, `goal_completion_window` and `nudge_step.delay_interval` clear, separate meanings.
- [ ] **Check order**: Run the goal check before consent and frequency, so completions are recorded for opted-out users and nobody who already reached the goal is logged as a frequency skip.
