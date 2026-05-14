# Multi-Agent Mode

## Overview
Multi-Agent Mode is an AI-DLC execution model in which a single **Orchestrator** agent runs the Inception Phase alone, then launches one **Sub-Agent** per Unit of Work to execute Construction Phase stages in parallel. The Orchestrator owns workflow state and inter-Unit coordination; Sub-Agents own per-Unit implementation and return structured completion contracts.

This extension defines two rule families:
- **TEAM-01 through TEAM-08**: cross-cutting constraints that make multi-team / multi-agent execution safe (Unit ownership, contract-first interfaces, sync gates, audit conventions).
- **MAGENT-01 through MAGENT-06**: rules specific to Orchestrator / Sub-Agent coordination (role separation, briefings, isolation, completion contracts, communication protocol).

**Enforcement**: When this extension is enabled (`## Extension Configuration` in `aidlc-docs/aidlc-state.md` shows `Multi-Agent Mode | Yes`), every TEAM and MAGENT rule below is a hard constraint. At each applicable stage, the model MUST verify compliance before presenting the stage completion message to the user.

### Blocking Finding Behavior
A **blocking finding** means:
1. The finding MUST be listed in the stage completion message under a "Multi-Agent Findings" section with the rule ID (TEAM-NN or MAGENT-NN) and description
2. The stage MUST NOT present the "Continue to Next Stage" option until all blocking findings are resolved
3. The model MUST present only the "Request Changes" option with a clear explanation of what needs to change
4. The finding MUST be logged in `aidlc-docs/audit.md` with the rule ID, description, and stage context

If a rule is not applicable to the current stage or Unit (e.g., MAGENT-04 worktree isolation when only one Sub-Agent is active at a time, or TEAM-08 interface propagation when no interfaces have changed), mark it as **N/A** in the compliance summary — this is not a blocking finding.

### Pre-conditions for Enabling
Before this extension is enabled, the model MUST verify all of the following (record results in `aidlc-docs/audit.md`):
- Units Generation has produced **3 or more Units of Work** (or the user explicitly confirms running Multi-Agent mode with fewer Units)
- Each Unit has an **assigned owner** (team name or sub-agent identifier) in `aidlc-docs/inception/units/unit-of-work.md` or equivalent
- Inter-Unit contracts (APIs, events, schemas) are **explicitly defined** in Application Design or Functional Design artifacts before any Sub-Agent is launched
- Per-Unit unit tests do **not require artifacts** from other Units to execute

If any pre-condition fails, present a blocking finding and require the user to either resolve it or disable Multi-Agent Mode for this project.

---

# Part 1 — TEAM Rules (Cross-Cutting Coordination Constraints)

These rules apply whenever multiple Units of Work are developed concurrently, whether by human teams or by Sub-Agents. They are the foundation that MAGENT rules build on.

## Rule TEAM-01: Unit Ownership Assignment

**Rule**: Every Unit of Work MUST have exactly one assigned owner (a team name OR a Sub-Agent identifier). Ownership is recorded in the Units of Work document and is referenced in every Sub-Agent briefing.

**Verification**:
- `aidlc-docs/inception/units/unit-of-work.md` (or equivalent) lists every Unit with an `Owner:` field
- No Unit has two owners or no owner
- Sub-Agent identifiers follow a consistent naming convention (e.g., `sub-agent-unit-NN`)

---

## Rule TEAM-02: Data Store Exclusive Write Access

**Rule**: Each persistent data store (database table, S3 bucket prefix, queue, etc.) MUST be writable by **at most one Unit**. Read access may be shared via well-defined interfaces, but cross-Unit writes MUST go through the owning Unit's API.

**Verification**:
- Infrastructure Design (per-Unit) declares which data stores the Unit writes to
- No data store appears in the write list of more than one Unit
- Cross-Unit data modification flows are routed through APIs or events, not direct writes

---

## Rule TEAM-03: Contract-First Interfaces

