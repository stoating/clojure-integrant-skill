# Agent instructions

This repository contains the **`integrant`** skill — a structured set of Markdown files that teach an AI coding agent how to work with Integrant, the Clojure/ClojureScript micro-framework for data-driven application architecture.

**Any agent that understands this `AGENTS.md` convention should:**

1. Treat `integrant/SKILL.md` as the entry point — it contains a decision table pointing to the right reference file for the task.
2. Load reference files on demand based on that table:
   - `core-concepts.md` — configuration maps, multimethods (init-key, halt-key!, suspend-key!, resume-key, resolve-key, expand-key, assert-key), refs, refsets, system maps, key naming conventions
   - `lifecycle.md` — init, halt!, suspend!, resume, expand, converge, profiles, vars
   - `advanced-features.md` — composite keys, derived keywords, composite references, annotations, load-namespaces, load-hierarchy, load-annotations, normalize-key
   - `workflows.md` — typical application setup, loading EDN config, REPL workflow with integrant-repl, testing patterns, building modules, migrating from Component
   - `anti-patterns.md` — common mistakes and things to avoid
3. Always use `ig/ref` or `ig/refset` in the configuration map to express dependencies — never wire dependencies programmatically in `init-key`.
4. Ensure `halt-key!` implementations are idempotent and that `ig/halt!` is always called in a `finally` block when managing system lifetime in tests or scripts.
5. Remind users to call `ig/expand` before `ig/init` whenever `expand-key` methods are defined.

This file follows the [agents.md](https://agents.md/) convention and is honored by OpenAI Codex CLI, Cursor, Aider, Zed, Amp, Gemini CLI, Google Jules, Windsurf, Factory, RooCode, and many others.

For Claude Code, the richer native format is `.claude-plugin/` + `integrant/SKILL.md`.
