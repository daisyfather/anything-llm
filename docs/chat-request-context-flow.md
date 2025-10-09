# Chat Request & Context Management Flow

This document explains how AnythingLLM handles an incoming chat request, how it assembles the
conversation context that will be sent to the language model, and how the system streams the
response back to the caller. Every step references the exact modules in the `server/` runtime so a
contributor can trace the execution in code.

## 1. Request intake and validation
1. **Endpoint** – Chats arrive through the SSE endpoint defined in
   `server/endpoints/chat.js` at `POST /workspace/:slug/stream-chat` (or its thread variant). The
   handler is wrapped with the `validatedRequest`, `flexUserRoleValid`, and `validWorkspaceSlug`
   middleware stack to ensure the session cookie/API key is valid and the workspace exists before
   any chat work begins.【F:server/endpoints/chat.js†L24-L63】
2. **Session & payload** – After middleware, the handler resolves the authenticated user with
   `userFromSession` and pulls the JSON body via `reqBody`, enforcing a non-empty `message` and
   optional `attachments` array.【F:server/endpoints/chat.js†L28-L46】
3. **SSE headers & quota** – Response headers are switched to Server-Sent Events so downstream code
   can stream tokens incrementally. In multi-user mode the handler calls `User.canSendChat`; if the
   sender exhausted their quota an `abort` chunk is written immediately and the flow stops.【F:server/endpoints/chat.js†L47-L76】
4. **Entry into chat pipeline** – Once validated the handler calls
   `streamChatWithWorkspace(response, workspace, message, chatMode, user, thread, attachments)` and
   defers the rest of the work to the chat utilities module.【F:server/endpoints/chat.js†L77-L105】

## 2. Command & agent short-circuits
Inside `streamChatWithWorkspace` the system performs two early exits before talking to any LLM:

1. **Slash commands** – `grepCommand` tests the message against builtin commands such as `/reset`. If
   the message maps to a command, the command handler executes immediately and writes its own SSE
   payload without touching the rest of the pipeline.【F:server/utils/chats/stream.js†L26-L44】
2. **Agent flows** – `grepAgents` inspects whether the workspace routes this chat through an agent
   flow (tool executors, automations, etc.). Agent chats manage their own streaming contract, so the
   standard retrieval flow halts when one is detected.【F:server/utils/chats/stream.js†L46-L57】

## 3. Provider bootstrap & workspace guards
1. **LLM connector** – `getLLMProvider` instantiates the chat connector for the workspace’s selected
   provider/model. This object exposes capabilities such as `promptWindowLimit`, message compression,
   streaming helpers, and embedding utilities.【F:server/utils/chats/stream.js†L59-L69】
2. **Vector store** – `getVectorDbClass` returns the configured vector database provider wrapper
   (LanceDB by default). The wrapper offers namespace management and similarity search helpers used
   later in the flow.【F:server/utils/chats/stream.js†L59-L69】
3. **Namespace sanity** – The code checks whether the workspace namespace exists and contains
   embeddings. For `query` chat mode (strict retrieval) an empty namespace triggers an immediate
   refusal response that is streamed back and persisted as an uninterpreted answer to avoid
   hallucinations.【F:server/utils/chats/stream.js†L71-L116】

## 4. Conversation history & fixed context assembly
The next section gathers every contextual artifact that should accompany the user prompt:

1. **Historical messages** – `recentChatHistory` fetches the most recent persisted chat turns for the
   `(workspace, user, thread)` combination, respecting the workspace message limit. It returns both
   the raw ORM rows and a `chatHistory` array already shaped for prompt injection.【F:server/utils/chats/index.js†L43-L71】
2. **Pinned documents** – `DocumentManager.pinnedDocs()` loads the JSON blobs for any documents a
   user has pinned in the workspace. Pinned context is added ahead of everything else, and the code
   tracks their identifiers so duplicate citations can be filtered out later.【F:server/utils/chats/stream.js†L118-L152】
3. **One-off parsed files** – `WorkspaceParsedFiles.getContextFiles` returns temporary files that the
   user uploaded for this chat or thread. Each parsed file contributes its plain text to the context
   list and a truncated preview for the eventual source list.【F:server/utils/chats/stream.js†L154-L170】

