# Backend review pipeline

Five review skills run in order. Each takes the previous reports as its input model, verifies that model against the code, and reports what it could not own to the skill that owns it.

```text
backend-architecture-review   →  Architecture summary (component table, layers, Context)
backend-dependency-review     →  Dependency map
backend-domain-review         →  Domain summary (rules, owners, transitions)
backend-persistence-review    →  Persistence summary (operations, boundaries, constraints)
backend-error-handling-review →  Error-handling summary (error sources, paths)
```

An input model is a starting point to verify, not a fact to trust: the previous agent may have missed something. Contradictions go to `Mismatches` in the `Scope` block with the owning skill named, and the model itself is not silently rewritten. Anything noticed outside a skill's scope goes to its `Вне scope` section with the owner named, and the owning skill must give every such line a verdict — finding, non-finding, or existing protection.

## Ownership matrix

One owner per finding category. Assembled from each skill's `Do NOT evaluate` list; update it whenever a skill is added or its scope changes.

| Finding category | Owner |
|---|---|
| Kind of work performed inside a component; mixed responsibilities | `backend-architecture-review` |
| Code sitting in the wrong layer; where a transaction, a rule, an error handler physically lives | `backend-architecture-review` |
| Dependency direction, coupling, cost of replacing a technology | `backend-dependency-review` |
| Leakage of types, DTO and error types across layer boundaries | `backend-dependency-review` |
| Dependency cycles | `backend-dependency-review` |
| Contract / port / adapter — needed or excessive; dependency inversion | `backend-dependency-review` |
| Existence and ownership of a business rule; bypass paths around a check | `backend-domain-review` |
| Duplicated business rules; reachable invalid state transitions | `backend-domain-review` |
| Domain concept represented by a primitive so that an invariant is easy to break | `backend-domain-review` |
| Transaction boundaries and atomicity; partial writes | `backend-persistence-review` |
| Concurrency: the outcome depends on the ordering of parallel operations | `backend-persistence-review` |
| Storage constraints; idempotency of a repeated write; query cost; migration compatibility | `backend-persistence-review` |
| Mapping of meaning between storage and application; side effects at rollback; what remains stored after a failure | `backend-persistence-review` |
| Errors lost or swallowed; operation reported as successful when it was not | `backend-error-handling-review` |
| Cause and context lost while wrapping | `backend-error-handling-review` |
| Outcomes indistinguishable for the caller that must react to them differently | `backend-error-handling-review` |
| Retry policy against the nature of the failure; unbounded retry, missing backoff | `backend-error-handling-review` |
| What crosses the outward boundary; partial failure reported wrongly | `backend-error-handling-review` |
| Duplicated or missing logging that hides what happened | `backend-error-handling-review` |
| API design: shape and versioning of the error contract, HTTP semantics | not assigned yet |
| Security: injection, auth mechanics, sensitive data in responses and logs | not assigned yet |
| Tests, naming, style, folder structure | not assigned yet |
| Log format and levels as such; performance outside persistence | not assigned yet |

## Boundaries that need a splitting rule

Categories that touch the same lines of code. Each pair has a one-line rule written in both skills.

| Pair | Rule |
|---|---|
| architecture / dependency | Fix moves code between components — architecture. Fix removes an import, changes a type, flips an arrow — dependency. |
| architecture / domain | In which layer the rule lives — architecture. Whether an existing path bypasses it — domain. |
| architecture / persistence | The transaction sits in the wrong component — architecture. Whether it covers both writes — persistence. |
| architecture / error handling | The handler sits in the wrong component — architecture. What it does with the error — error handling. |
| dependency / domain | Type changes so a component stops knowing a technology — dependency. Type changes so an invalid value cannot be built — domain. |
| dependency / persistence | Who knows the storage type and what replacing it costs — dependency. Whether the stored value and the application value mean the same — persistence. |
| dependency / error handling | The lower layer's error type is visible in the upper layer — dependency. The caller cannot make a decision from the error — error handling, whatever the types. |
| domain / persistence | Whether the rule exists and every path goes through it — domain. Whether storage holds it under concurrent, repeated or partial execution — persistence. |
| domain / error handling | Whether the rule and its domain error exist — domain. Whether that error reaches the reacting party distinguishable — error handling. |
| persistence / error handling | What remains stored after a failure at step two — persistence. What the caller was told about it — error handling. Whether a retry is idempotent — persistence. Whether to retry and how often — error handling. |

## Shared conventions

- **Six layers**, identical across all five skills: transport / application / domain / infrastructure / persistence / external service.
- **Scope block** in every report: `Code under review`, `Input`, `Context`, `Mismatches`, `Evaluated`, `Not evaluated` with an owner per line.
- **Confidence** — high / medium / low. An assumption with no source in Step 0 caps a finding at medium.
- **Severity** — P1 / P2 / P3, by consequence, never by taste. A stylistic preference is not a finding.
- **Base unit** — a mandatory Context line. Architecture and dependency record the base unit of *dependency*; error handling records the base unit of *error*. Both exist for the same reason: not to report a stack idiom as a problem. The mechanism changes evidence and cost, never the verdict.
- **Substitution test** — rewrite Problem, the failure scenario and Why it matters without framework names, then check the finding survives on at least three other stacks. If it disappears, it was not a finding.
- **Limits** — findings capped at 10 per report, the rest as one P3 "other places of the same kind"; one line per category in `Вне scope`.
- **Existing protections** (`Good decisions`, `Good dependencies`, `Protected invariants`, `Existing protections`) are as mandatory as findings: the next reviewer must know what not to touch.
