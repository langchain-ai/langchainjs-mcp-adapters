# LangChain.js MCP Adapters

This package provides adapters for using [Model Context Protocol (MCP)](https://github.com/modelcontextprotocol/specification) tools with LangChain.js. It enables seamless integration between LangChain.js and MCP servers, allowing you to use MCP tools in your LangChain applications.

[![npm version](https://img.shields.io/npm/v/langchainjs-mcp-adapters.svg)](https://www.npmjs.com/package/langchainjs-mcp-adapters)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Features

- Connect to MCP servers using stdio or SSE transports
- **Connect to MCP servers programmatically** using Server instances
- Connect to multiple MCP servers simultaneously
- Configure connections using a JSON configuration file
- **Support for custom headers in SSE connections** (great for authentication!)
- Integrate MCP tools with LangChain.js agents
- Comprehensive logging capabilities

## Installation

```bash
npm install langchainjs-mcp-adapters
```

For Node.js environments with SSE connections requiring headers, you need to install the optional dependency:

```bash
npm install eventsource
```

## Prerequisites

- Node.js >= 18
- For stdio transport: Python MCP servers require Python 3.8+
- For SSE transport: A running MCP server with SSE endpoint
- For SSE with headers in Node.js: The `eventsource` package

## Usage

### Connecting to an MCP Server

You can connect to an MCP server using either stdio, SSE transport, or programmatically:

```typescript
import { MultiServerMCPClient } from 'langchainjs-mcp-adapters';

// Create a client
const client = new MultiServerMCPClient();

// Connect to a server using stdio
await client.connectToServerViaStdio(
  'math-server', // A name to identify this server
  'python', // Command to run
  ['./math_server.py'] // Arguments for the command
);

// Connect to a server using SSE
await client.connectToServerViaSSE(
  'weather-server', // A name to identify this server
  'http://localhost:8000/sse' // URL of the SSE server
);

// Connect to a server using SSE with custom headers
await client.connectToServerViaSSE(
  'auth-server', // A name to identify this server
  'http://localhost:8000/sse', // URL of the SSE server
  {
    Authorization: 'Bearer your-token-here',
    'X-Custom-Header': 'custom-value',
  },
  true // Use Node.js EventSource (requires eventsource package)
);

// Connect to a server programmatically
import { Server } from '@modelcontextprotocol/sdk/server/index.js';

// Create a server instance
const server = new Server({
  name: 'example-server',
  version: '1.0.0',
});

// Register tools on the server
server.registerTool({
  name: 'greet',
  description: 'Greet a person by name',
  inputSchema: {
    type: 'object',
    properties: {
      name: {
        type: 'string',
        description: 'The name of the person to greet',
      },
    },
    required: ['name'],
  },
  handler: async params => {
    const { name } = params;
    return {
      content: [
        {
          type: 'text',
          text: `Hello, ${name}! Nice to meet you.`,
        },
      ],
    };
  },
});

// Connect to the server programmatically
await client.connectToServer('example-server', server);

// Get all tools from all connected servers
const tools = client.getTools();

// Use the tools
const result = await tools[0].invoke({ param1: 'value1', param2: 'value2' });

// Close the client when done
await client.close();
```

### Initializing Multiple Connections

You can also initialize multiple connections at once:

```typescript
import { MultiServerMCPClient } from 'langchainjs-mcp-adapters';

const client = new MultiServerMCPClient({
  'math-server': {
    command: 'python',
    args: ['./math_server.py'],
  },
  'weather-server': {
    transport: 'sse',
    url: 'http://localhost:8000/sse',
  },
  'auth-server': {
    transport: 'sse',
    url: 'http://localhost:8000/sse',
    headers: {
      Authorization: 'Bearer your-token-here',
      'X-Custom-Header': 'custom-value',
    },
    useNodeEventSource: true, // Use Node.js EventSource for headers support
  },
});

// Initialize all connections
await client.initializeConnections();

// Get all tools
const tools = client.getTools();

// Close all connections when done
await client.close();
```

### Using Configuration File

You can define your MCP server configurations in a JSON file (`mcp.json`) and load them:

```typescript
import { MultiServerMCPClient } from 'langchainjs-mcp-adapters';

// Create a client from the config file
const client = MultiServerMCPClient.fromConfigFile();
// Or specify a custom path: MultiServerMCPClient.fromConfigFile("./config/mcp.json");

// Initialize all connections
await client.initializeConnections();

// Get all tools
const tools = client.getTools();

// Close all connections when done
await client.close();
```

Example `mcp.json` file:

```json
{
  "servers": {
    "math": {
      "command": "python",
      "args": ["./examples/math_server.py"]
    },
    "weather": {
      "transport": "sse",
      "url": "http://localhost:8000/sse"
    },
    "auth-server": {
      "transport": "sse",
      "url": "http://localhost:8000/sse",
      "headers": {
        "Authorization": "Bearer your-token-here",
        "X-Custom-Header": "custom-value"
      },
      "useNodeEventSource": true
    }
  }
}
```

Note: Programmatic connections (using `server` instances) cannot be defined in the JSON configuration file as they require actual JavaScript objects. Use the `connectToServer` method or the constructor with a JavaScript object for programmatic connections.

The client will attempt to connect to all servers defined in the configuration file. If a server is not available, it will log an error and continue with the available servers. If no servers are available, it will throw an error.

```typescript
// Error handling when initializing connections
try {
  const client = MultiServerMCPClient.fromConfigFile();
  await client.initializeConnections();
  // Use the client...
} catch (error) {
  console.error('Failed to connect to any servers:', error.message);
}
```

### Using with LangChain Agents

You can use MCP tools with LangChain agents:

```typescript
import { MultiServerMCPClient } from 'langchainjs-mcp-adapters';
import { ChatOpenAI } from '@langchain/openai';
import { createOpenAIFunctionsAgent, AgentExecutor } from 'langchain/agents';
import { ChatPromptTemplate } from '@langchain/core/prompts';

// Create a client and connect to servers
const client = new MultiServerMCPClient();
await client.connectToServerViaStdio('math-server', 'python', ['./math_server.py']);

// Get tools
const tools = client.getTools();

// Create an agent
const model = new ChatOpenAI({ temperature: 0 });
const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'You are a helpful assistant that can use tools to solve problems.'],
  ['human', '{input}'],
]);

const agent = createOpenAIFunctionsAgent({
  llm: model,
  tools,
  prompt,
});

const agentExecutor = new AgentExecutor({
  agent,
  tools,
});

// Run the agent
const result = await agentExecutor.invoke({
  input: 'What is 5 + 3?',
});

console.log(result.output);

// Close the client when done
await client.close();
```

### Using with Google's Gemini Models

The package also supports integration with Google's Gemini models:

```typescript
import { MultiServerMCPClient } from 'langchainjs-mcp-adapters';
import { ChatGoogleGenerativeAI } from '@langchain/google-genai';
import { createGoogleGenerativeAIFunctionsAgent, AgentExecutor } from 'langchain/agents';
import { ChatPromptTemplate } from '@langchain/core/prompts';

// Create a client and connect to servers
const client = new MultiServerMCPClient();
await client.connectToServerViaStdio('math-server', 'python', ['./math_server.py']);

// Get tools
const tools = client.getTools();

// Create a Gemini agent
const model = new ChatGoogleGenerativeAI({
  modelName: 'gemini-pro',
  apiKey: process.env.GOOGLE_API_KEY,
});

// Create and run the agent
// ... similar to the OpenAI example
```

## Example MCP Servers

### Math Server (stdio transport)

Here's an example of a simple MCP server in Python using stdio transport:

```python
from mcp.server.fastmcp import FastMCP

# Create a server
mcp = FastMCP(name="Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers and return the result."""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two integers and return the result."""
    return a * b

# Run the server with stdio transport
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Weather Server (SSE transport)

Here's an example of an MCP server using SSE transport:

```python
from mcp.server.fastmcp import FastMCP

# Create a server
mcp = FastMCP(name="Weather")

@mcp.tool()
def get_temperature(city: str) -> str:
    """Get the current temperature for a city."""
    # Mock implementation
    temperatures = {
        "new york": "72°F",
        "london": "65°F",
        "tokyo": "25 degrees Celsius",
    }

    city_lower = city.lower()
    if city_lower in temperatures:
        return f"The current temperature in {city} is {temperatures[city_lower]}."
    else:
        return "Temperature data not available for this city"

# Run the server with SSE transport
if __name__ == "__main__":
    mcp.run(transport="sse")
```

## Running the Examples

The package includes several example files that demonstrate how to use MCP adapters:

1. `math_example.ts` - Basic example using a math server with stdio transport
2. `sse_example.ts` - Example using a weather server with SSE transport
3. `multi_transport_example.ts` - Example connecting to multiple servers with different transport types
4. `json_config_example.ts` - Example using server configurations from an `mcp.json` file
5. `gemini_example.ts` - Example using Google's Gemini models
6. `logging_example.ts` - Example demonstrating logging capabilities
7. `sse_with_headers_example.ts` - Example showing how to use custom headers with SSE connections

To run the examples:

```bash
# First build the project
npm run build

# Start the weather server with SSE transport
python examples/weather_server.py

# In another terminal, run the examples using Node.js
node dist/examples/math_example.js
node dist/examples/sse_example.js
node dist/examples/json_config_example.js
```

## Troubleshooting

### Common Issues

1. **Connection Failures**: Ensure the MCP server is running and accessible
2. **Tool Execution Errors**: Check the server logs for error messages
3. **Transport Issues**: Verify the transport configuration (stdio or SSE)
4. **Headers Not Applied**: When using headers with SSE, make sure you've installed the `eventsource` package and set `useNodeEventSource` to true

### Debugging

Enable debug logging to get more information:

```typescript
import { logger } from 'langchainjs-mcp-adapters';

// Set logger level to debug
logger.level = 'debug';
```

## Development

For information about contributing to this project, including GitHub Actions workflows, npm publishing, and more, please see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
