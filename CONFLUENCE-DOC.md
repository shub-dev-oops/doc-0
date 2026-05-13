# Team Forge - SRE Digest Overview

## Intelligent Alert Processing & Executive Reporting

*Transforming noise into actionable intelligence*

**Teams is the intake; AWS is the engine.** The SRE Digest turns Teams channel alerts into a clear, IST-timed operational brief for faster decisions. 🚀

---

## Quick Overview (TL;DR)

### What it does

```
🧹 Reduces noise: Groups and de-duplicates alerts so SREs & leaders see what matters.
🚨 Surfaces risk: Clear severity buckets (Critical/Warning/Unknown) with occurrences.
🗃 Creates traceability: Raw messages are retained in S3 for audit/backfill.
⏱ Speeds Handoff: Predictable, IST-aligned summary posted to Teams.
   • Every ~90 minutes for operational monitoring
   • Every 8 hours for SRE shift handoff / leadership summary
```

### How the flow works (3 Lambda functions)

```
📢 Vendors → Teams → API Gateway
    ↓
🔧 Lambda 1: sqs-ingest-lambda.py        (sre-digest-sqs-ingest)
   Parses Teams body, enriches metadata, sends to SQS
    ↓
📨 Amazon SQS: sre-alerts-queue
    ↓
🔧 Lambda 2: sqs-s3-writer.py           (sre-digest-sqs-s3-writer)
   Batches messages, writes JSONL to date-partitioned S3
    ↓
🗂 Amazon S3: sre-digest-alerts-data
    ↓
🔧 Lambda 3: s3_digest.py               (sre-digest-s3-process)
   Reads S3, calls Bedrock (Claude) to group/summarize,
   renders Markdown, posts back to Teams
    ↓
📱 Teams Channel: SRE-Govm Digest
```

### AWS Resources Quick Links

| Resource | Type | Name / Link |
|----------|------|-------------|
| Ingest Lambda | Lambda | `sre-digest-sqs-ingest` |
| S3 Writer Lambda | Lambda | `sre-digest-sqs-s3-writer` |
| Digest Lambda | Lambda | `sre-digest-s3-process` |
| Alerts Queue | SQS | `sre-alerts-queue` |
| Alert Data Bucket | S3 | `sre-digest-alerts-data` |
| State & Logs Bucket | S3 | `sre-digest-alerts-data` (shared) |
| API Gateway | API GW | `sre-graph-webhook` |
| EventBridge Rules | Scheduler | `sre-digest-regular`, `sre-digest-8h` |

---

## Detailed Architecture

### End-to-End Data Flow

```
Teams Workflow (Incoming Webhook)
    ↓ HTTP POST
Amazon API Gateway (sre-graph-webhook)
    ↓ Invoke
AWS Lambda — Ingest (sqs-ingest-lambda.py / sre-digest-sqs-ingest)
    • Parses request body (text/html/fromDisplay/timestamps)
    • Enriches: detects source, severity, status, environment, product
    • Generates stable incidentKey
    • Sends enriched JSON to SQS with MessageAttributes
    ↓ SQS SendMessage
Amazon SQS (sre-alerts-queue)
    • Buffers spikes and decouples ingest from storage
    • FIFO support for ordered processing per product-environment
    ↓ SQS Trigger (batch)
AWS Lambda — S3 Writer (sqs-s3-writer.py / sre-digest-sqs-s3-writer)
    • Receives batch of SQS records
    • Groups by event date (year/month/day partitioning)
    • Batches ~100 records per file
    • Writes JSONL to S3 with metadata
    • Returns partial batch failures for retry
    ↓ S3 PutObject
Amazon S3 (sre-digest-alerts-data)
    ├── alerts/year=YYYY/month=MM/day=DD/alerts-{ts}-{uuid}-{batch}.jsonl[.gz]
    ├── state/sre-digest-state.json
    ├── state/sre-digest-state-8h.json
    └── bedrock/digests/{session}/request/response logs (optional)
    ↓ EventBridge Schedule (every ~90 min / every 8 hrs)
AWS Lambda — Digest (s3_digest.py / sre-digest-s3-process)
    • Reads S3 JSONL for the time window
    • Filters: GovMeetings-only, timestamp window, non-empty body
    • Derives severity from channel names and content
    • Chunks alerts (max 90 per Bedrock call)
    • Parallel Bedrock invocations (6 workers) for AI grouping
    • Merges chunk results, recomputes KPIs
    • Optional: Executive summary via second Bedrock call
    • Optional: Product/component enrichment via third Bedrock call
    • Renders Markdown (Full or Lite mode)
    • Posts to Teams via Incoming Webhook
    • Saves state to S3 for next window calculation
    ↓ HTTPS POST
Microsoft Teams Channel (SRE-Govm Digest)
```

