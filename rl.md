# MDT Receiver Rate Limiting Tests

This document outlines and validates rate limiting behavior in `mdtreceiver` under a variety of configurations and edge cases. It is structured as a suite of reproducible test cases, each targeting a specific scenario including wildcard resolution, pre/post upload evaluation, system and user-level boundaries, warn mode behavior, and failure modes.

---

## Table of Contents

- [TC_MDT_Receiver_Config_MostSpecificMatch_General](#tc_mdt_receiver_config_mostspecificmatch_general)
- [TC_MDT_Receiver_Config_MostSpecificMatch_Specific](#tc_mdt_receiver_config_mostspecificmatch_specific)
- [TC_MDT_Receiver_Config_AllMatches](#tc_mdt_receiver_config_allmatches)
- [TC_MDT_Receiver_PreUpload_Rejected_429](#tc_mdt_receiver_preupload_rejected_429)
- [TC_MDT_Receiver_PostUpload_Files_Rejected_429](#tc_mdt_receiver_postupload_files_rejected_429)
- [TC_MDT_Receiver_SystemLimit_Exceeded_503](#tc_mdt_receiver_systemlimit_exceeded_503)
- [TC_MDT_Receiver_GlobalWarnMode_True](#tc_mdt_receiver_globalwarnmode_true)
- [TC_MDT_Receiver_SpecificWarnMode_Override](#tc_mdt_receiver_specificwarnmode_override)
- [TC_MDT_Receiver_BouncerError_FailOpen](#tc_mdt_receiver_bouncererror_failopen)
- [TC_MDT_Receiver_Metrics_Logging_Consistency](#tc_mdt_receiver_metrics_logging_consistency)

---

### TC_MDT_Receiver_Config_MostSpecificMatch_General

**Purpose:**  
Verify `mdtreceiver` applies the correct wildcard rule for a general channel pattern when multiple `granular_config.name_configs` entries exist.

**Setup:**
- Channel Name: `test.multi_1.file.Mapper.general.channel`
- User CN: `test_user_general`
- Payload: 1 request, 100KB, 1 file

**Relevant `mdt.config.groups` Excerpt:**

```json
"granular_config": {
  "name_configs": {
    "test.multi_1.*": {
      "rate_limit_requests": 10,
      "rate_limit_requests_burst": 10
    },
    "test.multi_1.file.Mapper.*": {
      "rate_limit_requests": 2,
      "rate_limit_requests_burst": 2
    }
  }
}
```

**Steps:**
1. Submit a request as `test_user_general` to the general channel
2. Verify that `mdtreceiver` selects the more specific wildcard: `test.multi_1.file.Mapper.*`
3. Construct `RateLimitRequest` accordingly

**Expected Result:**
- HTTP 200 OK returned
- No entry in `mdtreceiver_rate_limited`
- No alert triggered

---

### TC_MDT_Receiver_Config_MostSpecificMatch_Specific

**Purpose:**  
Validate that `mdtreceiver` applies a channel-specific config (`test.multi_1.file.Mapper.specific.channel`) over broader wildcard matches.

**Setup:**
- Channel: `test.multi_1.file.Mapper.specific.channel`
- User CN: `test_user_specific`
- Payload: 1 request, 100KB, 1 file

**Relevant Config:**

```json
"granular_config": {
  "name_configs": {
    "test.multi_1.file.Mapper.specific.channel": {
      "rate_limit_requests": 1,
      "rate_limit_requests_burst": 1
    },
    "test.multi_1.*": {
      "rate_limit_requests": 10,
      "rate_limit_requests_burst": 10
    },
    "test.multi_1.file.Mapper.*": {
      "rate_limit_requests": 2,
      "rate_limit_requests_burst": 2
    }
  }
}
```

**Steps:**
1. Submit a request to `test.multi_1.file.Mapper.specific.channel`
2. Verify `mdtreceiver` selects exact match rule over wildcard entries

**Expected Result:**
- Request evaluated using strictest limit (1/min)
- Correctly enforced if exceeded
- Logs show exact config selection

---

### TC_MDT_Receiver_Config_AllMatches

**Purpose:**  
Ensure that `mdtreceiver` stacks or evaluates all applicable rate limits across scopes, including user, channel, and system-level selectors.

**Setup:**
- Channel: `test.multi_1.dir.allmatches.channel`
- User CN: `test_user_allmatches`
- Payload: 1 request, 50KB, 1 file

**Relevant Configs:**

```json
"user_rate_limit_requests": 5,
"user_rate_limit_requests_burst": 5,
"granular_config": {
  "name_configs": {
    "test.multi_1.dir.allmatches.channel": {
      "rate_limit_requests": 2,
      "rate_limit_requests_burst": 2
    }
  }
}
```

**Steps:**
1. Submit a request triggering multiple scopes
2. `RateLimitRequest` should include:
   - User scope
   - Channel scope
3. Both scopes must pass evaluation

**Expected Result:**
- HTTP 200 if neither limit is exceeded
- HTTP 429 if either scope is exhausted
- Request logs detail multiple evaluations

---

### TC_MDT_Receiver_PreUpload_Rejected_429

**Purpose:**  
Verify that if a request exceeds the rate limit during **pre-upload evaluation**, `mdtreceiver` rejects it with HTTP 429 before accepting payloads.

**Setup:**
- Channel: `test.multi_2.dir.preupload.reject`
- User CN: `test_user_preupload`
- Payload: Empty (validation-only request)

**Relevant Config:**

```json
"user_rate_limit_requests": 1,
"user_rate_limit_requests_burst": 1
```

**Mock Bouncer Config:**

```json
{
  "should_exceed": true,
  "mock_retry_after_seconds": 15,
  "mock_remaining_tokens": [0]
}
```

**Steps:**
1. Send multiple requests quickly as `test_user_preupload`
2. On second request, Bouncer returns `exceeded: true`
3. `mdtreceiver` halts before upload processing

**Expected Result:**
- HTTP 429 Too Many Requests
- `Retry-After: 15` present
- Entry logged in `mdtreceiver_rate_limited`

---

### TC_MDT_Receiver_PostUpload_Files_Rejected_429

**Purpose:**  
Ensure that `mdtreceiver` correctly enforces file count limits during **post-upload evaluation**, rejecting submissions with too many files.

**Setup:**
- Channel: `test.multi_2.dir.TC_PostUpload_Files_Rejected`
- User CN: `test_user_postupload`
- Payload: 1 request, 10 files

**Relevant Config:**

```json
"user_rate_limit_files": 5,
"user_rate_limit_files_burst": 5
```

**Mock Bouncer Config:**

```json
{
  "should_exceed": true,
  "mock_retry_after_seconds": 20,
  "mock_remaining_tokens": [3, 2, 0]
}
```

**Steps:**
1. Submit upload request with 10 files after prior usage of 3
2. Pre-upload passes; post-upload triggers a `RateLimitRequest` for `files`
3. Bouncer responds with `exceeded: true` for user/files scope
4. `mdtreceiver` halts request and returns 429

**Constructed `RateLimitRequest` (Post-upload):**

```json
{
  "app": "mdt",
  "updates": [
    {
      "selectors": [
        { "key": "user", "value": "test_user_postupload" },
        { "key": "type", "value": "files" }
      ],
      "taking": 10,
      "limit": { "rate": 5, "time_unit": "MINUTE", "burst": 5 }
    }
  ],
  "include_headers": true
}
```

**Returned `RateLimitResponse`:**

```json
{
  "exceeded": true,
  "retry_after": { "seconds": 20 },
  "statuses": [
    {
      "exceeded": true,
      "limit": { "rate": 5, "time_unit": "MINUTE", "burst": 5 },
      "remaining": 0
    }
  ],
  "headers": {
    "Retry-After": "20"
  }
}
```

**Expected Result:**
- HTTP 429 Too Many Requests
- Header: `Retry-After: 20`
- Logging entry created in `mdtreceiver_rate_limited`
- Channel marked with `RATE LIMITED` status detail

---

### TC_MDT_Receiver_SystemLimit_Exceeded_503

**Purpose:**  
Confirm that `mdtreceiver` properly detects and reacts to a system-wide resource violation—returning HTTP 503 with logging and alerting.

**Setup:**
- Channel: `test.multi_2.dir.TC_SystemLimit_Exceeded_503`
- User CN: `test_user_system`
- Payload: 1 request, **500GB**, 1 file

**Relevant `mdt.config.groups` Excerpt:**

```json
"aggregate_limits": {
  "system_limits": {
    "freeflow": {
      "rate_limit_bytes": 1000000000,
      "rate_limit_bytes_burst": 1000000000
    }
  }
}
```

**Mock Bouncer Configuration:**

```json
{
  "should_exceed": true,
  "mock_retry_after_seconds": 60,
  "mock_remaining_tokens": [999, 10, 50, 0]
}
```

**Steps:**
1. Simulate a large upload from `test_user_system` to the system
2. Pre-upload request passes; post-upload evaluation is triggered for byte usage
3. `RateLimitRequest` includes a system `bytes` scope entry
4. Bouncer responds with `exceeded: true` on the **system-level bytes** scope
5. `mdtreceiver` issues an HTTP 503 and logs the incident

**Constructed `RateLimitRequest` (Post-upload):**

```json
{
  "app": "mdt",
  "updates": [
    {
      "selectors": [
        { "key": "system", "value": "freeflow" },
        { "key": "type", "value": "bytes" }
      ],
      "taking": 500000000000,
      "limit": { "rate": 1000000000, "time_unit": "MINUTE", "burst": 1000000000 }
    }
  ],
  "include_headers": true
}
```

**Returned `RateLimitResponse`:**

```json
{
  "exceeded": true,
  "retry_after": { "seconds": 60 },
  "statuses": [
    {
      "exceeded": true,
      "limit": { "rate": 1000000000, "time_unit": "MINUTE", "burst": 1000000000 },
      "remaining": 0
    }
  ],
  "headers": {
    "Retry-After": "60"
  }
}
```

**Expected Result:**
- HTTP 503 Service Temporarily Unavailable
- `Retry-After: 60` header returned
- Alert generated: `"System-wide rate limit exceeded"`
- Entry added to `mdtreceiver_rate_limited` and `alerts_triggered`

---

### TC_MDT_Receiver_GlobalWarnMode_True

**Purpose:**  
Confirm that when `rate_limit_warn_mode: true` is globally enabled, `mdtreceiver` allows the request even when a limit is exceeded—but logs a warning for traceability.

**Setup:**
- Channel: `test.multi_1.file.TC_GlobalWarnMode_True`
- User CN: `test_user_warn`
- Payload: 1 request, 50KB, 1 file

**Relevant `mdt.config.groups` Excerpt:**

```json
"rate_limit_warn_mode": true,
"user_rate_limit_requests": 1,
"user_rate_limit_requests_burst": 1
```

**Mock Bouncer Configuration:**

```json
{
  "should_exceed": true,
  "mock_remaining_tokens": [0],
  "mock_retry_after_seconds": 10
}
```

**Steps:**
1. Submit request as `test_user_warn` with a limit of 1 request/min already exhausted
2. `mdtreceiver` builds a `RateLimitRequest` including user scope
3. Mock Bouncer returns `exceeded: true`, but global warn mode is active
4. Instead of blocking, `mdtreceiver` allows the request and logs a warning
5. No retry-after header is enforced

**Constructed `RateLimitRequest`:**

```json
{
  "app": "mdt",
  "updates": [
    {
      "selectors": [
        { "key": "user", "value": "test_user_warn" },
        { "key": "type", "value": "requests" }
      ],
      "taking": 1,
      "limit": { "rate": 1, "time_unit": "MINUTE", "burst": 1 }
    }
  ],
  "include_headers": true
}
```

**Returned `RateLimitResponse`:**

```json
{
  "exceeded": true,
  "retry_after": { "seconds": 10 },
  "statuses": [
    {
      "exceeded": true,
      "limit": { "rate": 1, "time_unit": "MINUTE", "burst": 1 },
      "remaining": 0
    }
  ]
}
```

**Expected Result:**
- HTTP 200 OK
- `mdtreceiver_rate_limited` logs the event with `"warn_mode": true`
- No alert triggered
- No `Retry-After` header returned

---

### TC_MDT_Receiver_SpecificWarnMode_Override

**Purpose:**  
Verify that a specific channel-level `warn_mode: false` config overrides the global `warn_mode: true` setting, resulting in hard enforcement and request denial.

**Setup:**
- Channel: `test.multi_1.file.Mapper.critical.channel`
- User CN: `test_user_critical`
- Payload: 1 request, 25KB, 1 file

**Relevant `mdt.config.groups` Excerpt:**

```json
"rate_limit_warn_mode": true,
"granular_config": {
  "name_configs": {
    "test.multi_1.file.Mapper.critical.channel": {
      "rate_limit_requests": 1,
      "rate_limit_requests_burst": 1,
      "rate_limit_warn_mode": false
    }
  }
}
```

**Mock Bouncer Configuration:**

```json
{
  "should_exceed": true,
  "mock_retry_after_seconds": 5,
  "mock_remaining_tokens": [0]
}
```

**Steps:**
1. Simulate request submission to the critical channel
2. `mdtreceiver` detects both global and channel-specific warn mode settings
3. Chooses the stricter local config (`warn_mode: false`) over global default
4. Builds `RateLimitRequest` and sends to mock Bouncer
5. Receives `exceeded: true` for the channel scope
6. Returns HTTP 429 with proper headers and logs the rejection

**Constructed `RateLimitRequest`:**

```json
{
  "app": "mdt",
  "updates": [
    {
      "selectors": [
        { "key": "channel", "value": "test.multi_1.file.Mapper.critical.channel" },
        { "key": "type", "value": "requests" }
      ],
      "taking": 1,
      "limit": { "rate": 1, "time_unit": "MINUTE", "burst": 1 }
    }
  ],
  "include_headers": true
}
```

**Returned `RateLimitResponse`:**

```json
{
  "exceeded": true,
  "retry_after": { "seconds": 5 },
  "statuses": [
    {
      "exceeded": true,
      "limit": { "rate": 1, "time_unit": "MINUTE", "burst": 1 },
      "remaining": 0
    }
  ]
}
```

**Expected Result:**
- HTTP 429 Too Many Requests
- `Retry-After: 5` header returned
- `warn_mode: false` honored from channel-specific config
- Logged in `mdtreceiver_rate_limited`
- Entry in channel history with `status_detail: RATE LIMITED`

---

### TC_MDT_Receiver_BouncerError_FailOpen

**Purpose:**  
Ensure that `mdtreceiver` fails open when the Bouncer service is unavailable—allowing the request to proceed but logging the incident for observability and alerting.

**Setup:**
- Channel: `test.multi_1.file.TC_BouncerError_FailOpen`
- User CN: `test_user_error`
- Payload: 1 request, 20KB, 1 file

**Relevant `mdt.config.groups` Excerpt:**

```json
"rate_limit_enabled": true
```

**Mock Bouncer Configuration:**

```json
{
  "simulate_unavailability": true,
  "grpc_error_code": "UNAVAILABLE"
}
```

**Steps:**
1. Simulate a submission from `test_user_error`
2. `mdtreceiver` attempts to contact the Bouncer for rate-limiting decision
3. Bouncer returns a gRPC `UNAVAILABLE` error (e.g., timeout or backend failure)
4. `mdtreceiver` proceeds with request processing (fail-open behavior)
5. Logs the failure in `mdtreceiver_bouncer_errors` and triggers an alert if error rate threshold exceeded

**Constructed `RateLimitRequest`:**

```json
{
  "app": "mdt",
  "updates": [
    {
      "selectors": [
        { "key": "user", "value": "test_user_error" },
        { "key": "type", "value": "requests" }
      ],
      "taking": 1,
      "limit": { "rate": 5, "time_unit": "MINUTE", "burst": 10 }
    }
  ],
  "include_headers": true
}
```

**Returned `RateLimitResponse`:**

```json
// Not applicable – gRPC error returned:
{
  "error": "UNAVAILABLE",
  "message": "Failed to connect to Bouncer service"
}
```

**Expected Result:**
- HTTP 200 OK returned to the client
- Entry written to `mdtreceiver_bouncer_errors` table with timestamp and error details
- Alert potentially triggered if error rate exceeds acceptable threshold

---

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

---



