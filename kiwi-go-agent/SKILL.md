---
name: kiwi-go-agent
description: AI agent development standards using golanggraph for graph-based workflows, langchaingo for LLM calls, tool integration, MCP, and LLM best practices (context compression, prompt caching, attention raising, tool response trimming).
---

# Kiwi Go Agent Development

This skill defines the standards for building AI agents in Go using the golanggraph framework and langchaingo. All agent development MUST follow these patterns.

## Framework Overview

| Framework | Purpose | Import |
|-----------|---------|--------|
| golanggraph | Graph-based agent workflow engine | `github.com/futurxlab/golanggraph/*` |
| langchaingo | LLM calls, prompts, tool definitions | `github.com/tmc/langchaingo/*` |

Agents are DAGs (directed acyclic graphs -- with cycles allowed for loops) of **Nodes** connected by **Edges**. Each node performs one atomic step: call an LLM, execute tools, validate output, or transform state.

## Core Concepts

### State

All data flows through `state.State`:

```go
type State struct {
    History  []llms.MessageContent      // Conversation messages (system, human, AI, tool)
    Metadata map[string]interface{}      // Custom data shared between nodes
}
```

- `History`: The LLM message history. Append to it; the framework passes it between nodes.
- `Metadata`: A typed key-value map for passing structured data between nodes. Store custom state objects here.

### Node Interface

Every node implements `flowcontract.Node`:

```go
type Node interface {
    Name() string
    Run(ctx context.Context, currentState *state.State, streamFunc flowcontract.StreamFunc) error
}
```

- `Name()`: Unique identifier used in edge wiring.
- `Run()`: Executes the node logic. Modifies `currentState` in place.
- `streamFunc`: Optional callback for streaming events to the caller.

### State Helper Methods

`state.State` provides convenience methods:

```go
s.GetLastResponse() string          // Last AI message text
s.GetResumeValue() interface{}      // Value passed via ResumeWithValue()
s.IsInterrupted() bool              // Whether flow was interrupted
s.GetThreadID() string              // Current thread ID (for checkpointing)
s.SetInterruptPayload(payload)      // Set interrupt payload before Interrupt()
```

### Edges

Edges define execution order. Two types:

```go
// Direct edge: always flows From -> To
edge.Edge{From: "node_a", To: "node_b"}

// Conditional edge: ConditionFunc decides which target
edge.Edge{
    From:          "node_a",
    ConditionalTo: []string{"node_b", "node_c", flow.EndNode},
    ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
        // Return one of ConditionalTo values
        return "node_b", nil
    },
}
```

### Flow Building

```go
compiledFlow, err := flow.NewFlowBuilder(logger).
    SetName("my_agent").
    SetCheckpointer(checkpointer.NewInMemoryCheckpointer()).
    AddNode(nodeA).
    AddNode(nodeB).
    AddEdge(edge.Edge{From: flow.StartNode, To: nodeA.Name()}).
    AddEdge(edge.Edge{From: nodeA.Name(), To: nodeB.Name()}).
    AddEdge(edge.Edge{From: nodeB.Name(), To: flow.EndNode}).
    Compile()
```

### Flow Execution

```go
// Without streaming
finalState, err := compiledFlow.Exec(ctx, initialState, nil)

// With streaming callback
finalState, err := compiledFlow.Exec(ctx, initialState, func(ctx context.Context, event *flowcontract.FlowStreamEvent) error {
    // event.Chunk contains streamed data
    // event.FullState contains current flow state
    return nil
})
```

## Agent Patterns

See [references/agent-examples.md](references/agent-examples.md) for complete code examples of each pattern.

### Pattern 1: Prebuilt Agent (RECOMMENDED)

Use `agent.NewAgent()` for all standard chat-with-tools agents. It handles the ReAct loop, tool execution, context compression, and response validation internally.

