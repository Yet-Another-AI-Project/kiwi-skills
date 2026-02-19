# Tool Implementation

Complete patterns for implementing tools that agents can invoke.

## ITool Interface

```go
// From golanggraph/prebuilt/node/tools
type ITool interface {
    Tools(ctx context.Context) []llms.Tool                                          // Return tool definitions (JSON Schema)
    Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error)  // Execute a single tool call
}
```

The `Run` method receives a single `llms.ToolCall` and returns a `llms.ToolCallResponse`. The framework's `ToolsNode` handles:
- Iterating over all tool calls in the last message
- Matching each call to the correct `ITool` by function name
- Running matched tools concurrently via goroutines
- Appending all `ToolCallResponse` messages to history
- Returning "Tool not found" for unmatched tool calls

Tools no longer need to manage state, history, or streaming directly.

## Complete Tool Example: Search Tool

```go
package tools

import (
    "context"
    "encoding/json"

    "github.com/Yet-Another-AI-Project/kiwi-lib/logger"
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror"
    "github.com/tmc/langchaingo/llms"
)

type SearchTool struct {
    client ISearchClient
    logger logger.ILogger
}

func NewSearchTool(client ISearchClient, logger logger.ILogger) *SearchTool {
    return &SearchTool{client: client, logger: logger}
}

// Tools returns the tool definitions for the LLM
func (t *SearchTool) Tools(ctx context.Context) []llms.Tool {
    return []llms.Tool{{
        Type: "function",
        Function: &llms.FunctionDefinition{
            Name:        "search",
            Description: "Search for relevant information. Use when you need facts or context.",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "query": map[string]any{
                        "type":        "string",
                        "description": "The search query",
                    },
                    "max_results": map[string]any{
                        "type":        "integer",
                        "description": "Maximum number of results (default: 5)",
                    },
                },
                "required": []string{"query"},
            },
        },
    }}
}

// Run executes a single search tool call
func (t *SearchTool) Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error) {
    // Parse arguments
    var args struct {
        Query      string `json:"query"`
        MaxResults int    `json:"max_results"`
    }
    if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
        return llms.ToolCallResponse{}, xerror.Wrap(err)
    }
    if args.MaxResults <= 0 {
        args.MaxResults = 5
    }

    t.logger.Infof(ctx, "Executing search: query=%s, max_results=%d", args.Query, args.MaxResults)

    // Execute search
    results, err := t.client.Search(ctx, args.Query, args.MaxResults)
    if err != nil {
        // Return error as tool response (don't fail the agent)
        return llms.ToolCallResponse{
            ToolCallID: toolCall.ID,
            Name:       toolCall.FunctionCall.Name,
            Content:    "Error: " + err.Error(),
        }, nil
    }

    // Build truncated response for history (save context window)
    response := buildTruncatedResponse(results)

    return llms.ToolCallResponse{
        ToolCallID: toolCall.ID,
        Name:       toolCall.FunctionCall.Name,
        Content:    response,
    }, nil
}

func buildTruncatedResponse(results []SearchResult) string {
    const maxContentCharsPerResult = 500
    const maxResultsInResponse = 4

    truncatedResults := make([]map[string]string, 0, maxResultsInResponse)
    for i, r := range results {
        if i >= maxResultsInResponse {
            break
        }
        content := r.Content
        if len([]rune(content)) > maxContentCharsPerResult {
            content = string([]rune(content)[:maxContentCharsPerResult]) + "...[content truncated]"
        }
        truncatedResults = append(truncatedResults, map[string]string{
            "title":   r.Title,
            "content": content,
        })
    }

    resp := map[string]any{
        "results": truncatedResults,
        "total":   len(results),
        "note":    "Results are truncated for context efficiency.",
    }
    data, _ := json.Marshal(resp)
    return string(data)
}
```

> **Note on full results storage**: Since tools no longer have access to `state.State`, store full results via other mechanisms (e.g., a shared repository injected into the tool, or by including sufficient detail in the truncated response). If downstream nodes need full results, consider using a shared data store or encoding essential IDs in the response for later retrieval.

## File Search Tool (with Response Trimming)

Tool that searches a vector store and trims responses to save context window.

