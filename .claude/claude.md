# OpenCode

AI-powered development tool.

## Tech Stack

- **Runtime**: Bun 1.3.5
- **Language**: TypeScript
- **Frontend**: SolidJS
- **Backend**: Hono
- **Build**: Turbo (monorepo)
- **Styling**: TailwindCSS

## Project Structure

This is a monorepo with the following packages:

- `packages/opencode` - Main application
- `packages/console/*` - Console packages
- `packages/sdk/js` - JavaScript SDK
- `packages/slack` - Slack integration

## Commands

```bash
# Development
bun run dev

# Type checking
bun run typecheck
```

## Code Style

- No semicolons
- Print width: 120 characters
- Use Prettier for formatting