```go
a, err := agent.NewAgent(
    agent.WithName("my_agent"),
    agent.WithModel(llm),
    agent.WithTools([]tools.ITool{searchTool, fileTool}),
    agent.WithMaxToolCalls(10),                              // Prevent infinite loops (default: 10)
    agent.WithContextWindow(20),                             // Auto-compress history, keep last 20 msgs
    agent.WithResponseValidator(validatorFunc),               // Optional: validate LLM output
    agent.WithSubAgent("researcher", researcherAgent),        // Optional: delegate to sub-agents
    agent.WithBeforeModelHook(beforeHook),                    // Optional: run before LLM call
    agent.WithAfterModelHook(afterHook),                      // Optional: run after LLM call
    agent.WithBeforeToolsHook(beforeToolsHook),               // Optional: run before tool execution
    agent.WithAfterToolsHook(afterToolsHook),                 // Optional: run after tool execution
    agent.WithLogger(logger),
)
```

The prebuilt agent implements `flowcontract.Node`, so it can be used as a node in a larger flow or executed standalone:

```go
// Standalone execution
finalState, err := a.Run(ctx, &initialState, streamFunc)

// Or as a node in a flow
flow.NewFlowBuilder(logger).AddNode(a)...
```

Key: Replaces manual `ChatNode` + `ToolsNode` + `ToolCondition` wiring. Use this unless you need custom control flow.

### Pattern 2: Custom Flow with Manual Wiring (Advanced)

Custom nodes for specialized logic + prebuilt `tools.NewTools` + `model.NewModelNode` for manual control. Only use when the prebuilt agent doesn't support your flow requirements.

```
START -> CustomGenNode -> (has tool calls?) -> ToolsNode -> CustomGenNode (loop)
                       -> (no tool calls)  -> ValidationNode -> END
```

Key: Uses `toolcondition.NewToolCondition()` for the conditional edge. Use `model.NewModelNode` instead of the deprecated `chat.NewChatNode`.

### Pattern 3: Multi-Agent via Delegation (Prebuilt)

Use `agent.WithSubAgent()` to create agents that can delegate tasks to specialized sub-agents. The framework auto-creates a `delegate_task` tool.

```go
researcher, _ := agent.NewAgent(agent.WithName("researcher"), agent.WithModel(llm), agent.WithTools(researchTools))
writer, _ := agent.NewAgent(agent.WithName("writer"), agent.WithModel(llm))

orchestrator, _ := agent.NewAgent(
    agent.WithName("orchestrator"),
    agent.WithModel(llm),
    agent.WithSubAgent("researcher", researcher),
    agent.WithSubAgent("writer", writer),
)
```

The orchestrator LLM receives a `delegate_task` tool with `agent_name` (enum of registered sub-agents) and `task` parameters. Each sub-agent runs with a fresh state containing just the delegated task.

### Pattern 4: Generation + Validation Loop

LLM generates content, a validation node checks it, and a fix node corrects errors in a loop.

```
START -> GenerationNode -> ValidationNode -> (has errors?) -> FixNode -> ValidationNode (loop)
                                          -> (valid?)      -> END
```

## Prebuilt Components

| Component | Import | Purpose |
|-----------|--------|---------|
| `agent.NewAgent` | `golanggraph/prebuilt/agent` | **RECOMMENDED** All-in-one ReAct agent with tools, hooks, validation, delegation |
| `model.NewModelNode` | `golanggraph/prebuilt/node/model` | Generic LLM model node (replaces deprecated `chat.NewChatNode`) |
| `tools.NewTools` | `golanggraph/prebuilt/node/tools` | Executes tool calls from history |
| `toolcondition.NewToolCondition()` | `golanggraph/prebuilt/edge/toolcondition` | Conditional edge: routes to tools or next node |
| `native.NewChatLLM` | `golanggraph/prebuilt/langchaingoextension/native` | Creates langchaingo LLM instance |
| `checkpointer.NewInMemoryCheckpointer()` | `golanggraph/checkpointer` | In-memory state checkpointing |
| `checkpointer.NewRedisCheckpointer()` | `golanggraph/checkpointer` | Redis-backed state checkpointing (for production) |

## Tool Implementation

Tools implement `tools.ITool`:

```go
type ITool interface {
    Tools(ctx context.Context) []llms.Tool                                          // Return tool definitions
    Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error)  // Execute a single tool call
}
```

