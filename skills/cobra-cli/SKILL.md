---
name: cobra-cli
description: Scaffold a Go CLI application or subcommand using Cobra with table-driven tests, ldflags versioning, Makefile, and CLAUDE.md. Use when creating a new Go CLI tool or adding a subcommand.
argument-hint: "<project-name> [--commands cmd1,cmd2] [--add-command cmd-name]"
allowed-tools: Write, Bash, Read, Glob
---

# Go CLI Scaffold (Cobra)

Generate a Go CLI project or add subcommands, following conventions from modgit, mcpcli, and copypartygo.

## Step 1 — Parse Arguments

| Parameter | Default | Description |
|-----------|---------|-------------|
| `project-name` | (required unless `--add-command`) | Project name in kebab-case |
| `--commands` | (none) | Comma-separated subcommands to scaffold |
| `--add-command` | (none) | Add a single subcommand to an existing project |

**Mode detection:**
- If `--add-command` is provided: add a subcommand to the current project (Step 4)
- Otherwise: scaffold a full new project (Steps 2-3)

## Step 2 — Generate Full Project

Create:

```
<project-name>/
├── cmd/
│   └── <project-name>/
│       └── main.go          # Entry point
├── cmd/                     # (Alternative: commands in cmd/ for delegated pattern)
│   ├── root.go              # Root command
│   └── <subcommand>.go      # One file per subcommand
├── internal/                # Private packages
│   └── (empty, ready for domain logic)
├── testdata/                # Test fixtures directory
├── go.mod
├── go.sum
├── VERSION                  # Semantic version file
├── Makefile
├── .gitignore
├── CLAUDE.md
└── README.md
```

Use the **inline pattern** (modgit-style) when subcommands are few (0-2):
- Root command and subcommands in `cmd/<project-name>/main.go`

Use the **delegated pattern** (mcpcli-style) when subcommands are many (3+):
- `main.go` at root delegates to `cmd.Execute()`
- Each subcommand in its own file under `cmd/`

## Step 3 — File Contents

### main.go (inline pattern)

```go
package main

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var version = "dev"

var rootCmd = &cobra.Command{
    Use:     "<project-name>",
    Short:   "Short description",
    Long:    `Longer description of the CLI tool.`,
    Version: version,
    RunE:    runRoot,
}

func runRoot(cmd *cobra.Command, args []string) error {
    // Default action or show help
    return cmd.Help()
}

func init() {
    // Persistent flags (inherited by subcommands)
    rootCmd.PersistentFlags().StringP("config", "c", "", "Config file path")

    // Register subcommands
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### main.go + cmd/root.go (delegated pattern)

**main.go:**
```go
package main

import "github.com/adeshmukh/<project-name>/cmd"

func main() {
    cmd.Execute()
}
```

**cmd/root.go:**
```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var Version = "dev"

var rootCmd = &cobra.Command{
    Use:   "<project-name>",
    Short: "Short description",
    Long:  `Longer description.`,
    Run: func(cmd *cobra.Command, args []string) {
        if v, _ := cmd.Flags().GetBool("version"); v {
            fmt.Printf("<project-name> version %s\n", Version)
            return
        }
        cmd.Help()
    },
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    rootCmd.PersistentFlags().StringP("config", "c", "", "Config file path")
    rootCmd.Flags().BoolP("version", "v", false, "Show version")
}
```

### Subcommand file (cmd/<name>.go)

```go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
)

var <name>Cmd = &cobra.Command{
    Use:   "<name> [args]",
    Short: "Short description of <name>",
    Args:  cobra.ExactArgs(1),  // or MinimumNArgs, MaximumNArgs, NoArgs
    RunE: func(cmd *cobra.Command, args []string) error {
        // Implementation
        fmt.Printf("Running <name> with: %s\n", args[0])
        return nil
    },
}

func init() {
    rootCmd.AddCommand(<name>Cmd)

    // Local flags
    <name>Cmd.Flags().StringP("output", "o", "", "Output file path")
}
```

**Cobra patterns to follow:**
- Use `RunE` (not `Run`) for commands that can fail
- Use `cobra.ExactArgs(N)`, `cobra.MinimumNArgs(N)`, etc. for argument validation
- Use `StringVarP` for flags that bind to package-level vars, `StringP` for inline access
- Persistent flags on root, local flags on subcommands
- No Viper — keep config simple

### go.mod

```go
module github.com/adeshmukh/<project-name>