**Rule**: Every inter-Unit interface (REST/HTTP endpoint, GraphQL field, gRPC method, event schema, message topic) MUST be defined as a contract **before** any Sub-Agent begins implementation. Contracts are stored in Application Design or Functional Design artifacts and referenced by every dependent Unit's briefing.

**Verification**:
- All cross-Unit interfaces are listed in `aidlc-docs/inception/application-design/` or the producing Unit's Functional Design
- Each contract specifies: name, direction (producer/consumer), payload schema, error responses, SLA where relevant
- No Sub-Agent briefing references an interface that has no documented contract

---

## Rule TEAM-04: Independent Unit Test Execution

**Rule**: A Unit's unit tests MUST be runnable using only that Unit's source and test doubles / contract stubs for dependencies. They MUST NOT require live artifacts from other Units.

**Verification**:
- Each Unit's test suite executes successfully without artifacts from sibling Units
- Cross-Unit dependencies are stubbed using contract definitions (TEAM-03)
- Integration / contract tests that exercise multiple Units are explicitly labelled and scheduled in Build and Test, not in per-Unit unit tests

---

## Rule TEAM-05: Per-Unit Audit Log Prefix

**Rule**: `aidlc-docs/audit.md` is shared across Units. Every entry produced by or about a specific Unit MUST be prefixed with `## [Unit-<id>] <Stage> <Event>` to prevent ambiguity and merge conflicts when concurrent runs append in sequence.

**Verification**:
- Every audit entry that pertains to a single Unit begins with the `[Unit-<id>]` prefix
- Orchestrator-level entries (cross-cutting decisions, gate evaluations) use the prefix `## [Orchestrator]`
- The Orchestrator appends entries serially; Sub-Agents do not write `audit.md` directly (see MAGENT-03)

---

## Rule TEAM-06: Per-Stage Unit Test Execution

**Rule**: At the end of every Construction stage that produces code for a Unit (Code Generation, refactoring, bug fix), unit tests for that Unit MUST be executed and pass before the stage is reported complete. A skipped or failing test is a blocking finding.

**Verification**:
- Stage completion messages include `Unit Test Status: PASS (N/N)` for the Unit's tests
- No Code Generation stage is reported complete with failing tests
- The Orchestrator does not advance to the next stage until the Sub-Agent's completion contract reports `unit_test_status: "PASS"`

---

## Rule TEAM-07: Sync Gate Before Integration

**Rule**: Build and Test stage MUST NOT begin integration / E2E test execution until **every Unit** has reported a passing completion contract for its final Construction stage. This is the **Sync Gate**. The Orchestrator evaluates the gate; if any Unit is incomplete or failing, integration testing is blocked.

**Verification**:
- Build and Test stage opens with a Sync Gate evaluation step: list every Unit and its latest completion contract status
- If any Unit reports `unit_test_status != "PASS"` or has no completion contract, present a blocking finding
- The Sync Gate evaluation is logged in `audit.md` under `## [Orchestrator] Sync Gate Evaluation`

---

## Rule TEAM-08: Interface Change Propagation

**Rule**: When a Sub-Agent modifies an interface defined under TEAM-03, the Orchestrator MUST update every dependent Unit's briefing to reflect the change **before** that dependent Sub-Agent's next stage runs. Sub-Agents never modify sibling Units' contracts directly.

**Verification**:
- Completion contracts that report `interface_changes` are processed by the Orchestrator: the Orchestrator updates Application Design and the dependent Sub-Agent briefings
- A dependent Sub-Agent is not relaunched until its briefing reflects the latest contract
- Interface change events are recorded in `audit.md` under `## [Orchestrator] Interface Propagation`

---

# Part 2 — MAGENT Rules (Orchestrator / Sub-Agent Protocol)

These rules govern the Orchestrator + Sub-Agent execution model. They presume the TEAM rules above are enforced.

## Rule MAGENT-01: Orchestrator Role Definition

