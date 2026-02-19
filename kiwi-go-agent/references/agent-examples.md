# Agent Examples

Complete code examples for each agent pattern.

## Pattern 1: Prebuilt Agent (RECOMMENDED)

Minimal agent using `agent.NewAgent()` with tools. Handles the entire ReAct loop internally.

```go
package weatheragent

import (
    "context"
    "encoding/json"

    "github.com/Yet-Another-AI-Project/kiwi-lib/logger"
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror"
    "github.com/futurxlab/golanggraph/prebuilt/agent"
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"
    "github.com/futurxlab/golanggraph/state"
    "github.com/tmc/langchaingo/llms"
)

// --- Weather Tool ---

type WeatherTool struct{}

func (t *WeatherTool) Tools(ctx context.Context) []llms.Tool {
    return []llms.Tool{{
        Type: "function",
        Function: &llms.FunctionDefinition{
            Name:        "get_weather",
            Description: "Get current weather for a city",
            Parameters: map[string]any{
                "type": "object",
                "properties": map[string]any{
                    "city": map[string]any{"type": "string", "description": "City name"},
                },
                "required": []string{"city"},
            },
        },
    }}
}

func (t *WeatherTool) Run(ctx context.Context, toolCall llms.ToolCall) (llms.ToolCallResponse, error) {
    var args struct {
        City string `json:"city"`
    }
    if err := json.Unmarshal([]byte(toolCall.FunctionCall.Arguments), &args); err != nil {
        return llms.ToolCallResponse{}, xerror.Wrap(err)
    }

    // Simulated weather lookup
    result := map[string]any{"city": args.City, "temp": "22C", "condition": "sunny"}
    data, _ := json.Marshal(result)

    return llms.ToolCallResponse{
        ToolCallID: toolCall.ID,
        Name:       toolCall.FunctionCall.Name,
        Content:    string(data),
    }, nil
}

// --- Agent Factory ---

func NewWeatherAgent(llm llms.Model, log logger.ILogger) (*agent.Agent, error) {
    return agent.NewAgent(
        agent.WithName("weather_agent"),
        agent.WithModel(llm),
        agent.WithTools([]tools.ITool{&WeatherTool{}}),
        agent.WithMaxToolCalls(5),
        agent.WithLogger(log),
    )
}
```

**Usage:**

```go
a, err := weatheragent.NewWeatherAgent(llm, logger)

initialState := state.State{
    History: []llms.MessageContent{
        {Role: llms.ChatMessageTypeSystem, Parts: []llms.ContentPart{llms.TextContent{Text: "You are a weather assistant."}}},
        {Role: llms.ChatMessageTypeHuman, Parts: []llms.ContentPart{llms.TextContent{Text: "What's the weather in Tokyo?"}}},
    },
}

finalState, err := a.Run(ctx, &initialState, nil)
fmt.Println(finalState.GetLastResponse())
```

## Pattern 2: Prebuilt Agent with Response Validation

Agent that validates LLM responses and retries if validation fails.

```go
func NewValidatedAgent(llm llms.Model, log logger.ILogger) (*agent.Agent, error) {
    // Validator function: checks if response is valid JSON
    validator := func(ctx context.Context, response string) error {
        var result map[string]any
        if err := json.Unmarshal([]byte(response), &result); err != nil {
            return fmt.Errorf("response must be valid JSON: %w", err)
        }
        if _, ok := result["answer"]; !ok {
            return fmt.Errorf("response must contain 'answer' field")
        }
        return nil
    }

    return agent.NewAgent(
        agent.WithName("validated_agent"),
        agent.WithModel(llm),
        agent.WithTools([]tools.ITool{&SearchTool{}}),
        agent.WithResponseValidator(validator),
        agent.WithContextWindow(20), // Auto-compress history
        agent.WithLogger(log),
    )
}
```

When validation fails, the agent's built-in `responseValidationHook` appends a `[VALIDATION ERROR]` message to history and jumps back to the model node for a retry.

