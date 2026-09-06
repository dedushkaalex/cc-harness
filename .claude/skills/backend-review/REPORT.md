# Шаблон единого отчёта

Справочник к Stage F из [SKILL.md](SKILL.md). Отчёт пишется в `<output-dir>/backend-review.md`. Блок Scope, затем восемь разделов в этом порядке; пустой раздел остаётся с пометкой «не найдено», чтобы читатель знал, что проход был. Всё, что здесь сказано о коде, — текст владельцев из их отчётов; оркестратор добавляет только теги, ссылки, строки `Pipeline:` и статусы.

## Лимиты

- Findings — 15 в списке; остальное — одна строка на skill: «ещё N findings (P2 ×a, P3 ×b) — `<output-dir>/<skill>.md`».
- Existing protections — без лимита; одно место у двух скиллов — одна строка.
- Non-findings — только счётчик и ссылка, не перечислять.
- «Вне scope» — одна строка на категорию «владелец не назначен».

## Правила слияния

1. **Scope:** Evaluated — объединение списков Evaluated отработавших скиллов; Not evaluated — категории пропущенных и не запускавшихся скиллов с владельцем и причиной, плюс категории «владелец не назначен». Читатель видит не «этого не проверяли», а кто обязан был проверить и почему не проверил.
2. **Порядок findings:** P1 → P2 → P3; внутри уровня — порядок pipeline владельца, внутри владельца — порядок его отчёта. В 15 попадают первые по этому порядку.
3. **Текст finding** — из отчёта владельца целиком и без правок: заголовок и все поля. Заголовок получает тег владельца: `### [P1] `backend-persistence-review` — Название`. В конец добавляется одна строка `- **Pipeline:**` — ссылка на отчёт и пометки: «зависит от [Px] `<skill>` „…“», «дубль снят: `<skill>` описал то же место как [Px] „…“», «то же место с другой стороны: [Px] `<skill>` „…“», «см. Conflicts».
4. **Severity и Confidence** — владельца; дубль с другим уровнем на них не влияет.
5. **Existing protections** — объединение `Good decisions`, `Good dependencies`, `Protected invariants`, `Existing protections` (persistence), `Existing protections` (error handling). Строка = тег владельца + текст владельца. Одно место у двух — строка первого по pipeline с тегами обоих.
6. **Non-findings** — счётчик на skill и ссылка на раздел; у `backend-architecture-review` и `backend-dependency-review` раздела нет — так и записано. `Overengineering` dependency-review — одной строкой со счётчиком здесь же.
7. **«Вне scope»** — только категории «владелец не назначен»: security, API design, tests, naming, log format, performance outside persistence и что ещё встретилось. Строка = категория — места через запятую, каждое с тегом источника. Строки с назначенным владельцем сюда не попадают: закрытые учтены счётчиком в Scope, незакрытые — в Unresolved handoffs.
8. **Unresolved handoffs** — место, фраза, источник, владелец, причина: «вердикта нет после адресного прохода», «владелец пропущен», «владелец не запускался», «появилась в адресном проходе».
9. **Conflicts** — место; обе позиции дословно с именем skill и раздела; правило из README; строка «не разрешено оркестратором».
10. **Dropped** — finding владельца одной строкой: уровень, тег, название, номер проверки Stage E и причина, ссылка на отчёт.
11. **Skipped** — skill, причина обоих сбоев, кто работал без его отчёта, сколько строк ему адресовано.

## Структура

