---
name: mcp-server
description: Scaffold a new MCP (Model Context Protocol) server in Python (FastMCP) or Go (mcp-go). Generates tool definitions, transport config, Makefile, tests, and CI workflow. Use when creating a new MCP server.
argument-hint: "<server-name> [--lang python|go] [--tools tool1,tool2]"
allowed-tools: Write, Bash, Read, Glob
---

# MCP Server Scaffold

Generate a production-ready MCP server following established conventions from jyotishganit-mcp (Python) and mymcp (Go).

## Step 1 — Parse Arguments

| Parameter | Default | Description |
|-----------|---------|-------------|
| `server-name` | (required) | Server name in kebab-case (e.g. `my-data-server`) |
| `--lang` | `python` | Language: `python` or `go` |
| `--tools` | `hello` | Comma-separated list of tool names to scaffold |

Derive:
- `package_name` = server-name with hyphens to underscores (Python) or as-is (Go module)
- `binary_name` = server-name + `-mdc` suffix (Go only, e.g. `my-data-server-mdc`)

## Step 2 — Generate Python MCP Server

When `--lang python`, create:

```
<server-name>/
├── src/<package_name>/
│   ├── __init__.py
│   ├── __main__.py          # python -m entry point
│   └── server.py            # FastMCP server + tool definitions
├── tests/
│   ├── __init__.py
│   ├── test_server.py       # Server instantiation tests
│   └── test_tools.py        # Tool logic tests
├── .github/workflows/
│   └── ci.yml               # Lint + type-check + test matrix
├── pyproject.toml
├── Makefile
├── .gitignore
├── CLAUDE.md
└── README.md
```

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "<server-name>"
version = "0.1.0"
description = "MCP server for <server-name>"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-asyncio>=0.21.0",
    "ruff>=0.1.0",
    "mypy>=1.0",
]

[project.scripts]
<server-name> = "<package_name>.server:main"

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.ruff]
line-length = 100
target-version = "py310"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "ARG", "SIM"]
ignore = ["E501"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --tb=short"

[tool.mypy]
python_version = "3.10"
disallow_untyped_defs = true
check_untyped_defs = true
strict_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
ignore_missing_imports = true
```

### server.py

```python
"""<server-name> MCP server."""

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("<server-name>", json_response=True)


@mcp.tool()
def <tool_name>(<params>) -> str:
    """Tool description."""
    # Implementation
    return result


def main() -> None:
    """Run the MCP server over stdio."""
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

For each tool in `--tools`, generate a `@mcp.tool()` function with:
- Type-hinted parameters
- A docstring (used as the tool description by FastMCP)
- A placeholder implementation returning a JSON string

### __main__.py

```python
"""Allow running as python -m <package_name>."""
from .server import main
main()
```

### Makefile

```makefile
.PHONY: help install install-dev check lint format typecheck test

help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'

install: ## Install package
	pip install -e .

install-dev: ## Install with dev deps
	pip install -e ".[dev]"

check: lint typecheck test ## All checks

lint: ## Ruff check and format check
	ruff check src tests
	ruff format --check src tests

format: ## Format code
	ruff format src tests

typecheck: ## mypy type checking
	mypy src/

test: ## Run tests
	pytest
```

### .github/workflows/ci.yml

```yaml
name: CI
on: [push, pull_request]
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[dev]"
      - run: ruff check src tests
      - run: ruff format --check src tests
      - run: mypy src/
      - run: pytest
```

## Step 3 — Generate Go MCP Server

When `--lang go`, create:

```
<server-name>/
├── cmd/server/
│   └── main.go              # Entry point, transport selection
├── internal/
│   ├── server/
│   │   └── server.go        # MCP server struct, tool registration
│   ├── tools/
│   │   └── tools.go         # Tool args/result structs, implementations
│   └── models/
│       └── models.go        # Domain model structs
├── go.mod
├── Makefile
├── .gitignore
├── CLAUDE.md
└── README.md
```

### go.mod

```go
module github.com/adeshmukh/<server-name>

go 1.24

require github.com/mark3labs/mcp-go v0.41.1
```

Run `go mod tidy` after generating.

### cmd/server/main.go

```go
package main

import (
    "flag"
    "fmt"
    "log"
    "os"

    "github.com/adeshmukh/<server-name>/internal/server"
    mcpserver "github.com/mark3labs/mcp-go/server"
)

func main() {
    transport := flag.String("transport", "stdio", "Transport mode: stdio or http")
    host := flag.String("host", "localhost", "Host for HTTP transport")
    port := flag.Int("port", 8000, "Port for HTTP transport")
    flag.Parse()

    srv, err := server.New()
    if err != nil {
        log.Fatalf("Failed to create server: %v", err)
    }

    mcpSrv := srv.MCPServer()

    switch *transport {
    case "stdio":
        log.SetOutput(os.Stderr)
        log.Println("Starting <server-name> (stdio mode)")
        if err := mcpserver.ServeStdio(mcpSrv); err != nil {
            log.Fatalf("Server error: %v", err)
        }
    case "http":
        sseSrv := mcpserver.NewSSEServer(mcpSrv)
        addr := fmt.Sprintf("%s:%d", *host, *port)
        log.Printf("Starting <server-name> (HTTP mode) on %s", addr)
        if err := sseSrv.Start(addr); err != nil {
            log.Fatalf("Server error: %v", err)
        }
    default:
        log.Fatalf("Unknown transport: %s", *transport)
    }
}
```

### internal/server/server.go

```go
package server

import (
    "context"
    "encoding/json"

    "github.com/adeshmukh/<server-name>/internal/tools"
    "github.com/mark3labs/mcp-go/mcp"
    mcpserver "github.com/mark3labs/mcp-go/server"
)

type Server struct {
    mcpServer *mcpserver.MCPServer
    tools     *tools.Tools
}

func New() (*Server, error) {
    t := tools.New()

    mcpSrv := mcpserver.NewMCPServer(
        "<server-name>",
        "0.1.0",
        mcpserver.WithToolCapabilities(true),
    )

    s := &Server{mcpServer: mcpSrv, tools: t}
    s.registerTools()
    return s, nil
}

func (s *Server) MCPServer() *mcpserver.MCPServer {
    return s.mcpServer
}

func (s *Server) registerTools() {
    // Register each tool with schema and handler
}
```

For each tool in `--tools`, add to `registerTools()`:

```go
s.mcpServer.AddTool(mcp.Tool{
    Name:        "<tool_name>",
    Description: "Description of <tool_name>",
    InputSchema: mcp.ToolInputSchema{
        Type: "object",
        Properties: map[string]interface{}{
            "input": map[string]interface{}{
                "type":        "string",
                "description": "Input parameter",
            },
        },
        Required: []string{"input"},
    },
}, s.handle<ToolName>)
```

And add a handler method:

```go
func (s *Server) handle<ToolName>(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    var args tools.<ToolName>Args
    argsMap, ok := req.Params.Arguments.(map[string]interface{})
    if !ok {
        return mcp.NewToolResultError("invalid arguments"), nil
    }
    // Parse args from map
    result := s.tools.<ToolName>(args)
    resultJSON, _ := json.MarshalIndent(result, "", "  ")
    return mcp.NewToolResultText(string(resultJSON)), nil
}
```

### internal/tools/tools.go

For each tool, generate Args and Result structs:

```go
package tools

type Tools struct{}

func New() *Tools { return &Tools{} }

type <ToolName>Args struct {
    Input string `json:"input"`
}

type <ToolName>Result struct {
    Success bool   `json:"success"`
    Data    string `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
}

