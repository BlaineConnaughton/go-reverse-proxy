# AI Agent Instructions for go-reverse-proxy

This is a configuration-driven reverse proxy written in Go with zero dependencies outside the Go standard library. Here's what you need to know to work effectively with this codebase:

## Architecture Overview

The codebase is organized into these key components:

- `main.go`: Entry point that sets up the HTTP server and routing handlers
- `routing/`: Contains route configuration and request handling logic
- `proxy/`: Implements the core reverse proxy functionality
- `upstreams/`: Defines backend services to proxy requests to

### Key Data Flows

1. HTTP requests hit the server on port 9001
2. The router matches requests against configured patterns in `routing/configuration.go`
3. Matched requests are forwarded to appropriate upstreams with optional modifications

## Core Concepts

### Route Configuration

Routes are defined in `routing/configuration.go` as a slice of `Config` structs with these key fields:
- `Path`: URL pattern to match (supports regex with named capture groups)
- `Upstream`: Target backend service
- `ModifyPath`: Optional path transformation using capture groups
- `Override`: Optional conditional routing logic based on headers/query params

Example configuration:
```go
Config{
    Path:       `/(?P<cap>foo\w{3})`,
    Upstream:   upstreams.HTTPBin,
    ModifyPath: "/anything/${cap}",
}
```

### Upstreams

Backend services are defined in `upstreams/upstreams.go` as `Upstream` structs with:
- `Name`: Identifier for the service
- `Host`: Hostname of the backend

## Development Workflow

### Building and Running
```
make build   # Build the binary
make run     # Run the server on port 9001
make test    # Run test suite
```

### Testing Patterns
- Integration tests are in `proxy/proxy_test.go`
- Manual testing can be done using the load test targets in `load-test/targets.txt`

## Common Tasks

1. Adding a new route:
   - Add configuration to `routing/configuration.go`
   - Pattern: Define path regex, upstream, and any path modifications
   - Test with requests to `:9001/<your-path>`

2. Adding a new upstream:
   - Define in `upstreams/upstreams.go`
   - Follow existing pattern of `Name` and `Host` fields

## Project Conventions

1. Regular Expressions:
   - Use named capture groups with `(?P<name>pattern)` syntax
   - Reference captures in path modifications with `${name}`

2. Route Organization:
   - Routes are evaluated in order of definition
   - More specific routes should come before general ones