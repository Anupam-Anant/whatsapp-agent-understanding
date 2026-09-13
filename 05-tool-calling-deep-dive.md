# 05 — Tool-Calling Deep Dive

> **Scope.** The heart of the system: the hand-rolled agent loop, the tool abstraction, all six tools,
> dynamic tool generation from Krista conversations, JSON-schema generation, the two model-quirk
> workarounds, tool suppression, and end-to-end walkthroughs. Builds on `03-openai-integration.md`
> (the model) and `04-langchain4j-usage.md` (the API + message bridge).

## 0. Mental model

The LLM is used as a **router/agent**: given the system prompt, the conversation history, and a set of
**tool specifications**, it decides whether to answer in text or to **call tools**. The app executes those
tools, feeds the results back, and lets the model react — repeating until it produces a text answer or a
tool signals "stop." This loop is written by hand in `ToolExecutor` (not `AiServices`).

Tools fall into two groups:
- **Static tools** (singletons): `show_options`, `exit_session`, `switch_language`, `submit_answer` (the
  last is per-request, built from the current field state).
- **Dynamic tools**: one per Krista "conversation" (workflow), generated at startup and on config change.

## 1. Support types

### `RemoteTool` (`orchestrator/tools/RemoteTool.java`)
```java
public interface RemoteTool {
    ToolSpecification specification();                                // what the model sees
    ToolResult execute(ToolContext toolContext, Map<String,Object> arguments); // what runs
    default String name() { return specification().name(); }
}
```

### `ToolResult` (sealed) (`orchestrator/tools/ToolResult.java`) — the loop-control signal
```java
public sealed interface ToolResult {
    String PROCESSING_MESSAGE = "Processing";
    record Continue(String message)  implements ToolResult {}  // keep looping (e.g. validation error → re-ask)
    record StopLoop(String message)  implements ToolResult {}  // end the turn (silence to user)
    record TriggerExitFlow()         implements ToolResult {}  // hand off to exit-confirmation flow
    static ToolResult continueWith(String m) { return new Continue(m); }
    static ToolResult stopLoop()             { return new StopLoop(PROCESSING_MESSAGE); }
    static ToolResult stopLoop(String m)     { return new StopLoop(m); }
    static ToolResult triggerExitFlow()      { return new TriggerExitFlow(); }
}
```

### `ToolContext` (`orchestrator/tools/ToolContext.java`) — the capability surface handed to a tool
An interface with an inline record impl created per tool batch. It wraps `SessionContext` + `PlatformService`
+ `MessageSender` + `InteractiveMessageFactory` + `SessionMessageFactory`. Key capabilities:

```java
String launchConversation(String conversationId, Map<String,String> arguments); // 29-35:
    // platform.launchConversation(sessionId, lang, convId, accountId, args)
    // then session.newConversationExecution(executionId, convId); returns executionId
void   submitAnswer(String parentMessageId, Map<String,Object> answers);        // 47-52 (requires execution id)
void   transitionToIdle();
void   setPickOneState(String options, int page);  String getPickOneOptions();  int getPickOnePage();
Language getPreferredLanguage(); void setPreferredLanguage(Language);
String getPhoneNumber(); String getInvokerId(); String getInvokerPhoneNumber();
String sendMessage(OutGoingRequestMessage message);
InteractiveMessageFactory getInteractiveMessageFactory();
void   stageAssistantMessage(String content);
```

### `RemoteToolMap` (`orchestrator/RemoteToolMap.java`) — dual index
```java
public RemoteToolMap(Map<ToolSpecification, RemoteTool> toolMap) { /* also builds nameMap by tool.name() */ }
public RemoteTool get(String name);                      // for dispatch
public List<ToolSpecification> getToolSpecifications();  // for the ChatRequest
```

## 2. The agent loop — `ToolExecutor.executeChatModel` (`orchestrator/ToolExecutor.java:62-143`)

