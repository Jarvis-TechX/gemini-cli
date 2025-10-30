# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Essential Commands

### Development Workflow

```bash
# Install dependencies (use Node.js ~20.19.0 for development)
npm install

# Build the entire project
npm run build

# Build everything including sandbox container
npm run build:all

# Start Gemini CLI from source
npm start

# Run CLI outside repo using alias or link
alias gemini="node path/to/gemini-cli/packages/cli"
```

### Testing

```bash
# Run unit tests for all packages
npm run test

# Run integration tests (end-to-end)
npm run test:e2e

# Run integration tests with specific sandbox environments
npm run test:integration:sandbox:none
npm run test:integration:sandbox:docker

# Run single test file with Vitest
npx vitest run path/to/test.test.ts
```

### Quality Checks

```bash
# Run complete preflight check (REQUIRED before submitting PR)
npm run preflight

# Individual checks
npm run lint          # ESLint
npm run lint:fix      # Auto-fix linting issues
npm run format        # Prettier formatting
npm run typecheck     # TypeScript type checking
```

### Debugging

```bash
# Debug mode (pauses for debugger attachment)
npm run debug

# Enable dev tracing for debugging agent behavior
GEMINI_DEV_TRACING=true gemini

# Debug sandbox container
DEBUG=1 gemini

# React DevTools for UI debugging
DEV=true npm start
# Then: npx react-devtools@4.28.5
```

## Architecture Overview

### Monorepo Structure

This is a **TypeScript monorepo** with npm workspaces:

- **`packages/cli/`** - User-facing CLI: input processing, display rendering,
  history management, themes, UI customization. Built with **Ink** (React for
  CLIs).
- **`packages/core/`** - Backend engine: API client for Google Gemini API,
  prompt construction, tool registration/execution, state management,
  conversation orchestration.
- **`packages/a2a-server/`** - Agent-to-Agent server implementation
  (experimental).
- **`packages/test-utils/`** - Shared utilities for testing (temporary file
  system creation/cleanup).
- **`packages/vscode-ide-companion/`** - VS Code extension that pairs with
  Gemini CLI.
- **`integration-tests/`** - End-to-end integration tests.
- **`scripts/`** - Build, testing, and development automation scripts.
- **`docs/`** - All project documentation.

### Request Flow

1. User types command → **CLI package** receives input
2. CLI sends request → **Core package**
3. Core constructs prompt with conversation history and available tool
   definitions → **Gemini API**
4. Gemini API responds (either answer or tool request) → **Core package**
5. If tool requested:
   - Core prepares execution
   - User approval required for write operations (reads may auto-approve)
   - Core executes tool and sends result back to Gemini API
   - Gemini processes result and generates final response
6. Core sends response → **CLI package**
7. CLI formats and displays → **User**

### Key Architectural Principles

- **Modularity**: CLI (frontend) and Core (backend) are decoupled for
  independent development and future extensibility.
- **Extensibility**: Tool system designed for easy addition of new capabilities.
  MCP (Model Context Protocol) support for custom integrations.
- **User Experience**: CLI provides rich, interactive terminal experience with
  React-based UI using Ink.

### Tool System

Tools are individual modules in `packages/core/src/core/` that extend Gemini
model capabilities:

- **File System Operations** - Read, write, edit files
- **Shell Commands** - Execute bash commands
- **Web Fetch & Search** - Google Search grounding, web content fetching
- **MCP Servers** - Custom tool integrations via Model Context Protocol

Tools are registered in Core and invoked based on Gemini API requests. Read-only
tools may execute without user confirmation; write operations require approval.

## Development Guidelines

### TypeScript & JavaScript

**Prefer Plain Objects over Classes:**

- Use plain JavaScript objects with TypeScript `interface` or `type`
  declarations instead of class syntax
- Classes complicate React integration due to internal state encapsulation
- Plain objects are immutable-friendly, easier to serialize, and align with
  functional programming
- Use ES module syntax (`import`/`export`) for encapsulation instead of
  `private`/`public` class members

**Type Safety:**

- **Never use `any`** - it disables type checking and masks underlying issues
- **Prefer `unknown` over `any`** - requires explicit type narrowing before use
- **Minimize type assertions (`as Type`)** - they bypass type safety checks
- If you need type assertions in tests to access "private" internals, that's a
  code smell - refactor into a separate module with a public API

**Type Narrowing:**

- Use `checkExhaustive` helper (in `packages/cli/src/utils/checks.ts`) in
  `switch` statement default clauses to ensure all enum/union cases are handled

**Array Operations:**

- Leverage `.map()`, `.filter()`, `.reduce()`, `.slice()`, `.sort()` for
  immutable, functional transformations
- Promotes immutability and readability

### React Guidelines (Ink UI)

The CLI uses **Ink** (React for terminal interfaces). Follow these patterns:

**Component Structure:**

- Use **functional components with Hooks** only (no class components)
- Keep components **pure and side-effect-free** during rendering
- Break UI into small, reusable components

**State Management:**

