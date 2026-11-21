# Repository Guidelines

## Project Structure & Module Organization

This repository contains `DESIGN.md`, the original concept, and `PLAN.md`, the agreed implementation plan. Source code, build configuration, and tests have not yet been added. Consult the plan before implementing features.

The implementation direction is Go with Pion WebRTC, supporting Linux and macOS. Use `cmd/tentaicle/` for the CLI and agent, `cmd/tentaicle-server/` for the coordinator, and `internal/` for authentication, signaling, transport, and terminal packages. Keep tests alongside their packages. These directories are planned, not existing.

## Build, Test, and Development Commands

No build or development commands are configured yet. After initializing the Go module, use:

- `go build ./...` to compile all packages.
- `go test ./...` to run tests.
- `go test -race ./...` to detect concurrency issues.
- `go vet ./...` for static checks.
- `gofmt -w <files>` to format changed Go files.

Document service configuration and local startup commands when their implementations exist.

## Coding Style & Naming Conventions

Follow standard Go formatting and idioms. Use `gofmt`, short lowercase package names, exported `PascalCase` identifiers, and unexported `camelCase` identifiers. Keep terminal management separate from WebRTC transport and coordinator authorization.

Return contextual errors and propagate cancellation through `context.Context`. Bound network queues and terminal-output buffers.

## Testing Guidelines

Use Go’s standard `testing` package, `*_test.go` files, and `TestXxx` functions. No coverage threshold is established.

Prioritize authorization failures, credential revocation, reconnect behavior, output replay, PTY cleanup, and direct versus TURN-relayed connections. Keep external-service integration tests explicitly configured; ordinary tests must not require production credentials.

## Commit & Pull Request Guidelines

Git history is unavailable, so no existing commit convention is established. Use concise, imperative subjects such as “Add agent enrollment validation.”

PRs should explain the problem, resulting behavior, validation performed, and configuration changes. Link relevant issues when available. Include terminal transcripts for CLI behavior changes when useful.

## Security & Configuration

Never commit OAuth credentials, TURN secrets, device tokens, or terminal recordings. Avoid logging terminal contents or authentication material. Run agents under the installing user’s account and verify authorization before exposing sessions.
