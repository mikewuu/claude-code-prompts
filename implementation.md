# Agent System Implementation

This document explains how the agent loop, parallel tool calls, and result collation work in the Claude Code implementation.

## Agent Loop

The agent loop is the core execution mechanism that allows Claude to interact with tools and produce coherent responses. Here's how it works:

```javascript
async function *call({ prompt: userPrompt }, { abortController, options, readFileTimestamps }) {
  // Track execution time
  let startTime = Date.now();
  
  // Get available tools based on permission mode
  let availableTools = await getTools(permissionMode);
  
  // Initialize with user prompt
  let messages = [createUserMessage({ content: userPrompt })];
  
  // Send progress update with "Initializing..."
  yield { 
    type: "progress", 
    content: formatMessage({ content: "Initializing…" }),
    normalizedMessages: normalizeMessages(messages),
    tools: availableTools 
  };
  
  // Load system prompts and model configuration
  let [agentPrompt, systemInfo, modelName, tokenLimit] = await Promise.all([
    getAgentPrompt(),
    getSystemInfo(),
    getModelName(),
    calculateTokenLimit(messages)
  ]);
  
  // Tool usage tracking
  let toolUseCount = 0;
  
  // Message logging setup
  let logFunction = setupMessageLogging(messageLogName, forkNumber);
  
  // Main agent execution loop
  for await (let message of executeAgent(
    messages, 
    agentPrompt, 
    systemInfo, 
    modelConfiguration,
    { 
      abortController,
      options: {
        permissionMode,
        forkNumber,
        messageLogName,
        tools: availableTools,
        commands: [],
        verbose,
        model: modelName,
        maxTokens: tokenLimit
      },
      messageId: generateMessageId(messages),
      readFileTimestamps
    }
  )) {
    // Add message to conversation
    messages.push(message);
    
    // Log messages to file
    writeMessageLog(getLogPath(messageLogName, forkNumber, logFunction()), serializeMessages(messages));
    
    // Skip if not an assistant message
    if (message.type !== "assistant") continue;
    
    // Normalize messages for UI
    let normalizedMessages = normalizeMessages(messages);
    
    // Process tool uses in the message
    for (let content of message.message.content) {
      if (content.type !== "tool_use") continue;
      
      // Increment tool usage counter
      toolUseCount++;
      
      // Send progress update for each tool use
      yield {
        type: "progress",
        content: normalizedMessages.find(msg => 
          msg.type === "assistant" && 
          msg.message.content[0]?.type === "tool_use" && 
          msg.message.content[0].id === content.id
        ),
        normalizedMessages: normalizedMessages,
        tools: availableTools
      }
    }
  }
  
  // Get final normalized messages
  let finalMessages = normalizeMessages(messages);
  
  // Get last message
  let lastMessage = getLastMessage(messages);
  
  // Check if request was interrupted
  if (lastMessage?.message.content.some(content => 
    content.type === "text" && content.text === INTERRUPT_MESSAGE
  )) {
    yield {
      type: "progress",
      content: lastMessage,
      normalizedMessages: finalMessages,
      tools: availableTools
    };
  } else {
    // Send completion statistics
    let stats = [
      toolUseCount === 1 ? "1 tool use" : `${toolUseCount} tool uses`,
      formatTokenCount(getTokenCount(lastMessage)) + " tokens",
      formatExecutionTime(Date.now() - startTime)
    ];
    
    yield {
      type: "progress",
      content: formatMessage({ content: `Done (${stats.join(" · ")})` }),
      normalizedMessages: finalMessages,
      tools: availableTools
    }
  }
  
  // Extract text content from final message
  let textContent = lastMessage.message.content.filter(content => content.type === "text");
  
  // Return final result
  yield {
    type: "result",
    data: textContent,
    normalizedMessages: finalMessages,
    resultForAssistant: this.renderResultForAssistant(textContent),
    tools: availableTools
  }
}
```

## Parallel Tool Calls with BatchTool

The BatchTool allows running multiple tool operations in parallel for improved efficiency:

