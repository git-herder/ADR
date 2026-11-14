# ADR-26110801: Компоновка кода приложения по доменам

## Статус

✅ **Принято**

## Контекст

Утверждены первые прикладные спецификации (команды `profile` и `report`), в
`application/` появляется первый код. Целевая архитектура (ArchiMate) задаёт
набор application components: Bootstrap, Invocation, ProfileStore, Logger,
ConsoleUI, RepositoryCard, CommonScmSelection, ScmHttpTransport,
ScmClientFactory; интерфейсы ScmListingBuilder, ScmListing, ScmEnrichmentBuilder,
ScmEnrich; platform-компоненты GithubScmListing, GithubScmEnrichment,
GitlabScmListing, GitlabScmEnrichment; RetryingQueue, GitCliRunner,
RepositoriesReportBuilder и обработчики команд. Навигация по view —
[architecture/README.md](../documentation/architecture/README.md).

[ADR-26101701][adr-stack] фиксирует стек (Go, модуль
`github.com/Snowbridge/git-herder`) и **явно не задаёт** структуру каталогов.
Без принятого соглашения о компоновке каждый автор раскладывал бы код по-своему,
и границы пакетов перестали бы совпадать с границами компонентов архитектуры.

## Решение

Код компонуем по доменам («гибрид по доменам»): команды — отдельными пакетами
под дерево cobra, SCM-контур — поддеревом, сквозные механизмы — по одному
пакету на механизм. Всё, кроме точки входа, — в `internal/` (утилита не
предоставляет библиотечный API, каталог `pkg/` не вводим).

### Структура репозитория `application/`

```text
application/
├── go.mod                         # module github.com/Snowbridge/git-herder
├── cmd/
│   └── githerd/
│       └── main.go                # только вызов bootstrap.Run()
├── internal/
│   ├── bootstrap/
│   │   └── bootstrap.go           # сборка компонентов, запуск дерева команд
│   ├── cli/
│   │   ├── root.go                # корень дерева cobra, глобальные флаги
│   │   ├── resolve.go             # резолв argv + профиль
│   │   ├── profile/
│   │   │   └── profile.go         # команда profile и её flow
│   │   └── report/
│   │       └── report.go          # команда report и её flow
│   ├── profilestore/
│   │   ├── store.go               # чтение/запись каталога профилей (viper)
│   │   ├── model.go               # YAML-модель профиля
│   │   └── validate.go            # ошибки настроек при загрузке
│   ├── report/
│   │   └── builder.go             # RepositoriesReportBuilder
│   ├── diag/
│   │   ├── logging/
│   │   │   └── logging.go         # Logger: slog, NDJSON-файл, TSV-stderr
│   │   └── console/
│   │       └── console.go         # ConsoleUI: обёртка pterm
│   ├── retrying/
│   │   └── queue.go               # RetryingQueue
│   ├── tstamp/
│   │   └── tstamp.go              # fs-timestamp / data-timestamp
│   ├── scm/
│   │   ├── card/
│   │   │   └── card.go            # RepositoryCard: типы Card и Set
│   │   ├── selection/
│   │   │   ├── criteria.go        # переносимая модель критериев отбора
│   │   │   └── selection.go       # CommonScmSelection: пайплайн отбора
│   │   ├── http/
│   │   │   └── transport.go       # ScmHttpTransport поверх net/http
│   │   ├── listing.go             # ScmListing, ScmListingBuilder (Listing, ListingBuilder)
│   │   ├── factory/
│   │   │   └── factory.go         # ScmClientFactory
│   │   ├── github/
│   │   │   ├── listing.go         # GithubScmListing
│   │   │   └── enrichment.go      # GithubScmEnrichment
│   │   └── gitlab/
│   │       ├── listing.go         # GitlabScmListing
│   │       └── enrichment.go      # GitlabScmEnrichment
│   ├── gitexec/
│   │   └── runner.go              # GitCliRunner (первое использование — P1+)
│   └── workspace/
│       ├── resolve.go             # GitWorkingCopiesResolver
│       ├── walk.go                # обход каталога --workspace
│       └── match.go               # сопоставление --qp и --qn
```

### Соответствие компонентов архитектуры пакетам

| Компонент (ArchiMate) | Пакет |
| --------------------- | ----- |
| Bootstrap | `internal/bootstrap` |
| Invocation | `internal/cli` (`root.go`, `resolve.go`) |
| Profile command, Report command | `internal/cli/profile`, `internal/cli/report` |
| ProfileStore | `internal/profilestore` |
| RepositoriesReportBuilder | `internal/report` |
| Logger | `internal/diag/logging` |
| ConsoleUI | `internal/diag/console` |
| RetryingQueue | `internal/retrying` |
| Профили меток времени ([ADR-26101704][adr-time]) | `internal/tstamp` |
| RepositoryCard | `internal/scm/card` |
| CommonScmSelection | `internal/scm/selection` |
| ScmHttpTransport | `internal/scm/http` |
| ScmListing, ScmListingBuilder, ScmEnrichmentBuilder, ScmEnrich (интерфейсы) | `internal/scm` (`listing.go`, enrich — по мере реализации) |
| ScmClientFactory | `internal/scm/factory` |
| GithubScmListing, GithubScmEnrichment | `internal/scm/github` |
| GitlabScmListing, GitlabScmEnrichment | `internal/scm/gitlab` |
| GitCliRunner | `internal/gitexec` |
| GitWorkingCopiesResolver | `internal/workspace` |

