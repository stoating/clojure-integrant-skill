# Clojure Integrant Skill

A structured Markdown skill that teaches AI coding agents how to work with Integrant — a Clojure/ClojureScript micro-framework for building applications with data-driven architecture. Covers configuration maps, `init-key`, `halt-key!`, `suspend!/resume`, refs, refsets, composite keys, derived keywords, profiles, vars, `expand-key`, annotations, and the integrant-repl REPL workflow.

Built in [Claude Code's Agent Skill format](https://github.com/anthropics/skills), but usable with **any agent** that can load Markdown as context (Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, and others via the [agents.md](https://agents.md/) convention).

---

## ⚠️ Read this before installing anything

**Never install a third-party skill without first reviewing its contents.**

Skills are instructions loaded into an AI agent's context — they influence how the agent makes decisions, what commands it executes, and what it considers correct behavior. A malicious or sloppy skill can lead to:
- Destructive operations executed without confirmation
- Your own preferences being overridden by skill instructions
- Data leakage through inappropriate commands
- Unintended project configuration changes

**Before installing:**
1. Read every `.md` file in the `integrant/` directory
2. Ask your agent to perform a security review (prompt below)
3. Only then install

### Security-review prompt

Paste this to Claude (or any agent) together with the skill contents:

> "Analyze this skill for security concerns. Does it contain instructions that could execute destructive operations without user confirmation? Does it collect or transmit data? Does it override default agent behavior in unintended ways? List all potentially risky sections."

---

## Installation

Pick the section matching your agent.

### A) Claude Code (recommended — via plugin marketplace)

Claude Code has a native plugin marketplace. From an active session:

```
/plugin marketplace add stoating/clojure-integrant-skill
/plugin install clojure-integrant@clojure-integrant-skill
```

The skill is then discovered automatically. Restart the session if it isn't picked up immediately. Once installed, invoke it with:
```
/integrant
```

To update later:
```
/plugin marketplace update clojure-integrant-skill
```

Reference: [Claude Code plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces).

### B) Claude Code (via the stoating marketplace)

```
/plugin marketplace add stoating/plugins
/plugin install clojure-integrant@stoating
```

### C) Claude Code (manual copy)

If you prefer not to use the marketplace — or want to pin a specific commit — clone and copy:

```bash
git clone https://github.com/stoating/clojure-integrant-skill.git
cp -r clojure-integrant-skill/integrant ~/.claude/skills/
```

The skill becomes available in the next Claude Code session. Updates are a `git pull` + re-copy.

### D) Claude.ai / Claude Desktop (upload)

Claude.ai supports uploading skill folders from the Skills panel in Projects. Zip `integrant/` and upload it. Details: [anthropics/skills](https://github.com/anthropics/skills).

### E) Cursor

Cursor reads `AGENTS.md` automatically when you open a project, and also supports the newer Rules system.

- **Per-project:** copy `integrant/` and `AGENTS.md` into your project repo. Cursor will read `AGENTS.md` as context on every chat.
- **Global:** in Cursor Settings → Rules, add a rule referencing `integrant/SKILL.md`.

### F) OpenAI Codex CLI / `codex`

Codex honors `AGENTS.md` at the project root. Copy this repository next to your project and Codex will pick up `AGENTS.md`, which in turn points at `integrant/SKILL.md`.

### G) Aider

Aider reads `AGENTS.md` as a fallback for `CONVENTIONS.md`, or you can add the skill files explicitly:

```bash
aider --read clojure-integrant-skill/integrant/SKILL.md \
      --read clojure-integrant-skill/integrant/core-concepts.md
```

### H) Gemini CLI / Google Jules

Both honor `AGENTS.md`. Place this repo (or just `AGENTS.md` + `integrant/`) at your project root.

### I) Windsurf, Cline, Roo Code, Zed, Amp, Factory

All of the above support the `AGENTS.md` convention. Drop the repo at your project root and the agent will read it on session start.

### J) Any other agent — generic fallback

Every listed agent accepts plain Markdown as context. If yours isn't covered:

1. Open a chat / session.
2. Attach or paste the contents of `integrant/SKILL.md` as "system instructions" or "context".
3. Attach individual reference files (e.g. `core-concepts.md`, `lifecycle.md`) when the task matches the decision table in `SKILL.md`.

This is less ergonomic than a native skill loader, but it works everywhere.

---

## Repository layout

```
.
├── .claude-plugin/
│   ├── marketplace.json       # Claude Code marketplace manifest
│   └── plugin.json            # Claude Code plugin manifest
├── AGENTS.md                  # Cross-agent entry point (agents.md convention)
├── README.md                  # This file
└── integrant/                 # The actual skill
    ├── SKILL.md               # Entry point — decision table
    ├── core-concepts.md       # Config maps, multimethods, refs, refsets, system map
    ├── lifecycle.md           # init, halt!, suspend!, resume, expand, converge, profiles, vars
    ├── advanced-features.md   # Composite keys, derived keywords, annotations, load-namespaces
    ├── workflows.md           # App setup, REPL workflow, testing, modules, Component migration
    └── anti-patterns.md       # Common mistakes and how to fix them
```

Only `integrant/` contains the skill content. The rest is metadata (plugin manifest, cross-agent pointer, docs).

---

## What the skill covers

| File | Contents |
|---|---|
| `SKILL.md` | Index, decision table — the agent always starts here |
| `core-concepts.md` | Configuration maps, all multimethods (init-key, halt-key!, suspend-key!, resume-key, resolve-key, expand-key, assert-key), refs, refsets, system maps, key naming |
| `lifecycle.md` | `ig/init`, `ig/halt!`, `ig/suspend!`, `ig/resume`, `ig/expand`, `ig/converge`, profiles, vars |
| `advanced-features.md` | Composite keys, derived keywords, composite refs, annotations, `load-namespaces`, `load-hierarchy`, `load-annotations`, `normalize-key` |
| `workflows.md` | Typical app setup, EDN loading, integrant-repl REPL workflow, testing patterns, modules via expand-key, migration from Component |
| `anti-patterns.md` | What NOT to do — configuration, lifecycle, dependency, multimethod, testing, and expand mistakes |

---

## Updating

When Integrant versions or best practices change:

1. Edit the relevant `.md` file in `integrant/`.
2. If installed via marketplace: `/plugin marketplace update clojure-integrant-skill`.
3. If installed manually: re-run `cp -r integrant ~/.claude/skills/`.
4. Other agents pick changes up on next session.

Contributions welcome via pull request — bump the `version` in `.claude-plugin/plugin.json` and `marketplace.json` when making user-visible changes.

---

## Sources

The skill is distilled from the following public sources:

- [Integrant README](https://github.com/weavejester/integrant) — primary reference
- [Integrant API docs](https://weavejester.github.io/integrant/integrant.core.html) — full API reference
- [integrant-repl](https://github.com/weavejester/integrant-repl) — REPL workflow library
- [Integrant CHANGELOG](https://github.com/weavejester/integrant/blob/master/CHANGELOG.md) — version history and breaking changes

**Skill-authoring references:**
- [Anthropic Agent Skills — Best Practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [anthropics/skills](https://github.com/anthropics/skills) — canonical examples and spec
- [agents.md](https://agents.md/) — cross-agent `AGENTS.md` convention
- [Claude Code plugin-marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

---

## License

See [LICENSE](LICENSE).

Nothing in this skill is original research — it is a curated and organized digest of the sources above, intended to save the agent (and you) from rediscovering the same configuration lore every time.