---

## Lambda Functions Deep Dive

### 1. Ingest Lambda — `sqs-ingest-lambda.py`

**Function Name:** `sre-digest-sqs-ingest`

**Trigger:** API Gateway HTTP POST (`sre-graph-webhook`)

**Input:** Teams Workflow payload
```json
{
  "messageId": "optional-id",
  "text": "Alert body content",
  "html": "HTML version if available",
  "fromDisplay": "Alert source name",
  "createdDateTime": "ISO timestamp",
  "receivedAt": 1234567890
}
```

**What it does:**
1. Parses API Gateway request body (handles both string and dict)
2. Extracts alert content (prefers `text` over `html`)
3. Parses timestamp from `receivedAt` (epoch) or `createdDateTime` (ISO)
4. **Enriches metadata by scanning the alert text:**
   - **Source detection:** PagerDuty, Elastic, Pingdom, Datadog
   - **Severity extraction:** `sev1`/`sev2`/`critical`/`warning`/`high`/`info`
   - **Status detection:** resolved, acknowledged, open, triggered, recovered
   - **Environment detection:** production, staging, development, testing
   - **Product detection:** govmeetings, onemeeting, swagit, or generic `Service: X`
   - **Incident key extraction:** incident #, rule.id, check ID, or fallback to messageId
5. Builds enriched alert record with UTC timestamps
6. Sends to SQS with MessageAttributes for filtering
7. Supports FIFO queues with MessageGroupId (`{product}-{environment}`)

**Output:** `202 Accepted` with message metadata, or `400`/`500` on errors

**Key Environment Variables:**
- `SQS_QUEUE_URL` — Target SQS queue URL
- `DEBUG_JSON` — Log enrichment details (`true`/`false`)

---

### 2. S3 Writer Lambda — `sqs-s3-writer.py`

**Function Name:** `sre-digest-sqs-s3-writer`

**Trigger:** SQS event (batch of records)

**Input:** SQS event with enriched alert records

**What it does:**
1. Parses each SQS record body (JSON enriched alert)
2. Adds SQS processing metadata:
   - `sqs_message_id`, `receipt_handle`, `receive_count`, `first_receive_timestamp`, `processed_at`
3. **Partitions by event date** (critical design choice):
   - Uses `event_ts_utc` from the alert, NOT processing time
   - Ensures digest Lambda can reliably find alerts for any time window
   - Path: `alerts/year=YYYY/month=MM/day=DD/`
4. Batches alerts into files of `MAX_BATCH_SIZE` (default 100)
5. Writes to S3 as `.jsonl` or `.jsonl.gz` (if compression enabled)
6. Attaches rich S3 metadata for debugging
7. Returns `batchItemFailures` for partial batch failure handling

**Output:**
```json
{
  "batchItemFailures": [
    {"itemIdentifier": "failed-message-id"}
  ]
}
```

**Key Environment Variables:**
- `ALERTS_BUCKET` — S3 bucket for alert storage
- `COMPRESSION_ENABLED` — Enable gzip (`true`/`false`)
- `MAX_BATCH_SIZE` — Records per S3 file (default 100)

---

### 3. Digest Lambda — `s3_digest.py`

**Function Name:** `sre-digest-s3-process`

**Trigger:** Amazon EventBridge Schedule
- Regular digest: every ~90 minutes
- 8-hour summary: 3 times daily

**Input:** Event payload (optional overrides)
```json
{
  "digest_type": "regular",      // or "summary_8h"
  "minutes": 90,                 // override window size
  "window": {                    // manual window override
    "start_utc": "2025-01-01T00:00:00Z",
    "end_utc": "2025-01-01T08:00:00Z"
  }
}
```

**What it does:**

#### Phase 1: Window Calculation
- Loads previous state from S3 (`state/sre-digest-state.json` or `state/sre-digest-state-8h.json`)
- Computes time window based on last run or fallback to interval
- Caps lookback to `MAX_CATCHUP_MINUTES` (default 180) to avoid processing huge backlogs

