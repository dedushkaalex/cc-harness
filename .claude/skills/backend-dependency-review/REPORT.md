# Шаблон отчёта

Справочник к последнему шагу из [SKILL.md](SKILL.md). Блок Scope, затем четыре раздела в этом порядке плюс «Вне scope»; пустой раздел остаётся с пометкой «не найдено», чтобы читатель знал, что проход был. Блок Scope обязателен: следующий агент в pipeline должен видеть не «этого не проверяли», а кто обязан это проверить.

## Лимиты

- Dependency map — только стрелки, нужные для понимания findings и good dependencies, не больше 25.
- Findings — не больше 10. Остальное — одним P3 «прочие стрелки того же типа» со списком файл:строка.
- «Вне scope» — не больше одной строки на категорию.

## Severity

| Уровень | Когда | Пример |
|---|---|---|
| **P1** | Dependency создаёт существенный architectural coupling или dependency cycle, который реально ограничивает изменения: замена технологии или правка одного модуля требует правок в нескольких несвязанных местах, либо цикл блокирует тестирование и требует обходов при сборке | пять application services знают формат ключей Redis и его pipeline-команды; два Layer требуют друг друга, тесты поднимают оба |
| **P2** | Dependency заметно ухудшает maintainability или распространяет implementation details между важными boundaries: изменение возможно, но дороже, чем должно быть | application-эффект возвращает Prisma-тип, и три handler'а зависят от его полей; `SqlError` в канале `E` доходит до transport без преобразования |
| **P3** | Потенциальная проблема или improvement, зависящий от будущего развития системы; из кода вердикт не виден | один сервис напрямую использует SDK внешнего API; станет проблемой при втором consumer или смене версии API |

Architectural preference («я бы перевернул эту стрелку», «по Hexagonal здесь должен быть port») — не finding. Максимум P3, и только если можно назвать изменение, которое станет дороже.

## Confidence

| Уровень | Когда |
|---|---|
| **high** | evidence в коде однозначно, change scenario подтверждён источником из Context или не требует подтверждения |
| **medium** | evidence однозначно, но вердикт («замена не планируется», «второго consumer не будет») — допущение без источника |
| **low** | evidence косвенное, или finding условный и опирается больше чем на одно допущение |

## Структура

```markdown
## Scope

Code under review: `src/iam/**` (diff PR #42 плюс один шаг по графу зависимостей).

Input: Architecture summary из отчёта `backend-architecture-review`
(или «отсутствует, модель построена с нуля»).

Context (из отчёта architecture-review, дособрано: тестовая подмена):
- один transport (HTTP) — README, docker-compose
- замена Redis: не известно, в источниках нет
- тесты подменяют зависимости тестовыми Layer — `test/layers/*.ts`

Mismatches with input model:
- `UserService`: в отчёте application, в коде импортирует `ioredis` и сам
  собирает ключи. Стрелки посчитаны по модели из отчёта.
  Owner: `backend-architecture-review`
(или «не найдено»)

Evaluated:
- dependency direction, leakage, coupling, cycles
- abstractions: needed or excessive

Not evaluated:
- responsibilities and placement of work
  Owner: `backend-architecture-review`
- SQL, performance, security, error handling, tests, domain invariants
  Owner: см. ownership matrix

## Dependency map

Те же имена компонентов, что в отчёте architecture-review. Слой в скобках
там, где он не очевиден из имени. Условные стрелки помечены.

Controller → Application Service
Application Service → SessionStore (Tag)
SessionStoreLive → Redis (ioredis)
OrderService → Prisma            условная: после переноса из [P2] architecture-review
UsersLayer ⇄ SessionsLayer       cycle

## Findings

### [P1] Короткое название проблемы

- **Dependency:** `Consumer → Dependency`, слои обеих сторон; «условная»,
  если стрелка появится после переноса кода.
- **Problem:** одна фраза, что не так со связью.
- **Evidence:** `path/to/file.ts:42` — что именно там видно; для условной
  стрелки — «`order.controller.ts:31-58` → переносится в `OrderService`».
- **Why it matters:** что протекло или какое coupling возникло и почему это
  стоит денег.
- **Change scenario:** конкретное изменение, которое стало дорогим, и что
  именно придётся править.
- **Confidence:** high / medium / low — источник из Context или допущение.
- **Suggested solution:** минимальное изменение; если abstraction — какую
  проблему решает, что стоит, почему оправдана здесь; если abstraction
  не нужна — так и написано.
- **Trade-offs:** что становится хуже или сложнее после изменения; когда
  лучше оставить как есть.

### [P2] ...

## Good dependencies

- `Consumer → Dependency` (`path/to/file.ts`) — почему стрелка хорошо
  локализована и имеет понятное направление; что стоит сохранить.

## Overengineering

### Название структуры (`path/to/file.ts:10`)

- **Что это:** Tag поверх Tag / interface / adapter / фабрика / прослойка.
- **Почему лишняя:** какие критерии не выполнены (реализаций: 1,
  consumers: 1, замена не планируется, isolation не используется).
- **Стоимость сейчас:** файлы, переходы, токены.
- **Упрощение:** что убрать; при каком изменении контекста вернуть.

## Вне scope

Одна строка на замеченное, с именем skill-владельца: «`OrderService`
принимает бизнес-решения и формирует HTTP-ответ — backend-architecture-review
(responsibilities)»; «`x.ts:12` — SQL-строка склеивается вручную — владелец
по ownership matrix».
```

## Пример заполненного finding

```markdown
### [P1] Формат ключей Redis известен четырём application services

- **Dependency:** `AuthService`, `SessionService`, `RefreshTokenService`,
  `LogoutService` (application) → `ioredis` `Redis` (infrastructure).
- **Problem:** каждый сервис сам собирает ключ `session:{userId}:{jti}`,
  выставляет TTL и вызывает `pipeline`; `Redis` стоит в канале `R` всех
  четырёх эффектов.
- **Evidence:** `src/iam/auth.service.ts:44`, `src/iam/session.service.ts:21`,
  `src/iam/refresh-token.service.ts:37`, `src/iam/logout.service.ts:18` —
  четыре копии шаблона ключа; `session.service.ts:25` — `pipeline().hset().expire()`.
- **Why it matters:** формат ключа и TTL — implementation detail хранилища,
  а он размазан по четырём сценариям; Redis-типы в сигнатурах двух из них.
- **Change scenario:** смена схемы ключей (например, добавить префикс окружения)
  или переход на другое хранилище — четыре файла и все их тесты.
- **Confidence:** high — четыре consumer видны в коде; планов замены Redis
  в источниках нет, но сценарий «сменить схему ключей» от планов не зависит.
- **Suggested solution:** один Tag `SessionStore` с операциями `save`, `find`,
  `revoke` и один Layer `SessionStoreRedis`, который единственный знает ключи
  и TTL. Четыре сервиса получают `SessionStore` в `R` и не знают о Redis.
  Тестовый Layer — по образцу `test/layers/*.ts`, как принято в проекте.
- **Trade-offs:** ещё один файл и один переход при чтении; `SessionStore`
  становится контрактом, который нужно держать стабильным.
```
