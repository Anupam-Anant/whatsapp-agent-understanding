# 04 — LangChain4J Usage

> **Scope.** Every LangChain4J concept this project uses and exactly how. This is the API layer that sits
> on top of the OpenAI model (`03-openai-integration.md`) and underneath the tool loop
> (`05-tool-calling-deep-dive.md`). The key theme: this codebase uses LangChain4J's **low-level chat API**
> and deliberately avoids the high-level declarative `AiServices`/`@Tool` layer.

## 0. Version & artifacts

`build.gradle:45-46`:

```gradle
implementation 'dev.langchain4j:langchain4j:1.9.1'
implementation 'dev.langchain4j:langchain4j-open-ai:1.9.1'
```

- `langchain4j` — core abstractions (messages, `ChatModel`, `ChatRequest`/`ChatResponse`, tool specs, JSON
  schema).
- `langchain4j-open-ai` — the `OpenAiChatModel` implementation.

> LangChain4J is the **Java port of LangChain**. There is **no LangGraph** (Python-only). If the interview
> phrasing is "LangGraph," correct it to LangChain4J + a hand-rolled state machine.

## 1. Type catalog (map each to the OpenAI wire concept)

| LangChain4J type | Package | Role in this app |
|------------------|---------|------------------|
| `ChatModel` | `dev.langchain4j.model.chat` | The model abstraction; impl `OpenAiChatModel` |
| `ChatRequest` / `ChatResponse` | `dev.langchain4j.model.chat.request` / `.response` | Request (messages + tool specs) / response wrapper (`.aiMessage()`) |
| `SystemMessage` | `dev.langchain4j.data.message` | System prompt (assembled by `InstructionBuilder`) |
| `UserMessage` | `…data.message` | A user turn |
| `AiMessage` | `…data.message` | Assistant turn; `.text()`, `.hasToolExecutionRequests()`, `.toolExecutionRequests()` |
| `ToolExecutionResultMessage` | `…data.message` | Result of running a tool, fed back to the model |
| `ToolSpecification` | `dev.langchain4j.agent.tool` | A tool/function definition (name, description, params) |
| `ToolExecutionRequest` | `dev.langchain4j.agent.tool` | The model's request to call a tool (`id`, `name`, `arguments` JSON) |
| `JsonObjectSchema` + `JsonStringSchema` / `JsonNumberSchema` / `JsonBooleanSchema` / `JsonEnumSchema` / `JsonSchemaElement` | `dev.langchain4j.model.chat.request.json` | Programmatic JSON-Schema for tool parameters |
| `InvalidRequestException` | `dev.langchain4j.exception` | Thrown on malformed requests (caught in the loop) |

Note there is also a **local** `chat/ChatMessage` record (role/content/timestamp) and a `chat/ChatService`
interface — these are lightweight app-level types, *not* the LangChain4J `ChatMessage`. The LangChain4J
`dev.langchain4j.data.message.ChatMessage` is the one used on the wire.

## 2. Two calling styles

### 2.1 Single-shot (no loop) — `orchestrator/LlmService`

`LlmService` (`@Service`, injects `ChatLanguageModelProvider` + `SessionMessageFactory`) holds all the
non-looping calls. `MAX_HISTORY_MESSAGES = 20` (line 34).

- **`executeSimpleChat(context, systemPrompt)`** (lines 120-132) — system prompt + last-20 history → one
  `chat(messages)` → stage the assistant reply → return text. Used for "just relay/translate this":
  complex-field asks, agent instructions, inform messages.
- **`classify(input, options)`** (lines 134-167) — short-circuits 0/1 options; builds a numbered
  classification prompt; matches the model's text against the option list (case-insensitive `contains`);
  falls back to the first option. Used by `FieldCollector` for boolean fields.
- **`extract(input, fieldType)`** (lines 169-187) — "extract the X, return ONLY the value" → trimmed text.
- **`analyzeInitMessage(text)`** (lines 46-118) — **structured output via a tool schema** (see §5).
- **`detectLanguage(text)`** (lines 200-313) — a **non-LLM** heuristic fallback (romanized word lists +
  Devanagari ratio, Marathi tie-break). Present as a utility; the shipped init path uses the LLM classifier.