#### Phase 2: S3 Collection & Filtering
Scans S3 for each day in the window, reads JSONL line-by-line, applies filters:
1. **Timestamp filter** — `event_ts_utc` must be within `[start_utc, end_utc]`
2. **Non-empty body** — skips empty records
3. **Strict GovMeetings filter** — if channel is `alerts_prod`, drops non-GovM alerts
4. **Trivial unknown skip** — drops `Unknown` severity alerts with empty/trivial bodies
5. **Product inference** — scans for GovMeetings subproducts:
   - Canonical: GovMeetings, Legistar, iLegislate, MediaManager, Peak, UFC, Swagit, IQM, Hypitia, OneMeeting
   - Supports `govmeetings:xxx` syntax
   - Custom aliases via `PRODUCT_ALIAS_JSON`
6. **GovM-only policy** — non-GovMeetings alerts are discarded

#### Phase 3: Severity Derivation
Priority order:
1. Explicit `severity` field in record
2. Channel name matches `CRITICAL_CHANNELS` / `WARNING_CHANNELS`
3. Channel name contains hints (`critical`, `sev1`, `p1`, `warning`, `sev2`)
4. Fallback: `Unknown`

#### Phase 4: Parallel Bedrock Grouping
- Chunks alerts into batches of `MAX_BODIES_PER_CALL` (default 90)
- Invokes Bedrock **in parallel** (ThreadPoolExecutor, default 6 workers)
- Model: Anthropic Claude 3 Haiku (configurable via `MODEL_ID`)

**Bedrock tasks:**
1. **Main grouping:** Groups similar alerts, extracts titles, summaries, affected hosts, components, suggested actions, impact
2. **Executive summary** (optional): Ultra-brief summary for leadership
3. **Product/component enrichment** (optional): Normalizes unclear products to canonical GovMeetings subproducts

#### Phase 5: Merge & Render
- Merges chunk results by `(product, title, severity)` key
- Recomputes KPIs by **occurrences** (not just group count)
- Renders Markdown in two modes:
  - **Full mode:** Detailed sections per product with action items, hosts, components
  - **Lite mode:** Compact bullet-per-group format
  - **8-hour summary:** Includes summary table `| Product | Critical | Warning | Total |`
- Truncates to `TEAMS_MAX_CHARS` (default 24000) if too long

#### Phase 6: Post & State Save
- Posts Markdown to Teams via `TEAMS_WEBHOOK`
- Saves state to S3 for next window calculation
- Returns execution summary with KPIs

**Key Environment Variables:**

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ALERTS_BUCKET` | Yes | — | S3 bucket for alerts and state |
| `TEAMS_WEBHOOK` | Yes | — | Teams incoming webhook URL |
| `MODEL_ID` | No | claude-3-haiku | Bedrock model for grouping |
| `DIGEST_INTERVAL_MINUTES` | No | 90 | Default window size |
| `OUTPUT_TIMEZONE` | No | Asia/Kolkata | Timezone for digest timestamps |
| `MAX_ALERTS` | No | 1500 | Max alerts per run |
| `MAX_BODIES_PER_CALL` | No | 90 | Bedrock chunk size |
| `LITE_DIGEST` | No | false | Use compact format |
| `BEDROCK_CONCURRENCY` | No | 6 | Parallel Bedrock threads |
| `BEDROCK_MAX_RETRIES` | No | 4 | Retry attempts per chunk |
| `STRICT_GOVM_ONLY_IN_ALERTS_PROD` | No | true | Drop non-GovM from alerts_prod |
| `GOVM_SUBPRODUCTS_JSON` | No | `{}` | Override canonical subproducts |
| `PRODUCT_ALIAS_JSON` | No | `{}` | Additional product aliases |
| `GENERATE_EXEC_SUMMARY` | No | true | Enable exec summary |
| `GENERATE_PRODUCT_COMPONENTS` | No | true | Enable product enrichment |
| `TEAMS_MAX_CHARS` | No | 24000 | Markdown length limit |

---

## S3 Storage Layout

```
s3://sre-digest-alerts-data/
├── alerts/
│   └── year=2025/
│       └── month=05/
│           └── day=13/
│               ├── alerts-1747123456-abc12345-001.jsonl
│               ├── alerts-1747123512-def67890-002.jsonl
│               └── ...
├── state/
│   ├── sre-digest-state.json          (regular digest state)
│   └── sre-digest-state-8h.json       (8-hour summary state)
└── bedrock/
    └── digests/
        └── {session-id}/
            ├── request-chunk-000.json
            ├── response-chunk-000.json
            └── ...
