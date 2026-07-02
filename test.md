Nice job on getting the fake streaming up and running! Transforming that static DOM extraction into an asynchronous chunked event stream is a huge step toward making the proxy feel native to local terminal agents or CLI wrappers.
Now for **Tool Support (Function Calling)**. Because M365 Copilot's web interface doesn't natively accept or return OpenAI-formatted tools objects, your proxy layer (server.js) needs to act as a two-way translator.
To achieve this, you need to implement **System-Prompt Tool Injection** on the way in, and **Regex/String Trapping** on the way out.
## The Tool Support Architecture
Here is how the data flow changes when a downstream framework (like GitHub Copilot CLI or an autonomous agent) sends a request containing a tools array:
```
[Downstream Agent] ──(OpenAI JSON Request with tools[])──> [server.js Proxy]
                                                                │
                                            (Injects tool schema into system prompt)
                                                                ▼
[M365 Copilot Web] <──(Raw Text with strict JSON instructions)──┘
        │
(Outputs explicit text block: [TOOL_CALL: name]...)
        │
        ▼
[server.js Proxy] ──(Intercepts text, converts to tool_calls JSON chunk)──> [Downstream Agent]

```
## Step 1: Prompt Injection (Request Side)
When req.body.tools is present, your proxy must dynamically compile those tool schemas into a strict, machine-readable instruction set and append it to either the system message or the last user message before typing it into the browser.
Add this utility function to your code to map the OpenAI schema array into an explicit Markdown block:
```javascript
function injectToolsToPrompt(originalPrompt, tools) {
  if (!tools || tools.length === 0) return originalPrompt;

  let toolInstructions = `\n\n[SYSTEM NOTICE: TOOL AVAILABILITY]
You have access to external tools to help fulfill the user's request. If you determine that a tool call is necessary, you MUST respond ONLY with the exact block structure below and NO conversational filler, markdown formatting outside the block, or pre-text.

Available Tools:`;

  tools.forEach(t => {
    const fn = t.function;
    toolInstructions += `\n- Name: ${fn.name}\n  Description: ${fn.description}\n  Parameters Schema: ${JSON.stringify(fn.parameters)}`;
  });

  toolInstructions += `\n\nIf you decide to invoke a tool, you must use this EXACT syntax:
[TOOL_CALL: tool_name]
{
  "argument_name": "value"
}
[END_TOOL_CALL]

If no tool is required, respond with your normal conversational answer.`;

  return originalPrompt + toolInstructions;
}

```
## Step 2: Trapping and Parsing the Stream (Response Side)
As you scrape the text output from the Edge DOM for your fake streaming engine, your code must constantly monitor the incoming text chunks for the [TOOL_CALL: marker.
If a tool call is detected, your proxy changes the streaming chunk layout from a regular content delta to a tool_calls delta.
Here is an implementation example to bake directly into your streaming loop:
```javascript
const crypto = require('crypto');

function handleResponseStreaming(res, rawTextFromBrowser) {
  // 1. Check if the browser is outputting a tool call pattern
  const toolCallRegex = /\[TOOL_CALL:\s*([a-zA-Z0-9_-]+)\]([\s\S]*?)(\[END_TOOL_CALL\]|$)/;
  const match = rawTextFromBrowser.match(toolCallRegex);

  if (match) {
    const toolName = match[1];
    let toolArgs = match[2].trim();
    const callId = `call_${crypto.randomBytes(8).toString('hex')}`;

    // Clean up potential loose markdown ticks the LLM might have wrapped inside the block
    toolArgs = toolArgs.replace(/^```json\s*/i, '').replace(/```$/, '').trim();

    // 2. Format as a valid OpenAI streaming tool_call chunk
    const chunk = {
      id: `chatcmpl-${crypto.randomUUID()}`,
      object: "chat.completion.chunk",
      created: Math.floor(Date.now() / 1000),
      model: "m365-copilot-bridge",
      choices: [{
        index: 0,
        delta: {
          tool_calls: [{
            index: 0,
            id: callId,
            type: "function",
            function: {
              name: toolName,
              arguments: toolArgs
            }
          }]
        },
        finish_reason: "tool_calls"
      }]
    };

    res.write(`data: ${JSON.stringify(chunk)}\n\n`);
    res.write(`data: [DONE]\n\n`);
    res.end();
    return true; // Intercepted successfully
  }

  return false; // Regular text stream, proceed with your existing fake stream chunker
}

```
## Step 3: Handling the Return Cycle (role: "tool")
Once your client framework executes the tool locally, it will send a *new* request back to your proxy containing the tool's execution result. You need to ensure your message parser gracefully flattens this back into a conversational context for M365 Copilot.
When processing req.body.messages, verify how you stringify the history:
```javascript
function compileMessagesToWebPrompt(messages) {
  return messages.map(msg => {
    if (msg.role === 'user') return `User: ${msg.content}`;
    if (msg.role === 'assistant' && msg.tool_calls) {
      // Reconstruct the structural block so Copilot remembers what it called
      return `Assistant: [TOOL_CALL: ${msg.tool_calls[0].function.name}]\n${msg.tool_calls[0].function.arguments}\n[END_TOOL_CALL]`;
    }
    if (msg.role === 'tool') {
      // Present the execution output to the web interface as an external context update
      return `[SYSTEM: Tool execution result for call validation]\nOutput: ${msg.content}`;
    }
    return `Assistant: ${msg.content}`;
  }).join('\n\n');
}

```
> **Crucial Edge-Case Note:**
> Sometimes M365 Copilot will output text *before* deciding to call a tool (e.g., "Sure, let me check that for you... [TOOL_CALL: ...] "). Your text scraper needs to be robust enough to clear out or ignore the pre-text conversational fluff if your consuming downstream framework expects *either* a text output *or* a tool call, but not both in a single response cycle.
> 
