# Declarative System Specification: Document Standard

> A document format for defining software systems in a way that both humans and AI agents can consume without prior context, enabling agentic engineering workflows where agents derive implementation from declarations.

**Version:** 1.2  
**What this document is:** The definition of a document type. After reading this, you will understand what a Declarative System Spec is, why it exists, when to create one, how to write one, and how to implement from one.

**Who this is for:** Any human or AI agent who needs to either write a Declarative System Spec for a new project, or implement a system from an existing one.

---

## 1. The Problem This Solves

### 1.1 Context Loss in Agentic Engineering

When AI agents write code for a project, they operate within a context window. Between sessions, that context is lost. Between different agents, that context never existed. The result: agents make locally reasonable but globally incoherent decisions because they lack the full picture of what the system is, why it exists, and what constraints govern it.

Traditional documentation doesn't solve this because it's written for humans who accumulate understanding over time. Agents don't accumulate — they receive context fresh every session.

### 1.2 Imperative vs. Declarative Specifications

Most software documentation is **imperative** — it describes procedures, sequences, and how-to instructions:

```
Imperative: "First, create a database table for users. Then, write an API 
endpoint that accepts POST requests. The endpoint should validate the email 
field, hash the password, and insert a row..."
```

Imperative documentation is fragile. It assumes a specific implementation path, breaks when tools or frameworks change, and forces agents into step-by-step execution rather than architectural reasoning.

A **declarative** specification instead describes what the system IS — what states exist, what invariants hold, what contracts bind components, what behaviors the system exhibits:

```
Declarative: "A User exists in the system with the following state: id, email 
(unique, validated format), password_hash (never stores plaintext), created_at.
Invariant: A User's email MUST be unique across all Users. Invariant: 
password_hash MUST be generated via bcrypt with cost factor ≥ 12."
```

The declarative version contains the same information but is implementation-agnostic. An agent can derive the database schema, API validation logic, and hashing implementation from the declarations without being told which framework, language, or architecture to use.

### 1.3 The Terraform Analogy

This approach is structurally identical to Infrastructure as Code tools like Terraform:

- **Terraform** declares desired infrastructure state. The engine figures out how to reach that state.
- **A Declarative System Spec** declares desired system behavior and structure. The implementing agent figures out how to build it.

In both cases, the human describes WHAT they want, not HOW to build it. The execution layer handles the how.

### 1.4 Why This Enables Agentic Engineering

In an agentic engineering workflow:

1. A human writes (or collaborates with AI to write) a Declarative System Spec.
2. An AI agent reads the spec and derives: architecture, data models, service boundaries, API contracts, test cases.
3. The agent implements incrementally, checking each decision against the spec's invariants and contracts.
4. When the agent's context resets (new session, different agent), the spec provides complete re-grounding — no conversation history needed.
5. Multiple agents can work on different components simultaneously, using the spec as their shared contract.

The spec is the single source of truth. It replaces the need for the human to re-explain the system every session.

---

## 2. When to Write a Declarative System Spec

**Write one when:**
- Starting a new project that will be built with AI agent assistance
- The system is complex enough that implementation decisions in one component affect other components
- Multiple sessions or agents will be involved in implementation
- The human needs to maintain architectural control without micromanaging code
- The system has non-obvious constraints, invariants, or behavioral requirements that agents would otherwise miss

**Don't write one when:**
- The task is a single script or utility (just describe it inline)
- The implementation is trivial enough to complete in one session without architectural decisions
- You're prototyping and the system design isn't stable yet (write it after the design stabilizes)

**The investment tradeoff:** A Declarative System Spec takes significant effort to write well. The payoff is proportional to how many times agents will need to re-ground on the system — if the answer is "many times over months or years," the investment is worth it. If the answer is "once," it probably isn't.

---

## 3. Document Structure

A Declarative System Spec SHOULD contain the following sections. Not every system needs every section, but the ordering and purpose of each section is deliberate.

### 3.1 Required Sections

#### System Identity

**Purpose:** Establish what the system is, why it exists, and what it is NOT.

This is the most important section. An agent reading only this section should understand the system's reason for existing, its scope boundaries, and the mental model that governs all other decisions.

**Must include:**
- A one-paragraph description of what the system does and why
- The core problem being solved (what fails without this system)
- Explicit "what this system is NOT" boundaries to prevent scope creep
- The relationship model: who/what are the system's users, and what is the contract between them

