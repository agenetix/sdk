# @agenetix/sdk

Telemetry SDK for MCP (Model Context Protocol) servers. Track tool invocations, errors, and performance.

[![npm version](https://badge.fury.io/js/%40agenetix%2Fsdk.svg)](https://www.npmjs.com/package/@agenetix/sdk)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## What is this?

This SDK adds observability to your MCP servers. When AI agents call your tools, Agenetix tracks:

- **Tool invocations** - Which tools are called and how often
- **Errors** - Failures with full context for debugging
- **Performance** - Latency metrics and success rates
- **Metadata** - Custom attributes for filtering and analysis

View your data in the [Agenetix Dashboard](https://agenetix.com).

## Installation

```bash
npm install @agenetix/sdk
```

## Quick Start

```typescript
import { AgenetixTelemetry } from '@agenetix/sdk';

// Initialize with your API key
const agenetix = new AgenetixTelemetry({
  apiKey: process.env.AGENETIX_API_KEY!,
  endpoint: 'https://api.agenetix.com/api/v1/telemetry',
  mcpServerId: process.env.AGENETIX_MCP_SERVER_ID,
});

// Set server info for metadata
agenetix.setServerInfo('my-mcp-server', '1.0.0');

// Wrap your tool handlers with trace()
const result = await agenetix.trace('get_user', async () => {
  return await api.getUser(userId);
});
```

## API

### `AgenetixTelemetry`

The main class for telemetry.

```typescript
const agenetix = new AgenetixTelemetry({
  apiKey: string;           // Required: Your Agenetix API key
  endpoint?: string;        // Optional: Telemetry endpoint (default: https://api.agenetix.com/api/v1/telemetry)
  mcpServerId?: string;     // Optional: MCP server ID for grouping
  debug?: boolean;          // Optional: Enable debug logging
  flushInterval?: number;   // Optional: Batch flush interval in ms (default: 5000)
  maxBatchSize?: number;    // Optional: Max events per batch (default: 100)
});
```

### `setServerInfo(name, version)`

Set server metadata included with all events.

```typescript
agenetix.setServerInfo('my-server', '1.2.3');
```

### `trace<T>(toolName, fn)`

Wrap an async function to track its execution.

```typescript
const result = await agenetix.trace('search_products', async () => {
  return await api.searchProducts(query);
});
```

The trace automatically captures:
- Start time
- End time / duration
- Success or failure
- Error details if thrown

### `trackInvocation(invocation)`

Manually track a tool invocation.

```typescript
agenetix.trackInvocation({
  toolName: 'get_user',
  startTime: Date.now(),
  endTime: Date.now() + 150,
  success: true,
  metadata: { userId: '123' },
});
```

### `flush()`

Force send all pending events. Called automatically on interval.

```typescript
await agenetix.flush();
```

### `shutdown()`

Flush and stop the telemetry client.

```typescript
await agenetix.shutdown();
```

## Configuration

### Environment Variables

The SDK reads these environment variables:

| Variable | Description |
|----------|-------------|
| `AGENETIX_API_KEY` | Your Agenetix API key (required) |
| `AGENETIX_TELEMETRY_URL` | Telemetry endpoint URL |
| `AGENETIX_MCP_SERVER_ID` | MCP server ID for grouping |
| `AGENETIX_DEBUG` | Set to `true` for debug logs |

### With @agenetix/openapi-to-mcp

If you generated your MCP server with [@agenetix/openapi-to-mcp](https://www.npmjs.com/package/@agenetix/openapi-to-mcp) and the `--agenetix` flag, the SDK is already integrated. Just set your environment variables:

```bash
AGENETIX_API_KEY=your-api-key
AGENETIX_TELEMETRY_URL=https://api.agenetix.com/api/v1/telemetry
AGENETIX_MCP_SERVER_ID=mcp_xxxxxxxxxxxx
```

## Example: Manual Integration

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { CallToolRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { AgenetixTelemetry } from '@agenetix/sdk';

const agenetix = new AgenetixTelemetry({
  apiKey: process.env.AGENETIX_API_KEY!,
});

agenetix.setServerInfo('my-server', '1.0.0');

const server = new Server(
  { name: 'my-server', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name: toolName, arguments: args } = request.params;
  
  // Wrap the tool execution with telemetry
  return agenetix.trace(toolName, async () => {
    switch (toolName) {
      case 'get_data':
        return await getData(args);
      default:
        throw new Error(`Unknown tool: ${toolName}`);
    }
  });
});

// Graceful shutdown
process.on('SIGINT', async () => {
  await agenetix.shutdown();
  process.exit(0);
});
```

## Data Format

Events are batched and sent as:

```typescript
interface TelemetryBatch {
  apiKey: string;
  mcpServerId?: string;
  serverName?: string;
  serverVersion?: string;
  invocations: ToolInvocation[];
}

interface ToolInvocation {
  toolName: string;
  startTime: number;
  endTime: number;
  success: boolean;
  errorMessage?: string;
  errorStack?: string;
  metadata?: Record<string, unknown>;
}
```

## Self-Hosting

Point the SDK at your own telemetry endpoint:

```typescript
const agenetix = new AgenetixTelemetry({
  apiKey: 'your-key',
  endpoint: 'https://your-server.com/api/v1/telemetry',
});
```

The endpoint should accept POST requests with the `TelemetryBatch` JSON body.

## License

MIT © [Agenetix](https://agenetix.com)

