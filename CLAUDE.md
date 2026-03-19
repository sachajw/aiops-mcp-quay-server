# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development Commands

```bash
make build          # Build binary to ./bin/quay-mcp (requires cmd/quay-mcp/main.go)
make test           # Run all tests with go test -v ./...
make fmt            # Format code with go fmt
make lint           # Lint with golangci-lint (must be installed separately)
make deps           # Download and tidy dependencies
make clean          # Remove build artifacts
make run-example    # Build and run with -url https://quay.io -example
go test -v ./...    # Run tests directly
go test -v -run TestName ./test/  # Run a single test
```

**Note:** The project currently has no `cmd/quay-mcp/main.go` entry point — `make build` will fail until it is created. The `examples/example.go` and `test/main_test.go` files are `package main` but have no `main()` function.

## Architecture

This is a Go MCP (Model Context Protocol) server that bridges LLMs to the Quay container registry API. It communicates with MCP clients over **stdio**.

```
[LLM / MCP Client] --stdio--> [QuayMCPServer] --> [QuayClient] --> [Quay Registry API]
```

### How it works

1. **Swagger spec discovery** — At startup, `QuayClient.FetchSwaggerSpec()` fetches the OpenAPI 2.0 spec from `{registry}/api/v1/discovery` (with `/discovery` fallback) and parses it via `pb33f/libopenapi`.

2. **Tag-based filtering** — `DiscoverEndpoints()` and `GenerateTools()` only expose GET endpoints tagged with: `manifest`, `organization`, `repository`, `robot`, `tag`.

3. **Dynamic tool generation** — `GenerateTools()` creates one `mcp.Tool` per endpoint with auto-generated input schemas (path params required, query params optional, plus a `resource_uri` compat param). Tool names are prefixed with `quay_`.

4. **Unified handler** — A single `createToolHandler()` closure handles all tool calls. It strips the `quay_` prefix, looks up the endpoint by operation ID or derived path identifier, and delegates to `MakeAPICallWithParams()`.

### Package structure

- **`internal/client/quay_client.go`** — Core: HTTP client, Swagger parsing, endpoint discovery, tool generation, API calls with auth. All the heavy lifting is here.
- **`internal/server/mcp_server.go`** — MCP server wiring: creates `mcp-go` server, registers tools, starts stdio transport.
- **`internal/types/types.go`** — Single `EndpointInfo` struct (Method, Path, Summary, OperationID, Tags, Parameters).

### Key dependencies

- `github.com/mark3labs/mcp-go` v0.32.0 — MCP SDK (server, tool definitions, stdio transport)
- `github.com/pb33f/libopenapi` v0.22.3 — OpenAPI/Swagger 2.0 spec parsing

### CLI flags (defined in missing main.go)

- `-url <registry-url>` — Quay registry URL (required)
- `-token <oauth-token>` — OAuth token for authenticated access (optional)
- `-example` — Run in example/demonstration mode (optional)
