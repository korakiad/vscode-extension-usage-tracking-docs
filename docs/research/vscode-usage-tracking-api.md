# VS Code usage-tracking APIs for MCP Hub

Research date: 2026-08-17

Scope: official VS Code documentation and API definitions, the official Copilot monitoring documentation, the MCP specification, and OpenTelemetry documentation. This note records facts needed to plan tracking; it does not implement tracking.

## Short answer

Use two complementary data paths:

1. Instrument the extension and MCP Hub gateway for the operations that this code owns.
2. Export Copilot Chat OpenTelemetry (OTel) for the surrounding agent workflow.

Neither path is complete alone. Extension instrumentation can count MCP Hub commands and every outward `tools/call`, including calls from non-Copilot clients. Copilot OTel can connect agent sessions, LLM turns, skills, tool executions, edit outcomes, and feedback, but it does not expose a durable user identity suitable for a 28-day unique-user metric.

Do not count activation, `provideMcpServerDefinitions`, MCP initialization, `tools/list`, catalog refresh, or heartbeat traffic as engaged usage. The strongest common engagement event is a completed user-facing operation: participant request, command handler, Language Model tool invocation, or MCP `tools/call`.

## What the stable VS Code API can observe

| Surface | Stable observation point | What is available | Important limitation |
| --- | --- | --- | --- |
| Chat participant | The `ChatRequestHandler` passed to `chat.createChatParticipant` | Prompt, selected slash command (`request.command`), references, manually attached tool references, model, cancellation token | There is no stable global listener for requests handled by other participants and no stable request/session ID on `ChatRequest`. Do not record prompt or reference values. |
| Participant feedback | `ChatParticipant.onDidReceiveFeedback` | Helpful/unhelpful kind and the returned `ChatResult`, including JSON-safe metadata | Only feedback on this extension's participant. Most users do not vote, so missing feedback is not success. |
| Participant copy/insert/apply/follow-up actions | No publishable stable event | Copilot OTel reports aggregate user-action signals | `ChatParticipant.onDidPerformAction` exists only in the proposed `chatParticipantAdditions` API. Proposed APIs are unstable, Insiders-only, and should not be used in a published extension. |
| Language Model tool | The extension's `LanguageModelTool.invoke()` | Validated input, cancellation token, opaque tool-invocation token, tokenization hints | `invoke` does not identify whether the model selected the tool or the user explicitly attached it. `prepareInvocation` is not a usage event because it is not guaranteed to be followed by `invoke`. |
| MCP server definition | `provideMcpServerDefinitions` and `resolveMcpServerDefinition` | Server discovery/start lifecycle | The provider API has no tool-call event. It exposes only definitions, definition changes, and resolution before start. |
| MCP tool | The MCP server's `tools/call` handler | Public tool name, arguments, result, `isError`, duration, cancellation/exception | Stable MCP `CallToolRequest` has no human-vs-agent initiator field and no end-user identity. Do not retain arguments/results. |
| VS Code command | The callback registered with `commands.registerCommand` / `registerTextEditorCommand` | Command ID, outcome, duration, errors | A handler can be reached from the palette, keybinding, UI, or programmatically; the API does not include invocation origin. |
| Standalone script | No VS Code API after the script leaves the extension host | Whatever the script instruments itself | A shell command visible to Copilot OTel is not a reliable script-usage event, and the command string is content-capture-only. The script needs its own consent-aware wrapper/event if it must be counted. |