**Rule**: Exactly one Orchestrator MUST be designated for the workflow. The Orchestrator:
- Executes **all Inception Phase stages** (Workspace Detection through Units Generation) directly
- Executes **Build and Test** and **Operations** stages directly
- Does **NOT** execute Construction Phase per-Unit stages directly; instead launches Sub-Agents (see MAGENT-04)
- Owns all writes to `aidlc-docs/aidlc-state.md` (see MAGENT-03)

The Orchestrator identity is recorded in `aidlc-docs/aidlc-state.md` under `## Multi-Agent Roster`.

**Verification**:
- `aidlc-state.md` contains a `## Multi-Agent Roster` section listing the Orchestrator and all Sub-Agents
- Construction Phase per-Unit stages show evidence of Sub-Agent execution (completion contracts logged in audit.md), not Orchestrator execution
- Inception, Build and Test, and Operations entries in audit.md use the `[Orchestrator]` prefix

---

## Rule MAGENT-02: Unit Assignment Briefing

**Rule**: Before launching a Sub-Agent for a Unit, the Orchestrator MUST produce a complete briefing covering:

1. **Unit identity**: Unit ID, name, owner
2. **Owned resources**: data stores, services, code paths
3. **Incoming interfaces**: every external contract this Unit consumes (from TEAM-03)
4. **Outgoing interfaces**: every contract this Unit produces, with consumer list
5. **Assigned User Stories or Functional Requirements**: explicit IDs from upstream artifacts
6. **NFR scope**: which non-functional requirements apply to this Unit
7. **Acceptance criteria for completion**: what the Sub-Agent must prove before returning a completion contract

The briefing is the **only** information the Sub-Agent has about sibling Units. Sub-Agents MUST NOT read sibling Units' source or contracts directly (see MAGENT-04, MAGENT-06).

A canonical briefing template appears at the end of this document (Appendix A).

**Verification**:
- Every Sub-Agent launch in audit.md references a stored briefing (path or inline content)
- Briefings include all seven sections above
- Sibling Unit source / contracts are not referenced in any Sub-Agent's prompt beyond what the briefing exposes

---

## Rule MAGENT-03: Shared Resource Write Control

**Rule**: Concurrent writes to shared documentation files are forbidden. Write ownership:

| File | Writable by | Notes |
|---|---|---|
| `aidlc-docs/aidlc-state.md` | **Orchestrator only** | Stage transitions, Extension Configuration, Sub-Agent roster, completion contracts table |
| `aidlc-docs/audit.md` | **Orchestrator only** | Sub-Agents return audit entries via completion contract; Orchestrator appends them with `[Unit-<id>]` prefix per TEAM-05 |
| `aidlc-docs/inception/**/*.md` | **Orchestrator only** (frozen after approval) | Inception artifacts are produced once and treated as read-only contracts during Construction |
| `aidlc-docs/construction/<unit-id>/**/*.md` | Owning Sub-Agent (and Orchestrator on update) | Per-Unit design and code summaries |
| Application source files under the owning Unit's path | Owning Sub-Agent | One Sub-Agent per Unit; see TEAM-02 for data stores |

**Verification**:
- Sub-Agent prompts do not include write permission to `aidlc-state.md` or `audit.md`
- All `aidlc-state.md` and `audit.md` writes in the session trace back to the Orchestrator
- No two Sub-Agents have overlapping source-file write scopes

---

## Rule MAGENT-04: Sub-Agent Scope Isolation

**Rule**: Each Sub-Agent MUST run with scope isolation, with worktree isolation required only when multiple Sub-Agents are launched concurrently:

- **Parallel launch (2 or more Sub-Agents launched in the same message or running concurrently)**: the Agent tool MUST be invoked with `isolation: "worktree"` so each Sub-Agent operates on its own git worktree, preventing concurrent file conflicts. This is a hard requirement — without worktree isolation, parallel Sub-Agents can corrupt shared files.
- **Sequential or single-Sub-Agent launch (only one Sub-Agent active at a time)**: `isolation: "worktree"` SHOULD be used when available, but is not strictly required. When worktree isolation is not used, the Orchestrator MUST instead enforce scope discipline via the briefing: explicit write paths, no shared file overlap, and a clear "do not touch outside these paths" instruction.
- **In all cases**: the Sub-Agent's tool scope SHOULD be restricted to what is required for its Unit (no broad shell or repo-wide write access beyond the listed paths), and the Sub-Agent's prompt MUST NOT include unrelated Units' source or contracts beyond what the briefing exposes.