go 1.24

require github.com/spf13/cobra v1.10.1
```

Run `go mod tidy` after generation.

### VERSION

```
0.1.0
```

### Makefile

```makefile
.PHONY: help build test clean install fmt vet check coverage

VERSION_FILE := VERSION
VERSION_BASE := $(shell cat $(VERSION_FILE) 2>/dev/null || echo "0.1.0")
GIT_SHA := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
GIT_STATUS := $(shell git status --porcelain 2>/dev/null)
VERSION_SUFFIX := -$(GIT_SHA)
ifneq ($(strip $(GIT_STATUS)),)
    VERSION_SUFFIX := $(VERSION_SUFFIX)-uncommitted
endif
FULL_VERSION := $(VERSION_BASE)$(VERSION_SUFFIX)
LDFLAGS := -X main.version=$(FULL_VERSION)

help: ## Show targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'

build: ## Build binary
	go build -ldflags "$(LDFLAGS)" -o <project-name> ./cmd/<project-name>

install: ## Install (honors <UPPER_NAME>_INSTALL_PATH)
ifdef <UPPER_NAME>_INSTALL_PATH
	@mkdir -p $(<UPPER_NAME>_INSTALL_PATH)
	go build -ldflags "$(LDFLAGS)" -o $(<UPPER_NAME>_INSTALL_PATH)/<project-name> ./cmd/<project-name>
else
	go install -ldflags "$(LDFLAGS)" ./cmd/<project-name>
endif

test: ## Run tests
	go test -v ./...

coverage: ## Run tests with coverage
	go test -v -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

fmt: ## Format code
	gofmt -s -w .

vet: ## Run go vet
	go vet ./...

check: fmt vet test ## All checks

clean: ## Remove build artifacts
	rm -f <project-name>
	rm -f coverage.out
```

For the **delegated pattern**, adjust the build path to `./` or `./main.go`.

### Test file (table-driven)

For each subcommand or key function, generate a test file:

```go
package main_test  // or package cmd_test

import (
    "testing"
)

func TestSomething(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    string
        wantErr bool
    }{
        {
            name:  "valid input",
            input: "hello",
            want:  "expected",
        },
        {
            name:    "empty input",
            input:   "",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := SomeFunction(tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Errorf("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Errorf("unexpected error: %v", err)
                return
            }

            if got != tt.want {
                t.Errorf("got %q, want %q", got, tt.want)
            }
        })
    }
}
```

### .gitignore

```
# Binaries
<project-name>
*.exe
*.dll
*.so
*.dylib

# Test
*.test
coverage.out

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
```

### CLAUDE.md

Generate with:
- Project overview
- Prerequisites (`go 1.24+`)
- Build commands (`make build`, `make install`)
- Project structure tree
- Code standards: gofmt, go vet, error wrapping with `%w`, table-driven tests
- Testing: `go test -v ./...`, `t.Run()`, `t.TempDir()`
- CLI conventions: exit codes (0=success, 1=error, 2=invalid args), `RunE` over `Run`
- Dependencies: minimize, prefer stdlib, Cobra for CLI

### README.md

Generate with:
- Description
- Installation (`go install` or `make install`)
- Usage examples
- Commands reference (table)
- Development commands

## Step 4 — Add Subcommand to Existing Project

When `--add-command <name>` is provided:

1. Read the existing project structure to determine inline vs delegated pattern
2. If delegated: create `cmd/<name>.go` with the subcommand template
3. If inline: add the subcommand to the existing main.go
4. Generate a test file for the new command
5. Register the command with `rootCmd.AddCommand(<name>Cmd)` in the appropriate `init()` function

## Step 5 — Post-Generation

After writing all files:

1. Run `go mod tidy`
2. Run `gofmt -s -w .`
3. List files created
4. Show quick start:
   ```
   cd <project-name>
   make build
   ./<project-name> --version
   ./<project-name> --help
   ```
