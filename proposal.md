# Debugging/Triage Exercise Proposal

## Repo map (core paths + boundaries)
- **core/runner**: `Runner` orchestrates session retrieval, plugin hooks, agent selection, and event
  compaction. This is the main runtime entrypoint for agent execution.
- **core/agents**: `BaseAgent`, `LlmAgent`, `InvocationContext`, `RunConfig` define agent lifecycle,
  callbacks, streaming/live execution, and run configuration.
- **core/flows/llmflows**: `BaseLlmFlow`, `SingleFlow`, `AutoFlow`, `Contents`, `Functions`,
  `RequestConfirmationLlmRequestProcessor` handle request assembly, tool calls, agent transfer, and
  response processing.
- **core/tools**: `BaseTool`, `FunctionTool`, `ToolContext`, `BaseToolset`,
  `FunctionCallingUtils` implement tool execution, schema generation, and live streaming tool
  wiring.
- **core/events**: `Event`, `EventActions` encode model/user events, tool metadata, and state
  deltas.
- **core/sessions**: `BaseSessionService`, `InMemorySessionService`, `VertexAiSessionService`,
  `SessionJsonConverter`, `SessionUtils`, `State` manage session storage, state merging, and API
  serialization.
- **core/models**: `LlmRequest`, `LlmResponse`, `Model`, `LlmRegistry`, `BaseLlmConnection` define
  LLM request/response contracts and model instantiation/cache behavior.
- **core/memory**: `InMemoryMemoryService` implements lightweight memory search for user history.
- **core/summarizer**: `SlidingWindowEventCompactor`, `EventsCompactionConfig` provide event
  compaction and summarization triggers.