**Why "is NOT" matters:** Agents are completion machines — they will expand scope to be helpful unless explicitly told not to. Without clear boundaries, an agent implementing a monitoring system will add dashboard features, a notification system will add message composition, a data pipeline will add analytics. The "is NOT" section is the primary defense against this.

#### Core Abstractions

**Purpose:** Declare every entity the system manages, its state, and the invariants that govern it.

For each abstraction:
- **Name and definition** in plain language
- **Declared state** — every field, its type, and its meaning
- **Invariants** — rules that MUST hold true at all times, stated as assertions
- **Relationships** — how this abstraction connects to others

**Writing invariants well:**

```
Weak:     "Emails should be unique."
Strong:   "A User's email MUST be unique across all Users. Attempting to 
           create a User with a duplicate email is a system error that MUST 
           be prevented at the persistence layer, not just the application layer."
```

Strong invariants tell the agent not just WHAT must be true but WHERE the enforcement lives. This prevents the common failure of an agent adding validation in the API layer but forgetting the database constraint.

#### System Behaviors

**Purpose:** Declare what the system does in response to events and state changes.

Behaviors are described as state transitions, not procedures:

```
Imperative (avoid): "When a user signs up, send a welcome email, then create 
their profile, then log the event."

Declarative (prefer): "When a User transitions from non-existent to created:
- A WelcomeNotification MUST be queued (see Notification contract)
- A UserProfile MUST be created with default values
- A UserCreated event MUST be appended to the event log
These three outcomes are independent — failure of one MUST NOT prevent the others."
```

The declarative version lets the agent decide whether to use a message queue, database triggers, application events, or synchronous calls — the spec doesn't care about mechanism, only outcomes.

#### Component Contracts

**Purpose:** Define the boundaries and interfaces between components.

For each component boundary:
- What the component is responsible for
- What it MUST do (positive contract)
- What it MUST NOT do (negative contract — prevents coupling)
- What its interface looks like (inputs, outputs, not implementation)

**The negative contract is critical.** Without it, agents will add logic to whatever component they're currently implementing, regardless of whether it belongs there:

```
"Interface adapters MUST NOT contain business logic."
"Capabilities MUST NOT store state between invocations."
"The notification system MUST NOT decide urgency — that is the judgment 
system's responsibility."
```

#### Design Principles

**Purpose:** Provide decision-making heuristics for situations the spec doesn't explicitly cover.

No spec covers every implementation decision. Design principles tell the agent: "When you encounter a tradeoff the spec doesn't address, resolve it using these priorities."

Each principle should include:
- The principle itself (short, memorable)
- Why it exists (the failure mode it prevents)

```
Principle: "Fail Visible"
Why: "Silent failures erode trust invisibly. A user who thinks the system is 
monitoring something when it isn't has a false sense of coverage. All failures 
MUST be surfaced."
```

#### Acceptance Criteria

**Purpose:** Verifiable conditions that determine whether the system works.

These are the spec's test suite. Each criterion should be:
- **Binary** — it either passes or it doesn't
- **Testable** — an agent can write an automated test for it or verify it manually
- **Traceable** — it maps to a specific section of the spec

```
Good:  "A Watch with status: active MUST be checked within ±10% of its 
        declared cadence."
Bad:   "The system should be reliable."
```

Acceptance criteria serve double duty: they validate the implementation AND they help the agent prioritize. If an agent must choose between two approaches, the one that satisfies more acceptance criteria wins.

### 3.2 Optional Sections

#### Open Questions

Unresolved design decisions that will be answered through development. Including these prevents agents from making premature decisions on topics the human hasn't resolved yet. An agent encountering an open question should flag it to the human rather than guessing.

#### Glossary

Domain-specific terms and their precise definitions. Essential when the system uses common words with specific meanings (e.g., "Watch" meaning a monitoring declaration, not a timepiece).

#### Document History

Changes over time. Useful for agents that need to understand what changed since a previous implementation pass.

#### Prior Art / Context

References to existing systems, documents, or conversations that informed the spec. Useful for human readers, less useful for agents (who should derive everything they need from the spec itself).

---

## 4. Writing Principles

### 4.1 State What IS, Not What To Do

Every sentence should describe a state, constraint, or contract — not an action sequence.