Constants: `MAX_HISTORY_MESSAGES = 20`, `MAX_ITERATIONS = 10`.

Pseudocode (faithful to the code):
```
messages = [ SystemMessage(systemPrompt), ...context.loadLastNMessagesSafely(20).map(toChatMessage) ]
initialSize = messages.size()
responseFound=false; finalResponse=null; exitFlow=false; iteration=0

while (!responseFound && iteration < 10):
    iteration++
    request  = ChatRequest(messages, remoteToolMap.getToolSpecifications())
    response = getChatModel().chat(request)          // OpenAI call; InvalidRequestException → log history + rethrow
    ai       = response.aiMessage(); messages.add(ai)

    if ai.hasToolExecutionRequests():
        toolContext = ToolContext.create(context, platformService, messageSender,
                                         interactiveMessageFactory, sessionMessageFactory)
        result = executeTools(ai.toolExecutionRequests(), remoteToolMap, toolContext)
        messages.addAll(result.resultMessages)
        if result.exitFlowTriggered: exitFlow=true; responseFound=true; finalResponse=null
        elif result.shouldStopLoop:  responseFound=true; finalResponse=null
        else: /* Continue → loop again so the model reacts to tool results */
    else:
        finalResponse = ai.text(); responseFound=true

stageNewMessages(context, messages, initialSize)     // persist assistant + tool-result messages produced in the loop
if !responseFound: throw RuntimeException("Tool execution loop exceeded max iterations")
return exitFlow ? ExecutionResult.ofExitFlow()
                : (finalResponse==null ? ExecutionResult.ofSilence() : ExecutionResult.ofResponse(finalResponse))
```

Return type (`ToolExecutor.ExecutionResult`, 278-292): `(String response, boolean exitFlowTriggered)` with
factories `ofResponse` / `ofSilence` / `ofExitFlow`. The orchestrator reads it: `exitFlowTriggered` →
send Yes/No exit confirmation; non-null `response` → reply; null → stay silent.

### 2.1 `stageNewMessages` (249-262)
Everything appended to `messages` during the loop (assistant tool-call messages + tool-result messages) is
converted back via `sessionMessageFactory.fromChatMessage` and staged onto the `SessionContext`, so the full
tool round-trip is persisted (write-behind — flushed later by the orchestrator).

## 3. Tool dispatch & the two workarounds — `executeTools` (145-233)

```java
// partition the model's requests
for (ToolExecutionRequest req : requests) {
    if ("submit_answer".equals(req.name())) {
        submitAnswerRequests.add(req);
        mergedSubmitAnswerArgs.putAll(parseToolArguments(req.arguments()));   // WORKAROUND #1: merge
    } else {
        otherRequests.add(req);
    }
}
```

### Workaround #1 — `submit_answer` merge
Some models split a multi-field submit into **several** `submit_answer` calls. We merge all their JSON args
into one map and execute `submit_answer` **once**, then echo the single result back to **each** original
request id (so every tool-call has a matching `ToolExecutionResultMessage` — the API requires the pairing).

```java
// after executing the merged submit once:
for (ToolExecutionRequest req : submitAnswerRequests)
    resultMessages.add(ToolExecutionResultMessage.from(req, resultMessage));
```

### Other tools
Executed in order; each result becomes a `ToolExecutionResultMessage.from(request, text)`:
```java
RemoteTool executor = remoteToolMap.get(toolName);
if (executor == null) { resultMessages.add(ToolExecutionResultMessage.from(req, "Error: Unknown tool '"+toolName+"'")); continue; }
try {
    ToolResult result = executor.execute(toolContext, parseToolArguments(req.arguments()));
    resultMessages.add(ToolExecutionResultMessage.from(req, processToolResult(result, toolName)));
    if (result instanceof ToolResult.TriggerExitFlow) { exitFlowTriggered = true; shouldStopLoop = true; }
    else if (result instanceof ToolResult.StopLoop)    { shouldStopLoop = true; }
} catch (Exception e) {
    resultMessages.add(ToolExecutionResultMessage.from(req, "Error executing tool: " + e.getMessage()));  // loop can recover
}
```

