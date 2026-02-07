# LLM Best Practices

Detailed patterns for optimizing LLM usage in Go agents: context compression, tool response trimming, prompt caching, attention raising, and MCP integration.

## Context Compression (History Trimming)

Long-running agents accumulate messages that exceed context limits. Trim history before every LLM call in loop-based agents.

### Sliding Window Pattern

Keep the system prompt (always index 0) + optionally important anchors + the last N messages.

```go
const (
    maxHistoryLen = 10  // Trigger trimming above this
    keepRecent    = 5   // Keep the last 5 messages
)

// TrimHistory reduces history length while preserving critical context.
// Call BEFORE every LLM invocation in loop-based agents.
func (s *MyState) TrimHistory() {
    if len(s.History) <= maxHistoryLen {
        return
    }

    preserved := make([]llms.MessageContent, 0, keepRecent+2)

    // 1. ALWAYS keep system prompt (index 0)
    preserved = append(preserved, s.History[0])

    // 2. Optionally keep anchor messages (e.g., task header at index 1)
    // preserved = append(preserved, s.History[1])

    // 3. Keep last N messages for recency
    startIdx := len(s.History) - keepRecent
    preserved = append(preserved, s.History[startIdx:]...)

    // 4. Trim tool responses in the preserved history
    trimToolResponsesInHistory(preserved)

    s.History = preserved
}
```

### Multi-Agent History Trimming

When agents maintain separate histories in Metadata (multi-agent pattern), trim each independently:

```go
// Director: keep system prompt + last task header + last 5
func (s *MultiAgentState) TrimDirectorHistory() {
    if len(s.DirectorHistory) <= 10 {
        return
    }
    preserved := []llms.MessageContent{s.DirectorHistory[0]}
    // Keep the last task header (an anchor message with context)
    if lastHeader := s.findLastTaskHeader(); lastHeader != nil {
        preserved = append(preserved, *lastHeader)
    }
    startIdx := len(s.DirectorHistory) - 5
    preserved = append(preserved, s.DirectorHistory[startIdx:]...)
    trimToolResponsesInHistory(preserved)
    s.DirectorHistory = preserved
}

// Character: keep system prompt + last 5 (simpler, no anchors)
func (s *MultiAgentState) TrimCharacterHistory(characterID string) {
    hist := s.CharacterHistory[characterID]
    if len(hist) <= 10 {
        return
    }
    preserved := []llms.MessageContent{hist[0]}
    startIdx := len(hist) - 5
    preserved = append(preserved, hist[startIdx:]...)
    trimToolResponsesInHistory(preserved)
    s.CharacterHistory[characterID] = preserved
}
```

### Rules

- ALWAYS keep the system prompt (index 0) -- it defines the agent's role and behavior
- Call `TrimHistory()` BEFORE every `llm.GenerateContent()` call in loop nodes
- Choose `keepRecent` based on how much context the agent needs (5 is a good default)
- Anchor messages (task definitions, key context) should be explicitly preserved
- After trimming, also trim tool responses in the remaining history (see below)

## Tool Response Trimming

Tool responses (especially search results, file contents) can be very large. Trim at two levels.

### Level 1: At Tool Execution Time

Truncate each result when building the `ToolCallResponse`:

```go
const (
    maxContentCharsPerResult = 500
    maxResultsInResponse     = 4
)

func buildToolResponse(results []Result) string {
    truncated := make([]map[string]string, 0, maxResultsInResponse)
    for i, r := range results {
        if i >= maxResultsInResponse {
            break
        }
        content := r.Content
        if len([]rune(content)) > maxContentCharsPerResult {
            content = string([]rune(content)[:maxContentCharsPerResult]) + "...[content truncated]"
        }
        truncated = append(truncated, map[string]string{
            "title":   r.Title,
            "content": content,
        })
    }
    resp := map[string]any{
        "results": truncated,
        "total":   len(results),
        "note":    "Results truncated. Full results available in metadata.",
    }
    data, _ := json.Marshal(resp)
    return string(data)
}
```

Store full (untruncated) results in `state.Metadata` so downstream nodes can access them:

```go
// In tool Run():
currentState.Metadata[MetadataKeyFullResults] = results           // Full for other nodes
currentState.History = append(currentState.History, toolResponse)  // Truncated for LLM
```

### Level 2: Retroactive Trimming in History

When trimming history, also shrink old tool responses that may have been large:

```go
const maxToolResponseChars = 1500

func trimToolResponsesInHistory(history []llms.MessageContent) {
    for i, msg := range history {
        if msg.Role != llms.ChatMessageTypeTool {
            continue
        }
        for j, part := range msg.Parts {
            resp, ok := part.(llms.ToolCallResponse)
            if !ok {
                continue
            }
            // Only trim specific tool types known to have large responses
            if resp.Name != "file_search" && resp.Name != "search" {
                continue
            }
            if len([]rune(resp.Content)) > maxToolResponseChars {
                resp.Content = string([]rune(resp.Content)[:maxToolResponseChars]) + "\n...[truncated]"
                msg.Parts[j] = resp
            }
        }
        history[i] = msg
    }
}
```

