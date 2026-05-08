---
name: alphadesign-conventions
description: AlphaDesign coding conventions and project guidelines for chipagents-cli
---

# AGENTS.md

Guidelines for AI coding agents working on this codebase.

## Package Manager

Use **bun** exclusively. Do not use npm, yarn, or pnpm.

- `bun install` for dependencies
- `bun run <script>` for scripts
- `bun test` for tests
- `bun run typecheck` for type checking
- `bun run lint` for linting

## TypeScript Conventions

- Use `type` instead of `interface` for all type definitions.
- Use inline `import type` syntax: `import { type Foo } from "./bar"`.
- Use Zod for runtime validation of external inputs.

## Formatting & Linting

This project uses **Biome** (not ESLint or Prettier).

- Indent: 2 spaces
- Line width: 80
- Quote style: double quotes
- Fix lint issues with: `bun run lint`

## Git & Commits

- A pre-commit hook runs `bun run lint` on staged files.
- Never credit AI tools in commit messages.
- For multi-line commit messages, use `git commit -F - <<'EOF'` instead of command substitution (`$(cat <<'EOF' ... EOF)`).
- **Never push directly to `main`.** Always create a feature branch and open a PR.
- When opening a PR, always use the PR template with these four sections: **Motivation**, **Details**, **Proof of Function**, and **Additional Details**.

## Local Development

Starting the backend server:

- `bun run backend:chipagents:fast` — fast start (skips vite build)
- `bun run backend:chipagents` — full start (includes vite/dashboard build)

Port is set via `PORT=` in `backend/.env.chipagents.local` (defaults to 3000). Provider config is `LLM_PROVIDER` and `STRONG_MODEL_ID` in the same file.

Starting the CLI:

- `bun run cli` — connects to default port 3000
- `PORT=3001 bun run cli` — if backend uses a non-default port

PORT defaults to 3000; set it to match the backend if using a different port to avoid conflicts. First run requires `/login` (OAuth browser flow).

## Testing

Run tests with `bun test`. Run type checking with `bun run typecheck`. Run tests before pushing to remote.

## Additional Context

See `README.md` for helpful project information including architecture overview and setup instructions.