```markdown
# Backend review — PR #42

## Scope

Target: PR #42 (`main...feature/refund`), режим: полный.
Output: `reviews/2026-09-06-pr-42/`

Code under review: `src/billing/**`, `src/http/billing.controller.ts`,
`src/jobs/refund.job.ts` (diff PR #42 плюс один шаг по графу вызовов).
Граница одна для всех скиллов.

Shared context — `reviews/2026-09-06-pr-42/shared-context.md`:
- Известно: один transport (HTTP) плюс job возвратов — README, docker-compose
- Известно: два экземпляра API, rolling deploy — k8s/deployment.yaml, CI
- Не известно: планируется ли второе хранилище — в источниках нет
- Deployment model: два экземпляра, rolling deploy; очередь заказов at-least-once
- Источники повторов: webhook оплаты, очередь заказов; retry в HTTP-клиенте — не известно
- Базовая единица: class + параметр конструктора, регистрация в DI
- Базовая единица ошибки: исключение; на границе — exception filter

| Skill | Отчёт | Граница | Findings | Handoffs |
|---|---|---|---|---|
| `backend-architecture-review` | `…/backend-architecture-review.md` | = Shared context | P1 0 · P2 4 · P3 3 | closed 4 · unresolved 0 |
| `backend-dependency-review` | `…/backend-dependency-review.md` | = Shared context | P1 1 · P2 2 · P3 1 | closed 2 · unresolved 1 |
| `backend-domain-review` | `…/backend-domain-review.md` | шире: + `src/import/**` | P1 1 · P2 1 · P3 2 | closed 3 · unresolved 0 |
| `backend-persistence-review` | `…/backend-persistence-review.md` | = Shared context | P1 1 · P2 2 · P3 0 | closed 2 · unresolved 0 |
| `backend-error-handling-review` | — | — | — | пропущен, см. Skipped |

Не запускались (частичный режим): нет.

Evaluated: responsibilities and placement of work; dependency direction, leakage,
coupling, cycles, abstractions; business rules и invariants; transactions,
concurrency, constraints, query cost, migrations, mapping.

Not evaluated:
- error handling: потери и глотание, различимость исходов, retry, границы наружу,
  частичный отказ — Owner: `backend-error-handling-review` (пропущен, см. Skipped)
- security, API design, tests, naming, log format, performance outside persistence
  Owner: not assigned yet

## Findings

15 из 16 (18 у владельцев: один снят как дубль, один в Dropped); остальные — по скиллам ниже.

### [P1] `backend-persistence-review` — Параллельные списания проходят проверку остатка по одному снимку

- **Persistence operation:** …
- **Evidence:** `src/billing/withdraw.ts:31` — …
- … (все поля владельца без правок)
- **Pipeline:** `reviews/2026-09-06-pr-42/backend-persistence-review.md`; то же место
  с другой стороны: [P2] `backend-architecture-review` „Controller управляет транзакцией“.

### [P2] `backend-dependency-review` — OrderService будет знать ORM после переноса транзакции

- … (поля владельца)
- **Pipeline:** `…/backend-dependency-review.md`; зависит от [P2]
  `backend-architecture-review` „Controller управляет транзакцией и обращается к ORM напрямую“.

### [P2] `backend-architecture-review` — Controller управляет транзакцией и обращается к ORM напрямую

- … (поля владельца)
- **Pipeline:** `…/backend-architecture-review.md`; дубль снят: `backend-domain-review`
  описал то же место как [P3] „Транзакция в handler“ — уже описан у владельца.

Не вошли в список:
- `backend-domain-review` — ещё 1 finding (P3 ×1) — `…/backend-domain-review.md`

## Existing protections

- `backend-architecture-review` — `src/orders/order.service.ts:40-58` — транзакция и обе
  записи (заказ, резерв) в одном сценарии; см. Conflicts.
- `backend-domain-review`, `backend-persistence-review` — `migrations/004` — CHECK
  `balance >= 0` держит остаток при любом writer.

## Non-findings

- `backend-architecture-review` — раздела нет в шаблоне skill
- `backend-dependency-review` — раздела нет; Overengineering: 1 — `…/backend-dependency-review.md`
- `backend-domain-review` — 3 — `…/backend-domain-review.md`
- `backend-persistence-review` — 4 — `…/backend-persistence-review.md`

## Вне scope

- security — `src/billing/report.ts:12` (`backend-persistence-review`),
  `src/http/auth.ts:8` (`backend-architecture-review`) — владелец не назначен
- API design — `src/http/errors.ts:12` (`backend-domain-review`) — владелец не назначен

## Unresolved handoffs

- `src/orders/order.service.ts:40` — после сбоя на втором шаге заказ остаётся без резерва
  (из `backend-dependency-review`, Вне scope) → `backend-error-handling-review`: владелец пропущен.

## Conflicts

- `src/orders/order.service.ts:40-58`
  - `backend-architecture-review`, Good decisions: «транзакция и обе записи (заказ, резерв)
    лежат в одном сценарии — сохранить».
  - `backend-persistence-review`, [P1] «Резерв пишется после commit заказа»:
    «`order.service.ts:58` — запись резерва вне блока транзакции `order.service.ts:40-55`».
  - Правило README (architecture / persistence): «The transaction sits in the wrong
    component — architecture. Whether it covers both writes — persistence.»
  - Не разрешено оркестратором; обе записи остаются на своих местах.

## Dropped

- [P3] `backend-dependency-review` «Один сервис использует SDK платежей напрямую» —
  проверка 2: нет Change scenario — `…/backend-dependency-review.md`

## Skipped

- `backend-error-handling-review` — агент дважды завершился без файла отчёта (timeout);
  без его отчёта работал: никто (последний в порядке); адресовано ему строк: 3,
  все в Unresolved handoffs.
```

## Что оркестратор не пишет в этот отчёт

- Finding, которого нет в отчёте владельца, — в том числе «очевидный», «который все пропустили».
- Вердикт на строку handoff вместо владельца — даже «это явно не проблема».
- Изменённый уровень, confidence или переформулированный Problem.
- Разрешение конфликта: «прав persistence-review».
- Места вне границы кода под ревью, которые оркестратор увидел сам.