- **a2a/**: `A2ASendMessageExecutor`, `RemoteA2AAgent`, converters bridge ADK events to A2A protocol.
- **dev/** + **maven_plugin/**: developer web UI, agent loaders, and plugin integration (non-core,
  but important IO surfaces).

---

## Bug candidates
Ranks use a 1–5 scale (higher is better): **EV**=Exercise Value, **Stealth**, **Scorability**.

### B01 — Stale session used after state delta applied
- **Location**: `core/.../Runner.java`, `runAsync(Session, Content, RunConfig, Map)` (~468–508)
- **Core relevance**: Runner is the hot path for all agent invocations.
- **Bug type**: correctness / state consistency
- **Proposed change**: pass the original `session` into `runAgentWithFreshSession(...)` instead of
  `updatedSession` (or skip the `getSession` refresh).
- **Trigger conditions**: any run with `stateDelta` or plugin/state mutations before agent run.
- **Expected symptom**: agent sees stale state, producing responses that ignore fresh state updates.
- **Why it’s hard**: only manifests when state is mutated before agent execution.
- **Static-analysis discoverability**: Medium (looks like harmless refactor).
- **Suggested detection**: integration test asserting stateDelta visible to agent; log state size.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 5

### B02 — Shared stateDelta map leaks mutations
- **Location**: `core/.../Runner.java`, `appendNewMessageToSession(...)` (~332–378)
- **Core relevance**: every user message passes through this method.
- **Bug type**: data integrity / concurrency
- **Proposed change**: use `stateDelta` directly without copying into a new `ConcurrentHashMap`.
- **Trigger conditions**: caller reuses or mutates `stateDelta` after invocation.
- **Expected symptom**: later mutations retroactively alter persisted state, nondeterministically.
- **Why it’s hard**: depends on caller behavior and timing; looks like a safe micro-optimization.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: unit test that mutates stateDelta after call; assert session state stable.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B03 — Agent selection uses oldest event
- **Location**: `core/.../Runner.java`, `findAgentToRun(...)` (~757–782)
- **Core relevance**: determines which agent handles each turn.
- **Bug type**: correctness / routing
- **Proposed change**: remove `Collections.reverse(events)` (scan oldest → newest).
- **Trigger conditions**: sessions with sub-agent transfer history.
- **Expected symptom**: wrong agent chosen (often root), especially after transfers.
- **Why it’s hard**: only multi-agent sessions show it; logs still show valid agents.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: integration test with multiple sub-agents and transfer history.
- **Rank (EV/Stealth/Scorability)**: 5 / 3 / 5

### B04 — Live audio runs without input transcription
- **Location**: `core/.../Runner.java`, `newInvocationContextForLive(...)` (~598–618)
- **Core relevance**: live streaming is a flagship path.
- **Bug type**: reliability / edge-case config
- **Proposed change**: remove the block that sets `inputAudioTranscription` when a live queue exists.
- **Trigger conditions**: live audio runs where agent transfer or tool calls expect transcripts.
- **Expected symptom**: live sessions silently drop user audio → degraded or empty responses.
- **Why it’s hard**: only manifests in audio + live mode; easy to misattribute to model issues.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: live-mode integration test with audio input + tool transfer.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 3

### B05 — LLM call limit resets on context copies
- **Location**: `core/.../InvocationContext.java`, `Builder(InvocationContext)` (~421–439)
- **Core relevance**: `InvocationContext` is central to every flow step.
- **Bug type**: correctness / quota enforcement
- **Proposed change**: initialize `invocationCostManager` with `new InvocationCostManager()` instead
  of copying from the original.
- **Trigger conditions**: flows that create nested contexts; maxLlmCalls configured.
- **Expected symptom**: call-limit bypass, runaway tool/LLM loops.
- **Why it’s hard**: only visible under long-running or looping flows.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: test that enforces maxLlmCalls across nested agent runs.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 4

### B06 — Resumability pause check uses responses not calls
- **Location**: `core/.../InvocationContext.java`, `shouldPauseInvocation(...)` (~367–381)
- **Core relevance**: governs resumable flows and long-running tools.
- **Bug type**: correctness / resumability
- **Proposed change**: check `event.functionResponses()` instead of `functionCalls()` when matching
  long-running IDs.
- **Trigger conditions**: long-running tools; resumability enabled.
- **Expected symptom**: invocation never pauses before tool runs; resumability broken.
- **Why it’s hard**: only for long-running tools; behavior looks like "tool just slow".
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: integration test for resumable long-running tool call.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B07 — Callback-produced response doesn’t end invocation
- **Location**: `core/.../BaseAgent.java`, `callCallback(...)` (~306–353)
- **Core relevance**: callbacks are used by plugins and agent hooks.
- **Bug type**: correctness / control flow
- **Proposed change**: remove `invocationContext.setEndInvocation(true)` when a callback returns
  content.
- **Trigger conditions**: before/after callbacks that short-circuit agent execution.
- **Expected symptom**: duplicate responses (callback + model output).
- **Why it’s hard**: only for callback-enabled agents; looks like "model repeated itself".
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test that callback response terminates invocation.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B08 — Agent transfer silently disabled
- **Location**: `core/.../LlmAgent.java`, `determineLlmFlow()` (~655–660)
- **Core relevance**: controls whether AutoFlow (transfer) is used.
- **Bug type**: correctness / routing
- **Proposed change**: change `&&` to `||` in the SingleFlow selection condition.
- **Trigger conditions**: any agent with sub-agents and *either* disallow flag set.
- **Expected symptom**: agent stops transferring to sub-agents/peers unexpectedly.
- **Why it’s hard**: manifests only in mixed transfer configs.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: multi-agent integration test with disallow flags set.
- **Rank (EV/Stealth/Scorability)**: 5 / 3 / 5

### B09 — Output state stores hidden “thought” content
- **Location**: `core/.../LlmAgent.java`, `maybeSaveOutputToState(...)` (~663–698)
- **Core relevance**: outputKey populates session state for downstream logic.
- **Bug type**: security / data integrity
- **Proposed change**: invert the `!isThought(part)` filter so only thought parts are kept.
- **Trigger conditions**: models emit thought parts; outputKey configured.
- **Expected symptom**: session state contains hidden reasoning instead of user-visible output.
- **Why it’s hard**: depends on model behavior; often overlooked in reviews.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: unit test with thought parts; ensure state uses only visible content.
- **Rank (EV/Stealth/Scorability)**: 5 / 5 / 4

### B10 — LLM is called twice per step
- **Location**: `core/.../BaseLlmFlow.java`, `run(...)` (~417–439)
- **Core relevance**: `BaseLlmFlow` executes every LLM loop.
- **Bug type**: performance / correctness
- **Proposed change**: remove `.cache()` from `currentStepEvents`.
- **Trigger conditions**: any non-final response step.
- **Expected symptom**: duplicate LLM calls and duplicated events.
- **Why it’s hard**: looks like model repetition; double-call is non-obvious in logs.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: test that LLM request count equals step count; add tracing counters.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 5

### B11 — Error-only responses drop all events
- **Location**: `core/.../BaseLlmFlow.java`, `buildPostprocessingEvents(...)` (~623–635)
- **Core relevance**: governs output emission from LLM responses.
- **Bug type**: correctness / error handling
- **Proposed change**: change the early-return condition to require **all** fields to be empty
  (`&&` instead of `||`), which skips events when only `errorCode` or `interrupted` is set.
- **Trigger conditions**: model errors or interruptions without content.
- **Expected symptom**: silent failure (no events), leaving clients hanging.
- **Why it’s hard**: only during error paths; easy to miss in manual testing.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test that error responses always yield at least one event.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B12 — Tool-only events treated as “empty”
- **Location**: `core/.../Contents.java`, `isEmptyContent(...)` (~164–174)
- **Core relevance**: controls which events are sent to the model.
- **Bug type**: correctness / context loss
- **Proposed change**: treat events as empty unless **text** is present (ignore function calls).
- **Trigger conditions**: sessions with tool calls/responses but little text.
- **Expected symptom**: tool calls disappear from LLM context; inconsistent tool usage.
- **Why it’s hard**: only in tool-heavy sessions; looks like model ignored tools.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: test with functionCall-only events; assert they remain in context.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 4

### B13 — Branch filtering becomes too strict
- **Location**: `core/.../Contents.java`, `isEventBelongsToBranch(...)` (~307–317)
- **Core relevance**: branch logic affects multi-agent isolation.
- **Bug type**: correctness / multi-agent isolation
- **Proposed change**: require exact equality (use `equals`) instead of `startsWith`.
- **Trigger conditions**: nested sub-agent branches.
- **Expected symptom**: child agent loses parent context, producing less coherent answers.
- **Why it’s hard**: appears as “model forgot earlier context”.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: test nested agent branches with shared context.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B14 — Off-by-one removes correct function-call pairing
- **Location**: `core/.../Contents.java`, `rearrangeEventsForLatestFunctionResponse(...)` (~380–403)
- **Core relevance**: tool-call ordering impacts LLM tool reasoning.
- **Bug type**: correctness / ordering
- **Proposed change**: start the backward search from `events.size() - 2` instead of `-3`.
- **Trigger conditions**: async tool responses where the match is two events back.
- **Expected symptom**: missing or mismatched function call/response pairing.
- **Why it’s hard**: only in async tool histories; ordering bugs are subtle.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: property-based test for call/response ordering invariants.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 3

### B15 — Uses stale confirmation instead of most recent
- **Location**: `core/.../RequestConfirmationLlmRequestProcessor.java`,
  `findMostRecentConfirmations(...)` (~146–171)
- **Core relevance**: controls security confirmation flow for tools.
- **Bug type**: correctness / security workflow
- **Proposed change**: search from the start (oldest) rather than from the end.
- **Trigger conditions**: multiple confirmation events in history.
- **Expected symptom**: old confirmations applied to new tool calls.
- **Why it’s hard**: only in long sessions with repeated confirmations.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: test that latest confirmation overrides prior ones.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B16 — Sequential vs parallel tool execution swapped
- **Location**: `core/.../Functions.java`, `handleFunctionCalls(...)` (~156–164)
- **Core relevance**: tool execution is a primary runtime path.
- **Bug type**: concurrency / ordering
- **Proposed change**: invert the `SEQUENTIAL` check so parallel runs when sequential is requested.
- **Trigger conditions**: tools that depend on order or shared state.
- **Expected symptom**: nondeterministic tool results or data races.
- **Why it’s hard**: fails only under concurrent tool calls; tests often single-tool.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: integration test with ordered tool dependencies.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 4

### B17 — stopStreaming leaves active tool behind
- **Location**: `core/.../Functions.java`, `processFunctionLive(...)` (~284–297)
- **Core relevance**: live streaming tool lifecycle is a hot path.
- **Bug type**: resource leak / concurrency
- **Proposed change**: remove `activeStreamingTools().remove(functionNameToStop)`.
- **Trigger conditions**: live streaming tools stopped mid-run.
- **Expected symptom**: memory leak, stale streams, or duplicate tool updates later.
- **Why it’s hard**: only visible after long sessions with repeated stop/start.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: load test with repeated start/stop of streaming tools.
- **Rank (EV/Stealth/Scorability)**: 4 / 5 / 3

### B18 — Long-running tool IDs derived from name, not call ID
- **Location**: `core/.../Functions.java`, `getLongRunningFunctionCalls(...)` (~353–365)
- **Core relevance**: drives resumability and pause logic.
- **Bug type**: correctness / resumability
- **Proposed change**: add `functionCall.name()` to the set instead of `functionCall.id()`.
- **Trigger conditions**: long-running tools with confirmation/resume behavior.
- **Expected symptom**: resumability logic fails; pauses never triggered.
- **Why it’s hard**: only shows in long-running tool flows.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: test that longRunningToolIds match call IDs.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B19 — Parallel tool actions overwrite each other
- **Location**: `core/.../Functions.java`, `mergeParallelFunctionResponseEvents(...)` (~416–439)
- **Core relevance**: tool results feed session state.
- **Bug type**: data integrity
- **Proposed change**: use only the first event’s `EventActions` instead of merging all.
- **Trigger conditions**: multiple tools returning state deltas in parallel.
- **Expected symptom**: missing state updates from some tools.
- **Why it’s hard**: only in parallel tools; state loss looks like tool bug.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: integration test with two tools updating different state keys.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 4

### B20 — LiveConnectConfig never updated with tool declarations
- **Location**: `core/.../BaseTool.java`, `processLlmRequest(...)` (~112–154)
- **Core relevance**: tool availability for live streaming.
- **Bug type**: correctness / streaming contract
- **Proposed change**: remove `llmRequestBuilder.liveConnectConfig(...)` update.
- **Trigger conditions**: live streaming mode with tools.
- **Expected symptom**: tools unavailable only in live mode; non-live works.
- **Why it’s hard**: streaming-only defect; typical tests use non-live flows.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: live-mode test invoking a tool.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 3

### B21 — Tool confirmation bypass
- **Location**: `core/.../FunctionTool.java`, `runAsync(...)` (~230–247)
- **Core relevance**: security gate for tool execution.
- **Bug type**: security / policy bypass
- **Proposed change**: if confirmation is missing, fall through to `call(...)` instead of returning
  a rejection.
- **Trigger conditions**: tools marked `requireConfirmation=true`.
- **Expected symptom**: tools execute without user approval.
- **Why it’s hard**: appears as normal tool execution unless audit logs checked.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: security test that requires confirmation before execution.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 5

### B22 — Optional params treated as required
- **Location**: `core/.../FunctionTool.java`, `buildArguments(...)` (~323–333)
- **Core relevance**: parameter binding for function tools.
- **Bug type**: input validation / edge-case
- **Proposed change**: remove the `schema.optional()` branch so missing optional args throw.
- **Trigger conditions**: tools with optional parameters omitted by the model.
- **Expected symptom**: tool call fails unexpectedly.
- **Why it’s hard**: depends on model behavior; appears random.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test for optional args with missing fields.
- **Rank (EV/Stealth/Scorability)**: 3 / 3 / 4

### B23 — Confirmation request keyed to empty ID
- **Location**: `core/.../ToolContext.java`, `requestConfirmation(...)` (~86–93)
- **Core relevance**: drives user confirmation UX.
- **Bug type**: correctness / state integrity
- **Proposed change**: remove the `functionCallId` guard, allowing empty keys.
- **Trigger conditions**: functionCallId not set (e.g., custom tool invocations).
- **Expected symptom**: confirmations can’t be matched to tool calls.
- **Why it’s hard**: only when call IDs are missing or regenerated.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: unit test that confirmation requires call ID.
- **Rank (EV/Stealth/Scorability)**: 3 / 4 / 3

### B24 — Recursive schema generation returns empty objects
- **Location**: `core/.../FunctionCallingUtils.java`, `buildSchemaRecursive(...)` (~176–245)
- **Core relevance**: tool schema drives model tool calls.
- **Bug type**: API contract drift
- **Proposed change**: on recursion detection, return `Schema.builder().type("OBJECT").build()`
  (drop description) or skip `finishProcessing`, causing incomplete/empty schema.
- **Trigger conditions**: tools with recursive or self-referential parameter types.
- **Expected symptom**: malformed schema, model refuses tool or calls it incorrectly.
- **Why it’s hard**: only in specific type shapes; schema errors surface at runtime.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: schema-generation tests on recursive DTOs.
- **Rank (EV/Stealth/Scorability)**: 3 / 4 / 3

### B25 — Plugin errors silently ignored
- **Location**: `core/.../PluginManager.java`, `runMaybeCallbacks(...)` (~253–275)
- **Core relevance**: plugin system wraps core execution.
- **Bug type**: reliability / observability
- **Proposed change**: add `.onErrorComplete()` so plugin errors are swallowed.
- **Trigger conditions**: plugin exceptions in callbacks.
- **Expected symptom**: plugin failures disappear; debugging becomes difficult.
- **Why it’s hard**: behavior looks fine without logs; only observability changes.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: test plugin that throws; assert error propagation.
- **Rank (EV/Stealth/Scorability)**: 4 / 5 / 3

### B26 — Partial streaming events mutate session state
- **Location**: `core/.../BaseSessionService.java`, `appendEvent(...)` (~161–188)
- **Core relevance**: session state is central for all flows.
- **Bug type**: correctness / streaming
- **Proposed change**: remove the early return for `event.partial()`.
- **Trigger conditions**: streaming responses with partial chunks.
- **Expected symptom**: duplicated events and premature state updates.
- **Why it’s hard**: only in streaming mode; state drift is subtle.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: streaming test asserting partial events are ignored.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B27 — Timestamp filter compares ms vs seconds
- **Location**: `core/.../InMemorySessionService.java`, `getSession(...)` (~146–152)
- **Core relevance**: session retrieval affects every run.
- **Bug type**: time/precision
- **Proposed change**: compare `event.timestamp()` directly to `Instant.getEpochSecond()`.
- **Trigger conditions**: `afterTimestamp` filtering.
- **Expected symptom**: most events filtered out (or retained incorrectly).
- **Why it’s hard**: only hits when using `afterTimestamp` config.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test for `afterTimestamp` with ms timestamps.
- **Rank (EV/Stealth/Scorability)**: 3 / 3 / 4

### B28 — App/user state prefixes slice wrong index
- **Location**: `core/.../InMemorySessionService.java`, `appendEvent(...)` (~238–261)
- **Core relevance**: session state impacts all behavior.
- **Bug type**: data integrity
- **Proposed change**: use `substring(State.APP_PREFIX.length() - 1)` (off-by-one).
- **Trigger conditions**: app/user state updates.
- **Expected symptom**: wrong keys written; state lookups fail later.
- **Why it’s hard**: values appear to be stored, but under wrong keys.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: test that app/user prefixes round-trip correctly.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B29 — Event timestamps serialized in milliseconds as seconds
- **Location**: `core/.../SessionJsonConverter.java`, `convertEventToJson(...)` (~72–82)
- **Core relevance**: API contract for event persistence.
- **Bug type**: time/precision
- **Proposed change**: set `"seconds": event.timestamp()` (remove `/ 1000`).
- **Trigger conditions**: remote session storage/restore.
- **Expected symptom**: event ordering broken; timestamps appear far in the future.
- **Why it’s hard**: only visible when consuming persisted sessions.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: serialization round-trip tests with known timestamps.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B30 — Contract drift for long-running tool IDs
- **Location**: `core/.../SessionJsonConverter.java`, `convertEventToJson(...)` (~64–67)
- **Core relevance**: long-running tools and resumability.
- **Bug type**: API contract drift
- **Proposed change**: rename key `"long_running_tool_ids"` → `"longRunningToolIds"`.
- **Trigger conditions**: storing/reading events via API.
- **Expected symptom**: long-running tool IDs lost on restore; resumability fails.
- **Why it’s hard**: only when persisting across process boundaries.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: API contract test asserting exact JSON keys.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B31 — Content role dropped during encode/decode
- **Location**: `core/.../SessionUtils.java`, `toContent(...)` (~85–89)
- **Core relevance**: event content is sent to models and APIs.
- **Bug type**: data integrity
- **Proposed change**: remove `role.ifPresent(contentBuilder::role)`.
- **Trigger conditions**: any content with non-default roles.
- **Expected symptom**: role defaults to model/user incorrectly; model behavior shifts.
- **Why it’s hard**: only visible in role-sensitive flows (system/user split).
- **Static-analysis discoverability**: Low.
- **Suggested detection**: round-trip tests for content roles.
- **Rank (EV/Stealth/Scorability)**: 3 / 4 / 3

### B32 — State removals no longer produce deltas
- **Location**: `core/.../State.java`, `remove(...)` (~125–129)
- **Core relevance**: state deltas drive persistence and downstream logic.
- **Bug type**: data integrity / state propagation
- **Proposed change**: delete the `delta.put(..., REMOVED)` line.
- **Trigger conditions**: state key removals.
- **Expected symptom**: removed keys reappear after persistence/merge.
- **Why it’s hard**: state looks correct in-memory but reappears later.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: test that removals propagate via EventActions stateDelta.
- **Rank (EV/Stealth/Scorability)**: 4 / 4 / 4

### B33 — Memory ingestion includes empty events (performance regression)
- **Location**: `core/.../InMemoryMemoryService.java`, `addSessionToMemory(...)` (~69–79)
- **Core relevance**: memory search affects relevance and latency.
- **Bug type**: performance / data quality
- **Proposed change**: remove the filter that drops events without content parts.
- **Trigger conditions**: sessions with many state-only events.
- **Expected symptom**: memory search returns junk and slows down over time.
- **Why it’s hard**: slow degradation; no single failing test.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: load test measuring memory-search latency with many events.
- **Rank (EV/Stealth/Scorability)**: 3 / 4 / 3

### B34 — Duplicate tool names silently override
- **Location**: `core/.../LlmRequest.java`, `Builder.appendTools(...)` (~200–217)
- **Core relevance**: tool mapping is critical for function calling.
- **Bug type**: API contract drift / correctness
- **Proposed change**: change duplicate-handling to “last tool wins” instead of throwing.
- **Trigger conditions**: multiple toolsets with same tool name.
- **Expected symptom**: unexpected tool implementation chosen.
- **Why it’s hard**: depends on configuration; failures look like tool bugs.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test verifying duplicate tool names error.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B35 — Output schema no longer enforces JSON
- **Location**: `core/.../LlmRequest.java`, `Builder.outputSchema(...)` (~224–229)
- **Core relevance**: structured outputs are a key feature.
- **Bug type**: correctness / API contract
- **Proposed change**: remove `.responseMimeType("application/json")`.
- **Trigger conditions**: outputSchema configured on models that require MIME type.
- **Expected symptom**: model returns text; schema validation fails downstream.
- **Why it’s hard**: only for models enforcing response MIME type.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: integration test with outputSchema and JSON-only model.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B36 — Treats normal finishReason as error
- **Location**: `core/.../LlmResponse.java`, `Builder.response(...)` (~179–190)
- **Core relevance**: response decoding is central to all flows.
- **Bug type**: correctness / error handling
- **Proposed change**: set `errorCode` whenever `finishReason` is present, even when content exists.
- **Trigger conditions**: normal responses with finishReason=STOP.
- **Expected symptom**: downstream sees errors for successful responses.
- **Why it’s hard**: finishReason is always present; hard to spot in static review.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: unit test that content-bearing responses are not marked error.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B37 — Model cache keyed by regex pattern
- **Location**: `core/.../LlmRegistry.java`, `getLlm(...)` (~61–76)
- **Core relevance**: model instantiation and caching across agents.
- **Bug type**: caching / data integrity
- **Proposed change**: cache by matched regex pattern instead of modelName.
- **Trigger conditions**: multiple models matching the same pattern (e.g., gemini-2.0, gemini-1.5).
- **Expected symptom**: model instance bleed-over, wrong model used intermittently.
- **Why it’s hard**: looks like model misconfiguration; cache hides root cause.
- **Static-analysis discoverability**: Low–Medium.
- **Suggested detection**: unit test requesting two different model names and verifying instances.
- **Rank (EV/Stealth/Scorability)**: 5 / 4 / 4

### B38 — A2A history dedupe uses invocationId
- **Location**: `a2a/.../A2ASendMessageExecutor.java`, `filterNewHistoryEvents(...)` (~139–155)
- **Core relevance**: A2A runtime is a primary IO boundary.
- **Bug type**: data integrity
- **Proposed change**: dedupe on `event.invocationId()` instead of `event.id()`.
- **Trigger conditions**: multiple events per invocation (typical).
- **Expected symptom**: drops all but one event per invocation, truncating history.
- **Why it’s hard**: only in multi-event turns; looks like A2A model limitations.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: integration test with multi-event invocation history.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B39 — A2A preprocessor uses oldest user text
- **Location**: `a2a/.../ConversationPreprocessor.java`, `extractHistoryAndUserContent(...)`
  (~79–94)
- **Core relevance**: this defines the user turn for remote agents.
- **Bug type**: correctness / ordering
- **Proposed change**: scan from the start (oldest) and pick the first text event.
- **Trigger conditions**: any session with multiple user turns.
- **Expected symptom**: remote agent responds to stale user input.
- **Why it’s hard**: looks like remote model “ignoring” latest input.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: A2A integration test with multiple user messages.
- **Rank (EV/Stealth/Scorability)**: 4 / 3 / 4

### B40 — A2A role/author flipped for messages
- **Location**: `a2a/.../ResponseConverter.java`, `messageToEvents(...)` (~84–105)
- **Core relevance**: A2A conversions are a primary protocol boundary.
- **Bug type**: correctness / API contract
- **Proposed change**: map `Message.Role.AGENT` to author `"user"` (flip author/role mapping).
- **Trigger conditions**: any A2A response converted back to ADK events.
- **Expected symptom**: model responses treated as user input; conversation gets inverted.
- **Why it’s hard**: only in A2A paths; looks like remote agent misbehaving.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: converter unit test (likely too easy unless the direct test is removed).
- **Rank (EV/Stealth/Scorability)**: 4 / 2 / 5

---

## Top 10 recommended set
1. **B01** — Stale session used after state delta applied (core correctness, high impact).
2. **B10** — LLM called twice per step (high signal, tricky to diagnose).
3. **B16** — Sequential vs parallel tool execution swapped (ordering + concurrency).
4. **B19** — Parallel tool actions overwrite each other (state loss in core path).
5. **B05** — LLM call limit resets on context copies (quota enforcement, subtle).
6. **B08** — Agent transfer silently disabled (multi-agent routing regression).
7. **B09** — Output state stores hidden “thought” content (security/data leak).
8. **B27** — Timestamp filter compares ms vs seconds (time precision + retrieval).
9. **B37** — Model cache keyed by regex pattern (cross-model bleed, subtle).
10. **B38** — A2A history dedupe uses invocationId (protocol boundary + data loss).