`processToolResult` (235-247) maps the sealed result to the tool-result text the model sees; `TriggerExitFlow`
yields the fixed string `"Asking user's confirmation to exit the session."`. `parseToolArguments` (268-276)
uses a shared Jackson `ObjectMapper` and returns an empty map on parse failure.

## 4. API-validity guards (why the loop never sends a broken request)

### Workaround #2 — dangling tool-response filtering — `SessionContext.loadLastNMessagesSafely` (136-169)
When assembling the 20-message window, collect every tool-call id present in ASSISTANT messages, then **drop
any `TOOL_RESULT` whose `toolCallId` isn't backed by a call in the window**. Without this, a tool-call that
scrolled out of the window but whose result remained would make OpenAI reject the request
("tool result without matching call").

### Ordering + pairing validation — `SessionOrchestrator.validateMessageOrder` (473-510)
Before persisting a staged batch, throw `IllegalStateException` (with a full history dump) if:
1. message ids aren't strictly increasing (ULIDs are time-ordered → stored order == creation order), or
2. any `TOOL_RESULT` lacks a matching preceding assistant tool-call id in the batch.

## 5. The six tools

### 5.1 `ConversationRemoteTool` (dynamic) — launch a Krista workflow
`orchestrator/tools/ConversationRemoteTool.java`. A flyweight holding one `ConversationInfo` + its spec.
```java
public ToolResult execute(ToolContext ctx, Map<String,Object> arguments) {   // 35-47
    Map<String,String> stringArgs = convertArgumentsToStringMap(arguments);  // map clean names → real field names
    String executionId = ctx.launchConversation(conversation.id(), stringArgs);
    return ToolResult.stopLoop();   // workflow now runs async on the platform
}
```
Offered in **Idle** turns (unless a conversation is already running — see §7). Tool name = conversation name
sanitized to `[A-Za-z0-9_]`; description = the conversation's enhanced description.

### 5.2 `SubmitAnswerTool` (per-request) — submit collected field values
`orchestrator/tools/SubmitAnswerTool.java`. Name `submit_answer`. Constructed from the current
`CollectingFieldState`; its parameter schema is exactly the fields being collected.
- `buildSpecification()` (35-57): one property per field via `ConversationToolHelper.createSchemaProperty`;
  only truly-required fields marked required; clean field names preserve Unicode (Hindi/Marathi) while
  replacing punctuation with `_` (`getCleanFieldName`, 172-186).
- `execute()` (64-99): validate → convert → submit → transition:
```java
String validationError = validateFields(rawValues);
if (validationError != null) return ToolResult.continueWith(validationError);  // LLM re-asks
Map<String,Object> converted = /* FieldUtils.getFieldValue per field */;
toolContext.submitAnswer(this.parentMessageId, converted);   // → platform.submitAnswer
toolContext.transitionToIdle();                              // Collecting → Idle
return ToolResult.stopLoop();
```
- `validateFields`/`validateFieldValue` (101-170): required-present, number min/max, pickOne/pickList
  membership.

### 5.3 `PickOneFieldTool` (singleton) — render an interactive list
`orchestrator/tools/PickOneFieldTool.java`. Name `show_options`. `PAGE_SIZE = 9`.
```java
private static final ToolSpecification SPECIFICATION = ToolSpecification.builder()   // 26-38
    .name("show_options")
    .description("Display options to the user in a list")
    .parameters(JsonObjectSchema.builder()
        .addProperty("question", JsonStringSchema.builder().description("The question to ask the user").build())
        .addProperty("options",  JsonStringSchema.builder().description("All options separated by semicolon (;)").build())
        .required(List.of("question","options")).build())
    .build();
```
`execute()` (122-160): split options on `;`, `setPickOneState`, `buildSections(...)` (paginated WhatsApp
interactive list; row/section titles ≤24 chars, `~` splits row~section title, localized "View More" row when
more pages exist), send the list, and return `stopLoop("Success: The options are presented to the user")`.
`isViewMoreTrigger` recognizes localized "view more" strings (`view more` / `और देखें` / `अधिक दाखवा`); the
orchestrator's `handleCollecting` handles pagination *without* an LLM call.