Эта таблица — канон трассировки «компонент → пакет». Имена пакетов не обязаны
повторять имена компонентов; обязано сохраняться соответствие по этой таблице.

### Правила компоновки

- Имена пакетов — короткие, строчными, без подчёркиваний и CamelCase; имя типа
  внутри пакета не дублирует имя пакета.
- Одна команда CLI — один пакет под `internal/cli/<имя>`.
- Платформенный адаптер — один пакет `internal/scm/<upstream>`; обе реализации
  Builder'ов живут в нём.
- Один сквозной механизм — один пакет; общие для механизмов части выделяются
  в подпакеты (как `diag/logging` и `diag/console`).

### Смежные решения, зафиксированные вместе с компоновкой

1. **SCM-адаптер.** ScmClientFactory по `upstream` настраивает platform-Builder
   и возвращает готовые клиенты **ScmListing** / **ScmEnrich** (`Build()`).
   Builder'ы — внутренняя фаза сборки; шаги листинга и обогащения в пайплайне
   вызывают runtime-интерфейсы, а не Builder (см. view **SCM Listing &
   Enrichment** в [architecture/README.md](../documentation/architecture/README.md)).
   Реализации Builder'ов и HTTP — в `internal/scm/github` и `internal/scm/gitlab`.
2. **RepositoryCard.** Пакет `internal/scm/card`: тип `Card` (нормализованная
   запись по платформенной спецификации) и тип `Set` — хранилище массива
   карточек с CRUD-API и API фильтрации предикатами. Переносимая модель
   критериев — в `internal/scm/selection`; платформо-специфичные матчеры —
   в пакетах адаптеров ([ADR-26102001][adr-local-eval]).
3. **RetryingQueue и report.** Команды не используют RetryingQueue напрямую.
   Очередь работает внутри пайплайна CommonScmSelection: отдельный прогон
   по объектам «запрос страницы» на шаге перечисления и прогон по карточкам
   на шаге обогащения ([ADR-26110301][adr-retry]).
4. **Валидация настроек профиля** (секреты только в `root`, битый указатель
   `default`, синтаксис YAML) — зона ProfileStore: ошибки настроек выявляются
   при загрузке профиля. Invocation только резолвит значения.
5. **Формирование отчёта.** RepositoriesReportBuilder — единственная точка
   формирования отчёта: Report command вызывает его и через ConsoleUI выводит
   результат — TSV-строки на стандартный вывод либо путь к записанному
   JSON-файлу. Машиночитаемый вывод на stdout пишется мимо стилизации pterm.

## Аргументация

| Альтернатива | Почему не выбрана |
| ------------ | ----------------- |
| Строго 1:1: пакет на каждый компонент (`invocation`, `profilestore`, `consoleui`, …) | Неидиоматичные имена, около двадцати мелких пакетов; переименование компонента в модели вынуждает переименовывать пакеты; не всякий компонент — это пакет по смыслу (RepositoryCard) |
| Слоистая: `internal/cli` + `internal/platform` + `internal/scm` | Минимум пакетов, но `platform` собирает в кучу логирование, терминал, профили, очередь и метки времени — разный темп изменений и разные причины для правок |
| Гибрид по доменам | Команды, SCM-контур и сквозные механизмы разведены по причинам изменения; имена пакетов идиоматичны; трассировка к архитектуре сохраняется таблицей |

## Последствия

### Позитивные

- Границы пакетов совпадают с доменами, а не с библиотеками: правка отбора
  GitLab трогает только `internal/scm/gitlab`.
- Идиоматичные имена пакетов снижают порог входа для Go-разработчиков.
- Соответствие «компонент → пакет» зафиксировано в одном месте.

### Негативные

- Трассировка к ArchiMate — не самоочевидна по именам пакетов; таблицу в этом
  ADR нужно поддерживать при добавлении компонентов.
- Каждый новый компонент требует решения о своём пакете со ссылкой на этот ADR.

### Компромиссы

- Расположение тестов, порог покрытия и языковые правила кода — [ADR-26110802](ADR-26110802-go-code-quality.md).
- `pkg/` не вводим, пока нет внешних потребителей библиотечного API; при
  появлении — отдельное решение.

<!-- LINKS -->
[adr-stack]: ADR-26101701-go-application-stack.md
[adr-time]: ADR-26101704-timestamp-formats.md
[adr-local-eval]: ADR-26102001-scm-repository-selection-evaluation.md
[adr-retry]: ADR-26110301-retrying-queue-policy.md
