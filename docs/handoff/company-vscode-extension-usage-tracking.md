# Handoff: Implement usage tracking for the company VS Code extension

## Mission

Implement privacy-safe usage tracking for the company VS Code extension that exposes MCP tools, chat participants/skills, Language Model tools, and normal commands or scripts.

The first management question is **“Do developers actually use it?”** The first release must answer that reliably. It must not claim productivity impact from install counts, extension activation, token volume, or raw tool-call volume.

This document is an execution brief for an agent running on the company laptop, where the real private repository, telemetry infrastructure, identity system, and policies can be inspected.

## Context already established

Use one shared measurement model across every surface:

1. **Deployed** — a user or machine received the extension.
2. **Activated** — VS Code loaded the extension. This is operational data, not product engagement.
3. **Engaged** — a developer caused at least one meaningful workflow to run.
4. **Retained** — an engaged developer returned on distinct days or weeks.

The proposed primary KPI is **28-day engaged developers / eligible developers** after a durable pseudonymous person-level identity has been approved and validated. Until then, report **28-day engaged installations** and state the identity grain explicitly. Report retention, successful workflows, feature reach, errors, and latency alongside it.

Classify every operation by its initiator:

- `human_explicit`: an owned participant/UI flow proves the developer invoked the operation.
- `agent_selected`: host telemetry or owned orchestration proves the model selected the skill/tool.
- `system_auto`: activation, heartbeat, catalog refresh, health check, or other background work.
- `unknown`: the stable API reports execution but not its origin.

`agent_selected` is real extension usage, but it is not another human interaction. Attribute it to the parent workflow and count the developer once. A generic VS Code command, Language Model tool callback, or MCP `tools/call` does not prove its initiator; use `unknown` unless another trusted signal establishes the origin.

## Guardrails

- Default data is metadata only. Exclude prompts, responses, source code, file paths, repository URLs, command text, tool arguments, tool results, environment variables, tokens, and secrets.
- Telemetry failure must never break activation, chat, commands, tool calls, lazy startup, or extension shutdown.
- Keep automatic signals separate from engaged-user calculations.
- Treat `tools/list`, MCP definition discovery, activation, and heartbeat as availability signals. A real MCP use begins at `tools/call`.
- Respect the company privacy policy and the user's VS Code telemetry choice. If policy requires a different consent model, document the approved decision before implementation.
- Prefer a pseudonymous corporate user key produced with HMAC and an organization-controlled secret. Never transmit the raw employee identifier. If no approved identity exists, use a random installation ID and label user counts as estimates.
- One person can use several machines. Keep `user_key`, `installation_id`, and `session_id` as different concepts.
- Raw usage counters should not be sampled. High-volume diagnostic traces may be sampled after product counters are recorded.
- Copilot OTel can include repository remote, branch, commit, and organization metadata even while prompt-content capture is off. Drop every unapproved attribute at the collector with an allowlist processor.

## Phase 1 — Research the company environment

Read the repository instructions and architecture before editing. Locate every actual entry point; do not assume the company repo uses the same paths as the reference project.

Start with searches such as:

```sh
rg -n "createChatParticipant|chatParticipants|onDidReceiveFeedback|registerTool|languageModelTools|registerMcpServerDefinitionProvider|CallToolRequestSchema|ListToolsRequestSchema|registerCommand|sendRequest|selectChatModels|telemetry|OpenTelemetry|OTEL" .
```

Build an inventory with one row per surface:

| Surface | Registration point | Invocation handler | Human or agent initiated | Existing telemetry | Test seam |
| --- | --- | --- | --- | --- | --- |
| Chat participant | | | | | |
| Chat command | | | | | |
| Chat skill | | | | | |
| Language Model tool | | | | | |
| MCP tool | | | | | |
| Normal command/script | | | | | |
| Internal LLM call | | | | | |

Also determine:

- Minimum and deployed VS Code versions.
- How the extension is distributed and how many developers are eligible.
- Existing OTel collector, analytics backend, event SDK, data warehouse, and dashboards.
- Whether Copilot managed settings and OTel export are available in the managed fleet.
- Approved identity source and whether cross-machine user counting is required.
- Telemetry consent, retention, data residency, access-control, and deletion requirements.
- Whether chat skills are native Copilot/VS Code skills or custom application code.

Write the findings to `docs/usage-tracking/inventory.md`. Cite the exact code path, installed type definition, official document, or policy for every API and policy claim.

**Phase 1 is complete when every shipped surface has an invocation handler or is explicitly marked unobservable, and all policy/backend/identity unknowns have an owner.**

