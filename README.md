# chipagents-plugin-alphadesign

AlphaDesign internal chipagents plugin. Bundles the release workflow skill, Linear MCP server, team coding conventions, and a PR review command.

## What's included

| Component | Description |
|-----------|-------------|
| `/release` skill | Cut patch and minor releases with cherry-picks, changelog updates, tagging, backmerge, and AE notification |
| `/pr-review` command | Review any PR by number, or auto-detect the current branch's open PR |
| Linear MCP | Full Linear issue tracker access via `@linear/mcp-server` |
| `alphadesign-conventions` agent | Team coding conventions (bun, Biome, TypeScript patterns, commit rules) |

## Prerequisites

- [chipagents CLI](https://github.com/chipagents/chipagents-cli) installed
- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- Node.js / `npx` available (for Linear MCP server)
- A [Linear API key](https://linear.app/settings/api)

## Install

```bash
chipagents plugin marketplace add github:alphadesign/chipagents-plugin-alphadesign
chipagents plugin install alphadesign@alphadesign/chipagents-plugin-alphadesign
```

## Configuration

### Linear API key

The Linear MCP server reads `LINEAR_API_KEY` from the environment. Add it to your shell profile or project `.env`:

```bash
export LINEAR_API_KEY=lin_api_xxxxxxxxxxxx
```

## Usage

### Release skill

```
/release v0.14.1
/release v0.15.0
/release v0.14.1 1234 1235 1236
```

Pass a version and optionally explicit PR numbers. If no PRs are given, the skill discovers commits since the last tag and asks which to include.

### PR review command

```
/pr-review          # reviews the current branch's open PR
/pr-review 42       # reviews PR #42
```

### Conventions agent profile

The `alphadesign-conventions` agent profile is available from the `/agents` menu. Activating it loads team coding conventions into context for the current session.

To apply conventions automatically as an ambient rule instead, copy the file to your project:

```bash
cp ~/.chipagents/plugins/alphadesign/agents/alphadesign-conventions.md .chipagents/alphadesign-conventions.md
```

Then toggle it on with `/rules`.
