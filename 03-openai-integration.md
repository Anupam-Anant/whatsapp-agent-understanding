# 03 — OpenAI Integration

> **Scope.** Exactly how this project talks to OpenAI: the single provider class, the model parameters,
> runtime hot-swap of configuration, where the model is consumed, failure handling, security, and
> cost/model-selection reasoning. Companion files: `04-langchain4j-usage.md` (the API layer on top of the
> model) and `05-tool-calling-deep-dive.md` (the agent loop that drives it).

## 0. One-paragraph summary

Every OpenAI call in the app goes through **one** LangChain4J `ChatModel` instance, created and owned by
`chat/ChatLanguageModelProvider` (an HK2 `@Service` singleton). The concrete implementation is
LangChain4J's `OpenAiChatModel`, configured from two admin-managed extension fields — **"OpenAI Model"**
and **"OpenAI secret key"** — and rebuilt lazily whenever those change. Default model is **`gpt-4o-mini`**.
There is **no OpenAI Realtime API** and **no LangGraph** here (the old `heloai-waba-extension` used Realtime
+ a thread pool; this rewrite replaced it with the Completions/chat API).

## 1. Where OpenAI appears

| Consumer | Purpose |
|----------|---------|
| `orchestrator/ToolExecutor.executeChatModel` | The agent tool-calling loop (main path) |
| `orchestrator/LlmService.analyzeInitMessage` | First-message language + greeting/query detection (structured output via a tool) |
| `orchestrator/LlmService.executeSimpleChat` | "Just relay/translate this" single-shot chat (complex-field asks, agent instructions, inform messages) |
| `orchestrator/LlmService.classify` / `extract` | Small prompt-only helpers used by `FieldCollector` |

All of them obtain the model the same way: `modelProvider.getChatModel()`.

## 2. Configuration surface (the two OpenAI fields)

Declared on the extension class (`KristaWhatsAppExtension.java:40-41`):

```java
@Field.Text(name = Constants.LLM_MODEL)                    // "OpenAI Model"
@Field.Text(name = Constants.LLM_API_KEY, isSecured = true) // "OpenAI secret key" (secured)
```

- `LLM_API_KEY` is `isSecured = true` → stored encrypted, masked in the admin UI, never rendered back.
- Both are validated as non-blank in `VALIDATE_ATTRIBUTES` (`KristaWhatsAppExtension.validateAttributes`,
  lines 147-159) before the extension is allowed to run.

## 3. The provider — `chat/ChatLanguageModelProvider`

Full class (annotated for reference; `ChatLanguageModelProvider.java`):

```java
@Service
public class ChatLanguageModelProvider {

    private static final String DEFAULT_MODEL = "gpt-4o-mini";            // line 25
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private volatile String currentApiKey;
    private volatile String currentModel;
    private volatile ChatModel chatLanguageModel;    // dev.langchain4j.model.chat.ChatModel

    @Inject public ChatLanguageModelProvider() {}

    // Called from INVOKER_LOADED and INVOKER_UPDATED
    public void updateConfiguration(String apiKey, String model) {        // lines 47-73
        if (apiKey == null || apiKey.isBlank()) { /* warn + return */ return; }
        String effectiveModel = (model != null && !model.isBlank()) ? model : DEFAULT_MODEL;
        rwLock.writeLock().lock();
        try {
            boolean apiKeyChanged = !Objects.equals(this.currentApiKey, apiKey);
            boolean modelChanged  = !Objects.equals(this.currentModel, effectiveModel);
            if (apiKeyChanged || modelChanged) {
                this.currentApiKey = apiKey;
                this.currentModel  = effectiveModel;
                this.chatLanguageModel = null;    // invalidate → lazy rebuild on next getChatModel()
            }
        } finally { rwLock.writeLock().unlock(); }
    }

    public ChatModel getChatModel() {                                      // lines 82-119
        // fast path: read lock
        rwLock.readLock().lock();
        try { if (chatLanguageModel != null) return chatLanguageModel; }
        finally { rwLock.readLock().unlock(); }

        // slow path: write lock + double-check + build
        rwLock.writeLock().lock();
        try {
            if (chatLanguageModel != null) return chatLanguageModel;
            if (currentApiKey == null || currentApiKey.isBlank())
                throw new IllegalStateException("LLM API key not configured. Call updateConfiguration() first.");

            this.chatLanguageModel = OpenAiChatModel.builder()             // lines 107-113
                    .apiKey(currentApiKey)
                    .modelName(currentModel)
                    .timeout(Duration.ofSeconds(60))
                    .maxRetries(2)
                    .temperature(0.7)
                    .build();
            return chatLanguageModel;
        } finally { rwLock.writeLock().unlock(); }
    }
}
```

### 3.1 What to notice

- **Singleton, not per-call.** `OpenAiChatModel` wraps an HTTP client and connection pool; building it per
  request would be wasteful. It's built once and cached in a `volatile` field.
- **Double-checked locking with a `ReentrantReadWriteLock`.** Hot reads take the cheap **read lock**; the
  expensive rebuild happens under the **write lock** with a re-check. This keeps the read path (called on
  every LLM turn) contention-free while making config changes thread-safe.