```
Avoid:  "The system should check watches periodically and send notifications."
Prefer: "A Watch with status: active has a declared cadence. The system MUST 
         check the Watch at that cadence. A missed check is a system failure."
```

### 4.2 Use RFC 2119 Language Precisely

- **MUST / MUST NOT** — Absolute requirement. Violation is a system failure.
- **SHOULD / SHOULD NOT** — Strong recommendation. Violation requires justification.
- **MAY** — Optional. The agent can choose.

Don't use "should" when you mean "must." Agents will treat ambiguity as permission.

### 4.3 Make Invariants Exhaustive

If a field can only hold certain values, enumerate them. If two states are mutually exclusive, say so. If a transition is one-way (e.g., events are append-only), declare it.

Agents are better at following explicit constraints than inferring implicit ones. Anything left unstated will be interpreted as "unconstrained."

### 4.4 Separate Concerns by Rate of Change

Group things that change together. Interfaces change when platforms change. Data models change when requirements change. Business logic changes when the domain evolves. These should be in separate sections because agents working on one concern shouldn't need to understand the others in full detail.

### 4.5 Include the "Why"

For every principle, constraint, or unusual design decision, explain why it exists. Agents that understand intent make better tradeoff decisions than agents that follow rules blindly.

```
Without why: "Events are append-only."
With why:    "Events are append-only. They are never modified or deleted. This 
              enables: debugging (what happened?), learning (what patterns exist?), 
              recovery (rebuild state from events), and audit (why did the system 
              do that?)."
```

An agent that knows WHY events are append-only will correctly refuse to add an "event update" endpoint, even if a feature request seems to require one.

### 4.6 Be Precise About Boundaries

The most common implementation failure in agentic engineering is logic ending up in the wrong component. The spec must clearly state which component is responsible for which decision. Use negative contracts liberally:

```
"The interface adapter translates formats. It MUST NOT filter content."
"The judgment system decides urgency. It MUST NOT decide delivery mechanism."
"Capabilities gather data. They MUST NOT interpret results."
```

### 4.7 Write for Cold Start

Assume the reader has zero prior context. Every term is defined. Every relationship is explicit. Every constraint is stated, not implied. The document should be fully self-contained — a reader should never need to ask "but what does X mean?" or "why is this constraint here?"

This is the core difference from internal documentation (which assumes shared context) and a Declarative System Spec (which assumes none).

### 4.8 Mark the Provenance of Every Claim

Every normative statement should record where it came from: the human, or the agent.

Specs are written collaboratively, and agents write most of the words. In doing so an agent inevitably fills gaps — it picks a default, invents a threshold, adds a restriction nobody asked for. Written down without attribution, that invention is indistinguishable from a requirement. The next session reads it as a constraint the human imposed, honours it, and builds on it. Constraints accumulate that no one ever chose.

The failure mode is **constraint laundering** — a model's guess enters the spec as prose, and every subsequent reading promotes it to a requirement. It is the inverse of staleness (Section 6.4): not the spec falling behind reality, but the spec inventing a reality of its own.

Tag every normative statement with its origin:

```
[H]   Stated explicitly by the human.
[H✓]  Proposed by an agent, then explicitly approved by the human.
[M]   Model-derived. A default, a guess, or a convention nobody asked for.
```

```
Untagged: "Sandbox VMs MUST be destroyed within 24 hours."
Tagged:   "[M] Sandbox VMs MUST be destroyed within 24 hours."
```

The first is a requirement forever. The second is a default any session may change.

A spec using these tags MUST state their authority once, near the top:

> `[M]` items carry no authority. They are the model's own defaults, not constraints the human
> imposed. Any session may challenge, change, or delete an `[M]` item without asking. Never cite
> an `[M]` item as a reason something cannot be done.

**Why this matters more for agents than for humans:** a human author remembers which lines were their own idea. An agent reading cold has no such memory — provenance survives only if it was written down. Untagged prose defaults to maximum authority, which is exactly backwards for the parts the model made up.

---

## 5. How Agents Should Consume This Document

This section is guidance for AI agents implementing a system from a Declarative System Spec.

### 5.1 Reading Order

1. **System Identity** — understand what you're building and its boundaries before anything else.
2. **Core Abstractions** — these become your data models and the nouns of the system.
3. **System Behaviors** — these become your business logic and the verbs of the system.
4. **Component Contracts** — these become your service boundaries and API interfaces.
5. **Design Principles** — these become your decision-making heuristics.
6. **Acceptance Criteria** — these become your test suite and definition of done.