### Rules

- Level 1 (at execution) is ALWAYS applied -- never put unbounded content in history
- Level 2 (retroactive) is applied during history trimming for older messages
- Use `[]rune` for character counting to handle multi-byte UTF-8 correctly
- Always store full results in Metadata so downstream nodes are not affected by truncation
- Include a `note` field in truncated responses so the LLM knows results are incomplete

## Prompt Caching

LLM providers cache identical message prefixes. Structure prompts to maximize cache hits.

### Static-First Structure

```
[SYSTEM MESSAGE - Static, identical across calls]
  - Role definition ("You are a ...")
  - Rules and constraints
  - Output format specification (JSON schema, XML format)
  - Few-shot examples
  - Tool usage instructions

[CONVERSATION HISTORY - Semi-static]
  - Previous messages (grow over time, but prefix is stable)

[LAST HUMAN MESSAGE - Dynamic, per-request]
  - Current task input
  - Context-specific instructions
  - Ephemeral reminders
```

### Implementation

```go
//go:embed system_prompt.txt
var systemPrompt string  // Static, never changes per request

func (n *MyNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    // System prompt is IDENTICAL across calls -> cacheable
    if len(currentState.History) == 0 {
        currentState.History = append(currentState.History, llms.MessageContent{
            Role:  llms.ChatMessageTypeSystem,
            Parts: []llms.ContentPart{llms.TextContent{Text: systemPrompt}},
        })
    }

    // Dynamic content goes in the LAST human message
    currentState.History = append(currentState.History, llms.MessageContent{
        Role:  llms.ChatMessageTypeHuman,
        Parts: []llms.ContentPart{llms.TextContent{Text: dynamicInput}},
    })

    completion, err := n.llm.GenerateContent(ctx, currentState.History, ...)
    // ...
}
```

### Provider-Specific Notes

**Anthropic (Claude):**
- Cache control breakpoints at message boundaries
- System message is automatically cached if identical
- Minimum cacheable prefix: ~1024 tokens
- Cache TTL: 5 minutes (resets on hit)

**OpenAI:**
- Automatic prefix caching on identical message sequences
- No explicit cache control needed
- Caching kicks in after the first request with the same prefix

### Rules

- System prompt MUST be identical across calls (no timestamps, user IDs, or dynamic data)
- Put ALL static content in the system message: role, rules, format spec, few-shot examples
- Put changing context in the LAST human message, never in the system message
- For multi-turn: the conversation prefix stays stable, maximizing incremental cache hits
- Do NOT dynamically construct the system prompt per request (kills caching)
- Use `//go:embed` to load prompt templates from files (ensures consistency)

## Attention Raising in Long Context

LLMs lose focus over long contexts. Use structural and repetition techniques to maintain attention on critical instructions.

### XML Tags for Structure

Use XML tags to create clear semantic boundaries:

```go
systemPrompt := `<role>
You are a content generation assistant. You produce structured output.
</role>

<rules>
CRITICAL: All output MUST be valid JSON matching the schema below.
- Do not include markdown code fences
- Do not include explanatory text outside the JSON
- All string values must be properly escaped
</rules>

<output_schema>
{
  "title": "string",
  "items": [{"name": "string", "description": "string"}]
}
</output_schema>

<examples>
Input: "List fruits"
Output: {"title": "Fruits", "items": [{"name": "Apple", "description": "A red fruit"}]}
</examples>`
```

### Primacy and Recency Effect

Place the MOST IMPORTANT instructions at the START and END of the prompt:

```
[START - High attention]
CRITICAL RULES HERE (format requirements, constraints)

[MIDDLE - Lower attention]
Context, background information, reference data

[END - High attention]
Task instruction + format reminder
```

### Ephemeral Reminders

Append a reminder message to the message list for the LLM call, but do NOT save it to history. This reinforces key instructions without polluting the conversation:

```go
func (n *MyNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    // Build messages for this LLM call
    messagesForCall := make([]llms.MessageContent, len(currentState.History))
    copy(messagesForCall, currentState.History)

    // Append ephemeral reminder (NOT saved to currentState.History)
    messagesForCall = append(messagesForCall, llms.MessageContent{
        Role:  llms.ChatMessageTypeHuman,
        Parts: []llms.ContentPart{llms.TextContent{
            Text: "REMINDER: Respond ONLY with valid JSON. No markdown fences, no explanation.",
        }},
    })

    // Call LLM with the reminder-appended messages
    completion, err := n.llm.GenerateContent(ctx, messagesForCall, ...)

    // Continue using currentState.History (without the reminder)
    // ...
}
```

Reusable reminder constants:

```go
// In agent/utils/tool_utils.go
var JsonReminder = llms.MessageContent{
    Role:  llms.ChatMessageTypeHuman,
    Parts: []llms.ContentPart{llms.TextContent{
        Text: "IMPORTANT: Respond ONLY with valid JSON. No markdown, no explanation.",
    }},
}

// Usage in nodes:
messagesForCall := append(slices.Clone(currentState.History), utils.JsonReminder)
```