## Phase 2 — Confirm the collection boundary

Prefer the following order:

1. **Current Copilot Chat OTel export** for agent sessions, LLM calls, skill calls, and MCP/extension tool executions when the managed VS Code fleet supports it.
2. **Extension-owned instrumentation** for chat participants, commands/scripts, business outcomes, and any surface not visible in the host OTel stream.
3. **Gateway instrumentation** around MCP `tools/call` for a stable, extension-owned record of tool routing, result, latency, and upstream identity.

Do not duplicate the same logical event in host OTel and custom telemetry without a documented deduplication key. Use a shared `trace_id` or workflow ID only after verifying that the deployed VS Code and MCP SDK propagate trace context into the gateway. Otherwise keep the streams conceptually related without inventing an unreliable join.

If the extension follows the reference project's separation, keep VS Code imports in the extension layer. Pass a small telemetry interface into the gateway so the gateway remains testable without VS Code.

Record the decision in `docs/usage-tracking/architecture.md`, including:

- Chosen backend and exporter.
- Which source owns each metric.
- Deduplication strategy.
- Offline buffering, batching, retry, and shutdown behavior.
- Consent and identity behavior.
- Why rejected alternatives were rejected.

**Phase 2 is complete when every metric has exactly one source of truth and the privacy/security owner has approved the data fields.**

## Phase 3 — Define the event contract

Use a small stable contract instead of a separate ad-hoc event for every feature:

```json
{
  "event": "operation.completed",
  "schema_version": 1,
  "surface": "mcp | lm_tool | chat | skill | command",
  "feature_id": "stable.non-sensitive.name",
  "operation": "invoke",
  "initiator": "human_explicit | agent_selected | system_auto | unknown",
  "outcome": "succeeded | failed | cancelled",
  "duration_ms": 842,
  "error_code": "stable-code-or-omitted",
  "user_key": "approved-pseudonymous-id",
  "installation_id": "random-local-id",
  "session_id": "ephemeral-session-id",
  "trace_id": "workflow-correlation-id",
  "extension_version": "x.y.z"
}
```

Define a data catalog that lists every field, type, allowed values, purpose, sensitivity, retention, and owner. Prefer stable error codes over exception messages.

Minimum product events:

- `workflow.started`: one human-started workflow.
- `operation.completed`: command, skill, tool, or participant outcome.
- `feedback.submitted`: thumbs up/down or approved structured feedback.
- `extension.activated`: operational only.
- `catalog.observed`: optional operational snapshot for finding exposed-but-unused tools; never count it as engagement.

For LLM calls, record cost/health metadata only when approved: provider/model family, duration, outcome, input/output token counts, and rate-limit category. Keep LLM calls as children of the user workflow.

Write definitions and formulas to `docs/usage-tracking/measurement.md`.

**Phase 3 is complete when two independent readers calculate the same KPI from the definitions and no field can contain arbitrary user content.**

## Phase 4 — Implement tracer bullets with TDD

Implement one narrow vertical slice at a time:

1. Add a telemetry interface plus no-op and in-memory test implementations.
2. Instrument one normal command end to end.
3. Instrument one chat participant/skill path end to end.
4. Instrument MCP `tools/call` at the gateway boundary.
5. Add the production exporter, consent gate, batching, and retry.
6. Extend the proven wrapper to the remaining inventory rows.

For MCP, start timing as soon as the outward `tools/call` handler receives the request, before dispatch. Emit exactly one terminal event for Hub built-ins, resolved upstream tools, unresolved names, returned `isError`, thrown errors, cancellation, and timeout. Record the public tool ID and upstream ID when one exists, never arguments or result content.

For normal commands, wrap handlers owned by the extension. Do not depend on an undocumented global command observer, and label the initiator `unknown` unless an owned UI flow proves it.

For chat participants, count the request once at the handler boundary. Use the request's stable command/skill identifier. Attach feedback through the participant feedback event when available.

For Language Model tools, instrument the registered tool's `invoke` implementation. `prepareInvocation` is preparation, not proof of execution. For MCP tools, instrument the gateway call handler even if host OTel is also used for broader agent traces.

**Phase 4 is complete when all inventory rows use the shared contract and the full suite passes without changing normal extension behavior.**

## Required tests

