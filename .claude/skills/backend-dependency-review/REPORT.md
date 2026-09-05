# Шаблон отчёта

Справочник к последнему шагу из [SKILL.md](SKILL.md). Четыре раздела в этом порядке плюс «Вне scope»; пустой раздел остаётся с пометкой «не найдено», чтобы читатель знал, что проход был.

## Severity

| Уровень | Когда | Пример |
|---|---|---|
| **P1** | Dependency создаёт существенный architectural coupling или dependency cycle, который реально ограничивает изменения: замена технологии или правка одного модуля требует правок в нескольких несвязанных местах, либо цикл блокирует тестирование и требует `forwardRef` | пять application services знают формат ключей Redis и его pipeline-команды; `UsersModule` и `SessionsModule` инжектят друг друга через `forwardRef`, тесты поднимают оба |
| **P2** | Dependency заметно ухудшает maintainability или распространяет implementation details между важными boundaries: изменение возможно, но дороже, чем должно быть | application service возвращает Prisma-тип, и три controller зависят от его полей; `QueryFailedError` доходит до transport без преобразования |
| **P3** | Потенциальная проблема или improvement, зависящий от будущего развития системы; из кода вердикт не виден | один сервис напрямую использует SDK внешнего API; станет проблемой при втором consumer или смене версии API |

Architectural preference («я бы перевернул эту стрелку», «по Hexagonal здесь должен быть port») — не finding. Максимум P3, и только если можно назвать изменение, которое станет дороже.

## Структура

```markdown
## Dependency map

Наиболее важные стрелки — только те, что нужны, чтобы понять findings
и good dependencies. Слой в скобках там, где он не очевиден из имени.

Controller → Application Service
Application Service → SessionStore
RedisSessionStore → Redis (ioredis)
Application Service → PrismaClient
UsersModule ⇄ SessionsModule (cycle, forwardRef)

## Findings

### [P1] Короткое название проблемы

- **Dependency:** `Consumer → Dependency`, слои обеих сторон.
- **Problem:** одна фраза, что не так со связью.
- **Evidence:** `path/to/file.ts:42` — что именно там видно; при необходимости
  ещё одно–два места.
- **Why it matters:** что протекло или какое coupling возникло и почему это
  стоит денег.
- **Change scenario:** конкретное изменение, которое стало дорогим, и что
  именно придётся править.
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

- **Что это:** interface / port / adapter / фабрика / прослойка.
- **Почему лишняя:** какие критерии не выполнены (реализаций: 1,
  consumers: 1, замена не планируется, isolation не используется).
- **Стоимость сейчас:** файлы, переходы, DI-токены.
- **Упрощение:** что убрать; при каком изменении контекста вернуть.

## Вне scope

Одна строка на замеченное, что проверяют другие skills: «`OrderService`
принимает бизнес-решения и формирует HTTP-ответ — область
backend-architecture-review (responsibilities)».
```

## Пример заполненного finding

```markdown
### [P1] Формат ключей Redis известен четырём application services

- **Dependency:** `AuthService`, `SessionService`, `RefreshTokenService`,
  `LogoutService` (application) → `ioredis` `Redis` (infrastructure).
- **Problem:** каждый сервис сам собирает ключ `session:{userId}:{jti}`,
  выставляет TTL и вызывает `pipeline`.
- **Evidence:** `src/iam/auth.service.ts:44`, `src/iam/session.service.ts:21`,
  `src/iam/refresh-token.service.ts:37`, `src/iam/logout.service.ts:18` —
  четыре копии шаблона ключа; `session.service.ts:25` — `pipeline().hset().expire()`.
- **Why it matters:** формат ключа и TTL — implementation detail хранилища,
  а он размазан по четырём сценариям; Redis-типы в сигнатурах двух из них.
- **Change scenario:** смена схемы ключей (например, добавить префикс окружения)
  или переход на другое хранилище — четыре файла и все их тесты.
- **Suggested solution:** один модуль `session-store.ts` с функциями
  `save`, `find`, `revoke`; interface не нужен — реализация одна, тестовая
  подмена делается через мок модуля. Четыре сервиса вызывают функции и не
  знают о ключах.
- **Trade-offs:** ещё один файл и один переход при чтении; если позже
  появится вторая реализация (in-memory для e2e), тогда и вводить interface.
```
