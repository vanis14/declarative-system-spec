# Pulse: Personal Information Feed with Judgment

> A system that monitors information sources on a user's behalf, applies judgment about relevance and urgency, and delivers digests rather than a stream of raw alerts.

**Document Type:** Declarative System Specification  
**Version:** 1.0  
**Last Updated:** March 2026  
**Owner:** [Author]

---

## 1. System Identity

### 1.1 What Pulse Is

Pulse is a personal information monitoring system that sits between raw data sources and a human who doesn't want to check them manually.

The core problem: a person cares about things that change — prices, availability, news in specific domains, project statuses, weather patterns, job postings matching specific criteria. Today, they either check manually (expensive in attention) or set up naive alerts that fire on every change regardless of significance (expensive in notification fatigue). Pulse is the judgment layer between data sources and the human. It decides not just *whether* something changed, but whether the change *matters enough to surface right now*.

The operating model:

```
Human                              Pulse
─────                              ─────
Declares interests ("watches")     Monitors sources at declared cadence
Sets preferences (quiet hours,     Evaluates changes against conditions
  urgency thresholds, channels)    Applies judgment: urgency, timing, batching
Reviews digests                    Delivers via preferred channel and schedule
Provides feedback                  Learns from feedback to improve judgment
```

### 1.2 What Pulse Is NOT

- **Pulse is not a dashboard.** It does not provide a UI for browsing data. It delivers information to the human through existing channels (email, Discord, Slack, etc.).
- **Pulse is not a general-purpose automation tool.** It monitors and notifies. It does not take actions on the user's behalf (no auto-purchasing, no auto-responding, no write operations on external services).
- **Pulse is not a real-time stream processor.** Minimum check cadence is 1 minute. Sub-second latency is not a goal.
- **Pulse is not multi-tenant.** It serves one human. Access control, billing, and user management are out of scope.

---

## 2. Core Abstractions

### 2.1 Watch

A **Watch** is a declared monitoring interest — the human says "I care about this."

**Declared State:**

| Field | Type | Description |
|---|---|---|
| id | UUID | Immutable identifier |
| intent | string | Original natural language request, stored verbatim |
| parsed_condition | Condition | Machine-evaluable criteria derived from intent |
| source | SourceRef | What data source to check |
| cadence | Duration | How often to check (minimum: 1 minute) |
| urgency_hint | enum: low, normal, high, critical | Human-declared baseline urgency |
| status | enum: active, paused, expired, completed | Current lifecycle state |
| created_at | timestamp | When the Watch was created |
| last_checked_at | timestamp | When the source was last queried |
| expires_at | timestamp? | Optional auto-expiration |

**Invariants:**

- A Watch MUST always retain its original `intent` field unmodified. The `parsed_condition` may be re-derived as parsing improves, but the intent is ground truth.
- A Watch with `status: active` MUST be checked within ±10% of its declared `cadence`. A missed check is a system failure that MUST be logged.
- A Watch MUST be bound to exactly one source. If the source becomes unhealthy, the Watch enters a degraded state but does NOT auto-pause — it continues attempting checks and surfaces the health issue to the human.
- `status` transitions are one-directional: `active → paused` (reversible), `active → expired` (irreversible), `active → completed` (irreversible). A completed or expired Watch MUST NOT be reactivated — create a new Watch instead.

### 2.2 Condition

A **Condition** is a machine-evaluable expression that determines whether a Watch has triggered.

**Declared State:**

| Field | Type | Description |
|---|---|---|
| type | enum: threshold, change, match, compound | What kind of evaluation |
| expression | ConditionExpr | The evaluable criteria |
| last_result | boolean | Did the last check trigger? |
| consecutive_triggers | integer | How many checks in a row have triggered |

**Invariants:**

- A Condition MUST be evaluable without LLM inference during normal operation. Conditions are the fast path — they run on every check. LLM evaluation is reserved for ambiguous cases only.
- A Condition MUST produce a boolean result: triggered or not triggered. Partial or probabilistic results are not valid.

### 2.3 Notification

A **Notification** is a potential message to the human that requires judgment before delivery.

**Declared State:**

| Field | Type | Description |
|---|---|---|
| id | UUID | Immutable identifier |
| watch_id | UUID | Which Watch produced this |
| trigger_data | object | Raw data from the triggering check |
| assessed_urgency | enum: low, normal, high, critical | System-determined urgency after judgment |
| delivery_decision | enum: immediate, batch, suppress | How to deliver |
| delivered_at | timestamp? | When actually delivered (null if batched/suppressed) |
| feedback | enum: useful, not_useful, null | Human response |

**Invariants:**

