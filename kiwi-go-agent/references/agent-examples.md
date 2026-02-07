# Agent Examples

Complete code examples for each agent pattern.

## Pattern 1: Simple ReAct Chat Agent

Minimal agent using prebuilt components for chat + tools.

```go
package chat

import (
    "context"

    "github.com/futurxlab/golanggraph/checkpointer"
    "github.com/futurxlab/golanggraph/edge"
    "github.com/futurxlab/golanggraph/flow"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/prebuilt/edge/toolcondition"
    "github.com/futurxlab/golanggraph/prebuilt/node/chat"
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/tmc/langchaingo/llms"
)

type Opt struct {
    logger logger.ILogger
    llm    llms.Model
    tools  []tools.ITool
}
type Option func(*Opt)

func WithLogger(l logger.ILogger) Option { return func(o *Opt) { o.logger = l } }
func WithLLM(l llms.Model) Option        { return func(o *Opt) { o.llm = l } }
func WithTools(t []tools.ITool) Option    { return func(o *Opt) { o.tools = t } }

func NewChat(ctx context.Context, options ...Option) (*flow.Flow, error) {
    opts := &Opt{}
    for _, option := range options { option(opts) }
    if opts.logger == nil { return nil, xerror.New("logger is required") }
    if opts.llm == nil    { return nil, xerror.New("llm is required") }

    // Gather tool definitions for the chat node
    var toolDefinitions []llms.Tool
    for _, t := range opts.tools {
        toolDefinitions = append(toolDefinitions, t.Tools(ctx)...)
    }

    // Create prebuilt nodes
    chatNode, err := chat.NewChatNode(
        chat.WithLLM(opts.llm),
        chat.WithTools(toolDefinitions),
    )
    if err != nil { return nil, xerror.Wrap(err) }

    toolsNode, err := tools.NewTools(
        tools.WithTools(opts.tools),
        tools.WithMaxToolCalls(6),
    )
    if err != nil { return nil, xerror.Wrap(err) }

    // Conditional edge: if last message has tool calls -> tools, else -> end
    tc := toolcondition.NewToolCondition(toolsNode.Name(), flow.EndNode)

    cp := checkpointer.NewInMemoryCheckpointer()

    return flow.NewFlowBuilder(opts.logger).
        SetName("chat_agent").
        SetCheckpointer(cp).
        AddNode(chatNode).
        AddNode(toolsNode).
        AddEdge(edge.Edge{From: flow.StartNode, To: chatNode.Name()}).
        AddEdge(edge.Edge{
            From:          chatNode.Name(),
            ConditionalTo: []string{toolsNode.Name(), flow.EndNode},
            ConditionFunc: tc.Condition,
        }).
        AddEdge(edge.Edge{From: toolsNode.Name(), To: chatNode.Name()}).
        Compile()
}
```

**Usage:**

```go
chatFlow, err := chat.NewChat(ctx,
    chat.WithLogger(logger),
    chat.WithLLM(llm),
    chat.WithTools([]tools.ITool{searchTool}),
)

initialState := state.State{
    History: []llms.MessageContent{
        {Role: llms.ChatMessageTypeSystem, Parts: []llms.ContentPart{llms.TextContent{Text: "You are a helpful assistant."}}},
        {Role: llms.ChatMessageTypeHuman, Parts: []llms.ContentPart{llms.TextContent{Text: "Find me info about Go."}}},
    },
}

finalState, err := chatFlow.Exec(ctx, initialState, nil)
```

## Pattern 2: Custom Agent with Tool Loop + Validation

Agent with custom generation node, tool integration, and a validation step.