### 5.2 Deriving Architecture

The spec doesn't prescribe architecture. The agent derives it:

- **Core Abstractions** → database tables/collections, domain models
- **Component Contracts** → services, modules, or microservices with defined interfaces
- **System Behaviors** → event handlers, state machines, business logic layer
- **Invariants** → database constraints, validation logic, guard clauses
- **Acceptance Criteria** → integration tests, end-to-end tests

### 5.3 Handling Ambiguity

When the spec is ambiguous or silent on a topic:

1. Check **Open Questions** — if the topic is listed there, flag it to the human rather than guessing.
2. Check **Design Principles** — resolve the ambiguity using the stated priorities.
3. If neither helps, choose the most conservative option (least scope, least coupling, most explicit) and document the assumption for human review.

### 5.4 Checking Decisions Against the Spec

Before implementing any significant decision, the agent should verify:

- Does this violate any declared invariant?
- Does this place logic in a component whose contract forbids it?
- Does this create coupling between components the spec says should be decoupled?
- Does this move toward or away from the acceptance criteria?

### 5.5 Incremental Implementation

The acceptance criteria define natural implementation phases. Agents should implement in order of acceptance criteria tiers (if the spec has tiered criteria), verifying each tier passes before moving to the next.

---

## 6. How Humans Should Use This Document

### 6.1 Writing the Spec

The most effective workflow is collaborative:

1. **Human writes System Identity** — this requires human judgment about purpose, scope, and boundaries.
2. **Human and AI iterate on Core Abstractions** — the human knows what entities matter; the AI helps formalize state and invariants.
3. **AI drafts System Behaviors and Component Contracts** — based on the abstractions, the AI can derive most behavioral requirements. The human reviews and corrects.
4. **Human writes Design Principles** — these encode values and priorities that come from the human, not from the system's structure.
5. **AI drafts Acceptance Criteria** — derived from everything above. The human adjusts priority and adds subjective criteria the AI can't infer.

### 6.2 Maintaining the Spec

The spec is a living document. When implementation reveals that a declaration was wrong, incomplete, or impractical:

- Update the spec FIRST, then update the implementation.
- Never let the implementation diverge from the spec silently — this destroys the spec's value as single source of truth.
- Use Document History to track changes.

### 6.3 Handing Off to Agents

When starting a new agent session:

1. Provide the spec as context.
2. State which acceptance criteria tier is currently being implemented.
3. State what has already been built (if anything).
4. The agent should be able to resume from this point without additional conversation history.

This is the core value proposition: **the spec replaces conversation history as the source of truth.**

### 6.4 The Implementation Feedback Loop

Specs get stale during implementation. This is expected, not a failure.

No spec, however carefully written, fully anticipates what implementation reveals. Edge cases surface. Abstractions that looked clean on paper prove awkward in code. Performance constraints force structural changes. A component contract that seemed right turns out to allocate responsibility to the wrong boundary. This is normal — it's how systems get built.

The failure mode is not staleness itself. The failure mode is **silent divergence** — when the implementation evolves but the spec doesn't, and the spec gradually becomes a historical artifact rather than a living contract. Once this happens, new agent sessions re-ground on a spec that no longer reflects reality, and the spec's core value is destroyed.

**The reconciliation practice:**

1. **Agents should flag divergence, not just implement around it.** When an agent makes an implementation decision that conflicts with or isn't covered by the spec, it should document the divergence explicitly — not silently work around it. The output should include a note: "This deviates from Section X because [reason]. The spec should be updated to reflect [new understanding]."

2. **Periodically pause implementation to reconcile.** After completing a tier of acceptance criteria, or when accumulated divergences reach a threshold, stop building and update the spec. This is not overhead — it's maintenance of the source of truth. The cost of reconciliation is always less than the cost of an agent re-grounding on a stale spec and making decisions based on outdated declarations.

3. **Treat implementation discoveries as spec improvements.** When building reveals that an invariant was wrong, a component boundary was misplaced, or a behavior was underspecified — that's new knowledge. Feed it back into the spec. The spec gets *better* through implementation, not just maintained.