The `Run` method receives a single `llms.ToolCall` and returns a `llms.ToolCallResponse`. The framework's `ToolsNode` handles iterating over tool calls, matching them to the correct tool, and appending responses to history. Tools no longer need to manage state or history directly.

See [references/tools-implementation.md](references/tools-implementation.md) for full tool patterns.

### Tool Definition

```go
func (t *MyTool) Tools(ctx context.Context) []llms.Tool {
    return []llms.Tool{{
        Type: "function",
        Function: &llms.FunctionDefinition{
            Name:        "my_tool",
            Description: "What this tool does",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "query": map[string]any{"type": "string", "description": "Search query"},
                },
                "required": []string{"query"},
            },
        },
    }}
}
```

### Tool Execution

```go
func (t *MyTool) Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error) {
    // Parse arguments from the tool call
    var args struct {
        Query string `json:"query"`
    }
    if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
        return llms.ToolCallResponse{}, xerror.Wrap(err)
    }

    // Execute tool logic
    result, err := t.execute(ctx, args.Query)
    if err != nil {
        // Return error as tool response (don't fail the agent)
        return llms.ToolCallResponse{
            ToolCallID: toolCall.ID,
            Name:       toolCall.FunctionCall.Name,
            Content:    "Error: " + err.Error(),
        }, nil
    }

    return llms.ToolCallResponse{
        ToolCallID: toolCall.ID,
        Name:       toolCall.FunctionCall.Name,
        Content:    result,
    }, nil
}
```

## LLM Call Pattern

All LLM calls use `llms.Model.GenerateContent`:

```go
completion, err := llm.GenerateContent(
    ctx,
    messages,                        // []llms.MessageContent
    llms.WithTemperature(0.7),       // Creativity control
    llms.WithTools(toolDefinitions), // Optional tool definitions
    llms.WithMaxTokens(4096),        // Optional max tokens
)
if err != nil { return xerror.Wrap(err) }

choice := completion.Choices[0]
// choice.Content  = text response
// choice.ToolCalls = tool calls (if any)
```

### Model Override

To use a specific model for a node:

```go
ctx = context.WithValue(ctx, utils.OverrideModelKey, config.SpecificModel)
completion, err := llm.GenerateContent(ctx, messages, ...)
```

## Prompt Management

### Embedded Templates

Use `//go:embed` for prompt files:

```go
//go:embed prompt.txt
var promptTemplate string
```

### Template Formatting

Use `prompts.NewPromptTemplate` from langchaingo:

```go
tmpl := prompts.NewPromptTemplate(promptTemplate, []string{"var1", "var2"})
formatted, err := tmpl.Format(map[string]any{"var1": "value1", "var2": "value2"})
```

## State Management via Metadata

### Pattern: Custom State in Metadata

```go
const MetadataKeyMyState = "my_agent_state"

type MyAgentState struct {
    CurrentStep int
    Results     []Result
    // Keep separate histories for multi-agent
    AgentAHistory []llms.MessageContent `json:"-"` // Exclude from serialization
}

func getState(st *state.State) *MyAgentState {
    if st.Metadata == nil { st.Metadata = make(map[string]interface{}) }
    if v, ok := st.Metadata[MetadataKeyMyState]; ok {
        if s, ok := v.(*MyAgentState); ok { return s }
    }
    s := &MyAgentState{}
    st.Metadata[MetadataKeyMyState] = s
    return s
}
```

### Multi-Agent History Isolation

In multi-agent flows, keep each agent's history in Metadata (NOT in `state.History`). The shared `state.History` is used for tool call routing only.

```go
type MultiAgentState struct {
    DirectorHistory  []llms.MessageContent            // Director's conversation
    CharacterHistory map[string][]llms.MessageContent  // Per-character conversations
}
```

## LLM Best Practices

See [references/llm-best-practices.md](references/llm-best-practices.md) for detailed patterns and examples.

### Context Compression (History Trimming)

> **Prebuilt Agent**: Use `agent.WithContextWindow(N)` to enable automatic context compression. The agent's built-in `contextCompressHook` preserves system messages and keeps the last N non-system messages. Manual trimming is only needed for custom flows.

