# ArchPlugin

# Разработка плагина для автоматического контроля архитектурных зависимостей в многомодульном проекте

Топик семинара наставника. Автор и эксперт: Самсонова Карина.

| | |
|---|---|
| **Результат работы** | Плагин для системы сборки или статического анализа, который проверяет зависимости между модулями и роняет сборку при нарушении архитектуры |
| **Стек** | На выбор. Эталонный: Kotlin + Gradle, для него автотесты уже готовы. Другой стек: Maven, ESLint, ruff, import-linter и т.п. (см. раздел 10) |
| **Трудоёмкость** | ~11 астрономических часов на всё, включая видео (обязательная часть, этапы 1–3, около 7 часов) |
| **Проверка** | 24 автотеста по этапам (для другого стека — те же сценарии своими тестами) плюс короткое видео |
| **Сдача** | Pull Request в этот репозиторий + видео (ссылкой в PR или файлом в Telegram [@ka_rine_s](https://t.me/ka_rine_s)) |

---

## 1. Зачем это нужно

В многомодульных проектах (бэкенд на Kotlin/Java, Android-приложения) архитектура обычно описана словами: «domain ни от чего не зависит», «data может зависеть только от domain», «ui не ходит в базу напрямую». Со временем кто-нибудь торопится, добавляет в `domain/build.gradle.kts` строку `implementation(project(":data"))`, ревьюер её не замечает, и через полгода слои перепутаны так, что их уже не разделить.

Надёжное решение — проверять архитектуру **автоматически при каждой сборке**. Так делают большие команды: правила описываются один раз в конфигурации, а сборка падает с понятным сообщением, если кто-то их нарушил. В этой работе вы напишете такой инструмент сами, в виде плагина к системе сборки (эталонный вариант — Gradle-плагин на Kotlin), и по пути разберётесь, как устроены плагины, задачи и инкрементальная сборка Gradle. Эти знания пригодятся в любом проекте на JVM.

## 2. Что должно получиться

Дальше задание описано для эталонного стека (Kotlin + Gradle). Если вы выбрали другой стек, прочитайте раздел 10: функциональность та же, меняются только имена и способ подключения.

Плагин подключается к корневому проекту и настраивается так:

```kotlin
// build.gradle.kts корневого проекта
plugins {
    id("ru.hse.archguard")
}

architectureGuard {
    layer("domain", ":domain", ":core")   // слой и модули, которые в него входят
    layer("data", ":data")
    layer("ui", ":app")

    allow("data", "domain")               // слою data разрешено зависеть от domain
    allow("ui", "data", "domain")         // слою ui разрешено зависеть от data и domain

    failOnViolation = true                // по умолчанию true (этап 4)
}
```

Запуск `./gradlew checkArchitecture` на проекте, где `domain` зависит от `data`, заканчивается так:

```
> Task :checkArchitecture FAILED
Architecture violation: module ':domain' (layer 'domain') must not depend on module ':data' (layer 'data')

FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':checkArchitecture'.
> Found 1 architecture violation(s):
  Architecture violation: module ':domain' (layer 'domain') must not depend on module ':data' (layer 'data')
```

А в `build/reports/architecture-guard/report.json` появляется машиночитаемый отчёт.

## 3. Окружение

| Что | Версия | Комментарий |
|---|---|---|
| JDK | 17 или новее для запуска Gradle | Плагин компилируется под Java 17 через Java toolchain. Если JDK 17 на машине нет, Gradle скачает её сам |
| Gradle | 8.14.3 | Ставить не нужно: в репозитории есть Gradle wrapper (`gradlew`), он скачает нужную версию |
| Kotlin | 2.0.21 | Подключается автоматически плагином `kotlin-dsl` (это версия Kotlin, встроенная в Gradle 8.14.3) |
| IDE | любая | Рекомендуется IntelliJ IDEA Community: подсказки по Gradle API работают из коробки. Android Studio и Android SDK **не нужны** |
| Интернет | при первом запуске | Скачиваются Gradle, JDK 17 (если нужно) и зависимости тестов. Дальше всё работает из кэша |

Окружение изолировано: версии Gradle, Kotlin и JDK зафиксированы в репозитории, глобально ничего ставить не нужно, кроме любой JDK 17+.

## 4. Быстрый старт

1. Сделайте форк этого репозитория и склонируйте его:
   ```bash
   git clone <ссылка на ваш форк>
   cd architecture-guard
   git checkout -b solution/<ваша-фамилия-латиницей>
   ```
2. Запустите тесты:
   ```bash
   ./gradlew test          # Windows: gradlew.bat test
   ```
3. Ожидаемый результат на старте: **24 теста, из них 1 проходит, 23 падают.** Проходит только смоук-тест `plugin can be applied to the root project`: он подтверждает, что окружение настроено правильно. Остальные тесты и есть ваше задание.

Если первый запуск упал не на тестах, а раньше (не скачался Gradle, нет Java), смотрите раздел «Частые проблемы».

## 5. Структура репозитория

```
architecture-guard/
├── build.gradle.kts            # сборка плагина (kotlin-dsl, java-gradle-plugin, тестовые зависимости)
├── settings.gradle.kts         # имя проекта и автозагрузка JDK 17
├── gradlew, gradlew.bat, gradle/   # Gradle wrapper 8.14.3
├── src/main/kotlin/ru/hse/archguard/
│   └── ArchitectureGuardPlugin.kt  # ЗАГЛУШКА: здесь вы пишете плагин (можно добавлять свои файлы)
├── src/test/kotlin/ru/hse/archguard/
│   ├── TestProject.kt              # помощник: создаёт временный многомодульный проект для теста
│   ├── Stage1PluginSetupTest.kt    # этап 1
│   ├── Stage2DslTest.kt            # этап 2
│   ├── Stage3DependencyRulesTest.kt# этап 3
│   ├── Stage4ReportTest.kt         # этап 4
│   └── Stage5IncrementalTest.kt    # этап 5*
├── sample/                     # демо-проект для ручной проверки и видео
├── docs/SCENARIOS.md           # те же проверки в виде сценариев (для выполнения на другом стеке)
└── .github/ и .gitlab/         # шаблон Pull Request и CI
```

**Тесты менять нельзя.** Вся работа ведётся в `src/main/kotlin`. Если вы считаете, что тест ошибочен, напишите эксперту.

Как устроены тесты: каждый тест через [Gradle TestKit](https://docs.gradle.org/8.14.3/userguide/test_kit.html) создаёт во временной папке маленький многомодульный проект, подключает к нему ваш плагин, запускает настоящую сборку и проверяет результат (статус задачи, текст в выводе, файл отчёта). Поэтому тесты медленнее обычных юнит-тестов: весь набор идёт 2–4 минуты.

## 6. Контракт плагина

Тесты проверяют именно эти имена, сообщения и форматы. Остальное (структура классов, названия внутренних функций) на ваше усмотрение.

**Идентификаторы**
- id плагина: `ru.hse.archguard`, класс: `ru.hse.archguard.ArchitectureGuardPlugin` (уже прописаны в `build.gradle.kts`);
- расширение (DSL): `architectureGuard`;
- задача: `checkArchitecture`, группа `verification`.

**DSL**
- `layer(name: String, vararg modules: String)` объявляет слой и его модули (пути модулей вида `:domain`). Слой может содержать несколько модулей;
- `allow(from: String, vararg to: String)` разрешает слою `from` зависеть от слоёв `to`. Повторные вызовы для одного слоя дополняют список;
- `failOnViolation: Property<Boolean>` — по умолчанию `true`.

**Семантика проверки**
- проверяются только **прямые** зависимости вида `project(":x")`, объявленные в модулях;
- проверяются все конфигурации модуля, **кроме тестовых** (у которых в имени есть `test` в любом регистре: `testImplementation`, `testRuntimeOnly` и т.п.). То есть `implementation`, `api`, `compileOnly`, `runtimeOnly` и другие проверяются;
- зависимости внутри одного слоя разрешены всегда;
- если модуль-источник или модуль-цель не входит ни в один слой, такая зависимость не проверяется;
- зависимость модуля слоя A от модуля слоя B (A ≠ B) — нарушение, если B нет в `allow(A, ...)`.

**Сообщения об ошибках** (должны появиться в выводе сборки дословно)

| Ситуация | Сообщение | Этап |
|---|---|---|
| Плагин подключён не к корневому проекту | `Plugin 'ru.hse.archguard' must be applied to the root project` | 1 |
| В `allow()` указан необъявленный слой | `Unknown layer '<имя>' used in allow()` | 2 |
| В `layer()` указан несуществующий модуль | `Module '<путь>' declared in layer '<слой>' does not exist` | 2 |
| Модуль указан в нескольких слоях | `Module '<путь>' is assigned to multiple layers` (дальше можно перечислить слои) | 2 |
| Нарушение архитектуры (по строке на каждое) | `Architecture violation: module '<откуда>' (layer '<слой>') must not depend on module '<куда>' (layer '<слой>')` | 3 |

**Отчёт** (этап 4): файл `build/reports/architecture-guard/report.json` в корневом проекте.

```json
{
  "violations": [
    {"from": ":data", "fromLayer": "data", "to": ":app", "toLayer": "ui"},
    {"from": ":domain", "fromLayer": "domain", "to": ":data", "toLayer": "data"}
  ]
}
```

Нарушения отсортированы по `from`, затем по `to`. Если нарушений нет, файл всё равно создаётся с пустым массивом.

## 7. Этапы выполнения

Каждый этап можно проверить отдельно:

```bash
./gradlew test --tests "*Stage1*"
```

### Этап 1. Плагин и задача (≈1 ч) — обязательный

**Цель:** плагин подключается к корневому проекту и регистрирует задачу `checkArchitecture`.

Что сделать:
1. В `ArchitectureGuardPlugin.apply()` проверить, что `project` — корневой (`project.rootProject`). Если нет, бросить `GradleException` с сообщением из контракта.
2. Создать класс задачи (например, `CheckArchitectureTask : DefaultTask()`) с методом, помеченным `@TaskAction`. Пока он может ничего не делать.
3. Зарегистрировать задачу **лениво** через `project.tasks.register(...)` (не `create`), задать `group = "verification"` и `description`.

Тесты: `Stage1PluginSetupTest` (4 теста).

Документация: [Developing Custom Gradle Plugins](https://docs.gradle.org/8.14.3/userguide/custom_plugins.html), [Implementing Custom Tasks](https://docs.gradle.org/8.14.3/userguide/custom_tasks.html).

### Этап 2. DSL и проверка конфигурации (≈1,5 ч) — обязательный

**Цель:** плагин читает описание архитектуры из блока `architectureGuard { ... }` и сообщает об ошибках в нём.

Что сделать:
1. Создать класс расширения (например, `abstract class ArchitectureGuardExtension`) с методами `layer(...)`, `allow(...)` и свойством `abstract val failOnViolation: Property<Boolean>`. Абстрактные свойства Gradle реализует сам.
2. Зарегистрировать его в плагине: `project.extensions.create("architectureGuard", ArchitectureGuardExtension::class.java)`. Задать `failOnViolation` значение по умолчанию через `convention(true)`.
3. Передать описание слоёв и правил в задачу и в `@TaskAction` проверить конфигурацию: неизвестные слои в `allow`, несуществующие модули, модуль в нескольких слоях. Список существующих модулей — `project.allprojects.map { it.path }`.

Подсказка: удобно хранить в задаче описание как `MapProperty<String, List<String>>` («слой → модули», «слой → разрешённые слои»), а заполнять через `project.provider { ... }`. Это пригодится на этапе 5.

Тесты: `Stage2DslTest` (5 тестов).

Документация: [Making a plugin configurable (extensions)](https://docs.gradle.org/8.14.3/userguide/implementing_gradle_plugins_binary.html), [Lazy Configuration](https://docs.gradle.org/8.14.3/userguide/lazy_configuration.html).

### Этап 3. Проверка правил зависимостей (≈2,5 ч) — обязательный

**Цель:** задача находит все нарушения архитектуры и роняет сборку с понятным сообщением.

Что сделать:
1. Собрать прямые зависимости между модулями: для каждого модуля перебрать `configurations` (кроме тестовых), из каждой взять `dependencies.withType(ProjectDependency::class.java)` и путь модуля-цели (`ProjectDependency.path`).
2. Сопоставить модули слоям и найти нарушения по правилам из раздела «Семантика проверки».
3. Вывести **все** нарушения (по строке на каждое, формат из контракта) и, если они есть, бросить `GradleException`.

Важно: собирать зависимости нужно, когда все модули уже сконфигурированы. Если считать их прямо в `apply()`, подпроекты ещё не прочитаны и список будет пустым. Ленивый `project.provider { ... }` вычисляется позже и решает эту проблему.

Тесты: `Stage3DependencyRulesTest` (8 тестов).

Документация: [Declaring dependencies between subprojects](https://docs.gradle.org/8.14.3/userguide/declaring_dependencies_between_subprojects.html), [ProjectDependency (API)](https://docs.gradle.org/8.14.3/javadoc/org/gradle/api/artifacts/ProjectDependency.html).

После этапа 3 плагин уже полезен: подключите его к `sample/` и попробуйте нарушить архитектуру (см. раздел 9).

### Этап 4. Отчёт и мягкий режим (≈1,5 ч) — уровень П

**Цель:** результат проверки доступен другим инструментам (CI, дашбордам), а команда может внедрять проверку постепенно.

Что сделать:
1. Объявить в задаче выходной файл (`@get:OutputFile abstract val reportFile: RegularFileProperty`) и по умолчанию направить его в `build/reports/architecture-guard/report.json` через `project.layout.buildDirectory`.
2. Записывать отчёт в формате из контракта при **каждом** запуске, в том числе без нарушений. Библиотеку для JSON подключать не обязательно: формат простой.
3. Поддержать `failOnViolation = false`: нарушения выводятся предупреждением (`logger.warn`) и попадают в отчёт, но сборка проходит.

Тесты: `Stage4ReportTest` (3 теста).

### Этап 5*. Инкрементальность и configuration cache (≈2 ч) — уровень Э

**Цель:** проверка не тратит время сборки впустую и работает с современными режимами Gradle.

Что сделать:
1. **UP-TO-DATE.** Повторный запуск без изменений не должен выполнять проверку заново. Для этого Gradle должен знать все входы задачи (`@get:Input`: описание слоёв, правила, список модулей, граф зависимостей, `failOnViolation`) и выход (отчёт с этапа 4). Если что-то из этого изменилось, задача выполняется снова.
2. **Configuration cache.** Запуск с `--configuration-cache` должен проходить без ошибок, а при повторном запуске кэш должен переиспользоваться. Главное правило: в `@TaskAction` нельзя обращаться к `project` (и к `Project` вообще). Всё, что нужно задаче, должно быть заранее вычислено в её свойствах (`Property`, `MapProperty`, `ListProperty`). Если вы на этапах 2–3 заполняли свойства через `project.provider { ... }`, большая часть работы уже сделана.

Проверить вручную:
```bash
cd sample
../gradlew checkArchitecture      # SUCCESS
../gradlew checkArchitecture      # UP-TO-DATE
../gradlew checkArchitecture --configuration-cache   # Configuration cache entry stored
../gradlew checkArchitecture --configuration-cache   # Configuration cache entry reused
```

Тесты: `Stage5IncrementalTest` (4 теста).

Документация: [Incremental build](https://docs.gradle.org/8.14.3/userguide/incremental_build.html), [Configuration cache: requirements](https://docs.gradle.org/8.14.3/userguide/configuration_cache_requirements.html).

## 8. Оценивание

Основа оценки — какие этапы пройдены автотестами. Дальше эксперт корректирует её по качеству кода и видео.

| Пройдено | Оценка |
|---|---|
| Этапы 1–5, все 24 теста | **Э** (экспертный) |
| Этапы 1–4 | **П** (продвинутый) |
| Этапы 1–3 и часть тестов этапов 4–5 | **Б+** (базовый с заделом) |
| Этапы 1–3 | **Б** (базовый) |
| Этапы 1–2 и часть этапа 3 | **Б⁻** (базовый с недостатками) |
| Этапы 1–2 | **У** (удовлетворительно) |
| Только этап 1 | **Н** (недостаточный) |
| Ничего не работает или нет видео | **К** (критический) |

Что может снизить оценку на ступень:
- нет инструкции по запуску;
- изменены тесты;
- код, который сложно читать: всё в одной функции на сотню строк, копипаста, закомментированные куски.

Обратная связь даётся в формате: список замечаний с диапазоном оценки по каждому и итоговая оценка.

## 9. Демо-проект

В папке `sample/` лежит трёхмодульный проект (`app` → `data` → `domain`), к которому плагин подключается прямо из исходников репозитория (`includeBuild("..")`), публиковать его не нужно.

```bash
cd sample
../gradlew checkArchitecture            # проверка проходит
```

Чтобы показать нарушение, раскомментируйте строку `implementation(project(":data"))` в `sample/domain/build.gradle.kts` и запустите проверку снова. Сборка должна упасть с понятным сообщением.

## 10. Если хочется другой стек

Задачу можно решить и на другом стеке: плагин для Maven, правило для ESLint (зависимости между пакетами монорепозитория), плагин для import-linter или ruff в Python и т.п. В этом случае:
1. **До начала работы** напишите эксперту, какой стек выбрали.
2. Реализуйте ту же функциональность и покройте собственными автотестами **все сценарии** из [docs/SCENARIOS.md](docs/SCENARIOS.md). Таблица оценок та же, этапы засчитываются по сценариям.
3. В PR дайте таблицу «сценарий → тест».

Эталонный стек (Kotlin + Gradle) рекомендуется: для него тесты уже готовы, и не придётся тратить время на их написание.

## 11. Как сдавать работу

1. Работа ведётся в вашем форке, в ветке `solution/<фамилия>`.
2. Запишите короткое видео (см. ниже).
3. Откройте **Pull Request из вашей ветки в этот репозиторий**. Описание заполнится шаблоном, его нужно дополнить:

   ```markdown
   ## Выполненное задание
   <коротко, 1–2 строчки: что реализовано, до какого этапа>

   ## Видео
   <ссылка на диск / YouTube или «отправил(а) в Telegram»>

   ## Что сделано (или не сделано)
   - [x] Этап 1 ...
   - [ ] Этап 5* — не сделан configuration cache, потому что ...

   ## Инструкция по быстрому запуску (быстрой проверке)
   1. git clone <ваш форк> && cd architecture-guard && git checkout solution/<фамилия>
   2. ./gradlew test
   3. cd sample && ../gradlew checkArchitecture
   ```
4. PR не вливается в основную ветку: он нужен для проверки и обратной связи. CI автоматически запустит тесты, их результат будет виден прямо в PR. (На основной ветке репозитория CI красный — это нормально: там лежит заглушка.)

Если репозиторий топика размещён на GitLab, всё то же самое делается через Merge Request (шаблон лежит в `.gitlab/merge_request_templates`).

### Видео

Короткое видео (примерно до 4 минут) о том, что получилось. Запишите экран как удобно. Примерный план:
- коротко покажите код плагина;
- покажите работу на `sample/`: проверка проходит → добавляете запрещённую зависимость → сборка падает → убираете → снова зелёная (для П/Э ещё отчёт и повторный запуск с UP-TO-DATE);
- запустите `./gradlew test` и покажите, какие этапы проходят.

Видео можно загрузить куда удобно (Яндекс Диск, Google Drive, YouTube по ссылке) и вставить ссылку в PR, или просто прислать файл в Telegram [@ka_rine_s](https://t.me/ka_rine_s).

## 12. Частые проблемы

- **`./gradlew: Permission denied`** — выполните `chmod +x gradlew`.
- **`Cannot find a Java installation ... languageVersion=17`** — нет доступа в интернет для загрузки JDK. Установите JDK 17 вручную (например, Temurin 17) и запустите снова.
- **Тесты идут долго** — это нормально: каждый тест поднимает отдельную сборку Gradle. Запускайте только нужный этап (`--tests "*Stage3*"`).
- **В тестах непонятно, что пошло не так** — вывод вложенной сборки печатается целиком в консоль и в отчёт `build/reports/tests/test/index.html`. Ищите там `What went wrong`.
- **Список зависимостей пустой** — вы собираете их слишком рано (в `apply()`). См. подсказку в этапе 3.
- **Configuration cache problems found** — в `@TaskAction` где-то используется `project`. Gradle пишет, в какой строке; отчёт лежит в `build/reports/configuration-cache/`.

## 13. Полезные ссылки

- [Gradle: Developing Custom Plugins](https://docs.gradle.org/8.14.3/userguide/custom_plugins.html)
- [Gradle: Implementing Binary Plugins](https://docs.gradle.org/8.14.3/userguide/implementing_gradle_plugins_binary.html)
- [Gradle: Lazy Configuration](https://docs.gradle.org/8.14.3/userguide/lazy_configuration.html)
- [Gradle: Incremental Build](https://docs.gradle.org/8.14.3/userguide/incremental_build.html)
- [Gradle: Configuration Cache](https://docs.gradle.org/8.14.3/userguide/configuration_cache.html)
- [Gradle: Testing Build Logic with TestKit](https://docs.gradle.org/8.14.3/userguide/test_kit.html)