### 2.2 The agent loop — `orchestrator/ToolExecutor`

Iterative `chat(request)` with tool specs, executing tools and feeding results back until the model returns
plain text or a tool signals stop. Full treatment in `05-tool-calling-deep-dive.md`.

## 3. Request/response anatomy

**Building a request with tools (`ToolExecutor.executeChatModel`, 88-95):**
```java
List<ToolSpecification> toolSpecifications = remoteToolMap.getToolSpecifications();
ChatRequest chatRequest = ChatRequest.builder()
        .messages(messages)                     // List<dev.langchain4j.data.message.ChatMessage>
        .toolSpecifications(toolSpecifications)  // the tools the model may call this turn
        .build();
ChatResponse response = getChatModel().chat(chatRequest);
AiMessage aiMessage = response.aiMessage();
```

**Reading the response:**
```java
if (aiMessage.hasToolExecutionRequests()) {
    for (ToolExecutionRequest req : aiMessage.toolExecutionRequests()) {
        req.id();          // tool-call id (must be paired with a ToolExecutionResultMessage)
        req.name();        // tool/function name
        req.arguments();   // JSON string of args
    }
} else {
    String text = aiMessage.text();   // final natural-language answer
}
```

`OpenAiChatModel` also supports the overload `chat(List<ChatMessage>)` (no tools) used by
`LlmService.executeSimpleChat`.

## 4. Message ⟷ persistence bridge (the crux)

LangChain4J messages are **not persisted directly**. The **chat memory is the persisted transcript**
(Infinispan `SessionMessage` BOs), windowed per call. Two converters bridge the two worlds.

### 4.1 Stored BO → LangChain4J message — `SessionMessage.toChatMessage()` (`persistence/bo/SessionMessage.java:70-95`)

```java
public ChatMessage toChatMessage() {
    switch (this.getRole()) {
        case USER      -> { return UserMessage.from(content); }
        case ASSISTANT -> {
            List<ToolExecutionRequest> reqs = new ArrayList<>();
            for (ToolCall tc : toolCalls)
                reqs.add(ToolExecutionRequest.builder()
                        .id(tc.getId()).name(tc.getName()).arguments(tc.getArguments()).build());
            return AiMessage.from(content, reqs);   // rebuilds assistant tool-calls
        }
        case SYSTEM      -> { return SystemMessage.from(content); }
        case TOOL_RESULT -> { return ToolExecutionResultMessage.from(toolCallId, toolName, content); }
    }
    throw new IllegalArgumentException("Unknown message role: " + this.getRole());
}
```

`MessageRole` enum (proto-numbered): `USER(0)`, `ASSISTANT(1)`, `SYSTEM(2)`, `TOOL_RESULT(3)`. The nested
`ToolCall` record stores `(id, name, arguments)`.

### 4.2 LangChain4J message → stored BO — `SessionMessageFactory` (`service/SessionMessageFactory.java`)

```java
public SessionMessage fromChatMessage(String sessionId, ChatMessage chatMessage) {   // 53-61
    if (chatMessage instanceof AiMessage a)                    return assistant(sessionId, a);
    else if (chatMessage instanceof ToolExecutionResultMessage t) return toolResponse(sessionId, t);
    else throw new IllegalArgumentException("Unsupported chat message type: " + chatMessage.getClass());
}

public SessionMessage assistant(String sessionId, AiMessage aiMessage) {              // 154-167
    return SessionMessage.builder()
            .id(ulid()).sessionId(sessionId).tenantId(invokerContext.getInvokerId())
            .timestamp(now()).role(ASSISTANT)
            .content(aiMessage.text())
            .toolCalls(getToolCalls(aiMessage.toolExecutionRequests()))  // → List<ToolCall>
            .sequenceNumber(seq()).build();
}
```

Every message gets a **monotonic ULID** id (`UlidCreator.getMonotonicUlid()`) and a per-session sequence
number. ULIDs are lexicographically time-ordered — this underpins ordering (see §6) and
`findLastNBySessionId ORDER BY id DESC`.