```

### JSONL Record Format

Each line is a JSON object:
```json
{
  "messageId": "alert-12345",
  "incidentKey": "PD-12345",
  "event_ts_utc": "2025-05-13T10:30:00Z",
  "ingestion_ts_utc": "2025-05-13T10:30:05Z",
  "body": "CRITICAL: Database connection failed...",
  "fromDisplay": "PagerDuty",
  "source": "pagerduty",
  "severity": "critical",
  "status": "triggered",
  "environment": "production",
  "product": "govmeetings",
  "ingestion_source": "teams-workflow",
  "lambda_request_id": "req-abc-123",
  "sqs_metadata": {
    "sqs_message_id": "sqs-msg-456",
    "receive_count": "1",
    "processed_at": "2025-05-13T10:30:06Z"
  },
  "raw_event": { ... }
}
```

### Lifecycle Policy

Alerts transition to cheaper storage over time:
- **0–30 days:** STANDARD (hot, frequently accessed)
- **30–90 days:** STANDARD_IA (infrequent access)
- **90–2555 days:** GLACIER (archival)
- **>2555 days:** Expired and deleted (~7 years retention)

State files and Bedrock logs are NOT affected by this lifecycle.

---

## AI-Powered Alert Monitoring with AWS Bedrock

Our monitoring system uses AWS Bedrock AI to automatically analyze thousands of alerts, filter out noise, and deliver concise, actionable summaries to the team every ~90 minutes — with longer 8-hour executive reports for strategic oversight.

### How It Works

**1. Collection:** Monitoring alerts are continuously collected via Teams workflows and stored in Amazon S3 as partitioned JSONL files.

**2. AI Analysis:** Every 90 minutes (or 8 hours for summaries), AWS Bedrock (Claude AI) processes the latest alerts.

**3. Smart Filtering:** The AI automatically:
   - Filters out noise and benign info
   - Identifies severity (Critical/Warning/Unknown)
   - Groups related issues into incident clusters
   - Extracts affected products (Legistar, OneMeeting, Swagit, etc.)
   - Identifies affected hosts, components, and MAG IDs
   - Generates concise, actionable summaries
   - Suggests specific action items

**4. Delivery:** Processed summaries are posted to the SRE-Govm Digest Teams channel.

### Bedrock Invocation Details

The Digest Lambda reads raw JSONL lines from S3, chunks them into batches of up to 90 alerts, and sends each batch to Bedrock with a strict system prompt.

**Prompt Requirements:**
- Group very similar items (same problem) into single groups
- Extract hosts from free text (tokens that look like hostnames/IPs/FQDNs)
- Identify components if hinted (e.g., `component:`, `group:`)
- Suggested actions must be specific and actionable
- Assess impact briefly
- Choose product only from the canonical GovMeetings list
- Always include `source_channels` per group

**Model:** `anthropic.claude-3-haiku-20240307-v1:0` (configurable via `MODEL_ID`)

### Two Report Types

| Report Type | Frequency | Purpose | Audience | Mode |
|-------------|-----------|---------|----------|------|
| **Regular Digest** | Every ~90 minutes | Real-time operational monitoring | SRE and DevOps | Full or Lite |
| **8-Hour Summary** | 3 times daily | Strategic overview and trends | Leadership | Lite + Table |

**Note:** These reports run independently using separate state tracking files, so the 90-minute operational updates continue uninterrupted while 8-hour executive summaries provide higher-level insights.

### Key Benefits

- **Reduces alert fatigue** — teams see only what matters
- **Faster incident response** — clear, actionable information
- **Better context** — AI understands relationships between alerts
- **Scalable** — processes thousands of alerts automatically
- **Multi-level visibility** — tactical and strategic views

---

## What the Digest Shows

### Full Mode Output

```
🛡️ SRE Digest
Window: 2025-05-13 08:00 - 2025-05-13 09:30 IST
Timestamp (Asia/Kolkata): 2025-05-13 09:30

Summary KPIs
• 🔴 Critical : 5
• 🟠 Warning : 12
• ⚪ Other : 3
• Total alerts (occurrences): 20

Executive Summary
• 3 critical issues affecting Legistar database connectivity
• MediaManager experiencing intermittent API timeouts
• No customer-impacting outages detected

📦 Product — Legistar
🔴 Legistar - Database Connection Failure
• Severity: 🔴 Critical
• Component: Database
• Occurrences: 3
• Source Channels: mon_prod_crit_govmeetings
• Affected Hosts: db-primary-01, db-replica-02
• First Seen (UTC): 2025-05-13T02:30:00Z
• Last Seen (UTC): 2025-05-13T03:15:00Z
• Summary: Repeated connection timeouts to primary database
• Impact: User login blocked for ~15 minutes

