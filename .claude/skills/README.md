# Backend review pipeline

Five review skills run in order. Each takes the previous reports as its input model, verifies that model against the code, and reports what it could not own to the skill that owns it.

```text
backend-architecture-review   →  Architecture summary (component table, layers, Context)
backend-dependency-review     →  Dependency map
backend-domain-review         →  Domain summary (rules, owners, transitions)
backend-persistence-review    →  Persistence summary (operations, boundaries, constraints)
backend-error-handling-review →  Error-handling summary (error sources, paths)
```

`backend-review` runs the five in this order as one pipeline: one code boundary, one Context, closed handoffs, one merged report. It owns no finding category and has no row in the matrix — see [Orchestrator](#orchestrator-backend-review) below.

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

## Orchestrator: `backend-review`

`backend-review` is the executor of this README, not a sixth reviewer. It never creates a finding, never changes a severity or a confidence, never widens the code under review, and never edits a skill's report. Anything it notices in the code itself is not recorded anywhere.

**What it passes in.** Every skill runs as a separate agent — a `backend-<x>-reviewer` subagent from `.claude/agents/` that preloads the skill, restricts tools to reading plus writing the report, and pins the model — and receives:

- the `Shared context` block (format below) instead of running its own Step 0 search — the skill takes it as `Context` and as the code boundary, and collects only what its own Step 0 needs beyond it (rule sources, volumes, retry policies, global handlers, outward error contract), marked "дособрано";
- the `Scope` and summary blocks of every previous report — Architecture summary, Dependency map, Domain summary, Persistence summary — as input models to verify, not facts;
- the `Вне scope` lines of previous reports whose owner is this skill, including owner mentions inside findings ("Вне scope: backend-persistence-review" in Trade-offs). Each must get a verdict on the same file and line. A verdict that has no section in the skill's own template goes to a `## Handoff verdicts` section at the end of the report, written by the skill itself; the section exists only in pipeline runs, its format is in `backend-review/PROTOCOL.md`, Stage B.

Full findings of previous reports are not passed; a skill reads a specific place from `<output-dir>/<skill>.md` when it needs one. Mismatches always point at a skill that has already run, so they are closed after the sequence by a targeted pass: the owner is called once with only those lines and the instruction to give verdicts and do nothing else.

**What it checks on the way out.** Protocol, not content:

- each report exists at `<output-dir>/<skill>.md`, its `Code under review` equals the shared boundary, and every `Вне scope` line names an owner or "владелец не назначен";
- every `Вне scope` line and every Mismatch has a verdict in the owner's report on the same place — finding, Non-finding, existing protection or `Handoff verdicts`; what the targeted pass leaves open goes to `Unresolved handoffs`;
- one place is not described by two findings of one category: the owner by this matrix keeps it, the other finding becomes a "дубль снят" note under the owner's; two findings of different categories on one place are what the splitting-rule table designs, and they are cross-linked;
- a protection in one report against a finding in another that denies it goes to `Conflicts` with both positions and the splitting rule quoted — unresolved;
- every finding has evidence with file and line, the owner's scenario field, a confidence with a basis and a severity P1–P3 — four presence checks; a finding that fails one is listed in `Dropped` with the reason and stays in the owner's report. The substitution test on Problem and Why it matters is a note, not a check: a doubtful result is written into the finding's `Pipeline:` line and the finding stays, because that result is a judgement about content.

Then one merged report, finding texts verbatim from the owners with an owner tag and a `Pipeline:` line. Details: `backend-review/PROTOCOL.md`, template: `backend-review/REPORT.md`.

**`Shared context` format.** A block each skill pastes into its `Scope` as `Context` without rework; the lines match the `Context` block of every skill's REPORT.md, both base units included.

```text
## Shared context

Target: PR #42 (main...feature/refund) — режим: полный
Code under review: src/billing/**, src/http/billing.controller.ts, src/jobs/refund.job.ts
  (diff PR #42 плюс один шаг по графу вызовов; граница одна для всех скиллов)

Известно:    один transport (HTTP) плюс job возвратов — README, docker-compose
Известно:    два экземпляра API, rolling deploy — k8s/deployment.yaml, CI
Известно:    webhook оплаты повторяется до 5 раз при таймауте — docs/payments.md
Не известно: планируется ли второе хранилище — в источниках нет
Deployment model: два экземпляра, rolling deploy; очередь заказов at-least-once — infra/queue.yaml
Источники повторов: webhook оплаты — docs/payments.md; очередь заказов — infra/queue.yaml;
             retry в HTTP-клиенте платежей — не известно
Базовая единица: class + параметр конструктора, регистрация в DI — источник: все сервисы
             в src/; подмена в тестах: override provider, test/**/*.spec.ts
Базовая единица ошибки: исключение — источник: сценарии в src/** бросают и ловят, объявленных
             типов ошибок в сигнатурах нет; на границе — exception filter, src/main.ts:18
```
