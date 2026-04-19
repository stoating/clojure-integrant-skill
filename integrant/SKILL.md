---
name: integrant
description: Use when working with Integrant — a Clojure/ClojureScript micro-framework for data-driven application architecture. Activate when the user asks about ig/init, ig/halt!, ig/suspend!, ig/resume, ig/ref, ig/refset, init-key, halt-key!, suspend-key!, resume-key, expand-key, assert-key, resolve-key, composite keys, derived keywords, profiles, vars, load-namespaces, load-hierarchy, or structuring a Clojure application with Integrant.
version: 1.0.0
---

# Integrant

Integrant is a Clojure/ClojureScript micro-framework for building applications with a data-driven architecture. The entire system is described as a **configuration map**; keys are initialized in dependency order, and the system is torn down in reverse order.

## Quick Decision: What do you need?

| Task | Go to |
|------|-------|
| Understand config maps, multimethods, refs, refsets, the system model | [core-concepts.md](core-concepts.md) |
| init, halt!, suspend!, resume, expand, converge, profiles, vars | [lifecycle.md](lifecycle.md) |
| Composite keys, derived keywords, annotations, load-namespaces, asserting | [advanced-features.md](advanced-features.md) |
| Typical application setup, REPL workflow, testing, integrant-repl | [workflows.md](workflows.md) |
| Something isn't working / weird behavior / things to avoid | [anti-patterns.md](anti-patterns.md) |

## Core Mental Model

A **configuration map** is a plain EDN map where each top-level key represents one component of the system. Keys are namespaced keywords. Dependencies between keys are expressed with `ig/ref` (one resolved value) or `ig/refset` (a set of all matching values). Integrant topologically sorts the keys and calls `init-key` on each in order, replacing the config value with the initialized result. `halt!` runs `halt-key!` in reverse order. The system lives entirely in data — there is no global state unless you introduce it yourself.