- Activation, heartbeat, `tools/list`, and catalog refresh do not increase engaged users.
- One human chat request creates one workflow even when the model makes several LLM/tool calls.
- Every `tools/call` records exactly one terminal outcome.
- Returned MCP `isError`, thrown errors, cancellation, and timeout are distinguishable.
- Command and skill identifiers come from an allowlist; arbitrary input cannot become an event name or attribute key.
- Fixtures prove prompts, file paths, arguments, results, stack traces, and secrets are absent.
- Turning telemetry off stops new exports and clears or handles queued data according to policy.
- Export failure, slow collector, and offline mode do not block product operations.
- Startup remains lazy; telemetry must not spawn upstream processes.
- Shutdown flush is bounded and cannot hang extension deactivation.
- Schema changes are versioned and backward-compatible or migrated deliberately.
- The packaged VSIX contains the declared root `telemetry.json` event inventory.

Use unit tests for classification and privacy, gateway tests for MCP calls, and a real VS Code integration test for editor-facing consent and lifecycle behavior.

## Dashboard contract

The first management dashboard should show:

- Eligible developers.
- Deployed users/installations.
- 28-day engaged developers and engaged/eligible rate.
- Weekly retained developers.
- Successful workflows / total workflows.
- Engaged developers by surface: MCP, chat, skill, command/script.
- Top used and exposed-but-unused features.
- Failure rate and p50/p95 latency.
- Feedback rate and positive/negative split when the sample is large enough.

Keep tool calls, LLM calls, tokens, and active installations as diagnostic or depth metrics. Do not label them as unique users or productivity.

Before setting targets, collect a clean four-to-six-week baseline. Document any retention threshold, such as activity in two distinct weeks, as a product decision rather than an industry fact.

## Production acceptance criteria

- Management can answer “how many eligible developers used a meaningful feature in the last 28 days?” without mixing in automatic activity.
- Product owners can break that number down by surface and stable feature ID.
- Operations can see success rate, failure category, and p95 latency without viewing user content.
- A privacy reviewer can enumerate every exported field from the repository.
- A developer can disable telemetry as required by policy and verify the result locally.
- The extension works normally when the collector is unavailable.
- The implementation passes unit, gateway, typecheck, build, and real VS Code integration tests used by the company repository.

## Primary references to verify on the company laptop

- VS Code telemetry guide: <https://code.visualstudio.com/api/extension-guides/telemetry>
- VS Code API reference: <https://code.visualstudio.com/api/references/vscode-api>
- Chat Participant API: <https://code.visualstudio.com/api/extension-guides/ai/chat>
- Language Model Tool API: <https://code.visualstudio.com/api/extension-guides/ai/tools>
- Copilot agent monitoring with OpenTelemetry: <https://code.visualstudio.com/docs/agents/guides/monitoring-agents>
- MCP specification: <https://modelcontextprotocol.io/specification/>
- OTel GenAI semantic conventions: <https://opentelemetry.io/docs/specs/semconv/gen-ai/>

The public docs can be newer than the company's VS Code fleet. Confirm every API against the exact `@types/vscode` and editor version installed in the private repo before coding.

## Suggested skills

Use these Matt Pocock skills in this order on the company laptop:

If company policy permits installing external agent instructions, install the reviewed subset first with `npx skills@latest add mattpocock/skills`. If external installation is prohibited, copy only the approved skill files through the company's normal review process. The skills installed on the current laptop do not automatically appear on a separate company laptop.

1. `/setup-matt-pocock-skills` — configure the private repository's tracker and doc locations once.
2. `/research` — produce the cited environment/API inventory in Phase 1.
3. `/grill-with-docs` — resolve backend, identity, consent, retention, and KPI decisions; capture ADRs and shared terms.
4. `/to-spec` — turn the approved measurement and architecture docs into an implementation spec.
5. `/to-tickets` — split the work into tracer-bullet tickets with explicit dependencies.
6. `/implement` — execute the approved tickets.
7. `/tdd` — drive each instrumentation seam red-green-refactor.
8. `/code-review` — review both repository standards and fidelity to the approved spec before commit.

Use `/writing-for-agents` whenever editing `AGENTS.md`, `CLAUDE.md`, or documents linked from them.

## Current state

- No company implementation has been changed.
- The local public/reference project was inspected only to identify likely seams: extension activation, registered command handlers, MCP definition provider, and the gateway `tools/call` handler.
- A separate current-API research note may accompany this handoff as `docs/research/vscode-usage-tracking-api.md`; use it as evidence, then re-verify against the private repo's pinned versions.
- The remaining blockers can only be resolved on the company laptop: real code paths, deployed editor versions, telemetry backend, approved identity, privacy policy, and management's final KPI definitions.
