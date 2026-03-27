# Prompt Engine Architecture

This document outlines the architecture and logic flow for the Health Services Prompt Engine. The system is designed to separate **Strategy** (Marketing/Clinical definitions) from **Execution** (Delivery, Consent, Frequency Capping).

## 1. High-Level Logic Flow

1.  **Definition**: Marketers/Admins define *Segments* (Who) and *Nudge Definitions* (What).
2.  **Generation**:
    *   **Campaigns**: A scheduled job runs, translates Segment Criteria into SQL, finds matching users, and inserts them into the `prompt_queue`.
    *   **Automated Algorithms**: External clinical systems trigger events that insert rows directly into `prompt_queue`.
3.  **Filtration (The Gatekeeper)**: Before sending, the system checks **Consent** (Opt-in) and **Frequency Caps** (Anti-spam).
4.  **Delivery**: The message is sent via the appropriate channel provider.
5.  **Feedback Loop**:
    *   **Interaction**: User opens/clicks (logged in `interaction_log`).
    *   **Action**: The system checks if the goal is met using the `goal_segment_id`. If met, subsequent steps in the workflow are cancelled.

---

## 2. Dynamic SQL Generation (Segmentation)

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
4.  **Result**: A list of `person_id`s to insert into `prompt_queue`.

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
*   **Completion Window**: Defined on `nudge_definition` (e.g., "User has 14 days to complete this").

If a record is found matching these rules, the Nudge is marked as **Completed**, and any pending reminders are cancelled.

### 2.4. Experimentation (A/B Testing)

The engine supports randomized controlled trials (RCTs) to measure effectiveness.

**Structure:**
*   **Experiment**: Linked to a `nudge_definition`. Defines the test period.
*   **Groups**:
    *   **Control (Holdout)**: Users assigned here get `status='control_group'`. **No message is sent**, but they are tracked to compare "Natural Action Rate" vs "Nudged Action Rate".
    *   **Variants**: Users receive specific `message_template` versions (e.g., "Subject Line A" vs "Subject Line B").

**Logic:**
1.  During generation, the engine checks if an active Experiment exists for the Nudge.
2.  It randomly assigns the user to a Group based on `allocation_percent`.
3.  The assigned `experiment_group_id` is stamped on the `prompt_queue` record for analytics.

---

## 3. The Prompt Queue & Lifecycle

The `prompt_queue` is the central operational table. Every notification request lives here.

**Status Transitions:**

*   **PENDING**: Newly created.
*   **SCHEDULED**: If `scheduled_for` is in the future, it waits here.
*   **QUEUED**: Ready for processing. The "Dispatcher" picks these up.
*   **PROCESSING CHECKS**:
    1.  **Consent Check**: Query `channel_consent`. If false -> `SKIPPED_CONSENT`.
    2.  **Frequency Check**: Query `frequency_cap_policy` vs recent history. If limit exceeded -> `SKIPPED_FREQUENCY`.
    3.  **Action Check**: (For reminders) Evaluate `action_criteria` against external tables. If returns true -> `CANCELLED_COMPLETED`.
*   **SENT**: Successfully dispatched to provider (SendGrid, Twilio, etc.).
*   **FAILED**: Provider returned an error.

---

## 4. Nudge Workflows (Reminders & Sequences)

Nudges are not just single messages; they are **Sequences of Steps**.

**Structure:**
*   **Nudge Definition**: The container (e.g., "Flu Shot Campaign"). Links to the **Goal Segment**.
*   **Nudge Steps**: Ordered sequence of interactions.
    *   **Step 1** (Order=1, Delay=0): Initial Email.
    *   **Step 2** (Order=2, Delay=3 days): Follow-up SMS.
    *   **Step 3** (Order=3, Delay=2 days): Final Push Notification.

**Execution Logic:**
1.  **Trigger**: User enters the Target Segment -> System schedules **Step 1**.
2.  **Step Completion**: When Step 1 is sent, the System checks for Step 2.
3.  **Scheduling**: System calculates `Scheduled Time = NOW + Step 2 Delay`.
4.  **Goal Check (The Circuit Breaker)**:
    *   Before sending *any* step, the system checks the **Goal Segment**.
    *   **If User in Goal Segment** -> **CANCEL** entire remaining workflow (Success!).
    *   **If NOT in Goal Segment** -> **SEND** the step.

---

## 5. Frequency Capping Algorithm

To prevent user fatigue, we implement a "Token Bucket" or "Sliding Window" check.

**Inputs:**
*   `person_id`
*   `channel` (e.g., Email)
*   `category` (e.g., Marketing)

**Check:**
1.  Look up policy: `Max 2 Marketing Emails per 7 days`.
2.  Count sent messages in `prompt_queue` for this person/channel/category in the last 7 days.
3.  `Count < Max` ? **ALLOW** : **BLOCK**.

*Note: Service/Transactional messages usually have a NULL policy or high limit, effectively bypassing this.*