### 5.4 `ExitSessionTool` (singleton) — user wants to quit
`orchestrator/tools/ExitSessionTool.java`. Name `exit_session`, no params.
```java
public ToolResult execute(ToolContext ctx, Map<String,Object> args) { return ToolResult.triggerExitFlow(); }
```
Offered in Idle + Collecting turns. The loop turns this into `ExecutionResult.ofExitFlow()`; the orchestrator
then sends a Yes/No confirmation (the actual exit is a state, not a tool).

### 5.5 `SwitchLanguageTool` (singleton) — change language
`orchestrator/tools/SwitchLanguageTool.java`. Name `switch_language`, enum param `language_code` (en/hi/mr).
`execute` validates and calls `toolContext.setPreferredLanguage(...)`, returning `continueWith(...)`.
**Note:** present but not wired into the tool maps in the reviewed orchestrator paths (worth flagging).

### 5.6 `analyze_session_start` (init classifier)
Not a `RemoteTool` — a one-off `ToolSpecification` inside `LlmService.analyzeInitMessage` used for
structured output (language + greeting/query). See `04-langchain4j-usage.md` §5.

## 6. Dynamic tool generation from Krista conversations

### Registry — `ConversationToolRegistry` (`orchestrator/tools/ConversationToolRegistry.java`)
`@Service`. Holds `CopyOnWriteArrayList<ConversationInfo>` + `ConcurrentHashMap<ToolSpecification, RemoteTool>`.
```java
public void syncConversations() {                          // 47-59
    List<ConversationInfo> loaded = platformService.getConversationsFromBackend();
    conversations.clear(); conversationTools.clear();
    conversations.addAll(loaded);
    conversationTools.putAll(rebuildConversationTools());   // one ConversationRemoteTool per conversation
}
private static ConversationRemoteTool getConversationRemoteTool(ConversationInfo c) {   // 30-45
    String toolName = c.name().replaceAll("[^A-Za-z0-9]", "_");
    ToolSpecification spec = ToolSpecification.builder()
        .name(toolName).description(c.description())
        .parameters(ConversationToolHelper.buildParameters(c)).build();
    return new ConversationRemoteTool(c, spec);
}
```
Called from `INVOKER_LOADED`, `INVOKER_UPDATED`, and the "Conversations" custom tab
(`ConversationResource.reloadConversations`). `conversationTools()` is what the orchestrator injects into
Idle turns.

### Schema generation — `ConversationToolHelper` (`orchestrator/tools/ConversationToolHelper.java`)
```java
public static JsonSchemaElement createSchemaProperty(ConversationField field) {   // 37-84
    return switch (field.getType()) {
        case bool                 -> JsonBooleanSchema.builder().build();
        case pickOne, pickList    -> /* JsonEnumSchema of options (with a note to call show_options first) */;
        case number               -> /* JsonNumberSchema with min/max in the description */;
        case date                 -> JsonStringSchema.builder().description("format: date (YYYY-MM-DD)").build();
        case date_time            -> JsonStringSchema.builder().description("format: date-time (ISO 8601)").build();
        case time                 -> JsonStringSchema.builder().description("format: time (HH:MM:SS)").build();
        case location             -> /* JsonStringSchema with a lat;long;name;address example */;
        default                   -> JsonStringSchema.builder().build();
    };
}
```
`getCleanFieldName` = `name.toLowerCase().replace(' ', '_')`; `buildParameters` marks all conversation fields
required. `ConversationInfo` is the in-memory record (`id, name, description, topicId, topicName,
List<ConversationField> fields, invokerId, tenantId, roles`).