Trim conversation history to stay within context limits while preserving critical messages.

**Sliding Window Pattern** (REQUIRED for long-running agents with custom flows):

```go
func (s *MyState) TrimHistory() {
    const maxLen = 10
    const keepRecent = 5
    if len(s.History) <= maxLen { return }

    preserved := []llms.MessageContent{s.History[0]} // Keep system prompt
    // Optionally keep important anchors (e.g., task definition)
    startIdx := len(s.History) - keepRecent
    preserved = append(preserved, s.History[startIdx:]...)
    s.History = preserved
}
```

Rules:
- ALWAYS keep the system prompt (index 0)
- Keep the last N messages for recency
- Optionally preserve anchor messages (task headers, key context)
- Call before each LLM invocation in loop-based agents

### Tool Response Trimming

Truncate large tool responses in history to save context window:

```go
const maxContentChars = 500
const maxResultsInResponse = 4

// In tool execution:
if len([]rune(content)) > maxContentChars {
    content = string([]rune(content)[:maxContentChars]) + "...[content truncated]"
}

// Store full results in Metadata for other nodes
currentState.Metadata["full_results"] = fullResults
```

Also trim old tool responses retroactively when history grows:

```go
func trimToolResponsesInHistory(history []llms.MessageContent) {
    for i, msg := range history {
        if msg.Role != llms.ChatMessageTypeTool { continue }
        for j, part := range msg.Parts {
            resp, ok := part.(llms.ToolCallResponse)
            if !ok { continue }
            if len([]rune(resp.Content)) > maxToolResponseChars {
                resp.Content = string([]rune(resp.Content)[:maxToolResponseChars]) + "\n...[truncated]"
                msg.Parts[j] = resp
            }
        }
        history[i] = msg
    }
}
```

### Prompt Caching

Structure prompts so static content comes first (cacheable) and dynamic content comes last:

```
[SYSTEM MESSAGE - Static, cacheable]
  - Role definition
  - Rules and constraints
  - Output format specification
  - Few-shot examples

[HUMAN MESSAGE - Dynamic, per-request]
  - Current task/input
  - Context-specific instructions
```

Rules:
- System prompt MUST be identical across calls for caching to work
- Do NOT embed dynamic data (timestamps, user IDs) in the system prompt
- Put changing context in the LAST human message
- Anthropic: cache_control breakpoints at system message boundaries
- OpenAI: automatic prefix caching on identical message prefixes

### Attention Raising in Long Context

Use structural markers to ensure the LLM focuses on critical instructions:

```go
// XML tags for structure
prompt := `<rules>
CRITICAL: You MUST respond in valid JSON format.
</rules>

<context>
... long context here ...
</context>

<task>
Generate the output based on the rules above.
</task>`

// Ephemeral reminders (appended to messages but NOT saved to history)
messagesForCall := append(history, llms.MessageContent{
    Role:  llms.ChatMessageTypeHuman,
    Parts: []llms.ContentPart{llms.TextContent{Text: "REMINDER: Respond in valid JSON only."}},
})
```

Rules:
- Place the MOST IMPORTANT instructions at the START and END of the prompt (primacy/recency effect)
- Use XML tags (`<rules>`, `<context>`, `<output>`) for structural boundaries
- Add ephemeral reminders at the end of message lists (not saved to history)
- Use `IMPORTANT:`, `CRITICAL:`, `MUST` for emphasis on key constraints
- Repeat key format instructions near the end of long prompts

### MCP (Model Context Protocol)

MCP provides a standard protocol for LLM-tool communication. In Go agents, MCP is implemented through the tool interface:

- Tools define their schema via JSON Schema parameters (matching MCP tool definitions)
- Tool results are returned as `ToolCallResponse` messages (matching MCP tool results)
- The agent framework handles the MCP message flow: LLM -> tool call -> tool result -> LLM

For MCP server integration, wrap external MCP servers as `tools.ITool` implementations that proxy calls to the MCP server.

## Human-in-the-Loop (Interrupt / Resume)

The framework supports interrupting agent execution to request human input, then resuming with the provided value.

### Interrupting from a Node

Use `flowcontract.Interrupt()` inside a node's `Run()` to pause execution:

