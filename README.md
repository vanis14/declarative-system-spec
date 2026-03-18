# Declarative System Spec

A document standard for specifying software systems in a way that both humans and AI agents can consume without prior context.

## The Problem

When AI agents build software across multiple sessions, they lose context between each session. The result: locally reasonable but globally incoherent decisions, because no single session has the full picture of what the system is, why it exists, and what constraints govern it.

Traditional documentation doesn't solve this — it's written for humans who accumulate understanding over time. Agents don't accumulate. They receive context fresh every session.

## The Solution

A **Declarative System Spec** describes what a system *is* — what states exist, what invariants hold, what contracts bind components, what behaviors the system exhibits — rather than how to build it. The implementing agent derives architecture, data models, and service boundaries from the declarations.

Think Terraform, but for software systems. The human declares desired state. The agent figures out how to get there.

## What's in This Repo

- **[SPEC.md](SPEC.md)** — The full document standard. Covers when to write a spec, how to structure it, writing principles, guidance for both human authors and consuming agents, anti-patterns, and a template.

- **[examples/](examples/)** — Worked examples of Declarative System Specs built using this standard. *(coming soon)*

## Quick Start

1. Read **Section 1** (The Problem This Solves) to understand the approach.
2. Read **Section 3** (Document Structure) for what goes in a spec.
3. Use the **Template** in Section 7 to start your own.
4. Hand the spec to your AI agent as the implementation contract.

## Who This Is For

- Engineers building software with AI coding agents (Claude, Cursor, Copilot, etc.)
- Teams where multiple agents or sessions touch the same codebase
- Anyone who needs to maintain architectural control without micromanaging every line of code

## Origin

This standard was derived from building Spark, a cognitive partnership system, and from designing production AWS infrastructure — both built primarily through AI agent sessions. The standard emerged from solving the practical problem of maintaining architectural coherence across hundreds of agent sessions over months.

Blog post with full context: [https://blog.vanislim.com/declare-dont-instruct/](https://blog.vanislim.com/declare-dont-instruct/)

## License

MIT

## Contributing

Issues and pull requests welcome. If you've used this standard on a project and have feedback on what worked or didn't, I'd genuinely like to hear it.
