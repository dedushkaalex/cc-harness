---
name: backend-dependency-review
description: Ревью зависимостей между компонентами backend-системы — кто от кого зависит, в какую сторону, что протекает через границы (ORM/Redis/framework types в application или domain), какая технология держит на себе слишком большую часть системы, где есть dependency cycles. Каждую dependency оценивает по стоимости конкретного изменения («какое изменение станет дорогим?»), а не по соответствию Dependency Inversion Principle; отдельно ловит лишние interface/port/adapter. Продолжение backend-architecture-review — тот отвечает за responsibilities и boundaries, этот за relationships между boundaries. Use when нужен dependency review, анализ coupling, direction of dependencies, implementation leakage, circular dependencies, вопрос «нужен ли здесь port / adapter / dependency inversion», либо dependency-проход внутри многоагентного code review. Не покрывает responsibilities, folder structure, naming, SQL, performance, security, error handling, тесты, design patterns как тему.
---

# Backend dependency review

## Зона ответственности

Ровно пять вопросов к коду:

1. **Direction** — кто от кого зависит и в ту ли сторону идёт стрелка.
2. **Leakage** — не протекают ли implementation details (типы ORM, ключи Redis, DTO транспорта, декораторы framework, ошибки инфраструктуры) через architectural boundaries.
3. **Coupling** — сколько компонентов знают о конкретной технологии и что придётся править при её замене.
4. **Cycles** — есть ли циклы между компонентами и что они реально ломают.
5. **Overengineering** — обратная проблема: interface, port, adapter или фабрика, за которые платят сейчас, а выгоды не будет.

Это продолжение `backend-architecture-review`. Тот skill отвечает за responsibilities и boundaries — *где* проходят границы. Этот skill отвечает за relationships — *как* компоненты связаны через эти границы. Смешение ответственностей, folder structure, naming, форматирование, SQL, индексы, performance, корректность auth, security, error handling, тесты и «правильность паттернов» проверяют другие skills. Заметил такое — одна строка в разделе «Вне scope», без finding.

## Ключевой вопрос

> Какое изменение станет сложнее из-за этой зависимости?

Не спрашивай «нарушает ли это Dependency Inversion Principle». Dependency не плохая только потому, что существует. `Application → Redis` — повод для оценки, не finding. Finding появляется, когда есть конкретный ответ: «смена схемы ключей Redis потребует править четыре application service». Interface, port, adapter — возможный ответ на уже найденную проблему, а не то, что ты ищешь. Не предполагай заранее Clean, Hexagonal или другую методологию: восстанавливай связи из кода.

## Порядок работы

Вопросы для каждого шага — в [STEPS.md](STEPS.md), оценка abstraction в обе стороны — в [INVERSION.md](INVERSION.md). Шаг закрыт, когда выполнен его критерий.

- [ ] **Step 1 — Reconstruct dependency graph.** Восстанови стрелки `A → B` из импортов, конструкторов, DI-регистраций и глобальных объектов. Критерий: список стрелок, у каждой указан слой обеих сторон (transport / application / domain / infrastructure / persistence / external service) и файл, где стрелка видна.
- [ ] **Step 2 — Direction.** Для каждой подозрительной стрелки определи consumer, dependency, причину существования и природу цели: business concept или implementation detail. Критерий: у каждой подозрительной стрелки все четыре ответа записаны.
- [ ] **Step 3 — Leakage.** Найди места, где деталь одного компонента или слоя видна в другом. Критерий: для каждой протечки названы что протекло, через какую границу, какое изменение стало дороже, файл и строка.
- [ ] **Step 4 — Coupling.** Для каждой существенной стрелки оцени шесть параметров из STEPS.md. Критерий: у каждой стрелки вердикт «дорого / приемлемо в этом контексте» с одной фразой почему.
- [ ] **Step 5 — Dependency inversion.** Для стрелок с вердиктом «дорого» ответь на пять вопросов из INVERSION.md, прежде чем предлагать interface или port. Критерий: у каждой рекомендации названы решаемая проблема, локализуемые изменения, добавляемая complexity, существующая похожая abstraction, нужна ли граница вообще.
- [ ] **Step 6 — Cycles.** Найди циклы `A → B → A` на уровне модулей и компонентов. Критерий: у каждого цикла есть оценка влияния по пяти признакам из STEPS.md; цикл без реального влияния — не finding.
- [ ] **Step 7 — Stable vs volatile.** Для каждого finding подтверди, что изменчивая сторона — именно dependency, а не consumer. Критерий: finding, где цель стабильна (standard library, устоявшийся внутренний контракт), понижен или убран с объяснением.
- [ ] **Overengineering pass.** Пройди все interface, port, adapter, фабрики и слои-прослойки по критериям из INVERSION.md. Критерий: у каждой abstraction вердикт «оправдана / лишняя» с причиной.
- [ ] **Отчёт** по шаблону из [REPORT.md](REPORT.md).

## Правила для findings

- Каждый finding опирается на конкретный код: путь к файлу и строка. Без evidence finding не существует.
- У каждого finding есть **change scenario** — конкретное изменение, которое стало дорогим: «заменить Redis на Memcached», «добавить второй transport», «сменить ORM». Без сценария это architectural preference, а не finding.
- Severity — по стоимости изменений. P1 — coupling или cycle, который реально ограничивает изменения. P2 — заметно дороже maintainability или implementation details протекают между важными boundaries. P3 — зависит от будущего развития системы. Подробнее — в REPORT.md.
- Не предлагай dependency inversion ради dependency inversion. Если после оценки abstraction не нужна, так и напиши и назови, при каком изменении контекста вердикт поменяется.
- Хорошие зависимости фиксируются так же обязательно, как проблемы: следующий ревьюер должен знать, что трогать не нужно.
