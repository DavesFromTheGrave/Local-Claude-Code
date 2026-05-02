# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

**Everything Claude Code (ECC)** (`ecc-universal` on npm) is a production-ready AI coding plugin that ships agents, skills, hooks, commands, and rules for use across Claude Code, Cursor, Codex, OpenCode, and Gemini harnesses. It is not an application — it is configuration and behavior that gets installed into AI coding tools.

## Commands

```bash
# Install dependencies
npm install

# Run all tests
node tests/run-all.js

# Run an individual test file
node tests/lib/utils.test.js

# Lint all Markdown files
npx markdownlint-cli '**/*.md' --ignore node_modules

# Run ESLint
npx eslint .

# Doctor / health check
node scripts/doctor.js

# List installed components
node scripts/list-installed.js

# Validate curated skills
node scripts/ci/validate-skills.js
```

## Architecture

### Component Types and Formats

| Directory | Format | Purpose |
|-----------|--------|---------|
| `agents/*.md` | YAML frontmatter (`name`, `description`, `tools`, `model`) + body | Specialized subagents for task delegation |
| `skills/<name>/SKILL.md` | YAML frontmatter (`name`, `description`, `origin`) + body | Workflow/domain knowledge modules |
| `commands/*.md` | YAML frontmatter with `description:` + body | Slash commands (e.g. `/tdd`, `/plan`, `/e2e`) |
| `hooks/hooks.json` | JSON matcher array | Trigger-based automations |
| `rules/<lang>/` | Markdown per concern | Always-applied coding guidelines |
| `mcp-configs/` | JSON server configs | MCP server integrations |
| `scripts/` | CommonJS Node.js utilities | Hook execution, install, session management |
| `tests/` | `*.test.js` mirroring `scripts/` | Tests for scripts and utilities |

### How Components Interact

- **Hooks** fire on Claude Code lifecycle events (PreToolUse, PostToolUse, etc.) and invoke scripts in `scripts/hooks/`. All hook scripts must route through `scripts/hooks/run-with-flags.js` so `ECC_HOOK_PROFILE` and `ECC_DISABLED_HOOKS` gating works.
- **Agents** are invoked by Claude (or other agents) via subagent delegation. Proactive invocation rules are in `AGENTS.md`.
- **Skills** are loaded by Claude Code based on context. Curated skills live in `skills/` and are referenced by `manifests/install-modules.json`; generated/imported skills live under `~/.claude/skills/` and are never committed.
- **Rules** in `rules/common/` apply to all languages; language-specific rules live in `rules/<lang>/`.
- **Commands** are slash commands available in chat (e.g. `/tdd`, `/learn`, `/skill-create`).

### Multi-Harness Config Directories

The repo ships configuration for multiple harnesses in parallel:
- `.claude/` — Claude Code
- `.cursor/` — Cursor IDE (rules under `.cursor/rules/`, hooks under `.cursor/hooks/`)
- `.codex/` / `.codex-plugin/` — OpenAI Codex
- `.gemini/` — Google Gemini
- `.kiro/` — Kiro
- `.agents/` — Cross-harness agent registry

### scripts/lib Infrastructure

Cross-platform Node.js utilities used by hooks and install:
- `utils.js` — home dir detection, platform helpers, Claude config dir
- `package-manager.js` — detect npm/pnpm/yarn/bun; override via `CLAUDE_PACKAGE_MANAGER` env var or project config
- `session-manager.js` — session read/write (session data in `~/.claude/session-data/`)
- `resolve-ecc-root.js` — resolves ECC install root from `CLAUDE_PLUGIN_ROOT` env or plugin cache scan
- `install-state.js` — tracks installed components for incremental updates

### ECC 2.0 Alpha

`ecc2/` is a Rust control-plane prototype. Build with `cargo build` inside `ecc2/`. Exposes `dashboard`, `start`, `sessions`, `status`, `stop`, `resume`, and `daemon` subcommands. Alpha only — not part of the main install flow.

## Code Style

- **Runtime**: Node.js ≥18, CommonJS (`require`/`module.exports`). No ESM (`import`/`export`) unless the file ends in `.mjs`. No TypeScript.
- **File naming**: lowercase with hyphens (`session-start.js`, `tdd-workflow.md`).
- **Variables/symbols**: camelCase.
- Prefer `const` over `let`; never `var`.
- Keep hook scripts under 200 lines — extract helpers to `scripts/lib/`.
- Hooks must exit `0` on non-critical errors. Blocking hooks (PreToolUse, stop) must stay fast (<200ms, no network calls).
- Log to stderr with a `[HookName]` prefix.

## Testing

- Tests live in `tests/` mirroring the structure of `scripts/`.
- New `scripts/lib/` files require a matching test in `tests/lib/`.
- New hooks require at least one integration test in `tests/hooks/`.
- Run `node tests/run-all.js` before committing.
- Target 80%+ coverage.

## Skill Placement Policy

| Type | Location | Shipped |
|------|----------|---------|
| Curated | `skills/<name>/SKILL.md` (repo) | Yes |
| Learned | `~/.claude/skills/learned/` | No |
| Imported | `~/.claude/skills/imported/` | No |

Only curated skills belong in the repo. Use `origin: ECC` in frontmatter for first-party skills and `origin: community` for contributed ones.

## Commit Style

Conventional commits with these prefixes: `feat`, `fix`, `docs`, `test`, `chore`.

```
feat(skills): add pytorch-patterns skill
fix(hooks): repair session path on Windows
docs: update contributing guide
test(lib): add package-manager detection cases
```

## Agent Orchestration (from AGENTS.md)

Use agents proactively without being asked:
- Complex feature → **planner**
- Code just written/modified → **code-reviewer**
- Bug fix or new feature → **tdd-guide**
- Architectural decision → **architect**
- Security-sensitive code → **security-reviewer**
- Autonomous loop execution → **loop-operator**

Launch independent agents in parallel.
