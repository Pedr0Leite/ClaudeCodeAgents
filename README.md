# ClaudeCodeAgents
A team of specialized Claude Code agents for end-to-end ServiceNow scoped application development — with built-in change governance.

![Agent Orchestration Pipeline](orchestration.svg)

---

## Agents

### Orchestrator
Master pipeline controller that commands the full development team in sequence. Receives client requirements, creates a workspace, and drives BA → Architect → Governance → Developer → Tester through to a passing test result. Handles fix loops automatically (up to 5 iterations), routing failures back to the correct agent based on failure type.

### BA Agent (Business Analyst)
Transforms raw client requirements (free text, bullet points, meeting notes) into structured `rm_story` records grounded in official ServiceNow documentation. Consults the ServiceNowDocs repo via an index before writing stories, identifies ambiguities, and refines output iteratively.

### Architect
Translates rm_stories into a precise technical design and actionable developer instructions. Produces a full `architecture.md` (components, build order, scoped app rules, risks) and a structured `test-plan.md` that traces every test back to an acceptance criterion. In fix loops, revises only the affected sections.

### Governance Gate
Change control checkpoint between Architect and Developer. Reads the architecture plan without touching ServiceNow, validates that the active update set is correct, checks that no component uses Global scope unless explicitly approved, and lists every cross-scope call for user acknowledgement. Produces a `change-manifest.md` (a full preview of every planned write) and requests an explicit human YES before the Developer is allowed to proceed. A single unapproved Global usage or wrong update set blocks the pipeline entirely.

### Developer
Builds every component in ServiceNow following the Architect's instructions exactly and in dependency order. Reads `governance-approval.md` before doing anything — stops immediately if it is not APPROVED. Routes each component type to the correct dispatcher skill (Business Rules, Client Scripts, Flows, ACLs, etc.), enforces scoped app prefixing, and logs all results to `dev-log.md`. Never deploys — that step is human-controlled.