## 5. Vector retrieval & citation window management
1. **Similarity search** – If embeddings exist the system calls
   `VectorDb.performSimilaritySearch`. The LanceDB implementation embeds the updated user message via
   `LLMConnector.embedTextInput`, runs a cosine-similarity search (optionally reranking results), and
   returns both `contextTexts` and normalized `sources`. Pinned document identifiers are passed as a
   filter so that citations do not duplicate pinned context.【F:server/utils/chats/stream.js†L172-L205】【F:server/utils/vectorDbProviders/lance/index.js†L288-L458】
2. **Failure handling** – When the vector layer reports an error the function emits an `abort` chunk
   and stops so the client sees a graceful error instead of hanging.【F:server/utils/chats/stream.js†L207-L217】
3. **Backfilling citations** – The helper `fillSourceWindow` ensures the response includes a full set
   of references. If the live search returned fewer documents than requested, the system backfills
   citations from recent chat history that already carried vector-backed sources, again skipping any
   pinned identifiers.【F:server/utils/chats/stream.js†L219-L233】【F:server/utils/helpers/chat/index.js†L382-L442】
4. **Query-mode guard** – When the chat is in `query` mode and no context chunks survived the search
   and backfill, the flow refuses to call the LLM and responds with the workspace’s query refusal
   message, preventing unsupported answers.【F:server/utils/chats/stream.js†L235-L260】

## 6. Prompt construction & token budgeting
1. **System prompt expansion** – `chatPrompt` loads the workspace/system prompt template and expands
   any dynamic variables (user name, workspace metadata) via `SystemPromptVariables` before it is
   passed downstream.【F:server/utils/chats/index.js†L73-L95】
2. **Prompt compression** – `LLMConnector.compressMessages` receives the system prompt, user prompt,
   assembled `contextTexts`, chat history, and attachments. The connector delegates to the
   `messageArrayCompressor` helper which aggressively trims the system message and prior turns to fit
   within the model’s token window while preserving the current user message intact.【F:server/utils/chats/stream.js†L262-L279】【F:server/utils/helpers/chat/index.js†L1-L214】

## 7. Generation, streaming, and persistence
1. **Non-streaming connectors** – If a provider does not support streaming the connector performs a
   standard `getChatCompletion` call and immediately writes the full response as a single SSE chunk
   followed by a finalize event.【F:server/utils/chats/stream.js†L281-L304】
2. **Streaming connectors** – For streaming providers the connector opens a stream via
   `streamGetChatCompletion` and pipes partial tokens through `handleStream`, which repeatedly calls
   `writeResponseChunk` so the frontend can render incremental updates.【F:server/utils/chats/stream.js†L305-L316】
3. **Persistence** – Successful completions are inserted into `WorkspaceChats.new`, storing the user
   prompt, the assistant reply, the sources that powered the answer, metrics, and attachment metadata
   so they can be replayed in future history fetches.【F:server/utils/chats/stream.js†L318-L340】
4. **Finalize event** – Regardless of streaming mode the pipeline emits a final SSE action with the
   chat record ID and performance metrics. Clients treat this as the signal that the stream finished
   cleanly.【F:server/utils/chats/stream.js†L342-L354】

## 8. Conversation context lifecycle recap
Putting everything together, the system maintains conversational context through three cooperating
layers:

```mermaid
graph TD
  A[Workspace chat history (DB)] -- recentChatHistory --> D[Prompt compressor]
  B[Pinned docs & parsed files] -- DocumentManager / ParsedFiles --> C[Context text pool]
  C -- fillSourceWindow --> D
  E[Vector similarity search] -- performSimilaritySearch --> C
  D -- compressMessages --> F[LLM Connector]
  F -- response & metrics --> G[WorkspaceChats.new]
  G -- persisted history --> A
```

* **Persistent memory** – Past chats that opted into inclusion become part of both the history array
  and the backfill reservoir for future citation windows.
* **User-controlled anchors** – Pinned documents and ad-hoc uploads inject high-priority context that
  bypasses vector search yet still respect the token budget and duplicate filtering.
* **Retrieval-augmented context** – Similarity search results fill the remaining context budget so the
  prompt reflects the workspace’s most relevant knowledge at request time.

This choreography ensures that when a request leaves the REST layer, the conversation state has been
validated, merged with every relevant context source, compressed to fit the model, and is ready for a
single call to the configured LLM provider.
