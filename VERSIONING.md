# Firecrawl MCP versions and transports

The npm package version, the Firecrawl API version, and the MCP protocol revision are separate:

- **Package version:** `firecrawl-mcp` follows the version in [package.json](package.json). See [CHANGELOG.md](CHANGELOG.md) for changes.
- **Firecrawl API:** the current server uses the V2 API. It does not run a separate V1 server or expose the old credential-in-path routes such as `/:apiKey/sse` and `/:apiKey/v2/sse`.
- **MCP protocol:** the CLI supports revisions `2024-11-05`, `2025-03-26`, `2025-06-18`, and `2025-11-25`. The `modernProtocol` option for embedders using `createFirecrawlMcpServer` additionally enables revision `2026-07-28`; it is not enabled by the CLI.

The source retained under `src/legacy/` is historical reference, not a second executable entry point. The package's executable is `dist/index.js`.

## Local transports

The same tools are available over stdio and HTTP, subject to the configured profile and credentials. Transport selection does not select a different Firecrawl API version.

### stdio (default)

For an MCP client that launches a command:

```bash
npx -y firecrawl-mcp
```

Configure credentials through the client's environment when authenticated tools are needed. See the [README](README.md) for API-key, OAuth-token, keyless, and self-hosted API configuration.

### Streamable HTTP

```bash
HTTP_STREAMABLE_SERVER=true npx -y firecrawl-mcp
```

By default, this listens on `localhost:3000` and serves Streamable HTTP at `/mcp`. `PORT` and `HOST` control the listener. `FASTMCP_ENDPOINT` can select `/v2/mcp`, `/v2/mcp-oauth`, or `/v2/mcp-search`; the account and search profiles have additional authentication requirements described in the README and [search-profile guide](docs/search-profile.md).

For example, to explicitly select the full `/v2/mcp` endpoint:

```bash
HTTP_STREAMABLE_SERVER=true FASTMCP_ENDPOINT=/v2/mcp npx -y firecrawl-mcp
```

### Legacy HTTP+SSE transport

The HTTP listener also serves the older MCP transport at `GET /sse`. Its first `endpoint` event supplies `/messages?sessionId=...`; the client posts JSON-RPC messages to that URL and reads replies from the event stream. This is a legacy **MCP transport**, not Firecrawl API V1.

`SSE_LOCAL=true` also enables the HTTP listener. It does not restore the old credential-in-path routes.

### Health checks and network access

The HTTP listener exposes `/health` and `/ready`. These are health endpoints, not MCP connection URLs.

Local HTTP mode checks Host and Origin headers. See [Streamable HTTP local mode](README.md#running-with-streamable-http-local-mode) before configuring a reverse proxy or allowing clients on another machine. Do not expose a listener using a shared server API key to an untrusted network.

## Hardened HTTP preset

`CLOUD_SERVICE=true` selects HTTP, binds to `0.0.0.0`, requires a credential on every request, enables safe mode, and uses upload-only file access. This is the package's hardened preset, not the full configuration of the Firecrawl-operated hosted service.

For a source checkout after building:

```bash
npm run start:cloud
```

For hosted MCP connection URLs and authentication, use the [README's hosted setup](README.md). Do not construct a hosted URL by adding an API key to a local endpoint path.

## Migrating older configurations

- Replace old credential-in-path configurations with the appropriate current setup in the README. Use the client's supported credential configuration rather than embedding a key in a URL.
- Discover the available tools with `tools/list`. Historical tools such as `firecrawl_deep_research` and `firecrawl_generate_llmstxt` are not current tool names. The README describes the current Agent and research-paper tools.
- Use the current tool schemas and README examples for scrape formats and options. V1 SDK method names and option defaults do not describe the current MCP interface.

## Testing from source

Use Node.js 22 or newer. Install dependencies with the lockfile, then run the repository's test suite:

```bash
pnpm install --frozen-lockfile
pnpm test
```

`pnpm test` builds the package and runs `tests/*.test.mjs`. There is no `test:endpoints` npm script. To run just the protocol and HTTP transport checks after building:

```bash
pnpm run build
node --test --test-concurrency=1 tests/protocol-conformance.test.mjs tests/http-transport-parity.test.mjs
```

These tests use local mock services; they do not verify a deployed hosted endpoint or call the live Firecrawl API.
