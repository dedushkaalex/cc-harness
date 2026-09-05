# cc-harness

Skills for a multi-agent code review pipeline in Claude Code. Each skill answers one question about the code and hands everything else to the skill that owns it.

Skills live in `.claude/skills/<name>/`. `write-a-skill` is the meta-skill used to create the others; every new skill must pass its Scope ownership check.

## Pipeline

```text
                 CODE
                   │
                   ▼
┌──────────────────────────────────┐
│ backend-architecture-review      │
│ What kind of work is done here?  │
│ Does it belong in this component?│
└────────────────┬─────────────────┘
                 │ Architecture summary + component table
                 ▼
┌──────────────────────────────────┐
│ backend-dependency-review        │
│ What does this component know    │
│ about other components?          │
└────────────────┬─────────────────┘
                 │
                 ▼
             next skill
```

Each report starts with a `## Scope` block: what was evaluated, what was not, and which skill owns every skipped concern. The next agent reads that block, not the whole report.

## Ownership matrix

One owner per finding category. When adding or changing a skill, update this table first, then the `Do NOT evaluate` lists of every neighbour named here.

| Finding category | Owner |
|---|---|
| Mixed kinds of work in one component | `backend-architecture-review` |
| Code of another layer inside a component (persistence in transport, business rules in repository, HTTP response shaping in a use case) | `backend-architecture-review` |
| Component and layer map (reconstructed from code) | `backend-architecture-review` |
| Dependency direction | `backend-dependency-review` |
| Leakage of types, errors, DTOs across layer boundaries | `backend-dependency-review` |
| Coupling to a concrete technology (ORM, Redis, SDK) and cost of replacing it | `backend-dependency-review` |
| Dependency cycles | `backend-dependency-review` |
| Interface / port / adapter: needed or excessive (dependency inversion, overengineering) | `backend-dependency-review` |

Splitting rule for the two backend skills: if the fix is moving code between components, the finding belongs to `backend-architecture-review`; if the fix is changing an import, a type, a dependency direction, or an abstraction, it belongs to `backend-dependency-review`.

### Not owned yet

These categories are already referred to as "another skill's area" inside existing reports, but no skill checks them. Until a skill exists, an "Out of scope" line naming one of them is a dead end for the pipeline.

| Finding category | Planned owner |
|---|---|
| SQL quality, database queries, indexes | `backend-sql-review` (to add) |
| Error handling: conversion at boundaries, lost context, swallowed errors | `backend-error-handling-review` (to add) |
| Security and authentication correctness | `backend-security-review` (to add) |
| Performance | `backend-performance-review` (to add) |
| Tests: coverage of behaviour, test design | `backend-testing-review` (to add) |
| Naming, folder structure, formatting, framework best practices | `backend-style-review` (to add) |
| Domain invariants and business rule correctness | `backend-domain-review` (to add) |

## Adding a skill

1. Run `write-a-skill` and complete its Scope ownership check.
2. Add every finding category the new skill owns to the matrix above; a category that already has an owner is a conflict to resolve before writing the skill.
3. Move the category from "Not owned yet" to the main table.
4. Update the `Do NOT evaluate` lists and `## Scope` report blocks of the neighbour skills that referred to the category as "another skill".
5. If the skill is a pipeline step, state in its SKILL.md which report it takes as input and how it avoids a second finding on the same line of code.
