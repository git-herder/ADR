# ADR-26102201: Hub, options, listing, enrichment и enumeration для SCM-отбора

## Статус

✅ **Принято**

## Контекст

[Git Herder](../documentation/product-intent/README.md) задаёт **единую модель SCM-отбора** для массовых
операций ([repository-selection](../documentation/product-intent/capability/platform/repository-selection.md)). Переносимое ядро критериев
(`--qp`, `--qn`, `--qt`, `--qa`, `--scm-search`, `--scope`) зафиксировано в
платформенной спецификации SCM-отбора.

В [capability отбора](../documentation/product-intent/capability/platform/repository-selection.md) допускаются **платформо-специфичные**
параметры отбора (extension-флаги): часть сценариев требует флагов и ключей профиля,
имеющих смысл только на GitHub или только на GitLab. Эти параметры участвуют в том
же контуре, что и общие критерии: **разбор argv**, резолв с [профилем](../documentation/product-intent/capability/platform/profiles.md),
валидация **до** старта команды.

Первоначальная схема hub + extension + enumeration смешала в одном файле
`repository-selection-extensions.md` CLI-контракт, продуктовый эффект и HTTP list
и дозагрузки. Позже слой **mapping** объединил server-side list и REST-enrich;
[архитектура](../documentation/architecture/git-herder.archimate) (процесс
`id-649305ae`) разделила **листинг** и **обогащение**; контур приложения — view
**SCM Listing & Enrichment**
([architecture/README.md](../documentation/architecture/README.md)).
Уточнение: слой
**Enumeration** — **клиентская фильтрация** выборки (один файл на опцию), не
HTTP-транспорт. Базовый list, пагинация и схема карточки — `base-enumeration.md`;
server-side list — **listing**; дозагрузки полей карточки — **enrichment**.

## Решение

Документацию прикладного контракта SCM-отбора вести по схеме **hub + options +
base-enumeration + listing + enrichment + enumeration**; порядок фаз listing →
enrichment → enumeration — [scm/index.md](../documentation/specifications/scm/index.md):

| Слой                 | Назначение                                                                                                                                                         | Расположение (канон)                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Hub**              | Переносимые опции; синтаксис CLI; резолв с профилем; композиция `--q`*; продуктовый эффект `scope` / `scm-search`; пайплайн; общие ошибки; правила стыка с options | `documentation/specifications/platform/scm-repository-selection.md`                                                       |
| **Options**          | Таблица и семантика **только extension-флагов** для выбранного `upstream`; те же имена ключей в YAML, что и CLI; **без** REST                                      | `documentation/specifications/scm/<upstream>/repository-selection-options.md`                                             |
| **Base enumeration** | Базовый list/search, пагинация, **схема карточки** (своя per SCM)                                                                                                  | `documentation/specifications/scm/<upstream>/base-enumeration.md`                                                         |
| **Listing**          | CLI-опция → server-side list: эндпоинты, query/body, заголовки; без дозагрузок полей карточки                                                                    | `documentation/specifications/scm/<upstream>/repository-listing/index.md` и `…/listing/<parameter>.md`                    |
| **Enrichment**       | HTTP-дозагрузки полей карточки после листинга (джобы обогащения)                                                                                                   | `documentation/specifications/scm/<upstream>/repository-enrichment/index.md` и `…/enrichment/<parameter>.md`              |
| **Enumeration**      | Фильтрация **полученной** выборки на клиенте; один файл на опцию                                                                                                   | `documentation/specifications/scm/<upstream>/repository-enumeration/index.md` и `…/repository-enumeration/<parameter>.md` |

Правила:

1. **Один extension-флаг** — контракт и продуктовый эффект **только** в

   `repository-selection-options.md` своего `upstream`.
2. Hub **не** дублирует таблицы options; даёт ссылки и **общие** правила (резолв,

   композиция с `--q*`, допустимость после резолва `upstream`).
3. Options **не** дублирует переносимые опции hub, **не** описывает hub-параметры

   (в т.ч. `scope`) и **не** повторяет алгоритм резолва из [profiles](../documentation/specifications/platform/profiles.md) —
    только ссылка на hub.
