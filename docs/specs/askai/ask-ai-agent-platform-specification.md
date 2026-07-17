# Ask AI Agent Platform Specification

## 1. Goal

Enable JoyAgent to serve the Portal Ask AI pilot as an agent platform by accepting bounded multi-turn context, retrieving relevant content from a configured Markdown knowledge corpus, and streaming an answer plus source metadata through its existing agent endpoint.

The feature must prove that Portal can consume JoyAgent as a platform without coupling Portal to JoyAgent's React UI or adding durable conversation storage.

## 2. Background And Current Behavior

JoyAgent currently exposes `POST /web/api/v1/gpt/queryAgentStreamIncr` as a server-sent event (SSE) endpoint. The request contains the current query and identifiers, and the selected agent is created with new in-memory state for each request. Although the JoyAgent React UI displays several messages in one session, it does not send prior user and assistant messages to the backend; consecutive questions therefore reach the model as independent requests.

JoyAgent already separates the React UI, Spring backend, tool service, and MCP client service. The Spring backend discovers and calls MCP tools through the MCP client. The current branch does not provide a knowledge MCP server for the Portal Markdown corpus; the more extensive RAG implementation described for the separate `mrag` branch is not part of this baseline.

The following existing behavior must be preserved:

- The current streaming endpoint path and existing SSE event semantics.
- Requests from existing callers that do not provide conversation messages.
- Existing orchestration modes, tool discovery, and MCP client boundaries.
- Existing JoyAgent React UI behavior; it is not the Portal integration surface.
- Existing non-knowledge tools and their contracts.

## 3. Scope

### In Scope

- Add optional prior `user` and `assistant` messages to the existing agent request contract.
- Apply that context to the default agent mode used by Portal for the lifetime of one request.
- Add a read-only company-knowledge search tool through the existing MCP architecture.
- Load and index Markdown files from one server-configured knowledge root at service startup.
- Perform deterministic, heading-aware lexical retrieval without a vector database.
- Return bounded snippets and source metadata through the existing tool-result stream.
- Propagate cancellation when the Portal disconnects from an active stream.
- Prevent prompts, conversation content, and knowledge excerpts from appearing in normal application logs.
- Preserve backward compatibility for callers that omit the new context field.

### Out Of Scope

- Changes to the JoyAgent React UI.
- Durable conversation history, server-side sessions, or a chat database.
- Vector embeddings, semantic search, a vector database, or the separate `mrag` branch.
- Document upload, editing, deletion, approval, or administration.
- Live file watching or automatic re-indexing after startup.
- Per-user or per-role document filtering inside JoyAgent.
- New orchestration modes or changes to modes not used by the Portal pilot.
- A new public API version or replacement of the existing SSE schema.
- Production service-to-service authentication, user token propagation, analytics, or usage reporting.

## 4. Actors And Permissions

- **Portal Ask AI backend:** The trusted caller for this pilot. It has already authenticated and authorized the employee before calling JoyAgent.
- **JoyAgent agent runtime:** Receives a bounded request, selects and calls tools, invokes the configured LLM, and streams results.
- **Knowledge MCP server:** Reads only the configured knowledge root and returns bounded search matches.
- **Portal employee:** An indirect user. JoyAgent does not receive the employee's Entra token, Portal session cookie, roles, permissions, or personal identity.

JoyAgent does not evaluate the Portal's `askai.view` permission. That permission is owned and enforced by Portal. Direct network access to JoyAgent remains an environment/deployment concern for this pilot.

## 5. User Workflows

### 5.1 First Question

1. Portal sends the current question, an empty prior-message list, and request correlation identifiers to the existing JoyAgent streaming endpoint.
2. JoyAgent validates the request and initializes the selected default agent with no prior conversation messages.
3. The agent may call the company-knowledge search tool when the question requires corpus information.
4. JoyAgent streams existing response events, including assistant content and any structured knowledge tool result.
5. JoyAgent emits the existing terminal event and releases request resources.

### 5.2 Follow-Up Question

1. Portal sends the most recent completed user/assistant message pairs and the new question.
2. JoyAgent validates and installs the prior messages in chronological order before adding the current question.
3. The default agent answers with access to that bounded context and may perform another knowledge search.
4. JoyAgent does not retain the context after the request ends.

