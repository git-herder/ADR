# ADR-26102201: Hub, options, mapping и enumeration для SCM-отбора

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
`repository-selection-extensions.md` CLI-контракт, продуктовый эффект и маппинг на
REST (в том числе для переносимого `scope` из hub). Это нарушает разделение слоёв и
[изменяемость документации](../glossary/terms/documentation-modifiability.md). Уточнение: слой
**Enumeration** — **клиентская фильтрация** выборки (один файл на опцию), не
HTTP-транспорт. Базовый list, пагинация и схема карточки — `base-enumeration.md`;
REST от CLI — **Mapping**.

## Решение

Документацию прикладного контракта SCM-отбора вести по схеме **hub + options +
base-enumeration + mapping + enumeration**:

| Слой                 | Назначение                                                                                                                                                         | Расположение (канон)                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Hub**              | Переносимые опции; синтаксис CLI; резолв с профилем; композиция `--q`*; продуктовый эффект `scope` / `scm-search`; пайплайн; общие ошибки; правила стыка с options | `documentation/specifications/platform/scm-repository-selection.md`                                                       |
| **Options**          | Таблица и семантика **только extension-флагов** для выбранного `upstream`; те же имена ключей в YAML, что и CLI; **без** REST                                      | `documentation/specifications/scm/<upstream>/repository-selection-options.md`                                             |
| **Base enumeration** | Базовый list/search, пагинация, **схема карточки** (своя per SCM)                                                                                                  | `documentation/specifications/scm/<upstream>/base-enumeration.md`                                                         |
| **Mapping**          | CLI-опция → эндпоинты, query/body, заголовки; комбинации на API; все, что связано с запросами в SCM.                                                               | `documentation/specifications/scm/<upstream>/repository-selection-mapping/index.md` и `…/mapping/<parameter>.md`          |
| **Enumeration**      | Фильтрация **полученной** выборки на клиенте; один файл на опцию                                                                                                   | `documentation/specifications/scm/<upstream>/repository-enumeration/index.md` и `…/repository-enumeration/<parameter>.md` |

Правила:

1. **Один extension-флаг** — контракт и продуктовый эффект **только** в

   `repository-selection-options.md` своего `upstream`.
2. Hub **не** дублирует таблицы options; даёт ссылки и **общие** правила (резолв,

   композиция с `--q*`, допустимость после резолва `upstream`).
3. Options **не** дублирует переносимые опции hub, **не** описывает hub-параметры

   (в т.ч. `scope`) и **не** повторяет алгоритм резолва из [profiles](../documentation/specifications/platform/profiles.md) —
    только ссылка на hub.
4. **Маппинг на REST** — **только** `repository-selection-mapping/<parameter>.md`;

   `mapping/index.md` — реестр без тел справочников. Mapping ссылается на
    **base-enumeration** для базового list/pagination. По умолчанию `qp` **/** `qn` **в
    mapping не описываются** (клиентская фильтрация — enumeration). **Исключение:**
    GitHub — [mapping/qp.md и qn.md][github-mapping-qp-qn] и сборка `q` в
    [mapping/index.md][github-mapping-index].
5. **Клиентская фильтрация** — **только** `repository-enumeration/<parameter>.md`;

   `repository-enumeration/index.md` — реестр. Enumeration ссылается на
    **base-enumeration** для полей карточки. Параметры, действующие только на API
    (`scope`, `scm-search`, …), **не** дублируются в enumeration.
6. После резолва `upstream` флаг или ключ профиля из options **чужого** upstream —

   **ошибка до старта** (exit `1`).
7. Условия из options после резолва участвуют в отборе с **AND** к переносимым

   `--q*` (если для конкретного флага в options не оговорено иное).
8. Extension-флаг **может** считаться критерием отбора для проверки «не пустой набор

   критериев» в hub, если так помечено в options; **модификаторы** pipeline (например
    `include-inactive` на GitLab) критерием не считаются.

Спеки команд (`cli/<command>/`) ссылаются на hub и при необходимости на options,
base-enumeration, mapping, enumeration по `upstream`; таблицы флагов не копируют.

## Аргументация

| Альтернатива                                              | Почему не выбрана                                                                 |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Все флаги в одном `scm-repository-selection.md`           | Смешение переносимого и частного; рост файла; разный темп изменений GitHub/GitLab |
| SCM-специфичные ключи только в одном monolith enumeration | Enumeration — клиентская фильтрация; argv — hub/options; REST — mapping           |
| Enumeration как HTTP-транспорт                            | Транспорт и карточка — base-enumeration; фильтры — repository-enumeration/*       |
| CLI и REST в одном extension-файле                        | Смешение слоёв; hub `scope` оказывается «не в hub»; плохая изменяемость           |
| Один monolith `mapping.md` на все параметры upstream      | Сложно сопровождать; правка одного флага трогает весь документ                    |
| Только `--scm-opt key=value` без отдельных спек           | Allowlist всё равно нужен per upstream; options + mapping дают явный SSOT         |
| Дублировать таблицы в каждой команде                      | Нарушение SSOT и [spec-first](../playbook/principles/spec-first.md)               |

## Последствия

### Позитивные

- Пять границ: hub, options, base-enumeration, mapping (CLI→REST), enumeration
(клиентская фильтрация).
- Переносимое ядро остаётся одним документом; маппинг `scope` симметричен на GitHub и
GitLab в отдельных файлах.
- Новые upstream-флаги и новые query-параметры добавляются в свой файл без раздувания
hub.

### Негативные

- Опции, которые требуют и запросов в SCM, и отборов на клиенте, описываются в двух местах: mapping - часть про запросы, enumeration - часть про отборы.
- Нужно поддерживать согласованность hub ↔ options ↔ mapping ↔ enumeration (реестры,
SSOT map).
- Больше файлов на один `upstream`.

### Компромиссы

- Имена `base-enumeration.md`, `repository-selection-options.md`,
`repository-selection-mapping/`, `repository-enumeration/` фиксированы для GitHub и
GitLab в P0; другие SCM — по той же схеме.

## Ссылки

- [Замысел продукта Git Herder](../documentation/product-intent/README.md)
- [Единый отбор для массовых операций](../documentation/product-intent/capability/platform/repository-selection.md)
- [Именованные профили (capability)](../documentation/product-intent/capability/platform/profiles.md)
- [SCM-отбор (hub)](../documentation/specifications/platform/scm-repository-selection.md)
- [ADR-26102001: локальная оценка критериев SCM-отбора](ADR-26102001-scm-repository-selection-evaluation.md)
- [Spec-first](../playbook/principles/spec-first.md)

<!-- LINKS -->
[github-mapping-index]: ../documentation/specifications/scm/github/repository-selection-mapping/index.md
[github-mapping-qp-qn]: ../documentation/specifications/scm/github/repository-selection-mapping/qp.md