```go
package checklist

import (
    "context"
    _ "embed"

    "github.com/futurxlab/golanggraph/checkpointer"
    "github.com/futurxlab/golanggraph/edge"
    "github.com/futurxlab/golanggraph/flow"
    flowcontract "github.com/futurxlab/golanggraph/contract"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/prebuilt/edge/toolcondition"
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"
    "github.com/futurxlab/golanggraph/state"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/tmc/langchaingo/llms"
    "github.com/tmc/langchaingo/prompts"
    agentutils "myproject/infrastructure/ai/agent/utils"
)

//go:embed prompt_generation.txt
var generationPrompt string

//go:embed prompt_validation.txt
var validationPrompt string

const MetadataKeyState = "checklist_state"

type ChecklistState struct {
    Input     string
    Checklist *Checklist
}

// --- Generation Node ---

type GenerationNode struct {
    llm    llms.Model
    logger logger.ILogger
}

func NewGenerationNode(llm llms.Model, logger logger.ILogger) *GenerationNode {
    return &GenerationNode{llm: llm, logger: logger}
}

func (n *GenerationNode) Name() string { return "checklist_generation" }

func (n *GenerationNode) Run(ctx context.Context, currentState *state.State, streamFunc flowcontract.StreamFunc) error {
    s := getState(currentState)

    // Format prompt
    tmpl := prompts.NewPromptTemplate(generationPrompt, []string{"input"})
    formatted, err := tmpl.Format(map[string]any{"input": s.Input})
    if err != nil { return xerror.Wrap(err) }

    // Build history if empty
    if len(currentState.History) == 0 {
        currentState.History = append(currentState.History,
            llms.MessageContent{Role: llms.ChatMessageTypeSystem, Parts: []llms.ContentPart{llms.TextContent{Text: formatted}}},
            llms.MessageContent{Role: llms.ChatMessageTypeHuman, Parts: []llms.ContentPart{llms.TextContent{Text: "Generate the checklist."}}},
        )
    }

    // Call LLM
    completion, err := n.llm.GenerateContent(ctx, currentState.History,
        llms.WithTemperature(0.7),
        llms.WithTools(n.toolDefs),
    )
    if err != nil { return xerror.Wrap(err) }

    choice := completion.Choices[0]

    // Handle tool calls
    if agentutils.HasToolCalls(choice) {
        currentState.History = append(currentState.History, agentutils.CreateToolCallMessage(choice))
        return nil
    }

    // Parse response
    currentState.History = append(currentState.History, llms.MessageContent{
        Role: llms.ChatMessageTypeAI, Parts: []llms.ContentPart{llms.TextContent{Text: choice.Content}},
    })

    // Parse JSON into state
    s.Checklist = parseChecklist(choice.Content)
    saveState(currentState, s)
    return nil
}

// --- Validation Node (no LLM, pure logic) ---

type ValidationNode struct {
    logger logger.ILogger
}

func (n *ValidationNode) Name() string { return "checklist_validation" }

func (n *ValidationNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    s := getState(currentState)
    if s.Checklist == nil {
        return xerror.New("checklist is nil after generation")
    }
    // Validate structure, field counts, etc.
    return nil
}

// --- Agent Factory ---

func NewChecklistAgent(options ...Option) (*flow.Flow, error) {
    opts := applyOptions(options)

    genNode := NewGenerationNode(opts.llm, opts.logger)
    valNode := &ValidationNode{logger: opts.logger}
    toolsNode, _ := tools.NewTools(tools.WithTools(opts.tools), tools.WithMaxToolCalls(3))

    tc := toolcondition.NewToolCondition(toolsNode.Name(), valNode.Name())

    return flow.NewFlowBuilder(opts.logger).
        SetName("checklist_agent").
        SetCheckpointer(checkpointer.NewInMemoryCheckpointer()).
        AddNode(genNode).
        AddNode(valNode).
        AddNode(toolsNode).
        AddEdge(edge.Edge{From: flow.StartNode, To: genNode.Name()}).
        AddEdge(edge.Edge{
            From:          genNode.Name(),
            ConditionalTo: []string{toolsNode.Name(), valNode.Name()},
            ConditionFunc: tc.Condition,
        }).
        AddEdge(edge.Edge{From: toolsNode.Name(), To: genNode.Name()}).
        AddEdge(edge.Edge{From: valNode.Name(), To: flow.EndNode}).
        Compile()
}
```

## Pattern 3: Generation + Validation + Fix Loop

Agent that generates content, validates it, and loops through fixes if invalid.

```go
// Flow: START -> Generate -> Validate -> (errors?) -> Fix -> Validate (loop)
//                                     -> (valid?)  -> END

func NewOverviewAgent(options ...Option) (*flow.Flow, error) {
    opts := applyOptions(options)

    genNode := NewOverviewGenerationNode(opts.llm, opts.logger)
    valNode := NewValidationNode(opts.logger) // Pure logic, no LLM
    fixNode := NewFixNode(opts.llm, opts.logger)

    return flow.NewFlowBuilder(opts.logger).
        SetName("overview_agent").
        SetCheckpointer(checkpointer.NewInMemoryCheckpointer()).
        AddNode(genNode).
        AddNode(valNode).
        AddNode(fixNode).
        AddEdge(edge.Edge{From: flow.StartNode, To: genNode.Name()}).
        AddEdge(edge.Edge{From: genNode.Name(), To: valNode.Name()}).
        AddEdge(edge.Edge{
            From:          valNode.Name(),
            ConditionalTo: []string{fixNode.Name(), flow.EndNode},
            ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
                s := getState(&st)
                if len(s.ValidationErrors) > 0 {
                    return fixNode.Name(), nil
                }
                return flow.EndNode, nil
            },
        }).
        AddEdge(edge.Edge{From: fixNode.Name(), To: valNode.Name()}). // Loop back
        Compile()
}

// Fix node uses lower temperature for more deterministic corrections
func (n *FixNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    s := getState(currentState)
    errText := formatErrors(s.ValidationErrors)

    currentState.History = append(currentState.History, llms.MessageContent{
        Role: llms.ChatMessageTypeHuman,
        Parts: []llms.ContentPart{llms.TextContent{
            Text: "Fix the following errors:\n" + errText,
        }},
    })

    completion, err := n.llm.GenerateContent(ctx, currentState.History,
        llms.WithTemperature(0.3), // Lower temp for fixes
    )
    if err != nil { return xerror.Wrap(err) }

    // Parse corrected output, clear errors
    s.Output = parseOutput(completion.Choices[0].Content)
    s.ValidationErrors = nil
    saveState(currentState, s)
    return nil
}
```