- **Never mutate state directly** - use spread syntax for immutable updates
- Use `useState` or `useReducer` for state
- Use functional state updates for concurrent safety: `setCount(c => c + 1)`
- Pass data down through props (one-way data flow)

**Effects & Side Effects:**

- **Think hard before using `useEffect`** - primarily for synchronization with
  external state
- **NEVER `setState` inside `useEffect`** - degrades performance
- Put user-action logic (button clicks, form submissions) in event handlers, not
  effects
- Include all dependencies in effect dependency arrays
- Effects should return cleanup functions when subscribing to resources
- Never call Hooks conditionally or inside loops

**Refs:**

- Only use `useRef` when necessary (DOM focus, animations, non-React library
  integration)
- Never read/write `ref.current` during rendering (except lazy initialization)
- Don't use refs for reactive application state

**Optimization:**

- **React Compiler is enabled** - omit manual `useMemo`, `useCallback`,
  `React.memo`
- Focus on clear, simple components with direct data flow
- Avoid premature optimization

**Testing React Components:**

- Use `render()` from `ink-testing-library`
- Assert output with `lastFrame()`
- Wrap components in necessary Context Providers
- Mock custom hooks and complex child components

### Testing Conventions

**Framework:** Vitest (`describe`, `it`, `expect`, `vi`)

**File Organization:**

- Test files (`*.test.ts`, `*.test.tsx`) co-located with source files
- Configuration in `vitest.config.ts` files

**Setup/Teardown:**

- Use `beforeEach` and `afterEach`
- Call `vi.resetAllMocks()` in `beforeEach`
- Call `vi.restoreAllMocks()` in `afterEach`

**Mocking:**

- **ES Modules:** `vi.mock('module-name', async (importOriginal) => { ... })`
- Use `importOriginal` for selective mocking
- Place `vi.mock` at the **very top** of test files for critical dependencies
  (`os`, `fs`)
- **Hoisting:** `const myMock = vi.hoisted(() => vi.fn());` for mocks defined
  before use in factories
- **Mock Functions:** `vi.fn()` with `.mockImplementation()`,
  `.mockResolvedValue()`, `.mockRejectedValue()`
- **Spying:** `vi.spyOn(object, 'methodName')`, restore with `mockRestore()` in
  `afterEach`

**Commonly Mocked Modules:**

- Node.js: `fs`, `fs/promises`, `os`, `path`, `child_process`
- External SDKs: `@google/genai`, `@modelcontextprotocol/sdk`
- Internal: Cross-package dependencies

**Async Testing:**

- Use `async/await`
- For timers: `vi.useFakeTimers()`, `vi.advanceTimersByTimeAsync()`,
  `vi.runAllTimersAsync()`
- Test rejections: `await expect(promise).rejects.toThrow(...)`

### Code Style

**Comments:**

- Only write high-value comments when necessary
- Avoid explaining obvious code
- Never communicate with users through comments

**Naming:**

- Use **hyphens** in flag names: `--my-flag` (not `--my_flag`)

**Git:**

- Main branch: `main`
- Follow Conventional Commits standard for PR titles:
  `feat(cli): Add --json flag`
- Link all PRs to existing issues
- Keep PRs small and focused (one issue per PR)
- Use draft PRs for early feedback

## Authentication & Environment

Gemini CLI supports three authentication methods:

1. **OAuth (Login with Google)** - Best for individual developers
   - Free tier: 60 requests/min, 1,000 requests/day
   - Set `GOOGLE_CLOUD_PROJECT` for paid Code Assist licenses

2. **Gemini API Key**
   - Set `GEMINI_API_KEY` environment variable
   - Get key from https://aistudio.google.com/apikey

3. **Vertex AI**
   - Set `GOOGLE_API_KEY` and `GOOGLE_GENAI_USE_VERTEXAI=true`

## Sandboxing

**macOS Seatbelt:**

- Uses `sandbox-exec` with `permissive-open` profile by default (restricts
  writes to project folder)
- Switch profiles: `SEATBELT_PROFILE=restrictive-closed`
- Custom profiles: Create `.gemini/sandbox-macos-<profile>.sb`

**Container-based (Docker/Podman):**

- Set `GEMINI_SANDBOX=true|docker|podman`
- Run `npm run build:all` to build sandbox container
- Mounts project directory with read-write access
- Customize: Create `.gemini/sandbox.Dockerfile` and `.gemini/sandbox.bashrc`

## MCP Server Integration

Configure MCP servers in `~/.gemini/settings.json` to extend with custom tools:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

Usage: `@github List my open pull requests`

## Context Files

**GEMINI.md** in project root provides persistent context to Gemini CLI (similar
to this CLAUDE.md file but for Gemini CLI).

## Important Notes

- **Node.js version:** Development requires ~20.19.0 (use nvm); production >=20
- **Always run `npm run preflight`** before submitting PRs
- Integration tests require `GEMINI_API_KEY` secret in forked repos
- Examine existing tests before writing new ones to understand patterns
- Import restrictions enforced by ESLint between packages
