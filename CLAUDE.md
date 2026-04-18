# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm run build       # Compile TypeScript (cleans dist/, then tsc)
pnpm run dev         # Watch mode compilation
pnpm run lint        # ESLint on src/
pnpm test            # Run all tests once (Vitest)
pnpm test:watch      # Vitest watch mode
pnpm test:coverage   # Coverage report
```

Run a single test file:
```bash
pnpm test src/path/to/file.test.ts
```

## Architecture

OpenSpec is a CLI tool that generates AI-assistant instruction files (for Claude, Cursor, Windsurf, Copilot, etc.) from a schema-driven spec/change workflow. The core idea: a `schema.yaml` defines artifact types and their dependencies; the tool resolves them into a graph, enriches them with project context, and emits tool-specific command files.

### Key Modules

**`src/core/artifact-graph/`** - The engine. `graph.ts` does topological sort (Kahn's algorithm) over artifact dependencies defined in schema YAML. `schema.ts` parses/validates those schemas with Zod. `instruction-loader.ts` enriches artifacts with project context before sending to adapters.

**`src/core/command-generation/`** - Tool adapter system. `registry.ts` is a factory that maps tool names to `adapters/` (one file per AI tool). All adapters implement `ToolCommandAdapter`; they format the same content differently (e.g., `.claude/commands/` vs `.cursor/rules/`). Adding a new AI tool = new adapter file + registration.

**`src/core/templates/workflows/`** - Skill templates per artifact type (propose, apply-change, archive-change, etc.). `skill-templates.ts` is the unified entry point.

**`src/commands/workflow/`** - The newer artifact-driven CLI surface (`status`, `instructions`, `new-change`, `schemas`). The older surface lives directly in `src/commands/`.

**`schemas/spec-driven/`** - The default built-in workflow schema. User projects can define custom schemas in `openspec/config.yaml`.

### Data Flow

```
openspec/config.yaml
  → project-config.ts (Zod parse)
  → artifact-graph/schema.ts (load schema, resolve deps)
  → artifact-graph/graph.ts (topological sort)
  → instruction-loader.ts (inject context/rules)
  → command-generation/registry.ts (pick adapter)
  → adapter → writes files to .claude/commands/, .cursor/rules/, etc.
```

### Dual Workflow Modes

- **Core profile**: Lightweight — propose → apply → archive
- **Full workflow**: Explicit artifact stages with spec, design, task, and test artifacts tracked separately

### Change Isolation

Each change lives in its own folder under `openspec/changes/`. Specs live in `openspec/specs/`. Archiving a change merges its spec deltas (ADDED/MODIFIED/REMOVED/RENAMED) back into the canonical specs.

### Self-Hosting

The `openspec/` directory at the repo root is OpenSpec tracking itself — real specs and change proposals for the OpenSpec CLI.

## Config Layers

1. `~/.openspec/` — global config (profiles, tool preferences)
2. `openspec/config.yaml` — per-project config (schema, context, tool list)