**Rationale**: Parallel write conflicts are physical and cannot be prevented by prompt-level instructions alone. When only one Sub-Agent is active at a time, prompt-level scope restriction is sufficient and avoids worktree overhead in environments where worktrees are unavailable (e.g., CI sandboxes, container runtimes without write access to `.git/worktrees/`).

**Verification**:
- For each Sub-Agent launch in audit.md, the entry records either `isolation: "worktree"` OR `isolation: "scoped-prompt"` with the explicit write paths the Sub-Agent was permitted to touch
- Audit entries for parallel launches (2+ Sub-Agents launched in the same Orchestrator message or active simultaneously) MUST record `isolation: "worktree"` — anything else is a blocking finding
- Sub-Agent prompts do not embed sibling Units' source files
- On Sub-Agent completion under worktree isolation, the Orchestrator merges the worktree's changes into the main working tree under the Orchestrator's control. Under scoped-prompt isolation, the Orchestrator verifies that the Sub-Agent's modified files fall entirely within the permitted write paths.

---

## Rule MAGENT-05: Sub-Agent Completion Contract

**Rule**: A Sub-Agent's final response MUST be a structured completion contract conforming to the schema in Appendix B. Free-text responses are not acceptable. The Orchestrator parses the contract, validates required fields, and refuses to advance the workflow if any field is missing or malformed.

**Required fields**:
- `unit_id`, `unit_name`, `stage`, `unit_test_status`, `test_summary`, `interface_changes`, `audit_entry`, `next_blockers`

**Verification**:
- Every Sub-Agent's final message contains a parseable JSON block matching Appendix B
- Missing or malformed contracts trigger a Sub-Agent re-launch with corrective briefing
- The contract is recorded verbatim under `## [Unit-<id>] Completion Contract` in audit.md

---

## Rule MAGENT-06: Inter-Agent Communication Protocol

**Rule**: Sub-Agents MUST NOT communicate with each other directly. All inter-Agent communication flows through the Orchestrator:

```
  Sub-Agent A ──completion_contract──▶ Orchestrator
                                          │
                                          │ (interface change reflected
                                          │  in updated briefing)
                                          ▼
                                       Sub-Agent B (relaunched or messaged)
```

Allowed communication channels:
- Sub-Agent → Orchestrator: completion contract (final message) or status update via SendMessage (if multi-turn briefings are used)
- Orchestrator → Sub-Agent: launch with full briefing, or SendMessage with briefing update
- Orchestrator → Orchestrator: not applicable (single Orchestrator)
- Sub-Agent → Sub-Agent: **forbidden**

**Verification**:
- No Sub-Agent's prompt references another Sub-Agent by name
- No tool call in a Sub-Agent's trace targets another Sub-Agent's worktree or output
- Interface propagation (TEAM-08) is mediated by the Orchestrator, not by direct Sub-Agent coordination

---

# Part 3 — Execution Model for Construction Phase

When this extension is enabled, the Orchestrator follows the procedure below in place of the default Construction Phase Per-Unit Loop. The default loop runs each Unit's stages sequentially in the Orchestrator's own context; this procedure delegates per-Unit work to Sub-Agents and runs independent Units in parallel.

### Step 1 — Build the Dependency Graph
Read `aidlc-docs/inception/units/unit-of-work-dependency.md` (or equivalent). Produce a topological order; identify which Units have no unsatisfied dependencies and can be launched first.

### Step 2 — Prepare Briefings
For each ready Unit, produce a briefing per MAGENT-02 using the Appendix A template. Save each briefing under `aidlc-docs/construction/<unit-id>/briefing.md` for traceability.

