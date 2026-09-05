# cc-harness

Skills for a multi-agent code review pipeline in Claude Code. Each skill answers one question about the code and hands everything else to the skill that owns it.

Skills live in `.claude/skills/<name>/`. `write-a-skill` is the meta-skill used to create the others; every new skill must pass its Scope ownership check.

Primary stack of the reviewed projects is **Effect TS**. Skills treat `Context.Tag` + Live Layer + Test Layer as the baseline unit of dependency, not as an abstraction; NestJS and plain TS are covered as secondary styles. Both backend skills use the same six layers: transport, application, domain, infrastructure, persistence, external service.

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

Each report starts with a `## Scope` block: code under review, context sources, what was evaluated, what was not, and which skill owns every skipped concern. The next agent reads that block, not the whole report.

Handoff rules between the two backend skills:

- `backend-dependency-review` takes the component table from the architecture report as its input model. It verifies it against the code, adds only the arrows it needs, and reports contradictions in a `Mismatches` section instead of reclassifying components.
- Arrows that disappear when an architecture finding is applied are not evaluated; the arrow that appears after the move is evaluated as a **conditional finding** («при условии применения [P2] architecture-review»).

## Ownership matrix

One owner per finding category. When adding or changing a skill, update this table first, then the `Do NOT evaluate` lists of every neighbour named here.

| Finding category | Owner |
|---|---|
| Mixed kinds of work in one component (including business rules inside an ORM entity class) | `backend-architecture-review` |
| Code of another layer inside a component (persistence in transport, business rules in repository, HTTP response shaping in a use case) | `backend-architecture-review` |
| Component and layer map (reconstructed from code), code under review, context sources | `backend-architecture-review` |
| Dependency direction (including `Domain → ORM` for whatever remains after business rules are moved out) | `backend-dependency-review` |
| Leakage of types and DTOs across layer boundaries | `backend-dependency-review` |
| Lower-layer error **type** visible in an upper layer (`SqlError` in an application effect's `E` channel, `AxiosError` caught in a handler) | `backend-dependency-review` |
| Coupling to a concrete technology (ORM, Redis, SDK) and cost of replacing it | `backend-dependency-review` |
| Dependency cycles (imports, Layer requirements, DI) | `backend-dependency-review` |
| Tag / interface / port / adapter: needed or excessive (dependency inversion, overengineering) | `backend-dependency-review` |

Splitting rule for the two backend skills: a finding belongs to `backend-architecture-review` when the cause is the wrong kind of work inside a component (the fix moves code); it belongs to `backend-dependency-review` when the cause is what a component knows about another (the fix changes an import, a type, a dependency direction, or an abstraction). A line that fits both is owned by `backend-architecture-review`, and `backend-dependency-review` evaluates only the arrow that remains after the move.

### Not owned yet

These categories are already referred to as "another skill's area" inside existing reports, but no skill checks them. Until a skill exists, an "Out of scope" line naming one of them is a dead end for the pipeline.

| Finding category | Planned owner |
|---|---|
| SQL quality, database queries, indexes | `backend-sql-review` (to add) |
| Error handling correctness: lost context, swallowed errors, retry and recovery, mapping to responses (not whether the lower-layer error type is visible — that is `backend-dependency-review`) | `backend-error-handling-review` (to add) |
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
