# ClaudeCodeAgents

A team of specialized Claude Code agents for end-to-end ServiceNow scoped application development.

---

## Agents

### Orchestrator
Master pipeline controller that commands the full development team in sequence. Receives client requirements, creates a workspace, and drives BA → Architect → Developer → Tester through to a passing test result. Handles fix loops automatically (up to 5 iterations), routing failures back to the correct agent based on failure type.

### BA Agent (Business Analyst)
Transforms raw client requirements (free text, bullet points, meeting notes) into structured `rm_story` records grounded in official ServiceNow documentation. Consults the ServiceNowDocs repo via an index before writing stories, identifies ambiguities, and refines output iteratively.

### Architect
Translates rm_stories into a precise technical design and actionable developer instructions. Produces a full `architecture.md` (components, build order, scoped app rules, risks) and a structured `test-plan.md` that traces every test back to an acceptance criterion. In fix loops, revises only the affected sections.

### Developer
Builds every component in ServiceNow following the Architect's instructions exactly and in dependency order. Routes each component type to the correct dispatcher skill (Business Rules, Client Scripts, Flows, ACLs, etc.), enforces scoped app prefixing, and logs all results to `dev-log.md`. Never deploys — that step is human-controlled.

### Tester
Independent QA gate that validates the built solution against the original requirements. Executes the Architect's test plan, cross-checks the dev log for skipped or failed components, produces a requirements-coverage matrix, and outputs a final PASS/FAIL verdict with classified failures to route back into the fix loop.

### Dispatcher
General-purpose entry point for any ServiceNow or full-stack development task. Loads a skill index of 185+ skills at startup, classifies the request by domain, and routes to the correct skill automatically. Covers ITSM, CSM, HRSD, development, GenAI, admin, security, GRC, catalog, CMDB, and general coding (JS, Python, React, Node).
