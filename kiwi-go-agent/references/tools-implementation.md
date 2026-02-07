# Tool Implementation

Complete patterns for implementing tools that agents can invoke.

## ITool Interface

```go
// From golanggraph/prebuilt/node/tools
type ITool interface {
    Tools(ctx context.Context) []llms.Tool   // Return tool definitions (JSON Schema)
    Run(ctx context.Context, currentState *state.State, streamFunc flowcontract.StreamFunc) error
}
```

## Complete Tool Example: Search Tool

```go
package tools

import (
    "context"
    "encoding/json"

    flowcontract "github.com/futurxlab/golanggraph/contract"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/state"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/tmc/langchaingo/llms"
)

const MetadataKeySearchResults = "search_results"

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

// Run executes tool calls found in the last history message
func (t *SearchTool) Run(ctx context.Context, currentState *state.State, streamFunc flowcontract.StreamFunc) error {
    if len(currentState.History) == 0 {
        return nil
    }

    lastMsg := currentState.History[len(currentState.History)-1]

    for _, part := range lastMsg.Parts {
        toolCall, ok := part.(llms.ToolCall)
        if !ok {
            continue
        }

        if toolCall.FunctionCall.Name != "search" {
            continue
        }

        // Parse arguments
        var args struct {
            Query      string `json:"query"`
            MaxResults int    `json:"max_results"`
        }
        if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
            return xerror.Wrap(err)
        }
        if args.MaxResults <= 0 {
            args.MaxResults = 5
        }

        t.logger.Infof(ctx, "Executing search: query=%s, max_results=%d", args.Query, args.MaxResults)

        // Execute search
        results, err := t.client.Search(ctx, args.Query, args.MaxResults)
        if err != nil {
            // Return error as tool response (don't fail the agent)
            currentState.History = append(currentState.History, llms.MessageContent{
                Role: llms.ChatMessageTypeTool,
                Parts: []llms.ContentPart{
                    llms.ToolCallResponse{
                        ToolCallID: toolCall.ID,
                        Name:       toolCall.FunctionCall.Name,
                        Content:    "Error: " + err.Error(),
                    },
                },
            })
            continue
        }

        // Store full results in metadata for other nodes
        currentState.Metadata[MetadataKeySearchResults] = results

        // Build truncated response for history (save context window)
        response := buildTruncatedResponse(results)

        currentState.History = append(currentState.History, llms.MessageContent{
            Role: llms.ChatMessageTypeTool,
            Parts: []llms.ContentPart{
                llms.ToolCallResponse{
                    ToolCallID: toolCall.ID,
                    Name:       toolCall.FunctionCall.Name,
                    Content:    response,
                },
            },
        })

        // Stream tool event if callback is available
        if streamFunc != nil {
            eventData, _ := json.Marshal(map[string]any{
                "type":   "tool_result",
                "name":   "search",
                "result": response,
            })
            _ = streamFunc(ctx, &flowcontract.FlowStreamEvent{Chunk: string(eventData)})
        }
    }

    return nil
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
        "note":    "Results are truncated for context efficiency. Full results available in metadata.",
    }
    data, _ := json.Marshal(resp)
    return string(data)
}
```

## File Search Tool (with Response Trimming)

Tool that searches a vector store and trims responses to save context window.

```go
package tools

import (
    "context"
    "encoding/json"

    flowcontract "github.com/futurxlab/golanggraph/contract"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/state"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/tmc/langchaingo/llms"
)

const (
    maxContentCharsPerResult = 500
    maxResultsInResponse     = 4
    MetadataKeyFileResults   = "file_search_results"
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

func (t *FileSearchTool) Run(ctx context.Context, currentState *state.State, streamFunc flowcontract.StreamFunc) error {
    if len(currentState.History) == 0 {
        return nil
    }

    lastMsg := currentState.History[len(currentState.History)-1]
    for _, part := range lastMsg.Parts {
        toolCall, ok := part.(llms.ToolCall)
        if !ok || toolCall.FunctionCall.Name != "file_search" {
            continue
        }

        var args struct {
            Query     string `json:"query"`
            SectionID string `json:"section_id"`
        }
        if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
            return xerror.Wrap(err)
        }

        // Execute vector search
        results, err := t.vectorStore.Search(ctx, args.Query)
        if err != nil {
            return xerror.Wrap(err)
        }

        // Store FULL results in metadata (other nodes may need them)
        if currentState.Metadata == nil {
            currentState.Metadata = make(map[string]interface{})
        }
        currentState.Metadata[MetadataKeyFileResults] = results

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

        currentState.History = append(currentState.History, llms.MessageContent{
            Role: llms.ChatMessageTypeTool,
            Parts: []llms.ContentPart{
                llms.ToolCallResponse{
                    ToolCallID: toolCall.ID,
                    Name:       toolCall.FunctionCall.Name,
                    Content:    string(response),
                },
            },
        })
    }

    return nil
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

```go
// In agent factory
searchTool := NewSearchTool(searchClient, logger)
fileSearchTool := NewFileSearchTool(vectorStore, logger)

// Get all tool definitions for the chat node
allToolDefs := append(searchTool.Tools(ctx), fileSearchTool.Tools(ctx)...)

// Create the prebuilt tools node with all tool implementations
toolsNode, err := tools.NewTools(
    tools.WithTools([]tools.ITool{searchTool, fileSearchTool}),
    tools.WithMaxToolCalls(6), // Prevent infinite tool loops
)
```
