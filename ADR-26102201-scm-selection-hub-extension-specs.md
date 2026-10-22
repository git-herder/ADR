# ADR-26102201: Hub и extension-спеки для SCM-отбора

## Статус

✅ **Принято**

## Контекст

[Git Herder][product-intent-readme] задаёт **единую модель SCM-отбора** для массовых
операций ([repository-selection][repository-selection]). Переносимое ядро критериев
(`--qp`, `--qn`, `--qt`, `--qa`, `--scm-search`, `--scope`) зафиксировано в
платформенной спецификации SCM-отбора.

В [capability отбора][repository-selection] допускаются **платформо-специфичные**
параметры отбора как исключение: часть сценариев требует флагов и ключей профиля,
имеющих смысл только на GitHub или только на GitLab. Эти параметры участвуют в том
же контуре, что и общие критерии: **разбор argv**, резолв с [профилем][profiles-capability],
валидация **до** старта команды.

Альтернатива — описать все флаги в одном платформенном файле или смешать CLI-контракт
с [перечислением репозиториев через API][enumeration-layer] (пагинация, эндпоинты,
маппинг ответа). Перечисление выполняется **после** сборки критериев; смешение слоёв
усложняет сопровождение и нарушает разделение «контракт пользователя» и «интеграция
с API».

## Решение

Документацию прикладного контракта SCM-отбора вести по схеме **hub + extension**:

| Слой | Назначение | Расположение (канон) |
| ---- | ---------- | -------------------- |
| **Hub** | Переносимые опции отбора; синтаксис CLI; резолв с профилем; композиция AND/OR для `--q*`; `scm-search` и `scope`; пайплайн отбора; общие ошибки до старта; правила стыка с расширениями | `documentation/specifications/platform/scm-repository-selection.md` |
| **Extension** | Таблица опций и семантика **только** для выбранного `upstream` (`github` / `gitlab`); те же имена ключей в YAML-профиле, что и длинные флаги CLI | `documentation/specifications/scm/<upstream>/repository-selection-extensions.md` |
| **Enumeration** | Перечисление и обогащение карточки репозитория по HTTP; **без** таблиц CLI-флагов | `documentation/specifications/scm/<upstream>/repository-enumeration.md` |

Правила:

1. **Один флаг extension** описывается **только** в extension-файле своего `upstream`.
2. Hub **не** дублирует таблицы extension; даёт ссылки и **общие** правила (резолв,
   композиция с `--q*`, допустимость после резолва `upstream`).
3. Extension **не** дублирует таблицу переносимых опций и **не** повторяет алгоритм
   резолва из [profiles][profiles-spec] — только ссылка на hub.
4. После резолва `upstream` флаг или ключ профиля из extension **чужого** upstream —
   **ошибка до старта** (exit `1`), не предупреждение с игнорированием.
5. Условия из extension после резолва участвуют в отборе с **AND** к переносимым
   `--q*` (если для конкретного флага в extension не оговорено иное).
6. Флаг extension **может** считаться критерием отбора для проверки «не пустой набор
   критериев» в hub, если так помечено в extension-спеке; **модификаторы** pipeline
   (например `include-inactive` на GitLab) критерием не считаются.

Спеки команд (`cli/<command>/`) ссылаются на hub и при необходимости на extension по
`upstream`; таблицы флагов не копируют.

## Аргументация

| Альтернатива | Почему не выбрана |
| ------------ | ----------------- |
| Все флаги в одном `scm-repository-selection.md` | Смешение переносимого и частного; рост файла; разный темп изменений GitHub/GitLab |
| SCM-специфичные ключи только в `repository-enumeration.md` | Enumeration — пост-parse и REST; не SSOT для argv и профиля |
| Только `--scm-opt key=value` без отдельных спек | Allowlist всё равно нужен per upstream; отдельные extension-файлы дают тот же SSOT с явными флагами |
| Дублировать extension-таблицы в каждой команде | Нарушение SSOT и [spec-first][spec-first] |

## Последствия

### Позитивные

- Чёткая граница: CLI-контракт (hub + extension) отдельно от интеграции API (enumeration).
- Переносимое ядро остаётся одним документом для всех массовых SCM-команд.
- Новые upstream-флаги добавляются в свой extension без раздувания hub.

### Негативные

- Нужно поддерживать согласованность hub ↔ extension (стык, ошибки, SSOT map).
- До появления extension-файла ссылка из hub указывает на зарезервированный путь.

### Компромиссы

- Имена путей `repository-selection-extensions.md` фиксированы для GitHub и GitLab в
  P0; другие SCM — по той же схеме при появлении.

## Ссылки

- [Замысел продукта Git Herder][product-intent-readme]
- [Единый отбор для массовых операций][repository-selection]
- [Именованные профили (capability)][profiles-capability]
- [ADR-26102001: локальная оценка критериев SCM-отбора][adr-local-eval]
- [Spec-first][spec-first]

<!-- LINKS -->
[product-intent-readme]: ../documentation/product-intent/README.md
[repository-selection]: ../documentation/product-intent/capability/platform/repository-selection.md
[profiles-capability]: ../documentation/product-intent/capability/platform/profiles.md
[profiles-spec]: ../documentation/specifications/platform/profiles.md
[enumeration-layer]: ../documentation/specifications/scm/github/repository-enumeration.md
[adr-local-eval]: ADR-26102001-scm-repository-selection-evaluation.md
[spec-first]: ../playbook/principles/spec-first.md
