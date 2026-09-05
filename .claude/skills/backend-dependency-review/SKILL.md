---
name: backend-dependency-review
description: Review relationships between backend components — находит проблемные dependency direction, leakage типов и ошибок через границы, coupling с конкретной технологией, dependency cycles и лишние interface/port/adapter. Каждую dependency оценивает по стоимости конкретного изменения, а не по соответствию Dependency Inversion Principle. Второй шаг pipeline после backend-architecture-review, принимает его Architecture summary на вход. Use when нужен dependency review, анализ coupling или circular dependencies, вопрос «что этот компонент знает о других» или «нужен ли здесь port / adapter / dependency inversion», либо dependency-проход внутри многоагентного code review. Do not evaluate whether work is located in the correct component — это backend-architecture-review; не покрывает SQL, performance, security, error handling, тесты, naming, domain invariants.
---

# Backend dependency review

## Primary question

> Что этот компонент знает о других компонентах, и какое изменение из-за этого станет дорогим?

**Finding принадлежит этому skill**, когда проблема вызвана тем, что компонент знает о другом компоненте или implementation detail, импортирует его, выставляет наружу его типы или зависит от него дорогим способом: четыре сервиса знают формат ключей Redis, application service возвращает Prisma-тип, два модуля инжектят друг друга через `forwardRef`.

**Исправления** в этом skill меняют импорты, типы, направление стрелки, abstraction или dependency boundary: передать данные параметром, выделить модуль, который один знает технологию, ввести или убрать interface, перевернуть стрелку. Проверка: если исправление — перенести код в другой компонент, finding не отсюда.

## Do NOT evaluate

Владелец всех пунктов ниже — `backend-architecture-review`:

- лежит ли в компоненте не тот вид работы;
- принадлежит ли ответственность другому компоненту;
- не слишком ли много видов работы в одном компоненте;
- слой компонента: карта из отчёта architecture-review не переопределяется, этот skill работает со стрелками от уже названных компонентов.

SQL, индексы, database queries, performance, security, auth, error handling, тесты, naming, folder structure, domain invariants и корректность бизнес-правил — другие skills.

Примеры на границе:

- application service возвращает тип `Prisma.UserGetPayload` — **здесь**: код на месте, чужой только тип.
- controller, внутри которого написана транзакция и запись в две таблицы — **не здесь**: это код persistence в transport, finding architecture-review. Здесь оценивается стрелка, которая останется после переноса кода.
- два сервиса инжектят друг друга через `forwardRef` — **здесь**: цикл.

Всё замеченное вне scope — одной строкой в разделе «Вне scope» с именем skill-владельца.

## Ключевой вопрос для severity

> Какое изменение станет сложнее из-за этой зависимости?

Не спрашивай «нарушает ли это Dependency Inversion Principle». Dependency не плохая только потому, что существует. `Application → Redis` — повод для оценки, не finding. Finding появляется, когда есть конкретный ответ: «смена схемы ключей Redis потребует править четыре application service». Interface, port, adapter — возможный ответ на уже найденную проблему, а не то, что ты ищешь. Не предполагай заранее Clean, Hexagonal или другую методологию: восстанавливай связи из кода.

## Порядок работы

Вопросы для каждого шага — в [STEPS.md](STEPS.md), оценка abstraction в обе стороны — в [INVERSION.md](INVERSION.md). Шаг закрыт, когда выполнен его критерий.

- [ ] **Step 1 — Reconstruct dependency graph.** Если есть отчёт `backend-architecture-review`, его Architecture summary — начальная модель компонентов. Проверь её по коду и дополни только теми компонентами и стрелками, которые нужны для dependency analysis; предыдущий агент мог что-то пропустить. Стрелки, которые исчезнут после переноса кода из findings того отчёта, не оценивай — оценивай стрелку, которая появится после переноса. Без отчёта восстанови стрелки `A → B` из импортов, конструкторов, DI-регистраций и глобальных объектов. Критерий: список стрелок, у каждой указан слой обеих сторон (transport / application / domain / infrastructure / persistence / external service) и файл, где стрелка видна.
- [ ] **Step 2 — Direction.** Для каждой подозрительной стрелки определи consumer, dependency, причину существования и природу цели: business concept или implementation detail. Критерий: у каждой подозрительной стрелки все четыре ответа записаны.
- [ ] **Step 3 — Leakage.** Найди места, где деталь одного компонента или слоя видна в другом. Критерий: для каждой протечки названы что протекло, через какую границу, какое изменение стало дороже, файл и строка.
- [ ] **Step 4 — Coupling.** Для каждой существенной стрелки оцени шесть параметров из STEPS.md. Критерий: у каждой стрелки вердикт «дорого / приемлемо в этом контексте» с одной фразой почему.
- [ ] **Step 5 — Dependency inversion.** Для стрелок с вердиктом «дорого» ответь на пять вопросов из INVERSION.md, прежде чем предлагать interface или port. Критерий: у каждой рекомендации названы решаемая проблема, локализуемые изменения, добавляемая complexity, существующая похожая abstraction, нужна ли граница вообще.
- [ ] **Step 6 — Cycles.** Найди циклы `A → B → A` на уровне модулей и компонентов. Критерий: у каждого цикла есть оценка влияния по пяти признакам из STEPS.md; цикл без реального влияния — не finding.
- [ ] **Step 7 — Stable vs volatile.** Для каждого finding подтверди, что изменчивая сторона — именно dependency, а не consumer. Критерий: finding, где цель стабильна (standard library, устоявшийся внутренний контракт), понижен или убран с объяснением.
- [ ] **Overengineering pass.** Пройди все interface, port, adapter, фабрики и слои-прослойки по критериям из INVERSION.md. Критерий: у каждой abstraction вердикт «оправдана / лишняя» с причиной.
- [ ] **Отчёт** по шаблону из [REPORT.md](REPORT.md), с блоком Scope.

## Правила для findings

- Каждый finding опирается на конкретный код: путь к файлу и строка. Без evidence finding не существует.
- У каждого finding есть **change scenario** — конкретное изменение, которое стало дорогим: «заменить Redis на Memcached», «добавить второй transport», «сменить ORM». Без сценария это architectural preference, а не finding.
- Severity — по стоимости изменений. P1 — coupling или cycle, который реально ограничивает изменения. P2 — заметно дороже maintainability или implementation details протекают между важными boundaries. P3 — зависит от будущего развития системы. Подробнее — в REPORT.md.
- Не предлагай dependency inversion ради dependency inversion. Если после оценки abstraction не нужна, так и напиши и назови, при каком изменении контекста вердикт поменяется.
- Рекомендация не выходит за границу skill: «перенести код в другой компонент» — не отсюда, это уходит в «Вне scope».
- Хорошие зависимости фиксируются так же обязательно, как проблемы: следующий ревьюер должен знать, что трогать не нужно.