```javascript
const BatchTool = {
  name: "BatchTool",
  description: `
    - Batch execution tool that runs multiple tool invocations in a single request
    - Tools are executed in parallel when possible, and otherwise serially
    - Takes a list of tool invocations (tool_name and input pairs)
    - Returns the collected results from all invocations
    - Use this tool when you need to run multiple independent tool operations at once
    - Each tool will respect its own permissions and validation rules
  `,
  inputSchema: {
    description: "A short description of the batch operation",
    invocations: [{
      tool_name: "The name of the tool to invoke",
      input: "The input to pass to the tool"
    }]
  },
  
  async *call({ description, invocations }, context) {
    // Track results
    const results = {};
    
    // Run tool invocations in parallel where possible
    await Promise.all(invocations.map(async ({ tool_name, input }) => {
      try {
        // Find the tool by name
        const tool = findToolByName(tool_name);
        
        // Validate input against tool schema
        validateInput(input, tool.inputSchema);
        
        // Call the tool and collect results
        const result = await tool.call(input, context);
        
        // Store result
        results[tool_name] = result;
      } catch (error) {
        // Handle errors
        results[tool_name] = { error: error.message };
      }
    }));
    
    // Format results for return
    const formattedResults = Object.entries(results).map(([toolName, result]) => {
      return `${toolName}(${formatToolInput(toolName, result.input)}):${formatToolOutput(result)}`;
    }).join('');
    
    // Return collated results
    yield {
      type: "result",
      data: formattedResults,
      resultForAssistant: this.renderResultForAssistant(formattedResults)
    };
  },
  
  // Additional methods for rendering, validation, etc.
  renderResultForAssistant(result) {
    return result;
  }
};
```

## Result Collation

Results from tool invocations, especially parallel ones, need to be properly collated and presented to both the agent and the user:

```javascript
// Rendering tool results for the assistant
function renderResultForAssistant(result) {
  // Different tools might need different result formatting
  // For most tools, this just returns the raw result
  return result;
}

// Rendering tool results for the user interface
function renderToolResultMessage(result, { verbose }) {
  if (typeof result !== "string") return null;
  
  // Remove any prefix notes
  let cleanResult = result.replace(PREFIX_PATTERN, "");
  
  if (!cleanResult) return null;
  
  // Format result for display
  return createElement(
    Container, 
    { justifyContent: "space-between", width: "100%" },
    createElement(
      Container, 
      null,
      createElement(Text, null, "  ⎿  "),
      createElement(
        Container, 
        { flexDirection: "column", paddingLeft: 0 },
        // Split result into lines and render each
        cleanResult.split("\n")
          .filter(line => line.trim() !== "")
          .slice(0, verbose ? undefined : MAX_VISIBLE_LINES)
          .map((line, index) => createElement(Text, { key: index }, line)),
        
        // Show truncation indicator if needed
        !verbose && 
        cleanResult.split("\n").length > MAX_VISIBLE_LINES && 
        createElement(
          Text, 
          { color: getSecondaryTextColor() },
          "... (+", 
          cleanResult.split("\n").length - MAX_VISIBLE_LINES, 
          " items)"
        )
      )
    )
  );
}

// Collating BatchTool results
function formatBatchResults(results) {
  return Object.entries(results)
    .map(([toolName, result]) => {
      // Format the input parameters
      const inputStr = formatInputParameters(result.input);
      
      // Format the output based on result type
      const outputStr = formatOutput(result.output);
      
      // Combine into standard format
      return `${toolName}(${inputStr}):${outputStr}`;
    })
    .join('');
}
```

## Complete Flow

1. **User Request** - User sends a prompt to Claude Code
2. **Agent Initialization** - System loads available tools and initializes the agent
3. **Agent Processing** - The agent analyzes the request and determines required tools
4. **Tool Execution**:
   - For single tool use: tools are executed directly
   - For multiple tools: BatchTool executes them in parallel when possible
5. **Result Handling** - Tool results are formatted and returned to the agent
6. **Agent Response** - Agent continues processing with tool results
7. **Progress Updates** - System yields progress updates during execution
8. **Final Response** - Agent provides final response after all tool operations complete

This architecture allows efficient tool usage with parallel execution while maintaining a coherent conversation flow and providing appropriate progress updates to the user.