- A Notification MUST NOT be delivered without passing through the judgment system. There is no "bypass" path from trigger to delivery.
- A Notification with `assessed_urgency: critical` MUST be delivered within 60 seconds of creation.
- A suppressed Notification MUST be logged and queryable. The human can always ask "what did you suppress today?"

### 2.4 Source

A **Source** is an external data provider that Pulse can query.

**Declared State:**

| Field | Type | Description |
|---|---|---|
| id | UUID | Immutable identifier |
| type | enum: http, rss, api, scrape | How to access it |
| config | object | Connection details (URL, auth, headers, etc.) |
| health | enum: healthy, degraded, failed | Current status |
| last_success_at | timestamp? | Last successful query |

**Invariants:**

- A Source with `health: failed` for more than 24 hours MUST generate a health notification to the human. The human decides whether to fix it or remove dependent Watches.
- Source credentials MUST be stored encrypted at rest. They MUST NOT appear in logs, events, or notifications.

---

## 3. System Behaviors

### 3.1 Watch Check Cycle

When a Watch's cadence timer fires:

1. The bound Source is queried for current data.
2. The returned data is evaluated against the Watch's Condition.
3. If the Condition triggers, a Notification is created with the trigger data.
4. The Notification enters the judgment pipeline (Section 3.2).
5. Regardless of trigger, `last_checked_at` is updated.

If the Source query fails:
- The failure is logged as an event.
- The Source's `health` is updated (degraded after 3 consecutive failures, failed after 10).
- The Watch remains active — the next cadence tick will retry.
- A missed check due to source failure is recorded separately from a missed check due to system failure.

### 3.2 Judgment Pipeline

When a Notification is created:

1. **Urgency Assessment:** Compare the Watch's `urgency_hint`, the Condition's `consecutive_triggers`, recent notification history, and time-of-day against the human's preferences. Produce `assessed_urgency`.
2. **Delivery Decision:** Based on assessed urgency and current context:
   - `critical` → immediate delivery
   - `high` → immediate delivery unless within quiet hours (then deliver at quiet hours end)
   - `normal` → batch into next scheduled digest
   - `low` → batch into weekly digest or suppress if similar notification was recently delivered
3. **Channel Selection:** Based on urgency and human preferences, select delivery channel.

The judgment system SHOULD improve over time using feedback signals. After 20+ feedback signals, learned patterns SHOULD visibly affect future urgency assessment.

### 3.3 Digest Assembly

At the human's configured digest time (default: 8:00 AM local):

1. Collect all Notifications with `delivery_decision: batch` that haven't been delivered.
2. Group by domain or Watch similarity.
3. Order by assessed urgency (highest first).
4. Format and deliver as a single message through the configured channel.

### 3.4 Startup Recovery

When Pulse starts (or restarts):

1. Load all Watches with `status: active`.
2. Calculate missed checks during downtime.
3. Execute missed checks in priority order (highest urgency first).
4. Resume normal scheduling.
5. If downtime exceeded 1 hour, notify human: "I was offline for [duration]. Catching up on [N] missed checks."

---

## 4. Component Contracts

### 4.1 Scheduler

**Responsible for:** Maintaining the cadence timer for each active Watch. Firing check events at the correct time.

**MUST NOT:** Evaluate conditions, make judgment decisions, or interact with sources directly. The scheduler fires events — other components handle them.

**Interface:**
- Input: Watch created/paused/resumed/deleted events
- Output: CheckDue events at the Watch's cadence

### 4.2 Source Adapter

**Responsible for:** Querying external sources and returning structured data. Managing connection details, authentication, retries, and health tracking.

**MUST NOT:** Evaluate conditions against returned data. Must not interpret results — only fetch and structure them.

**Interface:**
- Input: Source config + query parameters
- Output: Structured data or error

### 4.3 Condition Evaluator

**Responsible for:** Evaluating a Condition expression against source data. Producing a boolean trigger result.

**MUST NOT:** Make urgency decisions or delivery decisions. A trigger is a fact — what to do about it is the judgment system's job.

**Interface:**
- Input: Condition + source data
- Output: { triggered: boolean, match_details: object }

### 4.4 Judgment System

**Responsible for:** Assessing urgency, deciding delivery timing, selecting channels, learning from feedback.

**MUST NOT:** Query sources, evaluate conditions, or format messages. The judgment system decides *what to do* — other components handle the doing.

**Interface:**
- Input: Notification + human preferences + historical context
- Output: { assessed_urgency, delivery_decision, channel }

### 4.5 Delivery Adapter

**Responsible for:** Formatting and sending notifications through configured channels (email, Discord, Slack, etc.). Managing channel-specific formatting and API details.

**MUST NOT:** Make judgment decisions. If the judgment system says "deliver immediately via Discord," the delivery adapter does exactly that. It does not second-guess urgency or timing.

**Interface:**
- Input: Notification + channel + format preferences
- Output: Delivery confirmation or failure

