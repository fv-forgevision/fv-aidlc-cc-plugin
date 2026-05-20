# Multi-Agent Mode — Opt-In

**Extension**: Multi-Agent Mode

## Opt-In Prompt

The following question is automatically included in the Requirements Analysis clarifying questions when this extension is loaded. Present this question using the `AskUserQuestion` tool:

**Question**: Should AI-DLC be executed in Multi-Agent mode for this project?

**Options**:
- **Yes** — enforce all TEAM and MAGENT rules as blocking constraints. The Orchestrator runs Inception alone, then launches one Sub-Agent per Unit in parallel during Construction Phase (recommended for projects with 3+ Units that can be developed in parallel, with clear interface contracts between Units)
- **No** — skip TEAM/MAGENT rules; run the workflow in single-agent mode (suitable for small projects with 1-2 Units, tightly coupled work, exploratory prototypes, or when running AI-DLC for the first time)

## Pre-conditions for Enabling

The user SHOULD be encouraged to enable Multi-Agent mode ONLY when ALL of the following hold. If any pre-condition is unmet, present the model's recommendation alongside the question:

- The project decomposes naturally into **3 or more Units** (verified after Units Generation)
- Each Unit has **its own owner team or owner sub-agent assignment** in the Units document
- Inter-Unit contracts (APIs, events, schemas) can be **explicitly defined before Construction starts**
- Each Unit has **independent test execution** (unit tests do not require artifacts from other Units)

If pre-conditions are not yet known (Units Generation has not run), record the answer as "Deferred" and re-confirm immediately after Units Generation completes. Update `## Extension Configuration` in `aidlc-docs/aidlc-state.md` accordingly.
