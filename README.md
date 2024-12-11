<!-- omit in toc -->
# MCP Go 🚀
[![Build](https://github.com/mark3labs/mcp-go/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/mark3labs/mcp-go/actions/workflows/ci.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/mark3labs/mcp-go?cache)](https://goreportcard.com/report/github.com/mark3labs/mcp-go)
[![GoDoc](https://pkg.go.dev/badge/github.com/mark3labs/mcp-go.svg)](https://pkg.go.dev/github.com/mark3labs/mcp-go)

<div align="center">

<strong>A Go implementation of the Model Context Protocol (MCP), enabling seamless integration between LLM applications and external data sources and tools.</strong>

</div>

```go
package main

import (
    "context"
    "fmt"

    "github.com/mark3labs/mcp-go/mcp"
    "github.com/mark3labs/mcp-go/server"
)

func main() {
    // Create MCP server
    s := server.NewMCPServer(
        "Demo 🚀",
        "1.0.0",
    )

    // Add tool
    tool := mcp.NewTool("hello_world",
        mcp.WithDescription("Say hello to someone"),
        mcp.WithString("name",
            mcp.Required(),
            mcp.Description("Name of the person to greet"),
        ),
    )

    // Add tool handler
    s.AddTool(tool, helloHandler)

    // Start the stdio server
    if err := server.ServeStdio(s); err != nil {
        fmt.Printf("Server error: %v\n", err)
    }
}

func helloHandler(ctx context.Context, arguments map[string]interface{}) (*mcp.CallToolResult, error) {
    name, ok := arguments["name"].(string)
    if !ok {
        return mcp.NewToolResultError("name must be a string"), nil
    }

    return mcp.NewToolResultText(fmt.Sprintf("Hello, %s!", name)), nil
}
```

That's it!

MCP Go handles all the complex protocol details and server management, so you can focus on building great tools. It aims to be high-level and easy to use.

### Key features:
* **Fast**: High-level interface means less code and faster development
* **Simple**: Build MCP servers with minimal boilerplate
* **Complete***: MCP Go aims to provide a full implementation of the core MCP specification

(\*emphasis on *aims*)

🚨 🚧 🏗️ *MCP Go is under active development, as is the MCP specification itself. Core features are working but some advanced capabilities are still in progress.* 


<!-- omit in toc -->
## Table of Contents

- [Installation](#installation)
- [Quickstart](#quickstart)
- [What is MCP?](#what-is-mcp)
- [Core Concepts](#core-concepts)
  - [Server](#server)
  - [Resources](#resources)
  - [Tools](#tools)
  - [Prompts](#prompts)
- [Examples](#examples)
- [Contributing](#contributing)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation-1)
  - [Testing](#testing)
  - [Opening a Pull Request](#opening-a-pull-request)

## Installation

```bash
go get github.com/mark3labs/mcp-go
```

## Quickstart

Let's create a simple MCP server that exposes a calculator tool and some data:

```go
package main

import (
    "context"
    "fmt"

    "github.com/mark3labs/mcp-go/mcp"
    "github.com/mark3labs/mcp-go/server"
)

func main() {
    // Create a new MCP server
    s := server.NewMCPServer(
        "Calculator Demo",
        "1.0.0",
        server.WithResourceCapabilities(true, true),
        server.WithLogging(),
    )

    // Add a calculator tool
    calculatorTool := mcp.NewTool("calculate",
        mcp.WithDescription("Perform basic arithmetic operations"),
        mcp.WithString("operation",
            mcp.Required(),
            mcp.Description("The operation to perform (add, subtract, multiply, divide)"),
            mcp.Enum("add", "subtract", "multiply", "divide"),
        ),
        mcp.WithNumber("x",
            mcp.Required(),
            mcp.Description("First number"),
        ),
        mcp.WithNumber("y",
            mcp.Required(),
            mcp.Description("Second number"),
        ),
    )

    // Add the calculator handler
    s.AddTool(calculatorTool, func(args map[string]interface{}) (*mcp.CallToolResult, error) {
        op := args["operation"].(string)
        x := args["x"].(float64)
        y := args["y"].(float64)

        var result float64
        switch op {
        case "add":
            result = x + y
        case "subtract":
            result = x - y
        case "multiply":
            result = x * y
        case "divide":
            if y == 0 {
                return mcp.NewToolResultError("Cannot divide by zero"), nil
            }
            result = x / y
        }

        return mcp.NewToolResultText(fmt.Sprintf("%.2f", result)), nil
    })

    // Start the server
    if err := server.ServeStdio(s); err != nil {
        fmt.Printf("Server error: %v\n", err)
    }
}
```
## What is MCP?

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io) lets you build servers that expose data and functionality to LLM applications in a secure, standardized way. Think of it like a web API, but specifically designed for LLM interactions. MCP servers can:

- Expose data through **Resources** (think of these sort of like GET endpoints; they are used to load information into the LLM's context)
- Provide functionality through **Tools** (sort of like POST endpoints; they are used to execute code or otherwise produce a side effect)
- Define interaction patterns through **Prompts** (reusable templates for LLM interactions)
- And more!


## Core Concepts


### Server

<details>
<summary>Show Server Examples</summary>

The server is your core interface to the MCP protocol. It handles connection management, protocol compliance, and message routing:

```go
// Create a basic server
s := server.NewMCPServer(
    "My Server",  // Server name
    "1.0.0",     // Version
)

// Start the server using stdio
if err := server.ServeStdio(s); err != nil {
    log.Fatalf("Server error: %v", err)
}
```

</details>

### Resources

<details>
<summary>Show Resource Examples</summary>

Resources are how you expose data to LLMs. They're similar to GET endpoints in a REST API - they provide data but shouldn't perform significant computation or have side effects. Some examples:

- File contents
- Database schemas
- API responses
- System information

Resources can be static:
```go
// Static text resource
s.AddResource("test://docs/readme", func() ([]interface{}, error) {
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://docs/readme",
                MIMEType: "text/markdown",
            },
            Text: "# Project Documentation\nThis is the main documentation...",
        },
    }, nil
})

// Binary/blob resource (e.g., image)
s.AddResource("test://images/logo", func() ([]interface{}, error) {
    imageData, err := os.ReadFile("logo.png")
    if err != nil {
        return nil, err
    }
    
    return []interface{}{
        mcp.BlobResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://images/logo",
                MIMEType: "image/png",
            },
            Blob: base64.StdEncoding.EncodeToString(imageData),
        },
    }, nil
})

// Dynamic resource that fetches data
s.AddResource("test://api/weather", func() ([]interface{}, error) {
    weather, err := fetchWeatherData()
    if err != nil {
        return nil, err
    }
    
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://api/weather",
                MIMEType: "application/json",
            },
            Text: weather,
        },
    }, nil
})

// Resource with multiple contents
s.AddResource("test://report", func() ([]interface{}, error) {
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://report/summary",
                MIMEType: "text/plain",
            },
            Text: "Monthly sales increased by 15%",
        },
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://report/details",
                MIMEType: "application/json",
            },
            Text: `{"sales": {"january": 100, "february": 115}}`,
        },
    }, nil
})

// Resource template for dynamic paths
s.AddResourceTemplate("test://users/{id}/profile", func() (mcp.ResourceTemplate, error) {
    return mcp.ResourceTemplate{
        Name:        "User Profile",
        Description: "Returns user profile information",
        MIMEType:    "application/json",
    }, nil
})

// Resource with metadata
s.AddResource("test://docs/api", func() ([]interface{}, error) {
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://docs/api",
                MIMEType: "text/markdown",
            },
            Text: "## API Documentation\n...",
            // Add annotations for the LLM
            Annotated: mcp.Annotated{
                Annotations: &struct {
                    Audience []mcp.Role  `json:"audience,omitempty"`
                    Priority float64     `json:"priority,omitempty"`
                }{
                    Audience: []mcp.Role{mcp.RoleAssistant},
                    Priority: 0.8,
                },
            },
        },
    }, nil
})

// Database-backed resource
s.AddResource("test://db/products", func() ([]interface{}, error) {
    products, err := db.QueryProducts()
    if err != nil {
        return nil, err
    }
    
    productsJSON, err := json.Marshal(products)
    if err != nil {
        return nil, err
    }
    
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI:      "test://db/products",
                MIMEType: "application/json",
            },
            Text: string(productsJSON),
        },
    }, nil
})
```

</details>

Resources can be:
- Static or dynamic
- Text or binary  
- Single or multi-part
- Template-based for dynamic paths
- Annotated with metadata
- Backed by various data sources

The URI scheme (e.g., `test://`) is arbitrary - you can use any scheme that makes sense for your application. The handler function is called whenever the resource is requested, allowing for dynamic content generation.

### Tools

<details>
<summary>Show Tool Examples</summary>

Tools let LLMs take actions through your server. Unlike resources, tools are expected to perform computation and have side effects. They're similar to POST endpoints in a REST API.

Simple calculation example:
```go
calculatorTool := mcp.NewTool("calculate",
    mcp.WithDescription("Perform basic arithmetic calculations"),
    mcp.WithString("operation",
        mcp.Required(),
        mcp.Description("The arithmetic operation to perform"),
        mcp.Enum("add", "subtract", "multiply", "divide"),
    ),
    mcp.WithNumber("x",
        mcp.Required(),
        mcp.Description("First number"),
    ),
    mcp.WithNumber("y",
        mcp.Required(),
        mcp.Description("Second number"),
    ),
)

s.AddTool(calculatorTool, func(args map[string]interface{}) (*mcp.CallToolResult, error) {
    op := args["operation"].(string)
    x := args["x"].(float64)
    y := args["y"].(float64)

    var result float64
    switch op {
    case "add":
        result = x + y
    case "subtract":
        result = x - y
    case "multiply":
        result = x * y
    case "divide":
        if y == 0 {
            return mcp.NewToolResultError("Division by zero is not allowed"), nil
        }
        result = x / y
    }
    
    return mcp.FormatNumberResult(result), nil
})
```

HTTP request example:
```go
httpTool := mcp.NewTool("http_request",
    mcp.WithDescription("Make HTTP requests to external APIs"),
    mcp.WithString("method",
        mcp.Required(),
        mcp.Description("HTTP method to use"),
        mcp.Enum("GET", "POST", "PUT", "DELETE"),
    ),
    mcp.WithString("url",
        mcp.Required(),
        mcp.Description("URL to send the request to"),
        mcp.Pattern("^https?://.*"),
    ),
    mcp.WithString("body",
        mcp.Description("Request body (for POST/PUT)"),
    ),
)

s.AddTool(httpTool, func(args map[string]interface{}) (*mcp.CallToolResult, error) {
    method := args["method"].(string)
    url := args["url"].(string)
    body := ""
    if b, ok := args["body"].(string); ok {
        body = b
    }

    // Create and send request
    var req *http.Request
    var err error
    if body != "" {
        req, err = http.NewRequest(method, url, strings.NewReader(body))
    } else {
        req, err = http.NewRequest(method, url, nil)
    }
    if err != nil {
        return mcp.NewToolResultError(fmt.Sprintf("Failed to create request: %v", err)), nil
    }

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return mcp.NewToolResultError(fmt.Sprintf("Request failed: %v", err)), nil
    }
    defer resp.Body.Close()

    // Return response
    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return mcp.NewToolResultError(fmt.Sprintf("Failed to read response: %v", err)), nil
    }

    return mcp.NewToolResultText(fmt.Sprintf("Status: %d\nBody: %s", resp.StatusCode, string(respBody))), nil
})
```

Tools can be used for any kind of computation or side effect:
- Database queries
- File operations  
- External API calls
- Calculations
- System operations

Each tool should:
- Have a clear description
- Validate inputs
- Handle errors gracefully 
- Return structured responses
- Use appropriate result types

</details>

### Prompts

<details>
<summary>Show Prompt Examples</summary>

Prompts are reusable templates that help LLMs interact with your server effectively. They're like "best practices" encoded into your server. Here are some examples:

```go
// Simple greeting prompt
s.AddPrompt(mcp.NewPrompt("greeting",
    mcp.WithPromptDescription("A friendly greeting prompt"),
    mcp.WithArgument("name",
        mcp.ArgumentDescription("Name of the person to greet"),
    ),
), func(args map[string]string) (*mcp.GetPromptResult, error) {
    name := args["name"]
    if name == "" {
        name = "friend"
    }
    
    return mcp.NewGetPromptResult(
        "A friendly greeting",
        []mcp.PromptMessage{
            mcp.NewPromptMessage(
                mcp.RoleAssistant,
                mcp.NewTextContent(fmt.Sprintf("Hello, %s! How can I help you today?", name)),
            ),
        },
    ), nil
})

// Code review prompt with embedded resource
s.AddPrompt(mcp.NewPrompt("code_review",
    mcp.WithPromptDescription("Code review assistance"),
    mcp.WithArgument("pr_number",
        mcp.ArgumentDescription("Pull request number to review"),
        mcp.RequiredArgument(),
    ),
), func(args map[string]string) (*mcp.GetPromptResult, error) {
    prNumber := args["pr_number"]
    if prNumber == "" {
        return nil, fmt.Errorf("pr_number is required")
    }
    
    return mcp.NewGetPromptResult(
        "Code review assistance",
        []mcp.PromptMessage{
            mcp.NewPromptMessage(
                mcp.RoleSystem,
                mcp.NewTextContent("You are a helpful code reviewer. Review the changes and provide constructive feedback."),
            ),
            mcp.NewPromptMessage(
                mcp.RoleAssistant,
                mcp.NewEmbeddedResource(mcp.ResourceContents{
                    URI: fmt.Sprintf("git://pulls/%s/diff", prNumber),
                    MIMEType: "text/x-diff",
                }),
            ),
        },
    ), nil
})

// Database query builder prompt
s.AddPrompt(mcp.NewPrompt("query_builder",
    mcp.WithPromptDescription("SQL query builder assistance"),
    mcp.WithArgument("table",
        mcp.ArgumentDescription("Name of the table to query"),
        mcp.RequiredArgument(),
    ),
), func(args map[string]string) (*mcp.GetPromptResult, error) {
    tableName := args["table"]
    if tableName == "" {
        return nil, fmt.Errorf("table name is required")
    }
    
    return mcp.NewGetPromptResult(
        "SQL query builder assistance",
        []mcp.PromptMessage{
            mcp.NewPromptMessage(
                mcp.RoleSystem,
                mcp.NewTextContent("You are a SQL expert. Help construct efficient and safe queries."),
            ),
            mcp.NewPromptMessage(
                mcp.RoleAssistant,
                mcp.NewEmbeddedResource(mcp.ResourceContents{
                    URI: fmt.Sprintf("db://schema/%s", tableName),
                    MIMEType: "application/json",
                }),
            ),
        },
    ), nil
})
```

Prompts can include:
- System instructions
- Required arguments
- Embedded resources
- Multiple messages
- Different content types (text, images, etc.)
- Custom URI schemes

</details>

## Examples

For examples, see the `examples/` directory.


## Contributing

<details>

<summary><h3>Open Developer Guide</h3></summary>

### Prerequisites

Go version >= 1.23

### Installation

Create a fork of this repository, then clone it:

```bash
git clone https://github.com/mark3labs/mcp-go.git
cd mcp-go
```

### Testing

Please make sure to test any new functionality. Your tests should be simple and atomic and anticipate change rather than cement complex patterns.

Run tests from the root directory:

```bash
go test -v './...'
```

### Opening a Pull Request

Fork the repository and create a new branch:

```bash
git checkout -b my-branch
```

Make your changes and commit them:


```bash
git add . && git commit -m "My changes"
```

Push your changes to your fork:


```bash
git push origin my-branch
```

Feel free to reach out in a GitHub issue or discussion if you have any questions!

</details>