Suggested Action Items
• [ ] Check database connection pool settings
• [ ] Verify network connectivity between app and DB tiers
• [ ] Review recent deployment for query plan changes

📦 Product — MediaManager
🟠 MediaManager - API Response Time Degradation
• Severity: 🟠 Warning
• Component: API Gateway
• Occurrences: 7
...

🕒 Last update: 2025-05-13 09:30 – next update at 2025-05-13 11:00
```

### Lite Mode Output

Compact bullet-per-group with metadata inline:
```
🛡️ SRE Digest (Lite)
Window: 2025-05-13 08:00 - 2025-05-13 09:30 IST
Last update: 2025-05-13 09:30 • Next: 2025-05-13 11:00

Summary
• 🔴 Critical (groups): 5
• 🟠 Warning (groups): 12
• ⚪ Other (groups): 3
• Total alerts (occurrences): 20

📦 Legistar
• 🔴 Legistar - Database Connection Failure — Repeated connection timeouts to primary database • Component: Database • Occurrences: 3 • Hosts: db-primary-01, db-replica-02 • Channels: mon_prod_crit_govmeetings  _(2025-05-13T02:30:00Z → 2025-05-13T03:15:00Z)_
```

---

## Input Channels

The following Teams channels are monitored as input sources:

```
mon_prod_crit_govmeetings
mon_prod_warn_govmeetings
mon_prod_crit_OneMeeting
mon_prod_warn_OneMeeting
mon_prod_crit_Swagit
mon_prod_warn_Swagit
alerts_prod
```

---

## Technical Notes

**Severity & Product Derivation:**
- Primarily from the Teams channel name (e.g., `_crit_`, `_warn_`, `_high_`)
- Product detection: `govmeetings`, `onemeeting`, `swagit`, `legistar`, etc.
- Fallback to content analysis via regex and Bedrock inference

**De-duplication:**
- Stable `incidentKey` extracted from incident #, rule.id, or check ID
- Prevents double-counting across repeated notifications

**Occurrences vs Groups:**
- **Groups** = unique incident clusters after AI grouping
- **Occurrences** = total alert firings/repeats within the window (what KPIs count)

**Storage:**
- JSONL files in S3 with Hive-style date partitions (`year=YYYY/month=MM/day=DD/`)
- Optional gzip compression
- Lifecycle policies for cost optimization

**Scheduling:**
- Regular digest: ~90 minutes (configurable)
- 8-hour summary: 3 times daily (independent state tracking)
- EventBridge rules trigger the Digest Lambda

**State Management:**
- Previous run timestamps saved in S3 JSON state files
- Separate state for regular vs 8-hour digests to prevent window collision
- `USE_PREV_WINDOW=true` ensures no gaps between runs
- `MAX_CATCHUP_MINUTES` prevents runaway backlogs

**Error Handling:**
- Partial batch failure handling in S3 Writer (SQS retry)
- Bedrock chunk failures logged but don't stop the digest
- Teams webhook failures return `ok=true posted=false` for monitoring
- JSON parse errors in S3 files are logged and skipped

---

## IAM Permissions Summary

### Ingest Lambda (`sre-digest-sqs-ingest`)
- `sqs:SendMessage` on `sre-alerts-queue`

### S3 Writer Lambda (`sre-digest-sqs-s3-writer`)
- `s3:PutObject` on `sre-digest-alerts-data`
- `sqs:ReceiveMessage`, `sqs:DeleteMessage` on `sre-alerts-queue`

### Digest Lambda (`sre-digest-s3-process`)
- `s3:ListBucket`, `s3:GetObject` on `sre-digest-alerts-data`
- `s3:PutObject` on `sre-digest-alerts-data` (for state and logs)
- `bedrock:InvokeModel` on model resource ARN
- Outbound HTTPS to Teams webhook endpoint

---

## Cost Optimization

- **SQS:** Long polling (`ReceiveMessageWaitTimeSeconds=20`) reduces empty responses
- **S3:** Lifecycle policies automatically tier old data to cheaper storage
- **Lambda:** Right-sized memory (512MB–1024MB based on function)
- **Bedrock:** Chunking and concurrency maximize throughput; Haiku model chosen for cost/speed balance
- **Compression:** Optional gzip reduces S3 storage and transfer costs

---