## Pattern 3: Multi-Agent via Delegation

Orchestrator agent that delegates tasks to specialized sub-agents.

```go
func NewResearchOrchestrator(llm llms.Model, log logger.ILogger) (*agent.Agent, error) {
    // Create specialized sub-agents
    researcher, err := agent.NewAgent(
        agent.WithName("researcher"),
        agent.WithModel(llm),
        agent.WithTools([]tools.ITool{&SearchTool{}, &FileSearchTool{}}),
        agent.WithMaxToolCalls(10),
        agent.WithLogger(log),
    )
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    writer, err := agent.NewAgent(
        agent.WithName("writer"),
        agent.WithModel(llm),
        agent.WithLogger(log),
    )
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    // Create orchestrator with sub-agents
    // The framework auto-creates a `delegate_task` tool with agent_name enum
    return agent.NewAgent(
        agent.WithName("orchestrator"),
        agent.WithModel(llm),
        agent.WithSubAgent("researcher", researcher),
        agent.WithSubAgent("writer", writer),
        agent.WithMaxToolCalls(15),
        agent.WithLogger(log),
    )
}
```

**Usage:**

```go
orchestrator, err := NewResearchOrchestrator(llm, logger)

initialState := state.State{
    History: []llms.MessageContent{
        {Role: llms.ChatMessageTypeSystem, Parts: []llms.ContentPart{llms.TextContent{Text: "You are a research orchestrator. Use the researcher to gather information, then the writer to produce a report."}}},
        {Role: llms.ChatMessageTypeHuman, Parts: []llms.ContentPart{llms.TextContent{Text: "Write a report about Go's concurrency model."}}},
    },
}

finalState, err := orchestrator.Run(ctx, &initialState, streamFunc)
```

Each sub-agent runs with a fresh `state.State` containing just the delegated task message. The sub-agent's `GetLastResponse()` is returned as the tool result to the orchestrator.

## Pattern 4: Human-in-the-Loop (Interrupt / Resume)

Agent that pauses for human approval before executing sensitive actions.

```go
package approval

import (
    "context"

    "github.com/Yet-Another-AI-Project/kiwi-lib/logger"
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror"
    "github.com/futurxlab/golanggraph/checkpointer"
    "github.com/futurxlab/golanggraph/edge"
    "github.com/futurxlab/golanggraph/flow"
    flowcontract "github.com/futurxlab/golanggraph/contract"
    "github.com/futurxlab/golanggraph/prebuilt/agent"
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"
    "github.com/futurxlab/golanggraph/state"
    "github.com/tmc/langchaingo/llms"
)

// ApprovalNode interrupts the flow to request human approval
type ApprovalNode struct{}

func (n *ApprovalNode) Name() string { return "approval" }

func (n *ApprovalNode) Run(ctx context.Context, currentState *state.State, _ flowcontract.StreamFunc) error {
    // Check if we're resuming from a previous interrupt
    if resumeValue := currentState.GetResumeValue(); resumeValue != nil {
        decision := resumeValue.(string)
        if decision == "approved" {
            return nil // Continue to execution
        }
        // Rejected — add rejection message and end
        currentState.History = append(currentState.History, llms.MessageContent{
            Role:  llms.ChatMessageTypeHuman,
            Parts: []llms.ContentPart{llms.TextContent{Text: "Action was rejected by user."}},
        })
        return nil
    }

    // First visit — present the plan and interrupt for approval
    lastResponse := currentState.GetLastResponse()
    currentState.SetInterruptPayload(map[string]any{
        "question": "The agent wants to execute the following plan. Approve?",
        "plan":     lastResponse,
    })
    return flowcontract.Interrupt(currentState.Metadata["interrupt_payload"])
}

// Build the flow: Agent -> Approval -> Execution
func NewApprovalFlow(llm llms.Model, log logger.ILogger) (*flow.Flow, error) {
    // Prebuilt agent for planning
    plannerAgent, err := agent.NewAgent(
        agent.WithName("planner"),
        agent.WithModel(llm),
        agent.WithTools([]tools.ITool{&SearchTool{}}),
        agent.WithLogger(log),
    )
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    approvalNode := &ApprovalNode{}

    // Use Redis checkpointer for production (required for interrupt/resume)
    cp := checkpointer.NewInMemoryCheckpointer()

    return flow.NewFlowBuilder(log).
        SetName("approval_flow").
        SetCheckpointer(cp).
        AddNode(plannerAgent).
        AddNode(approvalNode).
        AddEdge(edge.Edge{From: flow.StartNode, To: plannerAgent.Name()}).
        AddEdge(edge.Edge{From: plannerAgent.Name(), To: approvalNode.Name()}).
        AddEdge(edge.Edge{From: approvalNode.Name(), To: flow.EndNode}).
        Compile()
}
```