```go
func (n *ApprovalNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    // Prepare payload describing what approval is needed
    currentState.SetInterruptPayload(map[string]any{
        "question": "Do you approve this action?",
        "details":  actionDetails,
    })
    return flowcontract.Interrupt(currentState.Metadata["interrupt_payload"])
}
```

### Checking for Interrupts

The caller checks whether the flow was interrupted:

```go
finalState, err := compiledFlow.Exec(ctx, initialState, streamFunc)
if interruptErr, ok := flowcontract.IsInterrupt(err); ok {
    // Flow paused — interruptErr.Payload contains the interrupt payload
    // Present to user, collect input, then resume
}
```

### Resuming with a Value

Resume the flow from where it was interrupted by providing the human's response:

```go
finalState, err := compiledFlow.ResumeWithValue(ctx, threadID, userResponse, streamFunc)
```

Inside the node, access the resume value via `state.GetResumeValue()`:

```go
func (n *ApprovalNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    // Check if we're resuming from an interrupt
    if resumeValue := currentState.GetResumeValue(); resumeValue != nil {
        approval := resumeValue.(string)
        if approval == "approved" {
            // Proceed with the action
            return nil
        }
        // Handle rejection
        return xerror.New("action rejected by user")
    }

    // First visit — interrupt for approval
    currentState.SetInterruptPayload(map[string]any{"question": "Approve?"})
    return flowcontract.Interrupt(currentState.Metadata["interrupt_payload"])
}
```

### Requirements

- A `Checkpointer` MUST be set on the flow for interrupt/resume to work (state is persisted between calls)
- Use `checkpointer.NewRedisCheckpointer()` for production, `checkpointer.NewInMemoryCheckpointer()` for development
- `threadID` identifies the conversation thread and is used by the checkpointer to restore state

## LangChain Chains (Non-Agent)

For simple single-shot LLM tasks (no loops, no tools), use langchaingo chains:

```go
type MyChain struct {
    llm    llms.Model
    memory schema.Memory
}

func (c *MyChain) Call(ctx context.Context, inputs map[string]any, options ...chains.ChainCallOption) (map[string]any, error) {
    // Format prompt from template
    // Call LLM
    // Parse and return output
}

func (c *MyChain) GetMemory() schema.Memory      { return c.memory }
func (c *MyChain) GetInputKeys() []string         { return []string{"input_key"} }
func (c *MyChain) GetOutputKeys() []string        { return []string{"output_key"} }
```

See [references/langchain-chains.md](references/langchain-chains.md) for examples.

## Flow Orchestration from Domain/Application

Agents are constructed in domain/application services and executed via `flow.Exec`:

```go
// In domain service constructor
overviewAgent, err := overviewagent.NewOverviewAgent(
    overviewagent.WithLogger(logger),
    overviewagent.WithLLM(llm),
    overviewagent.WithConfig(config.LLM),
)

// In domain service method
initialState := state.State{
    Metadata: map[string]interface{}{
        overviewagent.MetadataKeyState: myState,
    },
    History: []llms.MessageContent{},
}
finalState, err := s.overviewAgent.Exec(ctx, initialState, streamCallback)

// Extract results from finalState.Metadata
result := finalState.Metadata[overviewagent.MetadataKeyState].(*MyState)
```

## Agent Construction Pattern (Functional Options)

> **Prefer `agent.NewAgent()`** for standard agents. The functional options pattern below is for custom flows or wrapping the prebuilt agent in a domain-specific factory.

Every agent factory follows the functional options pattern:

```go
type Opt struct {
    logger logger.ILogger
    llm    llms.Model
    config *config.LLMConfig
}
type Option func(*Opt)

func WithLogger(l logger.ILogger) Option { return func(o *Opt) { o.logger = l } }
func WithLLM(l llms.Model) Option        { return func(o *Opt) { o.llm = l } }
func WithConfig(c *config.LLMConfig) Option { return func(o *Opt) { o.config = c } }

func NewMyAgent(options ...Option) (*flow.Flow, error) {
    opts := &Opt{}
    for _, o := range options { o(opts) }
    // Validate required options
    if opts.logger == nil { return nil, xerror.New("logger is required") }
    if opts.llm == nil    { return nil, xerror.New("llm is required") }
    // Build and compile flow
    return flow.NewFlowBuilder(opts.logger).
        SetName("my_agent").
        // ... AddNode, AddEdge ...
        Compile()
}
```