4. **Version the reconciliation.** Use Document History to track what changed and why. An agent encountering a spec that was updated mid-implementation can understand the evolution, not just the current state.

**The mental model:** The spec is not a waterfall document that gets written once and handed to builders. It's closer to a constitution — it sets the framework and principles, it's the highest authority, but it has an amendment process. Implementation is the legislature that proposes amendments based on ground truth. The human ratifies them.

---

## 7. Template

A minimal skeleton for a new Declarative System Spec:

```markdown
# [System Name]: [One-Line Description]

> [One-sentence elevator pitch]

**Document Type:** Declarative System Specification  
**Last Updated:** [Date]  
**Owner:** [Name]

---

## 1. System Identity

### 1.1 What This System Is
[What it does, why it exists, the core problem it solves]

### 1.2 What This System Is NOT
[Explicit scope boundaries]

---

## 2. Core Abstractions

### 2.1 [Abstraction Name]
[Definition in plain language]

**Declared State:**
[Fields, types, meanings]

**Invariants:**
[Rules that MUST hold]

---

## 3. System Behaviors

### 3.1 [Behavior Name]
[State transitions, triggered by what, producing what outcomes]

---

## 4. Component Contracts

### 4.1 [Component Name]
**Responsible for:** [positive contract]
**MUST NOT:** [negative contract]
**Interface:** [inputs, outputs]

---

## 5. Design Principles

### 5.1 [Principle Name]
[Statement]
*Why:* [Failure mode this prevents]

---

## 6. Acceptance Criteria

### 6.1 [Tier Name]
- [ ] [Verifiable condition]
- [ ] [Verifiable condition]

---

## 7. Open Questions
1. [Unresolved decision]

## 8. Glossary
| Term | Definition |
|---|---|

## 9. Document History
| Date | Change |
|---|---|
```

---

## 8. Anti-Patterns

Things that degrade a Declarative System Spec's effectiveness:

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Prescribing implementation | "Use PostgreSQL with a users table" locks agents into choices that may not be optimal | Declare the data model. Let the agent choose storage. |
| Vague invariants | "Data should be consistent" is unenforceable | State specific consistency rules: "X MUST equal Y at all times" |
| Missing boundaries | No "is NOT" section | Agents will expand scope. Always include explicit exclusions. |
| Imperative sequences | "First do X, then do Y, then do Z" | Describe the end state and constraints, not the procedure |
| Implicit knowledge | "Obviously, this needs authentication" | If it's not in the spec, it doesn't exist to the agent. State everything. |
| Monolithic behaviors | One section describing everything the system does | Separate by component responsibility and rate of change |
| No acceptance criteria | The spec describes what to build but not how to verify it works | Always include testable conditions |
| Stale spec | Implementation diverged and the spec wasn't updated | Update spec first, then implementation. Never the reverse. |
| Unattributed model invention | An agent's guess is written as plain prose; later sessions honour it as a human requirement | Tag every normative statement `[H]`, `[H✓]` or `[M]` (Section 4.8) |

---

## 9. Relationship to Other Document Types

| Document | Purpose | Relationship to Declarative System Spec |
|---|---|---|
| **README** | How to run the project | Derived from the spec. The spec says what; the README says how to operate. |
| **API documentation** | Endpoint contracts | Derived from Component Contracts section. |
| **Architecture Decision Records** | Why specific implementation choices were made | Complementary. ADRs document choices the spec deliberately left open. |
| **User stories / tickets** | What to build next | Derived from Acceptance Criteria. Each criterion can become one or more tickets. |
| **Test plans** | How to verify the system works | Derived from Acceptance Criteria and Invariants. |
| **Runbooks** | How to operate the system in production | Not derivable from the spec — requires operational knowledge. |

The Declarative System Spec sits upstream of all implementation artifacts. It is the source from which everything else is derived.

---

## 10. Version History

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-02 | Initial release — core standard established |
| 1.1 | 2026-03 | Added Section 6.4: The Implementation Feedback Loop — addressing spec staleness during implementation based on production usage |
| 1.2 | 2026-08 | Added Section 4.8: Mark the Provenance of Every Claim — preventing model-invented constraints from being read as human requirements by later sessions |

---

*This document defines a document type. Use it to create Declarative System Specs for your projects. Hand those specs to AI agents as their implementation contract. Update the specs when reality diverges from declaration. The spec is the single source of truth — everything else is derived.*