func (t *Tools) <ToolName>(args <ToolName>Args) <ToolName>Result {
    // Implementation placeholder
    return <ToolName>Result{
        Success: true,
        Data:    "placeholder",
    }
}
```

### Makefile (Go)

```makefile
.PHONY: help build clean test deps install run-stdio run-http

BINARY_NAME=<binary_name>
BIN_DIR=bin

ifdef <UPPER_SERVER_NAME>_INSTALL_PATH
    INSTALL_PATH=$(<UPPER_SERVER_NAME>_INSTALL_PATH)
else
    INSTALL_PATH=$(shell go env GOPATH)/bin
endif

help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'

build: ## Build binary
	@mkdir -p $(BIN_DIR)
	@go build -o $(BIN_DIR)/$(BINARY_NAME) ./cmd/server

clean: ## Clean build artifacts
	@rm -rf $(BIN_DIR)

test: ## Run tests
	@go test -v ./...

deps: ## Download and tidy dependencies
	@go mod download
	@go mod tidy

install: build ## Install to GOPATH/bin or custom path
	@mkdir -p $(INSTALL_PATH)
	@cp $(BIN_DIR)/$(BINARY_NAME) $(INSTALL_PATH)/
	@chmod +x $(INSTALL_PATH)/$(BINARY_NAME)

run-stdio: build ## Run in stdio mode
	@$(BIN_DIR)/$(BINARY_NAME) -transport stdio

run-http: build ## Run in HTTP/SSE mode
	@$(BIN_DIR)/$(BINARY_NAME) -transport http -host localhost -port 8000

all: deps build test ## Full build pipeline
```

### CLAUDE.md

Generate a development guide covering:
- Project overview
- Prerequisites (Python 3.10+ or Go 1.24+)
- Setup commands
- Project structure
- How to add a new tool
- Testing
- MCP client configuration example (for Claude Desktop / Cursor)

### README.md

Generate with:
- Project description
- Installation
- Tools table (name, description, parameters)
- Usage / client configuration example
- Development commands

## Step 4 — Post-Generation

After writing all files:

1. If Go: run `go mod tidy` to resolve dependencies
2. List files created
3. Show how to test:
   - Python: `make install-dev && make test`
   - Go: `make deps && make test`
4. Show MCP client config snippet:
   ```json
   {
     "mcpServers": {
       "<server-name>": {
         "type": "stdio",
         "command": "<command>",
         "args": []
       }
     }
   }
   ```