The stable API details above are visible in the current [VS Code API reference](https://code.visualstudio.com/api/references/vscode-api): `ChatRequest.command` identifies a selected participant slash command, `ChatParticipant.onDidReceiveFeedback` reports helpful/unhelpful feedback, `LanguageModelTool.invoke` is the actual tool execution point, and `McpServerDefinitionProvider` contains only definition lifecycle methods. The official [Chat Participant guide](https://code.visualstudio.com/api/extension-guides/ai/chat) explicitly recommends counting handled requests and unhelpful feedback. The official [Language Model Tool guide](https://code.visualstudio.com/api/extension-guides/ai/tools) confirms that `invoke` runs after input validation and that `prepareInvocation` is preparatory UI/confirmation work.

The tempting `onDidPerformAction` API for copy/insert/apply is in [`vscode.proposed.chatParticipantAdditions.d.ts`](https://github.com/microsoft/vscode/blob/main/src/vscode-dts/vscode.proposed.chatParticipantAdditions.d.ts), not stable `vscode.d.ts`. VS Code's [proposed API policy](https://code.visualstudio.com/api/advanced-topics/using-proposed-api) says proposed APIs can change, require Insiders, and should not be used in published extensions.

## Extension telemetry and privacy controls

For telemetry emitted by MCP Hub itself, create a `TelemetryLogger` with `vscode.env.createTelemetryLogger(sender)`, then use `logger.logUsage` and `logger.logError`; do not call the sender directly. The logger applies telemetry-level checks, cleans potentially sensitive data, exposes `isUsageEnabled` / `isErrorsEnabled`, and emits enablement changes. A sender may implement `flush()` for buffered delivery. These guarantees are part of the stable [`TelemetryLogger` API](https://code.visualstudio.com/api/references/vscode-api#TelemetryLogger).

Requirements from the official [telemetry extension guide](https://code.visualstudio.com/api/extension-guides/telemetry):

- A custom backend must still respect `vscode.env.isTelemetryEnabled` and `onDidChangeTelemetryEnabled`. An extension-specific opt-in cannot override a global opt-out.
- Collect as little as possible, do not collect personally identifiable information, and disclose the event inventory.
- If MCP Hub adds its own setting, tag it `telemetry` and `usesOnlineServices`.
- Include a root `telemetry.json` so `code --telemetry` can show the extension's declared events. This repository's staging-based VSIX packaging must be checked to ensure that file survives packaging.

Treat `TelemetryTrustedValue` as an exceptional escape hatch: it disables cleaning for a value asserted to contain no identifiable information. It should not be used for prompts, paths, repository names, arguments, results, error messages, or arbitrary upstream text.

VS Code exposes `env.machineId` (computer identity) and `env.sessionId` (changes each editor start) in the [environment API](https://code.visualstudio.com/api/references/vscode-api#env). `machineId` is an installation/computer proxy, not a person: one developer may use several machines and several developers may share one machine. Therefore it cannot, by itself, support the proposed “28-day engaged developers” north-star metric. Any durable company-user key needs explicit privacy/security approval and pseudonymization; otherwise report unique installations and unique editor sessions honestly.

## MCP observability boundary

The VS Code MCP provider is the wrong counting point. `provideMcpServerDefinitions` is called eagerly for availability, while `resolveMcpServerDefinition` runs when the editor needs to start a server; neither means the user executed a tool ([API reference](https://code.visualstudio.com/api/references/vscode-api#McpServerDefinitionProvider)).

The correct MCP Hub point is the outward `CallToolRequestSchema` handler in `packages/gateway/src/server/gateway-server.ts`. The stable MCP tools specification defines `tools/list` as catalog discovery and `tools/call` as execution, with tool failures either returned as `isError: true` or represented by protocol errors ([MCP tools specification, 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)). Record one operation around every outward call, including built-in Hub tools and unresolved names, with:

- exposed public tool ID and routed upstream ID;
- success, tool error (`isError`), protocol/gateway error, cancellation, or timeout;
- duration and extension/gateway version;
- a local operation ID and session ID where available;
- low-cardinality MCP client product/version if useful.

Do not record arguments, results, prompts, file paths, URLs, command strings, or raw error messages. Count the outward call once; do not double-count the gateway-to-upstream call as another user operation.

MCP initialization includes client implementation information (`clientInfo.name`, title, version, etc.), so the gateway may distinguish VS Code from another MCP client, but the protocol provides no end-user identity ([MCP lifecycle specification](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)). Its generic logging facility is for server-to-client diagnostic messages, not product analytics. A 2026 draft deprecates protocol logging, reinforcing that analytics should remain an internal gateway concern rather than MCP `notifications/message` ([draft logging specification](https://modelcontextprotocol.io/specification/draft/server/utilities/logging)).

## Copilot Chat OpenTelemetry: what it adds

Current Copilot Chat can export traces, metrics, and log events via OTLP. It is off by default and turns on through `github.copilot.chat.otel.enabled`, a Copilot environment variable, or an OTLP endpoint. Exporters include OTLP/HTTP, OTLP/gRPC, console, and file. The current source of truth is VS Code's [Monitor agent usage with OpenTelemetry](https://code.visualstudio.com/docs/agents/guides/monitoring-agents) page, updated 2026-08-12.

### Most useful trace data

- `invoke_agent` span: `gen_ai.conversation.id`, agent name/type, requested/resolved model, total input/output/cache tokens, turn count, failure.
- `chat` span: one LLM call with model, tokens, finish reasons, latency/time-to-first-token, and failure.
- `execute_tool` span: `gen_ai.tool.name`, `gen_ai.tool.call.id`, failure and duration; skill calls add `github.copilot.tool.parameters.skill_name`; MCP calls add `github.copilot.tool.parameters.mcp_server_name_hash` and `github.copilot.tool.parameters.mcp_tool_name`.
- Resource attributes: `service.name`, extension `service.version`, and a `session.id` unique per VS Code window. Custom low-cardinality resource attributes such as team/department can be supplied with `OTEL_RESOURCE_ATTRIBUTES` or managed settings.

For new queries, prefer `gen_ai.*` where standardized and `github.copilot.*` for Copilot-specific fields. `copilot_chat.*` is the legacy namespace, although the VS Code docs promise continued dual emission for existing fields.

### Most useful metrics and events

| Signal | Use |
| --- | --- |
| `copilot_chat.tool.call.count` with `gen_ai.tool.name` and `success` | Tool volume and success; do not interpret raw calls as users |
| `copilot_chat.tool.call.duration` | Tool latency |
| `copilot_chat.session.count`, `copilot_chat.agent.invocation.duration`, `copilot_chat.agent.turn.count` | Workflow/session depth and duration |
| `copilot_chat.user.feedback.count` (`positive` / `negative`) | Explicit quality signal |
| `copilot_chat.user.action.count` (`copy`, `insert`, `apply`, `followup`) | Downstream engagement |
| `copilot_chat.edit.acceptance.count`, `chat_edit.outcome.count`, `edit.survival.*`, `lines_of_code.count` | Outcome/quality guardrails beyond tool execution |
| `copilot_chat.tool.call` event | Per-call tool name, duration, success, and error class |
| `copilot_chat.agent.turn` event | Per-turn token totals and tool-call count |

The exact metric dimensions and per-event fields are documented in the official [`agent_monitoring.md`](https://github.com/microsoft/vscode/blob/main/extensions/copilot/docs/monitoring/agent_monitoring.md) maintained with VS Code's bundled Copilot code.

### Privacy and identity caveats

Keep `captureContent` false on company laptops. With it false, prompts, responses, system instructions, tool schemas, arguments, results, file paths, and command strings are not exported. With it true, those fields may contain source code and sensitive company data.

However, “content capture off” does not mean “repository metadata off.” The current `invoke_agent` span includes repository remote URL, branch, commit SHA, and GitHub organization when available. These fields require privacy/security review and collector-side retention/access rules even when prompt capture is disabled ([official monitoring attributes](https://code.visualstudio.com/docs/agents/guides/monitoring-agents#_what-gets-collected)).

Copilot OTel provides window, conversation, request, and tool-call correlation, but the documented resource attributes contain no durable user/installation identifier. It cannot calculate 28-day engaged developers without an approved pseudonymous identity added as a custom resource attribute or a governed join to another data source. Never call conversation count “developer count.”

Enterprise administrators can mandate endpoint, protocol, service name, resource attributes, capture-content behavior, and exporter headers through Copilot managed settings. Managed values override environment variables and user settings. Current caveats include reload-to-apply for the agent host and managed authentication headers not being forwarded to the agent-host process in the documented release ([enterprise AI settings](https://code.visualstudio.com/docs/enterprise/ai-settings#_configure-telemetry-export-with-opentelemetry)). The company laptop must be checked for a sufficiently current VS Code/Copilot version; this repository's VS Code minimum of 1.101 does not guarantee that the 2026 Copilot OTel or enterprise policy surface exists.

OpenTelemetry's GenAI semantic conventions are still under active development, and `execute_tool` / `invoke_agent` convention values are currently marked development. Keep dashboards tolerant of added/renamed fields and version the ingestion mapping ([OpenTelemetry GenAI attribute registry](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/)).

## Implementation implications for MCP Hub

1. Define one low-cardinality operation schema shared by extension and gateway: surface, feature/tool ID, outcome, duration, extension version, schema version, session/operation ID, and pseudonymous installation/user key only if approved.
2. Create the privacy/consent boundary in the thin extension layer because `packages/gateway` must never import `vscode`. Pass a small telemetry sink or no-op sink into gateway options; do not put VS Code dependencies in gateway.
3. Instrument `createGatewayServer` around the complete outward `tools/call` handler, including Hub built-ins, unresolved tools, returned `isError`, thrown errors, and cancellation. Leave `tools/list` uncounted as engagement.
4. Wrap this extension's registered command callbacks. Record completion/outcome, not registration or activation. Label command initiator `unknown` unless the extension itself can prove the source.
5. If chat participants and Language Model tools are added, instrument their handler/`invoke` entry points. Return a generated operation ID in `ChatResult.metadata` so later `onDidReceiveFeedback` can correlate without storing prompts.
6. Do not claim a reliable `human_explicit` versus `agent_selected` split from generic command, LM-tool, or MCP callbacks. The stable APIs do not carry enough origin data. Use `human_explicit` only where an owned UI/participant flow proves it; otherwise use `agent_or_user` / `unknown`.
7. Send custom MCP Hub telemetry and Copilot OTel to the same governed collector/warehouse, but deduplicate conceptually: gateway calls are the authoritative MCP Hub tool execution count; Copilot spans provide the surrounding agent workflow. Join by trace/operation context only where an actual shared correlation ID exists.
8. Keep Copilot content capture disabled and use an OTel Collector allowlist/drop processor for repository metadata and any unapproved attributes. Do not depend only on producer defaults.
9. Add `telemetry.json`, a retention statement, a field inventory, and a user-visible setting if required by company policy. Verify the staged VSIX contains the disclosure file.
10. Report engaged installations/sessions until the company approves a user-resolution method. The management north star can become 28-day engaged developers only after identity resolution is validated.

## Unknowns to resolve on the company laptop

- Exact VS Code and GitHub Copilot Chat versions, and whether the documented OTel signals/enterprise managed settings are present in those pinned builds.
- Whether corporate policy permits extension telemetry at all, what consent language is required, and whether `vscode.env.isTelemetryEnabled` is centrally controlled.
- Approved collector endpoint, authentication delivery, offline buffering, retention, access controls, and deletion process.
- Whether repository URL/branch/commit attributes must be dropped at the collector.
- Approved identity grain: person, installation, managed device, or anonymous session; and the approved pseudonymization/key-rotation design.
- Whether the company needs non-Copilot MCP clients counted. If yes, gateway events are mandatory because Copilot OTel will not cover them.
- Whether scripts execute inside VS Code commands or as standalone processes. Standalone scripts need a separate consent and delivery design.
- Whether VS Code forwards W3C/MCP trace context into this HTTP gateway. MCP's finalized [SEP-414](https://modelcontextprotocol.io/seps/414-request-meta) specifies `traceparent`, `tracestate`, and `baggage` in request `_meta`, but the behavior must be tested against the exact VS Code and MCP SDK versions before relying on end-to-end trace joins.