```go
package tools

import (
    "context"
    "encoding/json"

    "github.com/Yet-Another-AI-Project/kiwi-lib/logger"
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror"
    "github.com/tmc/langchaingo/llms"
)

const (
    maxContentCharsPerResult = 500
    maxResultsInResponse     = 4
)

type FileSearchTool struct {
    vectorStore IVectorStore
    logger      logger.ILogger
}

func NewFileSearchTool(vs IVectorStore, logger logger.ILogger) *FileSearchTool {
    return &FileSearchTool{vectorStore: vs, logger: logger}
}

func (t *FileSearchTool) Tools(ctx context.Context) []llms.Tool {
    return []llms.Tool{{
        Type: "function",
        Function: &llms.FunctionDefinition{
            Name:        "file_search",
            Description: "Search uploaded files for relevant content sections.",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "query": map[string]any{
                        "type":        "string",
                        "description": "Semantic search query",
                    },
                    "section_id": map[string]any{
                        "type":        "string",
                        "description": "Optional: get full content of a specific section by ID",
                    },
                },
                "required": []string{"query"},
            },
        },
    }}
}

func (t *FileSearchTool) Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error) {
    var args struct {
        Query     string `json:"query"`
        SectionID string `json:"section_id"`
    }
    if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
        return llms.ToolCallResponse{}, xerror.Wrap(err)
    }

    // Execute vector search
    results, err := t.vectorStore.Search(ctx, args.Query)
    if err != nil {
        return llms.ToolCallResponse{
            ToolCallID: toolCall.ID,
            Name:       toolCall.FunctionCall.Name,
            Content:    "Error: " + err.Error(),
        }, nil
    }

    // Build TRUNCATED response for history (save context window)
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
            "id":      r.ID,
            "title":   r.Title,
            "content": content,
        })
    }

    response, _ := json.Marshal(map[string]any{
        "results": truncated,
        "total":   len(results),
        "note":    "Results are truncated for context efficiency. To get full content, call again with section_id.",
    })

    return llms.ToolCallResponse{
        ToolCallID: toolCall.ID,
        Name:       toolCall.FunctionCall.Name,
        Content:    string(response),
    }, nil
}
```

## Shared Tool Utilities

```go
package utils

import "github.com/tmc/langchaingo/llms"

// HasToolCalls checks if an LLM response contains tool calls
func HasToolCalls(choice *llms.ContentChoice) bool {
    return len(choice.ToolCalls) > 0
}

// HasToolCallsInHistory checks if the last message in history has tool calls
func HasToolCallsInHistory(st state.State) bool {
    if len(st.History) == 0 {
        return false
    }
    lastMsg := st.History[len(st.History)-1]
    for _, part := range lastMsg.Parts {
        if _, ok := part.(llms.ToolCall); ok {
            return true
        }
    }
    return false
}

// CreateToolCallMessage wraps an LLM response's tool calls into a history message
func CreateToolCallMessage(choice *llms.ContentChoice) llms.MessageContent {
    parts := make([]llms.ContentPart, 0, len(choice.ToolCalls))
    for _, tc := range choice.ToolCalls {
        parts = append(parts, tc)
    }
    return llms.MessageContent{
        Role:  llms.ChatMessageTypeAI,
        Parts: parts,
    }
}

// Reusable ephemeral reminders (append to messages, NOT saved to history)
var JsonReminder = llms.MessageContent{
    Role:  llms.ChatMessageTypeHuman,
    Parts: []llms.ContentPart{llms.TextContent{Text: "IMPORTANT: Respond ONLY with valid JSON. No markdown, no explanation."}},
}
```

## Registering Tools with the Agent

### With Prebuilt Agent (RECOMMENDED)

```go
searchTool := NewSearchTool(searchClient, logger)
fileSearchTool := NewFileSearchTool(vectorStore, logger)

// The prebuilt agent handles tool definitions, tool routing, and max tool calls internally
a, err := agent.NewAgent(
    agent.WithName("my_agent"),
    agent.WithModel(llm),
    agent.WithTools([]tools.ITool{searchTool, fileSearchTool}),
    agent.WithMaxToolCalls(6),  // Prevent infinite tool loops
    agent.WithLogger(logger),
)
```

### With Custom Flow (Manual Wiring)

```go
searchTool := NewSearchTool(searchClient, logger)
fileSearchTool := NewFileSearchTool(vectorStore, logger)

// Get all tool definitions for the model node
allToolDefs := append(searchTool.Tools(ctx), fileSearchTool.Tools(ctx)...)

// Create the prebuilt tools node with all tool implementations
toolsNode, err := tools.NewTools(
    tools.WithTools([]tools.ITool{searchTool, fileSearchTool}),
)

// Note: MaxToolCalls is no longer on ToolsNode — use agent.WithMaxToolCalls()
// or implement a custom hook for max tool call limiting in custom flows
```