- **Lazy rebuild on change.** `updateConfiguration` doesn't build a model — it just nulls the cached one.
  The next `getChatModel()` rebuilds with the new key/model. So a config change is O(1) and the cost is
  paid on the next actual use.
- **Model fallback.** Blank "OpenAI Model" → `gpt-4o-mini`.
- **Fail-fast if unconfigured.** `getChatModel()` throws `IllegalStateException` if no key is set.

### 3.2 Model parameters (and the reasoning)

| Param | Value | Why |
|-------|-------|-----|
| `modelName` | `gpt-4o-mini` (default) | Cheap + fast; the workload is intent-routing, field extraction, and short natural-language replies — not long-form generation. Overridable per deployment. |
| `temperature` | `0.7` | Natural phrasing across en/hi/mr. (Arguably high for a tool-routing bot — a lower value would make routing more deterministic; note this as a possible improvement.) |
| `timeout` | `60s` | WhatsApp users tolerate a short wait; the webhook already returned 200 and processing is async on a virtual thread. |
| `maxRetries` | `2` | Rides out transient 429/5xx without unbounded latency. |

## 4. Lifecycle wiring (how config reaches the provider)

`KristaWhatsAppExtension` injects the provider and updates it on load and on every settings change:

```java
private void updateChatLanguageModelConfiguration(Map<String, Object> attributes) { // lines 240-250
    String llmApiKey = (String) attributes.get(Constants.LLM_API_KEY);
    String llmModel  = (String) attributes.get(Constants.LLM_MODEL);
    if (llmApiKey != null && !llmApiKey.isBlank()) {
        chatLanguageModelProvider.updateConfiguration(llmApiKey, llmModel);
    } else { /* warn: not updated */ }
}
```

- `INVOKER_LOADED` → `onInvokerLoad()` calls it with `invokerContext.getAttributes()` (line 115).
- `INVOKER_UPDATED` → `onInvokerUpdate(old,new)` re-validates then calls it with `newAttributes` (line 181).

**Result:** an admin can change the OpenAI model or rotate the key from the Krista UI and the running
extension picks it up on the next message — **no redeploy**.

## 5. How the model is consumed (call sites)

`ToolExecutor` (the loop) and `LlmService` both just call `modelProvider.getChatModel()` and then
`chat(...)`. Two representative shapes:

**Loop call (`ToolExecutor.executeChatModel`, lines 88-95):**
```java
ChatRequest chatRequest = ChatRequest.builder()
        .messages(messages)
        .toolSpecifications(remoteToolMap.getToolSpecifications())
        .build();
ChatResponse response = getChatModel().chat(chatRequest);   // getChatModel() → modelProvider.getChatModel()
```

**Single-shot call (`LlmService.executeSimpleChat`, lines 120-132):**
```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(SystemMessage.from(systemPrompt));
for (SessionMessage m : context.loadLastNMessagesSafely(MAX_HISTORY_MESSAGES)) // 20
    messages.add(m.toChatMessage());
ChatResponse response = modelProvider.getChatModel().chat(messages);
```

See `04-langchain4j-usage.md` for the request/response anatomy and the message conversion.

## 6. Failure handling & degradation

- **No key configured:** `getChatModel()` throws `IllegalStateException`. In practice `VALIDATE_ATTRIBUTES`
  blocks this before runtime.
- **Init detection failure:** `LlmService.analyzeInitMessage` wraps the whole call in try/catch and returns
  a safe default `new InitMessageAnalysis("en", true)` (English greeting) on any exception (lines 113-117).
- **Loop-level:** `ToolExecutor` catches LangChain4J `InvalidRequestException`, logs the full message
  history, and rethrows (lines 96-99); the orchestrator's outer catch (`handleInternalFailure`) then sends
  the user a localized error message so they always get *some* reply.
- **Retries:** handled inside `OpenAiChatModel` (`maxRetries=2`).

## 7. Security & observability

- The OpenAI key is a **secured field** (`isSecured=true`) — encrypted at rest, masked in UI.
- `resources/log4j2.xml` sets `dev.langchain4j=INFO`, so request/response bodies (which could contain the
  prompt and user PII) are **not** emitted at DEBUG.
- The key is never logged by `ChatLanguageModelProvider` (only the model name is logged on rebuild).

## 8. Interview talking points

- "OpenAI access is centralized behind a thread-safe, hot-swappable provider singleton — read-write lock
  with double-checked lazy init, so the read path is contention-free and config changes are picked up
  without a redeploy."
- "We default to `gpt-4o-mini` because the task is routing + extraction + short replies, and cost matters
  at WhatsApp volume; the model is per-deployment configurable."
- "The key is a secured extension field and LangChain4J logging is pinned to INFO so prompts/keys don't
  leak."
- **Possible improvements to raise proactively:** lower `temperature` for more deterministic tool routing;
  make `timeout`/`maxRetries` configurable; add token-usage metrics; consider a cheaper model for the init
  classifier vs. the main loop.