4. **Server-side list** — **только** `repository-listing/<parameter>.md`;

   `listing/index.md` — реестр без тел справочников. Listing ссылается на
    **base-enumeration** для базового list/pagination. По умолчанию `qp` **/** `qn` **в
    listing не описываются** (клиентская фильтрация — enumeration). **Исключение:**
    GitHub — [listing/qp.md и qn.md][github-listing-qp-qn] и сборка `q` в
    [listing/index.md][github-listing-index].
5. **Обогащение** — **только** `repository-enrichment/<parameter>.md`; реестр —
   `enrichment/index.md`. Дозагрузки **не** дублировать в listing.
6. **Клиентская фильтрация** — **только** `repository-enumeration/<parameter>.md`;

   `repository-enumeration/index.md` — реестр. Enumeration ссылается на
    **base-enumeration** для полей карточки. Параметры, действующие только на API
    (`scope`, `scm-search`, …), **не** дублируются в enumeration.
7. После резолва `upstream` флаг или ключ профиля из options **чужого** upstream —

   **ошибка до старта** (exit `1`).
8. Условия из options после резолва участвуют в отборе с **AND** к переносимым

   `--q*` (если для конкретного флага в options не оговорено иное).
9. Extension-флаг **может** считаться критерием отбора для проверки «не пустой набор

   критериев» в hub, если так помечено в options; **модификаторы** pipeline (например
    `include-inactive` на GitLab) критерием не считаются.

Спеки команд (`cli/<command>/`) ссылаются на hub и при необходимости на options,
base-enumeration, listing, enrichment, enumeration по `upstream`; таблицы флагов не копируют.

## Аргументация

| Альтернатива                                              | Почему не выбрана                                                                 |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Все флаги в одном `scm-repository-selection.md`           | Смешение переносимого и частного; рост файла; разный темп изменений GitHub/GitLab |
| SCM-специфичные ключи только в одном monolith enumeration | Enumeration — клиентская фильтрация; argv — hub/options; REST — listing/enrichment |
| Enumeration как HTTP-транспорт                            | Транспорт и карточка — base-enumeration; фильтры — repository-enumeration/*       |
| CLI и REST в одном extension-файле                        | Смешение слоёв; hub `scope` оказывается «не в hub»; плохая изменяемость           |
| Один monolith `listing.md` на все параметры upstream      | Сложно сопровождать; правка одного флага трогает весь документ                    |
| Единый mapping (list + enrich)                           | Расходится с ArchiMate `id-649305ae`; смешивает RetryingQueue-фазы                |
| Только `--scm-opt key=value` без отдельных спек           | Allowlist всё равно нужен per upstream; options + listing дают явный SSOT         |
| Дублировать таблицы в каждой команде                      | Нарушение SSOT и [spec-first](../playbook/principles/spec-first.md)               |

## Последствия

### Позитивные

- Границы: hub, options, base-enumeration, listing, enrichment, enumeration,
  оркестрация [scm/index.md](../documentation/specifications/scm/index.md).
- Переносимое ядро остаётся одним документом; listing `scope` симметричен на GitHub и
GitLab в отдельных файлах.
- Новые upstream-флаги и новые query-параметры добавляются в свой файл без раздувания
hub.

### Негативные

- Опции, которые требуют и запросов в SCM, и отборов на клиенте, описываются в трёх
местах: listing (server-side), enrichment (дозагрузка), enumeration (клиент).
- Нужно поддерживать согласованность hub ↔ options ↔ listing ↔ enrichment ↔
enumeration (реестры, SSOT map).
- Больше файлов на один `upstream`.

### Компромиссы

- Имена `base-enumeration.md`, `repository-selection-options.md`,
`repository-listing/`, `repository-enrichment/`, `repository-enumeration/`
фиксированы для GitHub и GitLab в P0; другие SCM — по той же схеме.

## Ссылки

- [Замысел продукта Git Herder](../documentation/product-intent/README.md)
- [Единый отбор для массовых операций](../documentation/product-intent/capability/platform/repository-selection.md)
- [Именованные профили (capability)](../documentation/product-intent/capability/platform/profiles.md)
- [SCM-отбор (hub)](../documentation/specifications/platform/scm-repository-selection.md)
- [Загрузка из SCM и обогащение (spec)](../documentation/specifications/scm/index.md)
- [ADR-26102001: локальная оценка критериев SCM-отбора](ADR-26102001-scm-repository-selection-evaluation.md)
- [Spec-first](../playbook/principles/spec-first.md)

<!-- LINKS -->
[github-listing-index]: ../documentation/specifications/scm/github/repository-listing/index.md
[github-listing-qp-qn]: ../documentation/specifications/scm/github/repository-listing/qp.md