**Usage (caller side):**

```go
approvalFlow, err := approval.NewApprovalFlow(llm, logger)

initialState := state.State{
    History: []llms.MessageContent{
        {Role: llms.ChatMessageTypeHuman, Parts: []llms.ContentPart{llms.TextContent{Text: "Delete all expired records from the database."}}},
    },
}

// First run — will be interrupted at ApprovalNode
finalState, err := approvalFlow.Exec(ctx, initialState, streamFunc)
if interruptErr, ok := flowcontract.IsInterrupt(err); ok {
    // Present interruptErr.Payload to user, collect approval
    fmt.Println("Approval needed:", interruptErr.Payload)

    // Resume with user's decision
    threadID := finalState.GetThreadID()
    finalState, err = approvalFlow.ResumeWithValue(ctx, threadID, "approved", streamFunc)
}
```

## Pattern 5: Custom Flow with Manual Wiring (Advanced)

Agent with custom generation node, tool integration, and a validation step. Use when `agent.NewAgent()` doesn't support your flow requirements.

```go
package checklist

import (
    "context"
    _ "embed"

    "github.com/Yet-Another-AI-Project/kiwi-lib/logger"
    "github.com/Yet-Another-AI-Project/kiwi-lib/xerror"
    "github.com/futurxlab/golanggraph/checkpointer"
    "github.com/futurxlab/golanggraph/edge"
    "github.com/futurxlab/golanggraph/flow"
    flowcontract "github.com/futurxlab/golanggraph/contract"
    "github.com/futurxlab/golanggraph/prebuilt/edge/toolcondition"
    "github.com/futurxlab/golanggraph/prebuilt/node/tools"
    "github.com/futurxlab/golanggraph/state"
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
    toolsNode, _ := tools.NewTools(tools.WithTools(opts.tools))

    tc := toolcondition.NewToolCondition(6, toolsNode.Name(), genNode.Name(), "")

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

## Pattern 6: Generation + Validation + Fix Loop

Agent that generates content, validates it, and loops through fixes if invalid. This pattern doesn't use tools, so it requires manual flow wiring.

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

## Key Patterns Summary

| Pattern | When to Use | Approach |
|---------|------------|----------|
| **1. Prebuilt Agent** | Standard chat-with-tools agents (RECOMMENDED) | `agent.NewAgent()` |
| **2. Validated Agent** | Need response format validation | `agent.NewAgent()` + `WithResponseValidator()` |
| **3. Multi-Agent Delegation** | Specialized sub-agents for different tasks | `agent.NewAgent()` + `WithSubAgent()` |
| **4. Human-in-the-Loop** | Require human approval before actions | `flowcontract.Interrupt()` + `ResumeWithValue()` |
| **5. Custom Flow** | Complex control flow beyond prebuilt agent | Manual `flow.NewFlowBuilder()` wiring |
| **6. Gen + Validate + Fix** | Content generation with quality checks | Manual flow with validation loop |

> **Note**: The old "Director + Characters" multi-agent pattern using shared state and manual history isolation has been superseded by Pattern 3 (delegation via `WithSubAgent()`). For legacy code using the old pattern, consider migrating to delegation.