## 7. Tool suppression while a conversation runs

`SessionOrchestrator.executeIdleLogic` (730-765):
```java
Map<ToolSpecification, RemoteTool> conversationTools;
if (context.getConversationExecutionId() != null) {
    conversationTools = new HashMap<>();                          // suppress: a workflow is already running
} else {
    conversationTools = conversationToolRegistry.conversationTools();
}
Map<ToolSpecification, RemoteTool> toolMap = new HashMap<>(conversationTools);
toolMap.put(ExitSessionTool.INSTANCE.specification(), ExitSessionTool.INSTANCE);
var executionResult = toolExecutor.executeChatModel(context,
        InstructionBuilder.create().withCoreIdentity()
            .withLanguageEnforcement(context.getPreferredLanguage().getCode()).build(),
        new RemoteToolMap(toolMap));
```
The execution id is cleared by `conversationCompleted` (the ActiveMQ "inactive" event) — re-enabling
conversation tools on the next Idle turn.

## 8. Which tools are offered when

| Turn / state | Tool set assembled |
|--------------|--------------------|
| **Idle**, no conversation running | all dynamic conversation tools + `exit_session` |
| **Idle**, conversation running | *only* `exit_session` (conversation tools suppressed) |
| **Collecting** (`handleCollecting`, 800-805) | fresh `submit_answer` (this state) + `show_options` + `exit_session` |
| **Ask with fields** (`processAskMessage`, 263-267) | fresh `submit_answer` + `show_options` |
| **Agent instruction** (`processAgentInstruction`, 381-388) | conversation tools |
| **First message** | one-off `analyze_session_start` (structured output) |

## 9. End-to-end walkthroughs

**A) User asks a task in Idle → workflow launches**
1. `handleIdle` → `executeIdleLogic` builds tool map (conversation tools + `exit_session`).
2. `ToolExecutor` loop: model calls a `ConversationRemoteTool`.
3. Tool → `ctx.launchConversation(convId, args)` → `platform.launchConversation` + record execution id →
   `stopLoop()`.
4. Loop stops (silence). The Krista workflow runs async; subsequent Idle turns suppress conversation tools.

**B) Platform asks a question with fields → user answers**
1. `handlePlatformMessage` → `AskData` → `processAskMessage`: build fields, set `CollectingFieldState`
   (Idle→Collecting), tool map = `submit_answer` + `show_options`; loop asks the first question (using
   `show_options` for pickOne).
2. User replies → `handleCollecting`: loop runs; model calls `submit_answer`.
3. `SubmitAnswerTool.execute`: validate → `ctx.submitAnswer(parentMessageId, converted)` →
   `transitionToIdle()` → `stopLoop()`. Back to Idle.

**C) Multi-field submit split by the model** → `executeTools` merges the several `submit_answer` calls into
one execution, echoing the result to each call id (Workaround #1).

**D) User types "exit"** → global keyword or `exit_session` tool → `TriggerExitFlow` → orchestrator sends
Yes/No buttons. "Yes" → transcript event + cleanup; "No" → restore previous state and re-dispatch.

## 10. Gotchas & talking points

- **Bounded loop** (`MAX_ITERATIONS=10`) prevents infinite tool ping-pong; exceeding it throws (fail-loud).
- **`submit_answer` merge** and **dangling-tool-result filtering** are the two most interview-worthy details
  — both are defenses against real LLM/API behavior.
- **`ToolResult` sealed signals** cleanly separate "keep looping" / "stop" / "hand off to exit" from the
  tool's user-facing text.
- **Dynamic + per-request tools** are the concrete reason `AiServices`/`@Tool` wasn't used.
- **`SwitchLanguageTool` is unwired** in the reviewed paths — a good "cleanup candidate" answer.
- **Pagination without the LLM**: "View More" is handled in `handleCollecting` directly, saving a model call.