## 5. Structured output via a tool schema — `analyzeInitMessage`

First-message language + intent detection uses a **one-off `ToolSpecification` as a JSON-mode contract**
(`LlmService.analyzeInitMessage`, 46-118):

```java
ToolSpecification initTool = ToolSpecification.builder()
    .name("analyze_session_start")
    .description("Analyzes the first message to determine language and intent.")
    .parameters(JsonObjectSchema.builder()
        .addProperty("language", JsonEnumSchema.builder()
            .description("Detected language code").enumValues("en", "hi", "mr").build())
        .addProperty("type", JsonEnumSchema.builder()
            .description("Message type: 'greeting' ... or 'query' ...").enumValues("greeting", "query").build())
        .required("language", "type")
        .build())
    .build();

List<ChatMessage> messages = List.of(
    new SystemMessage("You are a helpful assistant. Call the analyze_session_start tool."),
    new UserMessage(prompt /* language + greeting heuristics + the user's message */));

ChatResponse response = chatModel.chat(ChatRequest.builder()
        .messages(messages).toolSpecifications(initTool).build());
AiMessage ai = response.aiMessage();

if (ai.hasToolExecutionRequests()) {
    ToolExecutionRequest tr = ai.toolExecutionRequests().get(0);
    Map<String,Object> args = OBJECT_MAPPER.readValue(tr.arguments(), new TypeReference<>() {});
    String language = (String) args.getOrDefault("language", "en");
    boolean isGreeting = "greeting".equalsIgnoreCase((String) args.getOrDefault("type", "greeting"));
    return new InitMessageAnalysis(language, isGreeting);
}
return new InitMessageAnalysis("en", true);   // safe fallback on any failure
```

**Interview framing:** "We force JSON structured output by declaring a tool with an enum schema and reading
the tool-call arguments, rather than parsing free text — it's robust and validated by the schema." Tool
arguments are parsed with a shared Jackson `ObjectMapper`.

## 6. History windowing & API-validity

Both the loop and single-shot calls prepend a `SystemMessage` then append the last
`MAX_HISTORY_MESSAGES = 20` stored messages via `SessionContext.loadLastNMessagesSafely`
(`orchestrator/SessionContext.java:136-169`). That method also **filters dangling tool results** — any
`TOOL_RESULT` whose `toolCallId` has no matching assistant tool-call *within the window* is dropped, because
OpenAI rejects a tool result with no matching call (details in `05-tool-calling-deep-dive.md`). Before
persisting, `SessionOrchestrator.validateMessageOrder` enforces strictly-increasing ULID order + tool-call
pairing.

## 7. Why NOT `AiServices` / `@Tool` (design rationale)

- **Dynamic tools.** Tools are generated at runtime from the tenant's Krista conversations
  (`ConversationToolRegistry`); the static, annotation-bound `@Tool` model can't express that.
- **Per-request stateful tools.** `SubmitAnswerTool` is constructed from the *current* `CollectingFieldState`
  (its parameter schema is the exact fields being collected). `@Tool` methods are singletons.
- **Explicit control.** The team needs to own the iteration bound (`MAX_ITERATIONS=10`), message
  persistence/ordering, the `submit_answer` merge workaround, dangling-result filtering, and the mapping of
  tool outcomes to state-machine signals (`ToolResult`). A hand-rolled loop makes all of that observable and
  unit-testable.

## 8. Interview talking points

- "We use LangChain4J's low-level chat API — `ChatModel.chat(ChatRequest)` with `ToolSpecification`s — and a
  hand-written agent loop. The high-level `AiServices` doesn't fit dynamic, per-request tools."
- "Chat memory *is* the persisted transcript. We convert both ways between our Protostream `SessionMessage`
  BOs and LangChain4J messages, so history survives restarts and is queryable/auditable."
- "Structured output is done with tool/enum schemas, not regex on free text."
- "We window to 20 messages and defensively strip dangling tool results so the OpenAI request always
  validates."
