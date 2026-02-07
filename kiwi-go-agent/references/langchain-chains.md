# LangChain Chains (Non-Agent)

For simple single-shot LLM tasks that do not require loops, tool calls, or multi-step workflows, use langchaingo `chains.Chain` instead of building a full golanggraph agent.

## When to Use Chains vs Agents

| Use Chain | Use Agent (golanggraph) |
|-----------|------------------------|
| Single LLM call, no loops | Multi-step workflow with loops |
| No tool calls needed | Tool integration required |
| Simple input -> output transform | Conditional routing between nodes |
| Stateless (no conversation history) | Stateful with message history |
| Title generation, summarization, classification | Chat, research, generation + validation |

## Chain Interface

```go
// From github.com/leeif/langchaingo/chains
type Chain interface {
    Call(ctx context.Context, inputs map[string]any, options ...ChainCallOption) (map[string]any, error)
    GetMemory() schema.Memory
    GetInputKeys() []string
    GetOutputKeys() []string
}
```

| Method | Purpose |
|--------|---------|
| `Call` | Execute the chain with input map, return output map |
| `GetMemory` | Return the memory instance (use `memory.NewSimple()` for stateless) |
| `GetInputKeys` | Declare expected input keys (for validation) |
| `GetOutputKeys` | Declare expected output keys (for downstream consumers) |

## Complete Example: Title Generation Chain

A chain that generates a title from conversation content.

### Directory Structure

```
infrastructure/ai/langchain/title/
    chain.go       // Chain implementation
    prompt.txt     // Embedded prompt template
```

### prompt.txt

```
You are a title generator. Given the following conversation content, generate a concise, descriptive title.

Content:
{{.content}}

Respond with only the title text. No quotes, no explanation.
```

### chain.go

```go
package title

import (
    "context"
    _ "embed"

    "github.com/leeif/langchaingo/chains"
    "github.com/leeif/langchaingo/llms"
    "github.com/leeif/langchaingo/memory"
    "github.com/leeif/langchaingo/prompts"
    "github.com/leeif/langchaingo/schema"

    "myproject/infrastructure/ai/agent/utils"
    "myproject/infrastructure/config"
    "myproject/pkg/logger"
    "github.com/futurxlab/golanggraph/xerror"
)

//go:embed prompt.txt
var promptTemplate string

type TitleChain struct {
    llm    llms.Model
    logger logger.ILogger
    memory schema.Memory
    config *config.LLMConfig
}

func NewTitleChain(llm llms.Model, logger logger.ILogger, cfg *config.LLMConfig) *TitleChain {
    return &TitleChain{
        llm:    llm,
        logger: logger,
        memory: memory.NewSimple(),
        config: cfg,
    }
}

func (c *TitleChain) Call(ctx context.Context, inputs map[string]any, options ...chains.ChainCallOption) (map[string]any, error) {
    content, ok := inputs["content"].(string)
    if !ok {
        return nil, xerror.New("input 'content' must be a string")
    }

    // Format the prompt template
    tmpl := prompts.NewPromptTemplate(promptTemplate, []string{"content"})
    formatted, err := tmpl.Format(map[string]any{"content": content})
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    // Override model for this chain (use a cheaper/faster model for titles)
    ctx = context.WithValue(ctx, utils.OverrideModelKey, c.config.TitleModel)

    // Build messages
    messages := []llms.MessageContent{
        {
            Role:  llms.ChatMessageTypeHuman,
            Parts: []llms.ContentPart{llms.TextContent{Text: formatted}},
        },
    }

    // Call LLM
    completion, err := c.llm.GenerateContent(ctx, messages,
        llms.WithTemperature(0.3),
        llms.WithMaxTokens(100),
    )
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    title := completion.Choices[0].Content

    return map[string]any{"title": title}, nil
}

func (c *TitleChain) GetMemory() schema.Memory  { return c.memory }
func (c *TitleChain) GetInputKeys() []string     { return []string{"content"} }
func (c *TitleChain) GetOutputKeys() []string    { return []string{"title"} }
```

### Usage from Application Layer

```go
// In application service
titleChain := title.NewTitleChain(llm, logger, config.LLM)

result, err := titleChain.Call(ctx, map[string]any{
    "content": conversationText,
})
if err != nil {
    return xerror.Wrap(err)
}

generatedTitle := result["title"].(string)
```

## Key Patterns

### Prompt Templates

Use `prompts.NewPromptTemplate` with Go template syntax (`{{.varName}}`):

```go
//go:embed my_prompt.txt
var myPrompt string

tmpl := prompts.NewPromptTemplate(myPrompt, []string{"var1", "var2"})
formatted, err := tmpl.Format(map[string]any{
    "var1": "value1",
    "var2": "value2",
})
```

- Template variables use `{{.varName}}` syntax
- The second argument to `NewPromptTemplate` lists expected variable names
- Use `//go:embed` to load templates from files (keeps code clean, enables caching)

### Model Override

Use a specific model for a chain (e.g., cheaper model for simple tasks):

```go
ctx = context.WithValue(ctx, utils.OverrideModelKey, config.SpecificModel)
completion, err := c.llm.GenerateContent(ctx, messages, ...)
```

### Stateless Memory

Chains are typically stateless. Use `memory.NewSimple()`:

```go
func (c *MyChain) GetMemory() schema.Memory {
    return memory.NewSimple() // No conversation memory
}
```

### Error Handling

Follow the same error handling rules as agents:

```go
// Use xerror.Wrap for all returned errors
if err != nil {
    return nil, xerror.Wrap(err)
}

// Use xerror.New for custom errors
if input == "" {
    return nil, xerror.New("input must not be empty")
}
```

## Coding Checklist for Chains

- [ ] Implements all 4 `chains.Chain` interface methods
- [ ] Prompt template loaded via `//go:embed`
- [ ] Uses `prompts.NewPromptTemplate` for variable interpolation
- [ ] All errors wrapped with `xerror.Wrap()` or created with `xerror.New()`
- [ ] Model override via `context.WithValue` if using a non-default model
- [ ] `GetInputKeys()` and `GetOutputKeys()` accurately reflect the `Call()` signature
- [ ] Temperature and MaxTokens set appropriately for the task
- [ ] Input validation at the start of `Call()`