### Step 3 — Launch Sub-Agents in Parallel
For all Units that are simultaneously ready (no inter-dependencies between them), launch their Sub-Agents in a single message containing multiple `Agent` tool calls. When 2 or more Sub-Agents are launched in the same message or otherwise run concurrently, each launch MUST set `isolation: "worktree"` (MAGENT-04). When the dependency graph reduces to a single ready Unit (sequential launch), worktree isolation is recommended but not required — fall back to scoped-prompt isolation by listing the Sub-Agent's permitted write paths explicitly in the briefing. Pass each briefing as the `prompt`. Suggested `subagent_type`: `general-purpose` unless a more specific type matches the Unit's character.

### Step 4 — Collect Completion Contracts
When each Sub-Agent returns, parse its completion contract per MAGENT-05. If the contract is malformed, re-launch the Sub-Agent with a corrective briefing. If `unit_test_status != "PASS"`, log the failure under `[Unit-<id>] Construction Failure` and follow Step 5.

### Step 5 — Propagate Interface Changes (TEAM-08)
For each completion contract containing `interface_changes`:
1. Update `aidlc-docs/inception/application-design/` (or per-Unit Functional Design) to reflect the new contract version
2. For every dependent Unit listed in `affected_consumers`, regenerate that Unit's briefing
3. If a dependent Sub-Agent has already started, send the updated briefing via `SendMessage`; if not yet launched, the briefing will be current on launch

### Step 6 — Advance the Dependency Graph
Mark completed Units. For Units whose dependencies are now satisfied, return to Step 2.

### Step 7 — Sync Gate (TEAM-07)
When all Units report `unit_test_status: "PASS"` and there are no pending dependencies, proceed to Build and Test. Open Build and Test with a Sync Gate Evaluation step listing every Unit's final completion contract. Log the evaluation under `[Orchestrator] Sync Gate Evaluation` in audit.md. If any Unit fails the gate, return to Step 3 with corrective briefings.

---

# Part 4 — `aidlc-state.md` Sections for Multi-Agent Mode

When this extension is enabled, the Orchestrator MAY extend `aidlc-state.md` with the following sections (in addition to the standard Extension Configuration table):

```markdown
## Multi-Agent Roster
- **Orchestrator**: <agent-id or "session-default">
- **Sub-Agents**:
  - `sub-agent-unit-01` → Unit-01 (Owner: <team-name>)
  - `sub-agent-unit-02` → Unit-02 (Owner: <team-name>)
  - ...

## Sub-Agent Completion Contracts
| Unit | Stage | Status | Timestamp | Completion Contract |
|---|---|---|---|---|
| Unit-01 | Code Generation | PASS | 2026-MM-DDTHH:MM:SSZ | audit.md#unit-01-code-completion |
| Unit-02 | Code Generation | PASS | 2026-MM-DDTHH:MM:SSZ | audit.md#unit-02-code-completion |

## Sync Gate Status
- **Last Evaluated**: 2026-MM-DDTHH:MM:SSZ
- **Result**: PASS / BLOCKED (reason)
- **Pending Units**: <list>
```

These sections are optional — they exist to give the Orchestrator a quick visual roster. The authoritative record remains `audit.md`.

---

# Appendix A — Sub-Agent Briefing Template

Save each briefing under `aidlc-docs/construction/<unit-id>/briefing.md` and embed it (or reference it) in the Sub-Agent's launch prompt.