### 4.6 Event Store

**Responsible for:** Persisting all system events as an append-only log. Providing query access to historical events.

**MUST NOT:** Interpret events or trigger behaviors based on events. The event store is passive storage — other components read from it to inform their decisions.

**Interface:**
- Input: Event to append
- Output: Events matching query criteria

---

## 5. Design Principles

### 5.1 Preserve Intent

Always store the original human request separately from parsed interpretations. Re-parsing may improve; original intent is ground truth.

*Why:* An early parsing that misinterprets "cheap flights to Tokyo" as "any flights to Tokyo" loses information. If the intent is preserved, re-parsing can recover the correct interpretation later.

### 5.2 Decouple Ruthlessly

Interfaces know nothing about business logic. Sources know nothing about conditions. Conditions know nothing about urgency. Urgency knows nothing about delivery format.

*Why:* Coupling is the primary way systems become unmaintainable. Every coupling point is a place where changing one component forces changes in others. Pulse should be a system where any single component can be replaced without affecting the rest.

### 5.3 Fail Visible

A failing source, a missed check, a delivery failure — all MUST be surfaced to the human. Silent failures erode trust invisibly. A human who thinks Pulse is monitoring something when it isn't has a false sense of coverage.

*Why:* The worst outcome isn't "Pulse failed." It's "Pulse failed and the human didn't know."

### 5.4 Less Notification, Not More

Default to batching and suppression, not immediate delivery. It's easier to turn up notification frequency than to recover from notification fatigue.

*Why:* A system the human ignores because it's too noisy is worse than no system at all.

### 5.5 Event-Source Everything

Store events, not just current state. Events are append-only and immutable. This enables debugging (what happened?), learning (what patterns exist?), recovery (rebuild state from events), and audit (why did the system do that?).

*Why:* State tells you where you are. Events tell you how you got there. For a system that learns from history, events are the raw material.

---

## 6. Acceptance Criteria

### 6.1 Core Functionality

- [ ] A Watch can be created from a structured input (natural language parsing is a later tier)
- [ ] An active Watch is checked within ±10% of its declared cadence
- [ ] A triggered Condition produces a Notification
- [ ] The Notification passes through judgment before delivery
- [ ] Critical urgency Notifications are delivered within 60 seconds
- [ ] Normal urgency Notifications are batched into the daily digest
- [ ] A daily digest is assembled and delivered at the configured time
- [ ] Human can query: "What did you check today?" and receive an accurate answer
- [ ] Human can query: "What did you suppress?" and see suppressed Notifications

### 6.2 Reliability

- [ ] A Source failure does not crash the system or affect other Watches
- [ ] A Source that fails 10+ consecutive checks is marked `failed` and the human is notified
- [ ] After a restart, missed checks are detected and executed in priority order
- [ ] Downtime exceeding 1 hour produces a recovery notification
- [ ] All events are persisted to the event store — no data loss on restart

### 6.3 Intelligence

- [ ] After 20+ feedback signals, judgment patterns visibly change (measurable through before/after urgency distributions)
- [ ] Human reports receiving fewer notifications than a naive "alert on every trigger" system would produce
- [ ] Quiet hours are respected for non-critical notifications

---

## 7. Open Questions

1. **Condition expression format:** JSON-based DSL? Natural language evaluated by LLM with caching? Custom grammar? Tradeoff is expressiveness vs. evaluation speed vs. authoring ease.

2. **Judgment bootstrapping:** How does the system make good urgency decisions without historical feedback data? Conservative defaults (batch everything until told otherwise)? Explicit calibration conversation?

3. **Channel abstraction depth:** Should channel adapters support rich formatting (embeds, buttons, reactions) or just plain text? Rich formatting increases per-channel implementation cost but improves UX.

4. **Watch creation interface:** CLI? Chat interface? Web form? The choice affects how intents are captured and how natural language parsing is integrated.

---

## 8. Glossary

| Term | Definition |
|---|---|
| **Watch** | A declared monitoring interest — "I care about this" |
| **Condition** | Machine-evaluable criteria for determining whether a Watch has triggered |
| **Notification** | A potential message to the human that must pass through judgment before delivery |
| **Source** | An external data provider that Pulse can query |
| **Judgment** | The decision about what to do with a trigger — urgency, timing, channel |
| **Digest** | A batch of notifications delivered together at a scheduled time |
| **Intent** | The original natural language request — immutable, never modified |
| **Cadence** | How often a Watch is checked |

---

## 9. Document History

| Date | Change |
|---|---|
| 2026-03 | Initial specification |

---

*This document declares what Pulse is and what states it maintains. Implementation agents derive architecture, data models, and service boundaries from these declarations. When implementation conflicts with this specification, this specification wins.*