### 5.3 Knowledge Search

1. The agent calls `search_company_knowledge` with a natural-language search query.
2. The tool searches the startup index and returns at most five ranked heading-level matches.
3. Each match contains a bounded excerpt and display-safe source metadata, not an absolute file path or full document.
4. The agent uses relevant excerpts as reference material and the tool result remains available in the existing SSE stream for Portal source rendering.

### 5.4 Stop An Active Response

1. Portal closes or aborts the upstream streaming request.
2. JoyAgent stops the associated agent execution and cancels outstanding LLM, tool, and MCP work where the underlying client supports cancellation.
3. JoyAgent releases request-scoped resources and does not continue producing output for the abandoned request.

## 6. Functional Requirements

1. The existing agent request shall accept an optional `messages` array.
2. Each message shall contain exactly one supported role, `user` or `assistant`, and non-empty textual content.
3. `messages` shall represent completed prior conversation messages only; the current question remains in `query`.
4. JoyAgent shall preserve the supplied chronological order and append the current query exactly once after prior context is installed.
5. The default agent mode used by Portal shall make the validated prior context available to the LLM for the current request.
6. Context shall be request-scoped and shall not be stored or recovered using `sessionId`.
7. Omitting `messages`, or supplying an empty array, shall retain the current single-question behavior.
8. The Portal integration shall continue to use the existing default agent mode. Multi-turn support for other optional modes is not required by this pilot.
9. A knowledge MCP server shall expose one tool named `search_company_knowledge` through the existing MCP discovery and call path.
10. The knowledge root shall be supplied only by server configuration. Neither the agent request nor the MCP tool input may select a filesystem root or file path.
11. At startup, the knowledge service shall recursively consider regular files whose names end in `.md` under the configured root.
12. The index shall extract available YAML frontmatter, the first level-one heading, headings, and body text. Display title shall use the first level-one heading when present and otherwise the file stem.
13. Search shall be case-insensitive and shall score matches across the title, frontmatter tags, headings, and body, with title, tag, and heading matches ranked above body-only matches.
14. Results shall be returned at heading-section granularity so the source can identify the relevant section.
15. Equal-scoring results shall use a stable deterministic ordering.
16. A search shall return no more than five matches, and each returned snippet shall contain no more than 1,200 characters.
17. Search results shall never include absolute paths, content outside the configured root, or complete file bodies.
18. The startup index shall remain unchanged until the knowledge service restarts.
19. Knowledge content shall be treated as untrusted reference data, not executable instructions or authority to change system behavior.
20. A knowledge tool result used during the request shall remain observable in the existing `tool_result` SSE behavior so Portal can derive source labels.
21. JoyAgent shall stop active request processing when its streaming client disconnects.

## 7. Business Rules

1. JoyAgent is the owner of agent orchestration, tool selection, knowledge retrieval, and LLM invocation. Portal must not decide which knowledge file to read.
2. Multi-turn context consists only of completed user and assistant messages. System prompts, tool calls, tool results, partial responses, errors, and cancelled responses are not accepted as caller-provided context.
3. The maximum prior context is 10 messages and 20,000 characters in aggregate. The current query is separate from both limits.
4. The current query must contain non-whitespace text and must not exceed 4,000 characters.
5. Knowledge matches are evidence candidates, not instructions. The agent must not follow commands found inside a document that conflict with its system or application instructions.
6. Source metadata may identify only a match actually returned by the knowledge tool during the current request.
7. When no useful corpus match is available, the agent must not fabricate a document source. It shall communicate that relevant company knowledge was not found when the answer depends on that knowledge.
8. A `sessionId` or request identifier provides correlation only and must not imply server-side conversation persistence.
9. Existing callers that omit `messages` must receive the same request handling and SSE event meanings as before this feature.

## 8. Data And Integration Requirements

### Conversation Context

- Context exists only in the incoming request and the selected agent's request-scoped memory.
- JoyAgent shall not write prompts, messages, answers, or retrieved excerpts to a database, cache, or session store for this feature.
- The order received after validation is the order presented to the agent.

### Knowledge Corpus