```markdown
# Sub-Agent Briefing — Unit-<id>

## Unit Identity
- **Unit ID**: Unit-<id>
- **Unit Name**: <name>
- **Owner**: <team-name or sub-agent-id>
- **Stage to Execute**: <Functional Design | NFR Requirements | NFR Design | Infrastructure Design | Code Generation>

## Owned Resources
- **Source paths**: <relative paths the Sub-Agent may write to>
- **Data stores** (write): <data store IDs, per TEAM-02>
- **Data stores** (read-only): <data store IDs accessed via interfaces>

## Incoming Interfaces (this Unit consumes)
| Interface | Producer Unit | Contract Reference | Notes |
|---|---|---|---|
| <name> | Unit-<id> | <doc path / section> | <e.g., latest version, deprecation notes> |

## Outgoing Interfaces (this Unit produces)
| Interface | Consumer Units | Contract Reference | Notes |
|---|---|---|---|
| <name> | Unit-<id>, Unit-<id> | <doc path / section> | <SLA / version> |

## Assigned User Stories / Functional Requirements
- <Story ID> — <one-line summary>
- <Story ID> — <one-line summary>

## NFR Scope
- <NFR ID> — <e.g., latency target, availability target>

## Acceptance Criteria for Completion Contract
- All Assigned Stories implemented with passing unit tests
- All Incoming Interface consumers updated to latest contract version
- No new interfaces introduced without recording them in `interface_changes`
- Coverage / quality thresholds: <project-specific>

## Reference Reading (read-only, for context)
- <path to Functional Design for this Unit>
- <path to Application Design — interfaces section>
- <path to relevant NFR design>

## Active Extensions
Extensions currently enabled for this workflow (mirrored from `aidlc-state.md` → `## Extension Configuration`). The Sub-Agent MUST apply every rule in every listed extension to its owned source, and report compliance per applicable rule in the `extension_compliance` field of the completion contract (Appendix B).

| Extension | Enabled | Rule Families to Apply |
|---|---|---|
| Security Baseline | <Yes / No> | SECURITY-01 .. SECURITY-NN |
| Property-Based Testing | <Yes / No> | PBT-01 .. PBT-NN |
| Multi-Agent Mode | Yes | TEAM-01 .. TEAM-08, MAGENT-01 .. MAGENT-06 |
| <other extensions> | <Yes / No> | <rule IDs> |

If a rule does not apply to this Unit's stage or scope, the Sub-Agent MUST still mention it in `extension_compliance` with `status: "n/a"` and a brief rationale.

## Constraints
- Tool scope: write only under the source paths listed above; do not modify `aidlc-state.md` or `audit.md`
- Isolation: this Sub-Agent runs with the isolation mode specified by the Orchestrator at launch (`isolation: "worktree"` when launched in parallel with other Sub-Agents; otherwise scoped-prompt isolation with the write paths listed above as the hard boundary)
- No communication with sibling Sub-Agents (MAGENT-06)
```

---

# Appendix B — Completion Contract Schema

Sub-Agents return a JSON block matching this schema as the final element of their response. The schema is validated by the Orchestrator before any state transition.

```json
{
  "unit_id": "Unit-01",
  "unit_name": "Order Service",
  "stage": "Code Generation",
  "unit_test_status": "PASS",
  "test_summary": "42/42 passed; 3 skipped (marked TODO with rationale)",
  "code_changes": [
    {
      "path": "src/order-service/handler.ts",
      "change_type": "added | modified | deleted",
      "summary": "Implemented POST /orders endpoint with validation"
    }
  ],
  "interface_changes": [
    {
      "interface_name": "PaymentAPI",
      "change_type": "new_endpoint | breaking_modification | non_breaking_modification | deprecation",
      "description": "Added POST /v2/process-payment with optional idempotency key",
      "affected_consumers": ["Unit-02"]
    }
  ],
  "nfr_compliance": [
    { "nfr_id": "NFR-LATENCY-01", "status": "compliant", "evidence": "p95 latency 120ms in load test" }
  ],
  "extension_compliance": [
    { "rule_id": "SECURITY-05", "status": "compliant | non-compliant | n/a", "rationale": "Input validation via Zod schema on all endpoints" }
  ],
  "audit_entry": "## [Unit-01] Code Generation Completion\n- Timestamp: 2026-MM-DDTHH:MM:SSZ\n- Status: PASS\n- Stories: US-10, US-11\n- ...",
  "next_blockers": [
    {
      "blocker_type": "interface_propagation | dependency_not_ready | nfr_review",
      "description": "Unit-02 must update to PaymentAPI v2 before integration testing"
    }
  ]
}
```

**Field semantics**:
- `unit_test_status`: `"PASS"` only if every unit test in this Unit's suite passes; `"FAIL"` otherwise. No third value.
- `interface_changes`: omit or use empty array if no interfaces changed. Every entry triggers TEAM-08 propagation.
- `extension_compliance`: REQUIRED if other extensions (security, PBT, etc.) are enabled. List each applicable rule with its status.
- `audit_entry`: Markdown content that the Orchestrator appends verbatim under the `[Unit-<id>]` section of `audit.md`.
- `next_blockers`: any Unit-external work the Orchestrator must do before this Unit's next stage can run.

---

# Enforcement Integration

When this extension is enabled:
1. The Orchestrator checks the Extension Configuration entry in `aidlc-state.md` at the start of each stage and applies the rules below.
2. During **Inception**, only MAGENT-01's role-recording requirement applies; the Orchestrator records itself in `## Multi-Agent Roster` and proceeds normally.
3. During **Construction Phase Per-Unit Loop**, the Orchestrator follows the Part 3 execution model instead of running the loop in its own context.
4. During **Build and Test**, TEAM-07 Sync Gate is evaluated first; integration testing is blocked until all completion contracts pass.
5. At every stage completion, include a **"Multi-Agent Findings"** section in the completion summary listing each applicable TEAM and MAGENT rule as compliant, non-compliant, or N/A.

