# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Read [AGENTS.md](AGENTS.md) first.** It contains the authoritative rules for code quality, commands, git, dependencies, testing, changelogs, and releasing. This file covers architecture; AGENTS.md covers process. When they overlap, AGENTS.md wins.

## What this is

`pi` is a self-extensible terminal coding agent. This is an npm monorepo of four lockstep-versioned packages that build on each other:

- **`packages/ai`** (`@earendil-works/pi-ai`) — Unified multi-provider LLM API. Bottom of the stack, no dependency on the others. Only includes models that support tool calling. `models.generated.ts` is generated (see AGENTS.md rule: edit `scripts/generate-models.ts`, never the generated file).
- **`packages/agent`** (`@earendil-works/pi-agent-core`) — Stateful agent runtime: the agent loop, tool execution, message/session types, compaction. Built on `pi-ai`. Provider-agnostic and tool-agnostic.
- **`packages/tui`** (`@earendil-works/pi-tui`) — Terminal UI library with differential rendering. Standalone, no dependency on the others.
- **`packages/coding-agent`** (`@earendil-works/pi-coding-agent`) — The `pi` CLI. Depends on all three. Contains the concrete tools (read/write/edit/bash/grep/find/ls), session management, config, and the extension system.

Dependency direction: `ai → agent`, and `coding-agent → {ai, agent, tui}`. `tui` is independent.

## Common commands

Run from the repo root unless noted. See AGENTS.md for the full rules (notably: never run `npm run build`/`npm test`/full vitest unless the user asks).

```bash
npm install --ignore-scripts   # hydrate deps (never run lifecycle scripts unless asked)
npm run check                  # biome + dep/import/shrinkwrap checks + tsgo --noEmit. Run after every code change; fix all errors/warnings/infos.
./test.sh                      # run non-LLM tests with all provider API keys/auth stripped (CI-safe)
./pi-test.sh                   # run pi from source in any directory (preserves caller cwd)
```

Run a single test from the package root (e.g. `packages/coding-agent`):

```bash
node ../../node_modules/vitest/dist/cli.js --run test/path/to.test.ts
```

Do **not** run the bare vitest suite — it activates e2e tests that hit real providers when endpoint/auth env vars are present. `./test.sh` strips those env vars; use it for everything non-e2e.

## coding-agent architecture (`packages/coding-agent/src`)

The bulk of the work happens here. Key structure:

- **`core/agent-session.ts`** (~100KB) — central orchestrator tying together the agent loop, tools, model registry, session persistence, and extensions. `agent-session-runtime.ts` and `agent-session-services.ts` split out runtime event handling and service wiring.
- **`core/tools/`** — the built-in tools: `read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`. `index.ts` registers them; `file-mutation-queue.ts` serializes concurrent file writes.
- **`core/extensions/`** — the TypeScript extension system (`loader.ts`, `runner.ts`, `types.ts`). Extensions are how users adapt pi without forking. This is the largest extension surface; see `docs/extensions.md`.
- **`core/`** also holds: `session-manager.ts` (persistence, JSONL sessions), `model-registry.ts` + `model-resolver.ts` (model discovery/selection), `settings-manager.ts` + `config.ts` (config layering), `compaction/` (context window management), `skills.ts`, `prompt-templates.ts`, `trust-manager.ts`, `auth-storage.ts`.
- **`modes/`** — pi's four run modes: `interactive/` (TUI), `rpc/` (process integration over JSONL), `print-mode.ts` (print/JSON one-shot). The SDK (`core/sdk.ts`) is the fourth, for embedding.
- **`main.ts`** / `cli.ts` / `cli/args.ts` — entrypoint, argument parsing, mode dispatch.
- **`config.ts`** — **always use this** (`getPackageDir`, `getThemeDir`, etc.) to resolve package assets. Never use `__dirname` directly: pi runs in three execution modes (npm install, standalone Bun binary, tsx-from-source) with different path layouts.

## ai providers (`packages/ai/src/providers`)

Each provider (anthropic, openai-completions, openai-responses, google, mistral, amazon-bedrock, cloudflare, ...) implements the unified streaming interface in `types.ts`. `register-builtins.ts` registers them. `faux.ts` is the deterministic fake provider used by tests — register it with `registerFauxProvider`.

## Testing the coding-agent

Two harnesses exist; prefer the new one:

- **New (preferred): `test/suite/`** — uses `test/suite/harness.ts` + the faux provider (`packages/ai/src/providers/faux.ts`). No real APIs, keys, network, or paid tokens. Deterministic and CI-safe. Broad lifecycle tests go directly under `test/suite/`; issue regressions go under `test/suite/regressions/` named `<issue-number>-<short-slug>.test.ts`.
- **Legacy: `test/test-harness.ts`** — don't extend it unless a missing capability forces you to.

## Testing interactive mode

The TUI can be driven in a controlled terminal via tmux (see AGENTS.md "Testing pi Interactive Mode with tmux" for the full recipe). `/debug` (hidden command) dumps rendered TUI lines and the last LLM messages to `~/.pi/agent/pi-debug.log`.

## Conventions specific to this repo

- **Erasable TypeScript only** in `packages/*/src`, `packages/*/test`, `packages/coding-agent/examples` (Node strip-only mode): no `enum`, `namespace`, parameter properties, `import =`, etc. Use explicit fields + constructor assignments. No `any` without strong justification. No inline/dynamic imports — top-level only.
- **Keybindings** must go through `DEFAULT_EDITOR_KEYBINDINGS` / `DEFAULT_APP_KEYBINDINGS`; never hardcode `matchesKey(..., "ctrl+x")` checks.
- **Pinned deps + supply-chain hardening**: direct external deps are pinned exact; lockfile changes are gated behind `PI_ALLOW_LOCKFILE_CHANGE=1`. See AGENTS.md "Dependency and Install Security".
- **Lockstep versioning**: all four packages share one version and release together (`patch` = fixes/additions, `minor` = breaking; no majors).
- **Changelogs**: per-package `CHANGELOG.md`, new entries under `## [Unreleased]` only; released sections are immutable.
- **Docs** for user-facing features live in `packages/coding-agent/docs/` (extensions, sdk, rpc, models, providers, skills, themes, settings, compaction, session-format, etc.) — a good map of feature areas.