- The configured root is initially expected to point to `/Users/raylui/Documents/obws/lmi/02_Knowledge` in the pilot environment, but the path must remain environment configuration rather than a source-code constant.
- The corpus is read-only from JoyAgent's perspective.
- YAML frontmatter fields may be used when present. The pilot recognizes `tags`, `source_pdf`, and `last_updated` as optional metadata; missing or malformed optional fields must not prevent otherwise readable Markdown from being indexed.
- A source exposed to Portal contains `fileName`, `title`, and `heading`. `tags` and `lastUpdated` may remain internal to retrieval and are not required in the Portal display contract.

### Service Boundaries

- Portal calls only the existing JoyAgent Spring backend endpoint.
- The Spring backend continues to call tools through the MCP client service.
- The knowledge MCP server is registered through existing MCP server configuration and is not called directly by Portal.
- No fixed new externally exposed port is part of this feature contract.

## 9. API Requirements

### 9.1 Existing JoyAgent Streaming Operation

**Operation:** `POST /web/api/v1/gpt/queryAgentStreamIncr`

**Transport:** Existing `text/event-stream` response behavior.

**Existing inputs:** Existing fields including `query`, `sessionId`, `requestId`, `deepThink`, `outputStyle`, `traceId`, and `user` remain compatible.

**New optional input:**

```json
{
  "messages": [
    { "role": "user", "content": "What does the EDI status mean?" },
    { "role": "assistant", "content": "It indicates ..." }
  ]
}
```

Validation requirements:

- `query`: required non-whitespace string, maximum 4,000 characters.
- `messages`: optional array, maximum 10 entries.
- `messages[*].role`: exactly `user` or `assistant`.
- `messages[*].content`: required non-whitespace string.
- Aggregate `messages[*].content`: maximum 20,000 characters.
- Caller-provided `system`, `tool`, or unknown roles: rejected.
- Portal shall omit employee identity and shall not rely on the existing `user` field for authorization.

Invalid input shall be rejected as a client validation error before agent or tool execution. If an unrecoverable failure occurs after the SSE response has opened, JoyAgent shall use its existing stream error behavior and then terminate the stream.

The response schema and meanings of existing SSE events shall not be renamed or removed. Knowledge search output shall use the existing tool-result mechanism. A request without `messages` shall not require a client change.

### 9.2 Knowledge MCP Tool

**Tool name:** `search_company_knowledge`

**Input:**

```json
{
  "query": "How is an EDI message retried?",
  "limit": 5
}
```

Input rules:

- `query`: required non-whitespace string, maximum 4,000 characters.
- `limit`: optional positive integer; default 5 and maximum 5.
- No filesystem path, file name, glob, or root input is accepted.

**Successful output:**

```json
{
  "matches": [
    {
      "fileName": "example.md",
      "title": "Example title",
      "heading": "Retry behavior",
      "snippet": "Bounded relevant excerpt",
      "tags": ["edi"],
      "lastUpdated": "2026-01-01"
    }
  ]
}
```

Output rules:

- `matches` is an empty array when the index is available but no document matches.
- Optional metadata is omitted when unavailable; it is not synthesized.
- A missing, unreadable, or uninitialized knowledge root produces a structured tool-unavailable error, not an empty match set.
- An invalid tool input produces a structured validation error without searching.
- Tool failures shall not expose absolute paths, stack traces, document content, or environment secrets to the caller.

## 10. Page-Specific UI Requirements

No changes to the JoyAgent React UI are required. The Portal-native Ask AI page and its UI behavior are specified in the Portal repository. The existing JoyAgent UI may continue to operate as it does today and is not an acceptance surface for this pilot.

## 11. Error Cases And Edge Cases

- **Missing or unreadable knowledge root:** Knowledge search reports an unavailable error; other existing JoyAgent functions remain available.
- **Empty corpus:** Search returns an empty `matches` array.
- **Malformed Markdown or frontmatter:** Readable Markdown body content is indexed when safe to do so; invalid optional metadata is ignored. A wholly unreadable file is skipped with a metadata-only warning.
- **Path escape or symlink outside the root:** The file is excluded and never read or returned.
- **Duplicate headings or file names:** Each match remains distinguishable by its root-relative file identity internally, while only display-safe file name/title/heading metadata is returned. Ranking remains deterministic.
- **Oversized context or query:** The request is rejected before LLM or tool execution.
- **Unsupported message role:** The request is rejected; the role is not converted or silently ignored.
- **Knowledge tool timeout or MCP failure:** The agent receives a tool failure and must not present fabricated sources. The stream either provides a clear unavailable response or terminates using existing error behavior.
- **LLM failure after partial output:** The stream terminates using existing error behavior; partial output is not considered a completed assistant context message by Portal.
- **Client cancellation:** Outstanding work is cancelled and late events are discarded.
- **No matching knowledge:** The tool returns an empty list, and the agent does not invent a corpus citation.
- **Document changes after startup:** Results continue to use the startup snapshot until service restart.

