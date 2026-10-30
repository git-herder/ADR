# ADR-26102001: SCM-отбор — локальная оценка критериев

## Статус

✅ **Принято**

## Контекст

[Git Herder][product-intent-readme] даёт [единую модель SCM-отбора][repository-selection]
для массовых операций: отчёты о репозиториях, клонирование с платформы, правки
метаданных и другие команды с одним и тем же смыслом «какой набор репозиториев
затронуть». Пользователь задаёт критерии в терминах продукта; утилита скрывает
различия API [SCM-платформ][scm-platform].

Для сопоставления **имени репозитория** и **namespace** (owner, group, path —
как классы, не как поля конкретного API) возникает вопрос: выполнять отбор на
стороне SCM через нативный поиск или в процессе Git Herder.

На практике:

- GitHub, GitLab и другие SCM **несовместимо** моделируют поиск и фильтрацию
  репозиториев; нет переносимого способа перевести один поисковый запрос Git Herder
  в эквивалентные вызовы каждого API.
- Не все SCM гарантируют **регулярные выражения** (или эквивалент) в серверном
  поиске, тогда как продукт обещает единообразную семантику шаблонов отбора.

Перечисление репозиториев через API идёт через SCM-адаптеры поверх общей HTTP-обёртки
([ADR-26101703][adr-http]). Какой **scope** перечисления допустим на конкретном хосте
(в том числе публичные `github.com` / `gitlab.com`) — [ADR-26102002][adr-saas-guard].
Сетевые повторы и очередь массовых шагов — в capability [отказоустойчивого выполнения
по сети][resilient-network], не здесь.

## Решение

Для критериев SCM-отбора, которые выражают **сопоставление имени репозитория и
namespace** с шаблонами пользователя (включая регэксп, где продукт это обещает):

1. SCM-адаптер **перечисляет** репозитории, **доступные текущему пользователю** в
   контексте [профиля][profiles] и **выбранного scope перечисления** (с пагинацией до
   полноты набора в рамках этого scope).
2. Адаптер возвращает **нормализованную** запись репозитория; схема полей карточки
   для каждого `upstream` — SSOT в `scm/<upstream>/base-enumeration.md` (например
   [GitHub][base-enumeration-github]).
3. Git Herder **применяет критерии отбора локально** к полученному набору в процессе
   CLI. Per-SCM алгоритмы фильтрации по `--qp`, `--qn`, `--qt`, `--qa` — в
   `scm/<upstream>/repository-enumeration/<parameter>.md` (реестр —
   `repository-enumeration/index.md`). **Не** строим общий слой трансляции «критерий
   → нативный search API SCM» для этих полей.

**Исключение (`upstream=github`):** критерии `--qp` и `--qn` транслируются в query
`q` базового `GET /search/repositories` ([mapping/index § сборка q][github-mapping-q]);
локальная фильтрация `qp`/`qn` **не** выполняется — отдельных файлов в
[repository-enumeration/][github-enum-index] для них нет. Семантика argv на GitHub —
литералы Search, не регэкп hub ([base-enumeration § ограничения API][github-base-limits]). Осознанный trade-off:
переносимость шаблонов GitLab/GitHub vs раннее сужение через GitHub Search API.

Под «локально» имеется в виду **процесс Git Herder**, не браузер и не UI SCM.

**Граница этого решения:** класс критериев «поля карточки репозитория» (имя,
namespace). Аналитические критерии (наличие ветки, расхождение веток и т.п. из
[отбора][repository-selection] и отчётов) **не** предполагают тот же одношаговый
контур «один список → один проход фильтра»; для них могут понадобиться дополнительные
вызовы API по каждому репозиторию — в прикладной спецификации.

Фильтрация по **метаданным** (`qt`, `qa`) согласована с hub: REST-дозагрузка —
mapping; проверка на карточке — repository-enumeration per [ADR-26102201][adr-hub-layout].

## Аргументация

| Альтернатива | Почему не выбрана |
| ------------ | ----------------- |
| Унифицированный «поисковый DSL → нативный query» per SCM | Несовместимые модели поиска; высокая стоимость поддержки; ложная переносимость между SCM |
| Проброс только server-side search GitHub/GitLab в параметры CLI | Разная семантика; регэксп не везде; ломает сценарий «одинаковый отбор» из capability |

## Последствия

### Позитивные

- Переносимая семантика имени и namespace между поддерживаемыми SCM в P0.
- Единый движок сопоставления шаблонов (в том числе регэксп) в Git Herder.

### Негативные

- Больше HTTP-запросов при перечислении (пагинация); рост времени и памяти на
  больших организациях.
- Зависимость от rate limits SCM — перечисление должно участвовать в общей политике
  массовых сетевых шагов ([resilient-network][resilient-network]).

### Компромиссы

- Сознательно не используем server-side search там, где он мог бы сузить выборку
  раньше перечисления.
- Кэш списка репозиториев, инкрементальное обновление и прочие оптимизации — в
  спецификации и коде, не в этом ADR.

## Ссылки

- [Замысел продукта Git Herder][product-intent-readme]
- [Единый отбор для массовых операций][repository-selection]
- [Именованные профили][profiles]
- [SCM-платформа (термин)][scm-platform]
- [Отказоустойчивое выполнение по сети][resilient-network]
- [ADR-26101703: HTTP-клиент для SCM API][adr-http]
- [ADR-26102201: раскладка hub / options / mapping / enumeration][adr-hub-layout]
- [ADR-26102002: affiliation-scope на публичных SaaS][adr-saas-guard]

<!-- LINKS -->
[base-enumeration-github]: ../documentation/specifications/scm/github/base-enumeration.md
[adr-hub-layout]: ADR-26102201-scm-selection-hub-extension-specs.md
[product-intent-readme]: ../documentation/product-intent/README.md
[repository-selection]: ../documentation/product-intent/capability/platform/repository-selection.md
[profiles]: ../documentation/product-intent/capability/platform/profiles.md
[scm-platform]: ../glossary/terms/scm-platform.md
[resilient-network]: ../documentation/product-intent/capability/platform/resilient-network-execution.md
[adr-http]: ADR-26101703-scm-http-client-wrapper.md
[adr-saas-guard]: ADR-26102002-public-saas-scm-affiliation-guard.md
[github-mapping-q]: ../documentation/specifications/scm/github/repository-selection-mapping/index.md#сборка-q
[github-enum-index]: ../documentation/specifications/scm/github/repository-enumeration/index.md
[github-base-limits]: ../documentation/specifications/scm/github/base-enumeration.md#ограничения-api