## Key Import Paths

```go
import (
    // golanggraph core
    "github.com/futurxlab/golanggraph/flow"             // flow.NewFlowBuilder, flow.Flow, flow.StartNode, flow.EndNode
    "github.com/futurxlab/golanggraph/edge"             // edge.Edge
    "github.com/futurxlab/golanggraph/state"            // state.State
    "github.com/futurxlab/golanggraph/checkpointer"     // checkpointer.NewInMemoryCheckpointer(), NewRedisCheckpointer()
    flowcontract "github.com/futurxlab/golanggraph/contract" // StreamFunc, FlowStreamEvent, Node, Interrupt(), IsInterrupt()

    // kiwi-lib (shared utilities — moved from golanggraph)
    "github.com/Yet-Another-AI-Project/kiwi-lib/logger" // logger.ILogger
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror" // xerror.Wrap, xerror.New

    // golanggraph prebuilt
    "github.com/futurxlab/golanggraph/prebuilt/agent"              // agent.NewAgent (RECOMMENDED)
    "github.com/futurxlab/golanggraph/prebuilt/node/model"         // model.NewModelNode (replaces chat.NewChatNode)
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"         // tools.NewTools, tools.ITool
    "github.com/futurxlab/golanggraph/prebuilt/edge/toolcondition" // toolcondition.NewToolCondition

    // langchaingo
    "github.com/tmc/langchaingo/llms"     // llms.Model, MessageContent, Tool, ToolCall
    "github.com/tmc/langchaingo/prompts"  // prompts.NewPromptTemplate
    "github.com/tmc/langchaingo/chains"   // chains (for chain pattern)
    "github.com/tmc/langchaingo/memory"   // memory.NewSimple()
    "github.com/tmc/langchaingo/schema"   // schema.Memory
)
```

## Shared Utilities

```go
// agent/utils/tool_utils.go
func HasToolCalls(choice *llms.ContentChoice) bool
func HasToolCallsInHistory(st state.State) bool
func CreateToolCallMessage(choice *llms.ContentChoice) llms.MessageContent

// Reusable format reminders
var JsonReminder = llms.MessageContent{
    Role:  llms.ChatMessageTypeHuman,
    Parts: []llms.ContentPart{llms.TextContent{Text: "IMPORTANT: Respond ONLY with valid JSON."}},
}
```

## Coding Checklist

Before submitting agent code, verify:

- [ ] **Prefer `agent.NewAgent()`** for standard chat-with-tools agents over manual wiring
- [ ] All custom nodes implement `Name()` and `Run()` from `flowcontract.Node`
- [ ] Agent factory uses functional options pattern with validation
- [ ] All `return err` use `xerror.Wrap(err)` (never raw errors) — import from `kiwi-lib/xerror`
- [ ] Flow starts with `flow.StartNode` and ends with `flow.EndNode`
- [ ] Long-running agents have history trimming (or use `WithContextWindow()` for prebuilt agents)
- [ ] Tool responses are truncated in history; full results stored in Metadata
- [ ] Tool `Run()` uses the new signature: `Run(ctx, llms.ToolCall) (llms.ToolCallResponse, error)`
- [ ] Prompts use `//go:embed` for templates
- [ ] Static prompt content is in system messages (cacheable)
- [ ] Dynamic content is in the last human message
- [ ] Conditional edges list ALL possible targets in `ConditionalTo`
- [ ] MaxToolCalls is set (via `agent.WithMaxToolCalls()` or hook) to prevent infinite tool loops
- [ ] Streaming events use `streamFunc` with proper nil checks
- [ ] State is saved to Metadata after mutations in nodes
- [ ] Human-in-the-Loop flows have a `Checkpointer` set
- [ ] `logger` and `xerror` are imported from `kiwi-lib`, NOT from `golanggraph`