## Pattern 4: Multi-Agent (Director + Characters)

Complex agent where a Director coordinates multiple Character agents, each with isolated history.

```go
func NewMultiAgent(options ...Option) (*flow.Flow, error) {
    opts := applyOptions(options)

    initNode := NewInitNode(opts.logger)
    directorNode := NewDirectorNode(opts.llm, opts.logger)
    characterNode := NewCharacterNode(opts.llm, opts.logger)
    transitionNode := NewTransitionNode(opts.logger)
    toolsNode, _ := tools.NewTools(tools.WithTools(opts.tools))

    return flow.NewFlowBuilder(opts.logger).
        SetName("multi_agent").
        SetCheckpointer(checkpointer.NewInMemoryCheckpointer()).
        AddNode(initNode).
        AddNode(directorNode).
        AddNode(characterNode).
        AddNode(transitionNode).
        AddNode(toolsNode).
        // Init -> Director
        AddEdge(edge.Edge{From: flow.StartNode, To: initNode.Name()}).
        AddEdge(edge.Edge{From: initNode.Name(), To: directorNode.Name()}).
        // Director routes to character, transition, or end
        AddEdge(edge.Edge{
            From:          directorNode.Name(),
            ConditionalTo: []string{characterNode.Name(), transitionNode.Name(), flow.EndNode},
            ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
                s := getState(&st)
                if s.CurrentIndex >= s.TotalItems { return flow.EndNode, nil }
                if s.AdvanceToNext { return transitionNode.Name(), nil }
                return characterNode.Name(), nil
            },
        }).
        // Character -> tool call? -> tools -> character (loop) OR -> director
        AddEdge(edge.Edge{
            From:          characterNode.Name(),
            ConditionalTo: []string{toolsNode.Name(), directorNode.Name()},
            ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
                if agentutils.HasToolCallsInHistory(st) {
                    return toolsNode.Name(), nil
                }
                return directorNode.Name(), nil
            },
        }).
        // Tools -> back to caller (tracked via ToolCallCaller in metadata)
        AddEdge(edge.Edge{
            From:          toolsNode.Name(),
            ConditionalTo: []string{characterNode.Name(), directorNode.Name()},
            ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
                s := getState(&st)
                if s.ToolCallCaller == "character" {
                    s.ToolCallCaller = ""
                    return characterNode.Name(), nil
                }
                return directorNode.Name(), nil
            },
        }).
        // Transition -> director (next item) or end
        AddEdge(edge.Edge{
            From:          transitionNode.Name(),
            ConditionalTo: []string{directorNode.Name(), flow.EndNode},
            ConditionFunc: func(ctx context.Context, st state.State) (string, error) {
                s := getState(&st)
                if s.CurrentIndex >= s.TotalItems { return flow.EndNode, nil }
                return directorNode.Name(), nil
            },
        }).
        Compile()
}
```

**Key Multi-Agent Patterns:**

1. **History Isolation**: Each agent role keeps its own history in Metadata, NOT in `state.History`. The shared history is only used for tool call routing.

2. **Tool Call Routing**: When multiple nodes can trigger tool calls, track the caller in Metadata (`ToolCallCaller`) so the tools node can route back to the correct caller.

3. **Transition Node**: A lightweight node that resets per-step state (turn counters, flags) and advances to the next item.

4. **Streaming from Character Nodes**: Use `streamFunc` to emit partial results as they're generated:

```go
if streamFunc != nil {
    data, _ := json.Marshal(result)
    if err := streamFunc(ctx, &flowcontract.FlowStreamEvent{Chunk: string(data)}); err != nil {
        return xerror.Wrap(err)
    }
}
```
