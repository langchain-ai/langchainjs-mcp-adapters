# LangChain.js MCP Adapters

[![npm version](https://img.shields.io/npm/v/@langchain/mcp-adapters.svg)](https://www.npmjs.com/package/@langchain/mcp-adapters)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This library provides a lightweight wrapper that makes [Anthropic Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) tools compatible with [LangChain.js](https://github.com/langchain-ai/langchainjs) and [LangGraph.js](https://github.com/langchain-ai/langgraphjs).

## Features

- 🔌 **Transport Options**

  - Connect to MCP servers via stdio (local) or SSE (remote)
  - Support for custom headers in SSE connections for authentication
  - Configurable reconnection strategies for both transport types

- 🔄 **Multi-Server Management**

  - Connect to multiple MCP servers simultaneously
  - Auto-organize tools by server or access them as a flattened collection
  - Convenient configuration via JSON file

- 🧩 **Agent Integration**

  - Compatible with LangChain.js and LangGraph.js
  - Optimized for OpenAI, Anthropic, and Google models

- 🛠️ **Development Features**
  - Uses `debug` package for debug logging
  - Flexible configuration options
  - Robust error handling

## Installation

```bash
npm install @langchain/mcp-adapters
```

### Optional Dependencies

For SSE connections with custom headers in Node.js:

```bash
npm install eventsource
```

For enhanced SSE header support:

```bash
npm install extended-eventsource
```

# Example: Manage the MCP Client yourself

This example shows how you can manage your own MCP client and use it to get tools that you can pass to a LangGraph prebuilt ReAcT agent.

```bash
npm install @langchain/mcp-adapters @langchain/langgraph @langchain/core @langchain/openai

export OPENAI_API_KEY=<your_api_key>
```

## Client

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { loadMcpTools } from "@langchain/mcp-adapters";

// Initialize the ChatOpenAI model
const model = new ChatOpenAI({ modelName: "gpt-4" });

// Automatically starts and connects to a MCP reference server
const transport = new StdioClientTransport({
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-math"],
});

// Initialize the client
const client = new Client({
  name: "math-client",
  version: "1.0.0",
});

try {
  // Connect to the transport
  await client.connect(transport);

  // Get tools
  const tools = await loadMcpTools("math", client);

  // Create and run the agent
  const agent = createReactAgent({ llm: model, tools });
  const agentResponse = await agent.invoke({
    messages: [{ role: "user", content: "what's (3 + 5) x 12?" }],
  });
  console.log(agentResponse);
} catch (e) {
  console.error(e);
} finally {
  // Clean up connection
  await client.close();
}
```

# Example: Connect to one or more servers via config

The library also allows you to connect to multiple MCP servers and load tools from them:

## Client

```ts
import { MultiServerMCPClient } from "@langchain/mcp-adapters";
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

// Create client and connect to server
const client = new MultiServerMCPClient({
  // adds a STDIO connection to a server named "math"
  math: {
    transport: "stdio",
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-math"],
  },

  // add additional servers by adding more keys to the config
  // here's a filesystem server
  filesystem: {
    transport: "stdio",
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-filesystem"],
  },
});

const tools = await client.getTools();

// Create an OpenAI model
const model = new ChatOpenAI({
  modelName: "gpt-4o",
  temperature: 0,
});

// Create the React agent
const agent = createReactAgent({
  llm: model,
  tools,
});

// Run the agent
const mathResponse = await agent.invoke({
  messages: [{ role: "user", content: "what's (3 + 5) x 12?" }],
});
const weatherResponse = await agent.invoke({
  messages: [{ role: "user", content: "what is the weather in nyc?" }],
});

await client.close();
```

For more detailed examples, see the [examples](./examples) directory.

## Browser Environments

When using in browsers:

- Native EventSource API doesn't support custom headers
- Consider using a proxy or pass authentication via query parameters
- May require CORS configuration on the server side

## Troubleshooting

### Common Issues

1. **Connection Failures**:

   - Verify the MCP server is running
   - Check command paths and network connectivity

2. **Tool Execution Errors**:

   - Examine server logs for error messages
   - Ensure input parameters match the expected schema

3. **Headers Not Applied**:
   - Install the recommended `extended-eventsource` package
   - Set `useNodeEventSource: true` in SSE connections

### Debug Logging

This package makes use of the [debug](https://www.npmjs.com/package/debug) package for debug logging.

Logging is disabled by default, and can be enabled by setting the `DEBUG` environment variable as per
the instructions in the debug package.

To output all debug logs from this package:

```bash
DEBUG='@langchain/mcp-adapters:*'
```

To output debug logs only from the `client` module:

```bash
DEBUG='@langchain/mcp-adapters:client'
```

To output debug logs only from the `tools` module:

```bash
DEBUG='@langchain/mcp-adapters:tools'
```

## Examples

To see available examples, run:

```bash
yarn run
```

Alternatively, here are some common examples:

| Example                           | Command                                    |
|-----------------------------------|--------------------------------------------|
| Weather Server                    | yarn start:weather                         |
| Math Server                       | yarn start:math                            |
| Filesystem LangGraph Example      | yarn start:filesystem_langgraph_example    |
| Config LangGraph Test             | yarn start:config_langgraph_test           |
| Firecrawl Custom Config Example   | yarn start:firecrawl_custom_config_example |
| Firecrawl Default Config Example  | yarn start:firecrawl_default_config_example|
| Firecrawl Enhanced Config Example | yarn start:firecrawl_enhanced_config_example|
| Firecrawl Mixed Loading Example   | yarn start:firecrawl_mixed_loading_example |
| LangGraph Example                 | yarn start:langgraph_example               |
| MCP Over Docker Example           | yarn start:mcp_over_docker_example         |

  

## License

MIT

## Acknowledgements

Big thanks to [@vrknetha](https://github.com/vrknetha), [@cawstudios](https://caw.tech) for the initial implementation!

## Contributing

Contributions are welcome! Please check out our [contributing guidelines](CONTRIBUTING.md) for more information.