---

# Appendix C — Worked Mini-Example (Two Units, One Interface Change)

Illustrative trace for a project with `Unit-01: Payment` and `Unit-02: Order` where `Unit-02` consumes `PaymentAPI`.

1. **Orchestrator** completes Inception, produces `unit-of-work.md` listing Unit-01 and Unit-02 with the `PaymentAPI` contract documented.
2. **Orchestrator** writes briefings for both Units (no inter-Unit blocker yet — Unit-02 uses the documented contract).
3. **Orchestrator** launches `sub-agent-unit-01` and `sub-agent-unit-02` in parallel with `isolation: "worktree"`.
4. **sub-agent-unit-01** finishes Code Generation. Completion contract reports:
   - `unit_test_status: "PASS"`
   - `interface_changes: [{ interface_name: "PaymentAPI", change_type: "new_endpoint", description: "POST /v2/process-payment", affected_consumers: ["Unit-02"] }]`
5. **Orchestrator** (per TEAM-08):
   - Updates Application Design — PaymentAPI section — to v2
   - Regenerates Unit-02's briefing to reflect PaymentAPI v2
   - Sends the new briefing to `sub-agent-unit-02` via `SendMessage`
6. **sub-agent-unit-02** finishes Code Generation against PaymentAPI v2; completion contract reports `unit_test_status: "PASS"`, `interface_changes: []`.
7. **Orchestrator** evaluates the Sync Gate: both Units PASS. Build and Test begins.
8. **Orchestrator** runs integration tests; failures (if any) are routed back to the responsible Unit's Sub-Agent with a corrective briefing.

---

## Appendix D — Compatibility with Other Extensions

The Orchestrator is responsible for propagating every enabled extension into each Sub-Agent's briefing via the `## Active Extensions` section in the Appendix A template. Sub-Agents read that section and apply the listed rules to their owned source; their completion contracts MUST report `extension_compliance` for every applicable rule.

- **Security Baseline**: When `security-baseline` is enabled, Sub-Agents apply every SECURITY rule to their owned source. Each completion contract's `extension_compliance` array MUST include every applicable SECURITY rule with status (`compliant` / `non-compliant` / `n/a`).
- **Property-Based Testing**: When `property-based-testing` is enabled, Sub-Agents apply PBT rules to their owned source; completion contracts report PBT compliance per applicable rule.
- **Future extensions**: any new opt-in extension is loaded by the Orchestrator at workflow start and reflected in the `## Extension Configuration` table of `aidlc-state.md`. The Orchestrator mirrors that table into the `## Active Extensions` section of every Sub-Agent briefing (see Appendix A) so Sub-Agents apply the same rule set. Adding a new extension does not require modifying this document — the Sub-Agent picks up the rules from its briefing.