**Dependency: [ponytail](https://github.com/DietrichGebert/ponytail)**
The Developer agent uses ponytail to enforce a "laziest senior dev" mindset — preferring OOTB platform capabilities, existing APIs, and native flows over custom code. The best code is the code you never wrote.

### Tester
Independent QA gate that validates the built solution against the original requirements. Executes the Architect's test plan, cross-checks the dev log for skipped or failed components, produces a requirements-coverage matrix, and outputs a final PASS/FAIL verdict with classified failures to route back into the fix loop.

### Dispatcher
General-purpose entry point for any ServiceNow or full-stack development task. Loads a skill index of 185+ skills at startup, classifies the request by domain, and routes to the correct skill automatically. Covers ITSM, CSM, HRSD, development, GenAI, admin, security, GRC, catalog, CMDB, and general coding (JS, Python, React, Node).

### Bug Hunter
Independent code auditor invoked after a passing test run (or standalone). Scans business rules, client scripts, script includes, and flows for concrete, doc-backed defects — N+1 queries, broken async/`gs.getUser()` usage, scope violations, insecure ACLs, deprecated APIs — cross-referenced against ServiceNowDocs and now-sdk, never style or opinion. Writes a severity-ranked `BUGS.md` and stops; it never fixes issues or invokes other agents itself. CRITICAL/HIGH findings route back to the Developer for a fix loop; MEDIUM/LOW are advisory only.

---

## Rule of thumb
- Small task → **Dispatcher**
- Full feature → **Orchestrator**
- Defect scan on existing code, independent of a test plan → **Bug Hunter** directly

---

## Pipeline

```
Requirements → BA → Architect → Governance Gate → Developer → Tester → Bug Hunter → Ready to deploy
                        ↑              |    ↑                             |              |
                        |     REJECTED/BLOCKED                    (fix loop, No)     (fix loop,
                        |              |    |                              |          Crit/High)
                        └──────────────┘    └───────────── Architect / Developer ◄─────┘
```

The Governance Gate is the read/write boundary. Everything before it is planning. Everything after it writes to ServiceNow. No write reaches the platform without a human YES. Bug Hunter runs after a PASS as a final defect scan — CRITICAL/HIGH findings loop back to the Developer before deployment; MEDIUM/LOW are reported but don't block.

See [`orchestration.svg`](orchestration.svg) above for the full visual flow, including the Governance approval/rejection path and the shared supporting resources available to every agent.

---

## Invocation

### Start a full development pipeline
```bash
claude --agent orchestrator "build a proactive case communication feature for CSM"
```

### Quick one-off task
```bash
claude --agent dispatcher "fix business rule on incident table"
```

### Inside a Claude Code session
```
/agent orchestrator
```
Then describe your requirement when prompted.

---

## Agent structure
```
~/.claude/agents/
  orchestrator.md   ← commands everyone
  ba-agent.md
  architect.md
  governance.md     ← change control gate
  developer.md      ← requires ponytail; blocked without governance approval
  tester.md
  bug-hunter.md     ← final defect scan, runs after a PASS
  dispatcher.md
```

---

## Workspace

Each pipeline run creates a workspace with handoff artifacts:

```
~/.claude/workspace/[project-slug]/
  requirements.md          ← raw client input
  stories.md               ← BA output (rm_stories)
  architecture.md          ← Architect design + dev instructions
  test-plan.md             ← Architect test plan
  change-manifest.md       ← Governance: full preview of planned writes
  governance-approval.md   ← Governance: approval token read by Developer
  dev-log.md               ← Developer build log
  test-results.md          ← Tester results
  BUGS.md                  ← Bug Hunter findings (optional, on-demand)
  status.md                ← Current pipeline state
```

---

## Supporting Resources

Every ServiceNow agent (BA, Architect, Governance, Developer, Tester, Bug Hunter, Dispatcher) can reach for these when they add real signal — used sparingly, not queried on every step, to keep token spend low:

| Resource | Skill | Use for |
|---|---|---|
| **Fluent / now-sdk** | `servicenow-sdk:now-sdk` | Fluent syntax, SDK types, live instance lookups (sys_id, schema, choices, roles) |
| **second-brain** | `second-brain` | Prior notes, decisions, or known issues on the current topic before re-deriving them |
| **obsidian-cli** | `obsidian-cli` | Pulling relevant specs/notes from the user's vault, when available |

These are optional lookups, not required steps — each agent's instructions cap it at roughly one lookup per tool per task. The Orchestrator itself doesn't use these directly since it only delegates to the agents above.

---

## Governance Model

The Governance Gate enforces three rules before any write reaches ServiceNow:

| Rule | Behaviour |
|---|---|
| Update set must be active and `In Progress` | Pipeline BLOCKED if wrong or missing |
| Global scope is never the default | Any Global usage requires explicit user approval |
| Cross-scope calls must be listed | User must acknowledge them before proceeding |

Governance outcomes:

| Outcome | Next step |
|---|---|
| **APPROVED** | Developer proceeds |
| **REJECTED** | User's change request sent back to Architect |
| **BLOCKED** | Pipeline stops; violations must be resolved before re-running |

---

## Dependencies

| Agent | Dependency | Purpose |
|---|---|---|
| BA + Architect + Developer | [ServiceNowDocs](https://github.com/ServiceNow/ServiceNowDocs) | Official platform docs via `search_docs` MCP tool |
| Developer | [ponytail](https://github.com/DietrichGebert/ponytail) | Enforces OOTB-first, minimal custom code approach |
| All agents (optional) | `servicenow-sdk:now-sdk`, `second-brain`, `obsidian-cli` skills | See [Supporting Resources](#supporting-resources) — used sparingly, no setup required beyond having the skills installed |

### Setup

**ServiceNowDocs**
```bash
git clone https://github.com/ServiceNow/ServiceNowDocs.git
export SERVICENOW_DOCS_PATH=/path/to/ServiceNowDocs
```

**ponytail**
Follow install instructions at https://github.com/DietrichGebert/ponytail

---

## Tool Priority

All agents that interact with ServiceNow **must** follow this order:

| Priority | Tool | When |
|---|---|---|
| 1st | **ServiceNow MCP server** (`/mcp`) | Always — use first |
| 2nd | **REST API** | Only if MCP is unavailable or unsupported |
| 3rd | **Manual / scripted** | Last resort only |

The Developer agent enforces this on every build step. MCP unavailability is logged in `dev-log.md`.
