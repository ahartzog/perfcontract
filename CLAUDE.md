# PerfContract

## What This Is

PerfContract is an **open standard** — a portable, version-controlled format for declaring runtime performance budgets (latency, memory, throughput, and more) in a form both humans and AI coding agents can read, enforce, and reason about. It closes the gap between performance *intent* — today locked in design docs, monitoring dashboards, and tribal knowledge — and machine-checkable *constraints that travel with the source*.

This repo **is the specification**, not an implementation. There is no runtime, library, or CLI here — only the spec prose, the JSON Schema, and examples. Changes are edits to a standard, not to software.

## Source of Truth

- **[README.md](README.md) is the primary, canonical source of truth.** It is the human- and agent-facing spec; when prose and schema disagree, the README wins and the schema gets fixed.
- **[v1/schema.json](v1/schema.json) is the machine-checkable contract.** It must stay in lockstep with the README.

**The sync rule:** README.md and v1/schema.json are two representations of one spec. Any change to a field, type, default, or constraint must land in *both*. They drift the instant you touch one and forget the other — treat that as the primary failure mode when editing.

## Design Goals

These shape every decision. The full rationale lives in the README's Core Principles table; the load-bearing ones:

1. **Self-interpreting artifacts.** A `.perfcontract.yml` an agent meets for the first time should be comprehensible *without* fetching an external spec or loading a special skill. Discovery is built into the file — the recommended header, inline field docs, and the top-level `guidance` block. The spec URL serves the 5% case (authoring new objectives or extending the schema), not the 95% case (reading and respecting budgets).
2. **Environment is required.** A budget without deployment context is a wish list.
3. **Inheritance by default; children may only tighten.**
4. **Two representations, one grammar.** Full YAML for service/system scope, shorthand for inline.
5. **LLM-native.** `rationale` and `guidance` give agents the *why*, not just the value.
6. **Declarative, not prescriptive.** The spec defines data; agent behavior lives in tooling, not in the contract.

## Conventions When Editing This Repo

- **Public standard — keep it neutral.** Examples must be free of proprietary or internal terms. Use generic domains (route planning, delivery, generic services); no company, product, or internal system names.
- **Versioning.** The spec is `v1`. Breaking changes bump the version; non-breaking additions (new metrics, new verification methods) do not.
- **Examples must validate.** Reference contracts live in [examples/](examples/) and are linted against [v1/schema.json](v1/schema.json) in CI ([.github/workflows/validate.yml](.github/workflows/validate.yml)). Run locally with `check-jsonschema --schemafile v1/schema.json examples/*.perfcontract.yml`. The README links to these files rather than inlining them, so there is no second copy to keep in sync — keep it that way.
