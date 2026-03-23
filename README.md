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
    *   **Action**: The system checks if the goal is met using `action_criteria` against external tables (e.g., "Did they book the appointment?").

---

## 2. Dynamic SQL Generation (Segmentation)

The `segment_criteria` table stores the rules. The `segment` table now stores a `logic_expression` (e.g., `(A OR B) AND C`) to combine them.

**Table Structure Example:**
| segment_id | label | field_name | operator | value |
|:---|:---|:---|:---|:---|
| 101 | A | age | > | 45 |
| 101 | B | has_asthma | = | true |
| 101 | C | last_screening | < | NOW() - 2 months |

**Logic Expression:** `(A AND B) OR C`

### 2.1. Composition & Negation (Advanced Logic)

The engine supports nesting segments and exclusion using `NOT`.

**Composition (Nesting)**
Criteria can now reference another Segment (`child_segment_id`).
*   *Example*: Define "Vulnerable" segment. Then define "Vulnerable AND Diabetes".

**Negation (Exclusion)**
To exclude users (e.g., "No Diabetes Diagnosis"), define the positive criteria and negate it in the expression.
*   **Criteria A**: Has Diabetes Diagnosis.
*   **Expression**: `NOT A`

**Generated Query Logic:**
1.  Start with `SELECT id FROM master_person WHERE ...`
2.  Iterate through criteria for `segment_id = 101`.
3.  Construct the query:
    ```sql
    SELECT id, email, first_name 
    FROM master_person 
    WHERE 
      (
        (age > 45 AND has_asthma = true) -- (A AND B)
        OR 
        (last_checkup_date < CURRENT_DATE - INTERVAL '2 months') -- C
      )
    ```
4.  **Result**: A list of `person_id`s to insert into `prompt_queue`.

### 2.1. Multi-Table & Campaign Intrication

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

**Campaign Intrication (Ad-Hoc Filters)**
Marketers can refine a base Segment for a specific campaign without creating a new permanent Segment.
*   **Base Segment**: "Seniors" (Age > 65)
*   **Campaign Criteria**: "Has Asthma" (Ad-hoc filter)
*   **Resulting Target**: `(Age > 65) AND (Has Asthma)`

### 2.2. Action Criteria (Defining Success)

Just like Segments, "Actions" (Goals) are defined by dynamic rules against the External Data.

**Example: Goal "Diabetic Screening Complete"**

**Table Structure Example:**
| action_id | label | table_name | field_name | operator | value |
|:---|:---|:---|:---|:---|:---|
| 205 | A | lab_result | loinc_code | = | 4548-4 |
| 205 | B | lab_result | value_numeric | < | 7.0 |

**Logic Expression:** `A AND B`

If a record is found matching these rules, the Action is marked as **Completed**, and any pending reminders are cancelled.

### 2.3. Experimentation (A/B Testing)

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

## 4. Handling Reminders & Expiration

Nudges are not just fire-and-forget; they have goals.

*   **Nudge Goal**: Linked to `action_definition` (e.g., "Book Appointment").
*   **Action Criteria**: Defines what "Success" looks like dynamically (e.g., `table=appointment_history`, `status='completed'`).
*   **Reminder Logic**:
    *   A nightly job checks `prompt_queue` for sent nudges that have a `reminder_policy`.
    *   It executes the **Action Criteria** query for the user.
    *   **If Result is EMPTY**: It creates a new `prompt_queue` item (the reminder).
    *   **If Result EXISTS**: It does nothing (silence is golden).

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