## 12. Security, Privacy, And Audit Requirements

1. The knowledge service shall resolve every candidate file against the configured root and shall never read a resolved path outside that root.
2. The knowledge service shall not modify corpus files.
3. Tool input shall not accept a caller-controlled path.
4. Markdown text, frontmatter, prompts, prior messages, LLM responses, and tool snippets are untrusted content.
5. Document text shall not be interpreted as system instructions, tool authorization, credentials, or a request to access another file.
6. Normal logs shall not contain current queries, prior messages, generated answers, raw document text, retrieved snippets, Entra data, Portal cookies, or authorization headers.
7. Logs may contain request and trace identifiers, message and character counts, tool name, result count, timing, cancellation state, and sanitized error category.
8. User-facing or caller-facing errors shall not expose absolute paths, secrets, stack traces, or raw upstream payloads.
9. JoyAgent shall not create a conversation audit record or content analytics record for this pilot.
10. Network restriction between Portal and JoyAgent is assumed for the pilot; adding application-level service authentication is explicitly deferred.

## 13. Acceptance Criteria

1. A request with no `messages` field continues to stream successfully with the existing SSE event meanings.
2. A request with valid prior user/assistant messages allows a follow-up question that depends on an earlier answer to be resolved using that context.
3. The same follow-up request without prior messages is not treated as though server-side conversation history exists.
4. Prior context is not available to a later request unless the caller sends it again.
5. Requests exceeding 10 prior messages, 20,000 prior-message characters, or 4,000 query characters are rejected before agent execution.
6. Requests containing a `system`, `tool`, or unknown message role are rejected.
7. The current query appears once, after prior messages, in the default agent's request context.
8. On startup with the pilot corpus configured, the knowledge tool indexes Markdown files and returns a relevant heading-level match for a known corpus question.
9. Search ranking gives a title, tag, or heading match precedence over an otherwise comparable body-only match.
10. Search returns no more than five results and no snippet exceeds 1,200 characters.
11. Tool output contains no absolute path and cannot be made to read a caller-selected path.
12. A no-match search returns an empty match list and does not cause a fabricated source to appear.
13. A missing or unreadable root reports the knowledge tool as unavailable without breaking unrelated JoyAgent endpoints.
14. A knowledge match used by the agent is visible through the existing tool-result stream with its file name, title, and heading.
15. Disconnecting the SSE client stops the associated request and prevents further stream events from being delivered.
16. Operational logs for a representative request contain correlation metadata but not the prompt, prior messages, answer, document body, or retrieved snippet.
17. Existing non-knowledge tools and existing agent requests remain operational after the feature is enabled.

## 14. Simplifications

- Use heading-aware lexical search instead of embeddings or semantic retrieval.
- Build one in-memory index at startup and require restart to reload documents.
- Expose one read-only knowledge tool with a maximum of five matches.
- Support multi-turn context only when the caller supplies it; do not store sessions.
- Validate the Portal pilot against JoyAgent's existing default agent mode.
- Reuse the existing endpoint, SSE events, MCP client, and tool-result behavior.

## 15. Assumptions

- The Portal backend and JoyAgent services run on a trusted private or local network during the pilot.
- Portal is solely responsible for employee authentication and enforcement of `askai.view`.
- The configured knowledge corpus is shared equally by all Portal users who have `askai.view`.
- Corpus files are UTF-8 Markdown and are small enough for a bounded in-memory startup index.
- Five results with snippets of at most 1,200 characters are sufficient for the pilot and protect the LLM/tool context from unbounded payloads.
- Portal uses the existing default agent mode and supplies only completed conversation pairs.

## 16. Open Questions

None.