### Emphasis Markers

Use uppercase keywords for critical constraints:

| Marker | Usage |
|--------|-------|
| `CRITICAL:` | Must-follow rules that affect correctness |
| `IMPORTANT:` | High-priority instructions |
| `MUST` / `MUST NOT` | Hard requirements |
| `ALWAYS` / `NEVER` | Absolute behavioral rules |
| `NOTE:` | Contextual clarification |

### Rules

- Use XML tags (`<rules>`, `<context>`, `<output>`, `<examples>`) for semantic boundaries
- Place critical rules at START and END of prompts (primacy/recency)
- Use ephemeral reminders for format enforcement in loop-based agents
- Never save ephemeral reminders to history (they are per-call only)
- Use `slices.Clone()` or `copy()` before appending ephemeral messages to avoid mutating the original history
- Repeat key format instructions at the end of long system prompts
- Use emphasis markers (`CRITICAL:`, `MUST`, `NEVER`) sparingly -- overuse reduces their impact

## MCP (Model Context Protocol)

MCP provides a standardized protocol for LLM-tool communication. In Go agents, tool interactions already follow MCP-compatible patterns.

### How MCP Maps to golanggraph

| MCP Concept | golanggraph Equivalent |
|-------------|----------------------|
| Tool Definition (JSON Schema) | `llms.Tool` with `FunctionDefinition.Parameters` |
| Tool Call (from LLM) | `llms.ToolCall` in `ContentChoice.ToolCalls` |
| Tool Result (to LLM) | `llms.ToolCallResponse` appended to history |
| Tool List | `ITool.Tools(ctx)` returns `[]llms.Tool` |

### Native Tool Pattern (Built-in)

The standard `ITool` interface is MCP-compatible:

```go
type ITool interface {
    Tools(ctx context.Context) []llms.Tool                           // MCP: list_tools
    Run(ctx context.Context, st *state.State, sf StreamFunc) error   // MCP: call_tool
}
```

The message flow matches MCP:
1. Agent sends tool definitions to LLM (`Tools()` -> `llms.WithTools()`)
2. LLM returns tool calls (`llms.ToolCall`)
3. Agent executes tools (`Run()`)
4. Agent returns results to LLM (`llms.ToolCallResponse`)

### Wrapping External MCP Servers

To integrate an external MCP server (e.g., a Python service exposing tools via MCP), wrap it as an `ITool`:

```go
type MCPProxyTool struct {
    client  MCPClient          // HTTP/stdio client to MCP server
    logger  logger.ILogger
    toolDef []llms.Tool        // Cached from MCP server's list_tools
}

func NewMCPProxyTool(client MCPClient, logger logger.ILogger) (*MCPProxyTool, error) {
    // Fetch tool definitions from MCP server at init time
    mcpTools, err := client.ListTools(context.Background())
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    // Convert MCP tool definitions to langchaingo format
    var defs []llms.Tool
    for _, mt := range mcpTools {
        defs = append(defs, llms.Tool{
            Type: "function",
            Function: &llms.FunctionDefinition{
                Name:        mt.Name,
                Description: mt.Description,
                Parameters:  mt.InputSchema, // JSON Schema, same format
            },
        })
    }

    return &MCPProxyTool{client: client, logger: logger, toolDef: defs}, nil
}

func (t *MCPProxyTool) Tools(ctx context.Context) []llms.Tool {
    return t.toolDef
}

func (t *MCPProxyTool) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    lastMsg := currentState.History[len(currentState.History)-1]
    for _, part := range lastMsg.Parts {
        toolCall, ok := part.(llms.ToolCall)
        if !ok {
            continue
        }

        // Proxy the call to the MCP server
        result, err := t.client.CallTool(ctx, toolCall.FunctionCall.Name, toolCall.FunctionCall.Arguments)
        if err != nil {
            result = "Error: " + err.Error()
        }

        // Truncate result for history
        if len([]rune(result)) > maxToolResponseChars {
            result = string([]rune(result)[:maxToolResponseChars]) + "...[truncated]"
        }

        currentState.History = append(currentState.History, llms.MessageContent{
            Role: llms.ChatMessageTypeTool,
            Parts: []llms.ContentPart{
                llms.ToolCallResponse{
                    ToolCallID: toolCall.ID,
                    Name:       toolCall.FunctionCall.Name,
                    Content:    result,
                },
            },
        })
    }
    return nil
}
```

### MCP Transport Options

| Transport | When to Use |
|-----------|-------------|
| stdio | MCP server runs as a subprocess (same machine) |
| HTTP/SSE | MCP server is a remote service |
| In-process | MCP server is a Go library (use `ITool` directly) |

### Rules

- Native `ITool` implementations are already MCP-compatible -- no extra abstraction needed
- For external MCP servers, create a thin proxy `ITool` that translates calls
- Cache MCP tool definitions at init time (`NewMCPProxyTool`) to avoid repeated `list_tools` calls
- Apply the same response trimming rules to MCP tool results as to native tools
- Handle MCP server errors gracefully: return error as tool response content, do not crash the agent
