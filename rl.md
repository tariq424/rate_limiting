### TC_MDT_Receiver_Metrics_Logging_Consistency

**Purpose:**  
Validate that `mdtreceiver` accurately and consistently updates all expected metrics and logging tables across both accepted and rejected submissions, regardless of status.

**Setup:**
- Channel: `test.multi_1.dir.metrics.logging`
- User CN: `test_user_metrics`
- Payloads: Mix of successful and rate-limited submissions across types (requests, bytes, files)

**Validation Scope (Not Configuration-Driven):**
This is a **meta-level test case**. The goal is not to assert rate-limiting decisions but to ensure the internal tables are populated correctly **in every outcome path**:
- Acceptance (HTTP 200)
- Rejection (HTTP 429, 503)
- Warn Mode Allowed
- Bouncer Fail-Open

**Validation Targets:**

| Table                          | Triggered When                                | Validation Check                                      |
|-------------------------------|------------------------------------------------|-------------------------------------------------------|
| `mdtreceiver_rate_limited`    | On 429 or 503 rejection                        | Entry includes timestamp, scope, user/channel details |
| `mdtreceiver_bouncer_errors`  | On Bouncer unavailability                      | Entry includes gRPC error and service context         |
| `mdtreceiver_channel_history` | On *every* submission                          | Records status, timestamp, status detail              |
| `alerts_triggered`            | When alertable thresholds are breached         | Entry includes alert type and context string          |

**Mock Configuration / Fixtures:**
A test framework fixture iterates across outcome scenarios:
1. Simulates each edge case (429, warn mode, bouncer fail-open, allow)
2. Forces `mdtreceiver` to run through all logging conditions
3. Captures in-memory “tables” after submission

**Expected Result:**
- All applicable tables are updated as expected for each test outcome
- No gaps in coverage regardless of request status
- Alerts fire only when required, not erroneously
- Tests assert structure and content of each table using `assert_table_contains`
