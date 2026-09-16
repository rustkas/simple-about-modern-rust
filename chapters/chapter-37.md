# Глава 37. Cargo

Cargo — это не просто менеджер пакетов для Rust. Это **вся экосистема разработки**: сборка, управление зависимостями, тестирование, документирование, публикация и многое другое.

В этой главе мы изучим Cargo во всех деталях: от структуры `Cargo.toml` до продвинутых функций, таких как профили сборки и настройка рабочего пространства.

Все примеры этой главы предполагают установленный Rust и Cargo.

---

## 37.1. Что такое Cargo?

**Cargo** — официальный инструмент Rust для работы с **пакетами (packages)**: он создаёт проекты, разрешает зависимости, компилирует targets, запускает тесты, строит документацию и публикует пакеты.

Здесь важно сразу различать три понятия:

- **package** — единица, которой управляет Cargo и которая описывается `Cargo.toml`;
- **target** — конкретная цель сборки внутри package: библиотека, бинарная программа, пример, тест или benchmark;
- **crate** — единица компиляции Rust, создаваемая из target.

Например, один package может содержать библиотеку и несколько бинарных программ:

```text
package
│
├── library target  ──────► library crate
│
├── binary target ────────► binary crate
│
├── binary target ────────► binary crate
│
├── example target ───────► binary crate
│
└── integration test ─────► test crate
```

Поэтому утверждение:

> «Cargo управляет крейтами»

полезно как упрощение, но технически точнее сказать:

> **Cargo управляет packages и targets, а targets компилируются в crates.**

Это различие становится особенно важным в больших проектах и workspace.

### Основные задачи Cargo

```bash
# Создать package
cargo new my_project

# Проверить исходный код
cargo check

# Собрать package
cargo build

# Собрать оптимизированную версию
cargo build --release

# Запустить бинарную программу
cargo run

# Запустить тесты
cargo test

# Построить документацию
cargo doc

# Добавить зависимость
cargo add serde

# Удалить зависимость
cargo remove serde

# Обновить зависимости согласно Cargo.toml
cargo update
```

Cargo также предоставляет инструменты для анализа проекта:

```bash
# Показать дерево зависимостей
cargo tree

# Показать метаданные проекта
cargo metadata
```

**Важно:** `cargo update` не означает «обновить вообще всё до последних возможных версий». Cargo обновляет lock-файл в рамках требований, указанных в `Cargo.toml`.

---

## 37.2. Структура Cargo-проекта

Простейший binary package:

```text
my_project/
├── Cargo.toml
├── Cargo.lock
└── src/
    └── main.rs
```

Если package содержит библиотеку:

```text
my_library/
├── Cargo.toml
├── Cargo.lock
└── src/
    └── lib.rs
```

Но package может быть значительно сложнее:

```text
my_project/
├── Cargo.toml
├── Cargo.lock
│
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── server.rs
│       └── cli.rs
│
├── examples/
│   ├── basic.rs
│   └── advanced.rs
│
├── tests/
│   └── integration.rs
│
├── benches/
│   └── benchmark.rs
│
└── target/
```

Здесь:

- `src/lib.rs` — библиотечный target;
- `src/main.rs` — основной binary target;
- `src/bin/*.rs` — дополнительные binary targets;
- `examples/*.rs` — примеры;
- `tests/*.rs` — интеграционные тесты;
- `benches/*.rs` — benchmarks;
- `target/` — генерируемые Cargo артефакты.

Таким образом, **один package не обязательно соответствует одному crate**.

Например:

```text
my_project/
└── Cargo.toml
    │
    ├── src/lib.rs
    │       └── library crate
    │
    ├── src/main.rs
    │       └── binary crate
    │
    ├── src/bin/server.rs
    │       └── binary crate
    │
    └── src/bin/client.rs
            └── binary crate
```

Cargo допускает **не более одного library target**, но binary targets может быть несколько. ([Rust Documentation][7])

---

## 37.3. `Cargo.toml` — манифест package

`Cargo.toml` — это **манифест package**, а не «описание крейта» в узком смысле.

В нём Cargo узнаёт:

- имя и версию package;
- используемую edition;
- минимальную поддерживаемую версию Rust;
- targets;
- зависимости;
- features;
- настройки публикации;
- параметры сборки.

Пример современного манифеста:

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"

description = "Example Rust application"
license = "MIT"
repository = "https://github.com/user/my_app"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

[dev-dependencies]
pretty_assertions = "1.0"

[build-dependencies]
cc = "1.0"

[features]
default = []
native = ["dep:cc"]

[profile.release]
opt-level = 3
lto = true
```

Для Edition 2024 действует **resolver 3** — механизм разрешения зависимостей, учитывающий `rust-version`. Поэтому в обычном package с `edition = "2024"` указывать `resolver = "3"` отдельно не требуется. ([Rust Documentation][8])

Однако для **virtual workspace** значение resolver следует задавать в корневом `[workspace]`, поскольку там нет `[package]`, из которого Cargo мог бы вывести edition. ([Rust Documentation][8])

---

## 37.4. Секции `Cargo.toml`

Основные секции:

| Секция                     | Назначение                                       |
| -------------------------- | ------------------------------------------------ |
| `[package]`                | Метаданные package                               |
| `[lib]`                    | Настройка library target                         |
| `[[bin]]`                  | Настройка binary target                          |
| `[[example]]`              | Настройка example target                         |
| `[[test]]`                 | Настройка test target                            |
| `[[bench]]`                | Настройка benchmark target                       |
| `[dependencies]`           | Основные зависимости                             |
| `[dev-dependencies]`       | Зависимости тестов, examples и benchmarks        |
| `[build-dependencies]`     | Зависимости `build.rs`                           |
| `[features]`               | Возможности условной сборки                      |
| `[profile.*]`              | Профили компиляции                               |
| `[workspace]`              | Workspace                                        |
| `[workspace.dependencies]` | Общие зависимости workspace                      |
| `[target.*]`               | Зависимости и настройки для конкретной платформы |
| `[lints]`                  | Настройка lint'ов                                |

Например, если package содержит две программы:

```toml
[[bin]]
name = "server"
path = "src/bin/server.rs"

[[bin]]
name = "client"
path = "src/bin/client.rs"
```

то один package создаёт два binary targets:

```text
my_project
├── server
└── client
```

---

## 37.5. Зависимости (Dependencies)

Зависимость указывается в `Cargo.toml`:

```toml
[dependencies]
serde = "1.0"
```

Запись

```toml
serde = "1.0"
```

означает совместимое с SemVer требование, эквивалентное:

```toml
serde = "^1.0"
```

Для версий `1.x` это означает:

```text
>= 1.0.0
< 2.0.0
```

Можно указать более точные ограничения:

```toml
[dependencies]

# Совместимая версия
serde = "1.0"

# Точная версия
serde = "=1.0.0"

# Явный caret
serde = "^1.0.0"

# Только patch-релизы внутри 1.0
serde = "~1.0.0"
```

Но нужно помнить о SemVer-правилах для версий `0.x`: ограничения для `0.2`, `0.0.3` и подобных версий существенно строже, чем для стабильных `1.x`.

### Git-зависимость

```toml
[dependencies]
my_lib = {
    git = "https://github.com/user/my_lib.git",
    branch = "main"
}
```

Можно зафиксировать конкретный commit:

```toml
[dependencies]
my_lib = {
    git = "https://github.com/user/my_lib.git",
    rev = "abcdef123456"
}
```

Cargo сохраняет выбранную ревизию Git-зависимости в `Cargo.lock`. ([Rust Documentation][9])

### Локальная зависимость

```toml
[dependencies]
my_lib = { path = "../my_lib" }
```

Это особенно удобно при разработке нескольких связанных packages.

### Опциональная зависимость

Современный способ скрыть внутреннее имя optional dependency:

```toml
[dependencies]
serde = { version = "1.0", optional = true }

[features]
json = ["dep:serde"]
```

Теперь пользователь включает не техническое имя зависимости `serde`, а смысловую возможность:

```bash
cargo build --features json
```

Запись `dep:serde` предотвращает автоматическое создание feature с именем `serde`. Это позволяет отделить **публичный API features** от внутренних названий зависимостей. ([Rust Documentation][10])

---

## 37.6. `Cargo.lock` — фиксация разрешённых версий

`Cargo.toml` отвечает на вопрос:

> Какие версии зависимостей **допустимы**?

`Cargo.lock` отвечает на другой вопрос:

> Какие конкретно версии Cargo **выбрал**?

Например, в `Cargo.toml`:

```toml
[dependencies]
serde = "1.0"
```

может разрешаться множество версий `1.x`.

`Cargo.lock` фиксирует конкретный результат разрешения:

```text
serde 1.x.y
serde_derive 1.x.y
syn 2.x.y
quote 1.x.y
proc-macro2 1.x.y
...
```

`Cargo.lock` создаётся и изменяется Cargo. Его **не следует редактировать вручную**. ([Rust Documentation][4])

### Нужно ли хранить `Cargo.lock` в Git?

Для приложений ответ однозначный:

> **Да.**

Для библиотек старое правило «никогда не коммить `Cargo.lock`» больше не стоит использовать как универсальное правило. Современная документация Cargo рекомендует:

> **Если сомневаетесь — включайте `Cargo.lock` в систему контроля версий.**

Это особенно полезно для воспроизводимых сборок и CI. ([Rust Documentation][4])

### Обновление зависимостей

```bash
# Пересчитать зависимости
cargo update

# Обновить конкретную зависимость
cargo update serde

# Проверить, что lock-файл не будет изменён
cargo check --locked
```

Последняя команда особенно полезна в CI:

```bash
cargo test --locked
```

Она заставляет Cargo использовать существующий `Cargo.lock` и завершиться с ошибкой, если для сборки потребовалось бы изменить его.

---

## 37.7. Профили сборки (Profiles)

Cargo предоставляет четыре встроенных профиля:

| Профиль   | Типичное использование  |
| --------- | ----------------------- |
| `dev`     | обычная разработка      |
| `release` | оптимизированная сборка |
| `test`    | тесты                   |
| `bench`   | benchmarks              |

Команды выбирают профиль автоматически:

```bash
cargo build          # dev
cargo run            # dev
cargo test           # test
cargo bench          # bench
cargo build --release # release
```

`test` по умолчанию наследует настройки `dev`, а `bench` — `release`. ([Rust Documentation][5])

Например:

```toml
[profile.dev]
opt-level = 0
debug = true

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
```

### Почему `dev` и `release` так сильно отличаются?

В `dev` Cargo стремится обеспечить:

- быструю компиляцию;
- удобную отладку;
- debug assertions;
- incremental compilation.

В `release` приоритет меняется:

- оптимизация кода;
- уменьшение runtime overhead;
- производительность конечного приложения.

Поэтому сравнивать производительность программы, запущенной через:

```bash
cargo run
```

и:

```bash
cargo run --release
```

не всегда имеет смысл: это разные профили компиляции.

---

## 37.8. Артефакты сборки

Cargo складывает результаты сборки в `target/`.

Упрощённо:

```text
target/
├── debug/
│   ├── my_app
│   ├── deps/
│   ├── build/
│   └── incremental/
│
└── release/
    ├── my_app
    ├── deps/
    └── build/
```

При использовании `--target` появляется дополнительный уровень:

```text
target/
└── aarch64-unknown-linux-gnu/
    └── debug/
```

В `target/` находятся:

- исполняемые файлы;
- библиотеки;
- промежуточные результаты компиляции;
- результаты build scripts;
- incremental cache;
- fingerprint-информация;
- другие внутренние артефакты Cargo.

**Не следует полагаться на точную внутреннюю структуру `target/`:** это рабочий каталог Cargo, а не стабильный формат данных для приложений. ([Rust Documentation][6])

Обычно `target/` не помещают в Git.

### Очистка

```bash
# Удалить все артефакты
cargo clean

# Удалить только release
cargo clean --release
```

`cargo clean --release` действительно является поддерживаемой командой и удаляет артефакты `release`. ([Rust Documentation][11])

---

## 37.9. Основные команды Cargo

| Команда                 | Назначение                                      |
| ----------------------- | ----------------------------------------------- |
| `cargo new`             | Создать новый package                           |
| `cargo init`            | Создать Cargo package в существующей директории |
| `cargo build`           | Собрать package                                 |
| `cargo build --release` | Собрать release                                 |
| `cargo run`             | Собрать и запустить binary                      |
| `cargo check`           | Проверить код без генерации конечного бинарника |
| `cargo test`            | Запустить тесты                                 |
| `cargo bench`           | Запустить benchmarks                            |
| `cargo doc`             | Построить документацию                          |
| `cargo doc --open`      | Построить и открыть документацию                |
| `cargo add`             | Добавить зависимость                            |
| `cargo remove`          | Удалить зависимость                             |
| `cargo update`          | Обновить lock-файл                              |
| `cargo tree`            | Показать дерево зависимостей                    |
| `cargo metadata`        | Вывести машинно-читаемые метаданные             |
| `cargo clean`           | Удалить артефакты                               |
| `cargo package`         | Проверить и собрать пакет для публикации        |
| `cargo publish`         | Опубликовать package в registry                 |
| `cargo install`         | Установить бинарный package                     |
| `cargo fetch`           | Загрузить зависимости без сборки                |

Полезно различать:

```bash
cargo check
```

и:

```bash
cargo build
```

`check` выполняет необходимые проверки компиляции, но не создаёт конечный executable/library artifact. Поэтому он обычно заметно быстрее при повседневной разработке.

---

## 37.10. Рабочие пространства (Workspaces)

**Workspace** — это набор связанных Cargo packages, которые управляются как единое целое.

Например:

```text
my_workspace/
├── Cargo.toml
├── Cargo.lock
│
├── app/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
│
├── core/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
│
└── cli/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

Корневой `Cargo.toml`:

```toml
[workspace]
members = [
    "app",
    "core",
    "cli",
]

resolver = "3"

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
```

`core/Cargo.toml`:

```toml
[package]
name = "core"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { workspace = true }
```

Теперь зависимость объявлена один раз:

```toml
[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
```

а package подключает её:

```toml
[dependencies]
serde = { workspace = true }
```

Workspace имеет общий `Cargo.lock` и общий каталог сборки `target/`. ([Rust Documentation][12])

Это особенно удобно для больших проектов:

```text
workspace
│
├── core library
├── backend
├── CLI
├── WASM package
└── integration tools
```

Каждая часть остаётся отдельным package, но все они развиваются в одном workspace.

---

## 37.11. Команды Cargo с флагами

Аргументы, предназначенные **самой программе**, передаются после `--`:

```bash
cargo run -- --verbose --config config.json
```

Cargo получает:

```text
cargo run
```

а программа получает:

```text
--verbose --config config.json
```

### Фильтрация тестов

```bash
cargo test add
```

запустит тесты, имена которых соответствуют фильтру `add`.

Для просмотра `println!` внутри тестов:

```bash
cargo test -- --nocapture
```

### Features

```bash
cargo build --features "full"
```

или:

```bash
cargo build --no-default-features
```

или:

```bash
cargo build --all-features
```

### Работа с конкретным package workspace

```bash
cargo build -p core
cargo test -p core
```

### Воспроизводимая сборка

```bash
cargo build --locked
```

или ещё строже:

```bash
cargo build --frozen
```

`--frozen` объединяет ограничения `--locked` и `--offline`.

---

## 37.12. `cargo add` и `cargo remove`

Современный Cargo позволяет редактировать зависимости через CLI:

```bash
cargo add serde
```

С feature:

```bash
cargo add serde --features derive
```

Опциональная зависимость:

```bash
cargo add serde --optional
```

Dev dependency:

```bash
cargo add pretty_assertions --dev
```

Build dependency:

```bash
cargo add cc --build
```

Git-зависимость:

```bash
cargo add my_lib --git https://github.com/user/my_lib.git
```

Path dependency:

```bash
cargo add my_lib --path ../my_lib
```

Удаление:

```bash
cargo remove serde
```

После изменения `Cargo.toml` Cargo автоматически пересчитает необходимые зависимости и обновит `Cargo.lock`.

---

## 37.13. Настройка Cargo

Cargo поддерживает конфигурационные файлы:

```text
.cargo/config.toml
```

Например:

```toml
[build]
jobs = 4
target-dir = "target"

[target.x86_64-unknown-linux-gnu]
linker = "clang"

[target.'cfg(windows)']
runner = "cmd /c"

[net]
git-fetch-with-cli = true

[alias]
check-all = "check --all-targets"
```

Особенно полезны aliases:

```toml
[alias]
c = "check"
t = "test"
r = "run"
rr = "run --release"
```

После этого можно использовать:

```bash
cargo c
cargo t
cargo rr
```

Cargo ищет конфигурацию `.cargo/config.toml` в текущем каталоге и родительских каталогах, а также в глобальном `$CARGO_HOME/config.toml`. ([Rust Documentation][13])

**Важно:** конфигурация Cargo и `Cargo.toml` — разные вещи.

`Cargo.toml` описывает **сам package**.

`.cargo/config.toml` настраивает **поведение Cargo в конкретном окружении**.

---

## 37.14. Cargo и кросс-компиляция

Cargo позволяет указать target:

```bash
# Посмотреть доступные targets
rustc --print target-list

# Установить target
rustup target add aarch64-unknown-linux-gnu

# Собрать для ARM64 Linux
cargo build --target aarch64-unknown-linux-gnu
```

Для WebAssembly:

```bash
rustup target add wasm32-unknown-unknown

cargo build --target wasm32-unknown-unknown
```

Но здесь важно понимать:

> **Установка Rust target ещё не означает наличие всех инструментов, необходимых для линковки.**

Например, для некоторых native targets потребуется соответствующий linker и системные библиотеки.

Для WASM это особенно заметно: `wasm32-unknown-unknown` позволяет получить WebAssembly target, но для полноценного Rust → WebAssembly приложения могут потребоваться дополнительные инструменты и runtime-интеграция.

---

## 37.15. Оффлайн-сборка

Cargo умеет работать без доступа к сети:

```bash
cargo build --offline
```

В этом режиме Cargo не обращается к сети и использует только доступные локально данные.

Перед поездкой или работой в изолированной среде зависимости можно заранее загрузить:

```bash
cargo fetch
```

После этого:

```bash
cargo build --offline
```

Для CI особенно полезен:

```bash
cargo build --frozen
```

Он одновременно требует:

- использовать существующий `Cargo.lock`;
- не обращаться к сети.

Поэтому `--frozen` хорошо подходит для проверки того, что сборка действительно воспроизводима в заранее подготовленном окружении. ([Rust Documentation][11])

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Cargo может использовать несколько версий одной зависимости

Важно исправить распространённое заблуждение: **две версии одной зависимости не обязательно являются конфликтом**.

Например, если:

```text
application
├── library_a
│   └── serde 1.x
│
└── library_b
    └── serde 2.x
```

и требования совместимы с одновременной установкой обеих версий, Cargo может собрать обе:

```text
serde 1.x
serde 2.x
```

Это видно через:

```bash
cargo tree
```

Например:

```bash
cargo tree -d
```

показывает зависимости, которые присутствуют в нескольких версиях.

**Настоящая проблема возникает**, когда требования невозможно удовлетворить одновременно.

Например:

```text
library_a требует:
foo >= 1.0, < 2.0

library_b требует:
foo >= 2.0, < 3.0

а сам проект требует:
foo = 1.5
```

В этом случае Cargo может завершить разрешение зависимостей ошибкой.

### Эксперимент 2: Посмотреть дерево зависимостей

Создайте проект:

```bash
cargo new dependency_tree
cd dependency_tree
```

Добавьте:

```bash
cargo add serde
cargo add serde_json
```

Теперь:

```bash
cargo tree
```

покажет не только `serde`, но и транзитивные зависимости.

Это один из самых полезных инструментов Cargo при диагностике:

> «Почему эта библиотека вообще попала в мой проект?»

### Эксперимент 3: Проверить воспроизводимость

После первой успешной сборки:

```bash
cargo build
```

выполните:

```bash
cargo check --locked
```

Если `Cargo.toml` не требует изменения разрешённых зависимостей, команда завершится успешно.

Теперь намеренно измените ограничение версии зависимости так, чтобы существующий `Cargo.lock` больше ему не соответствовал, и снова выполните:

```bash
cargo check --locked
```

Cargo должен остановиться вместо автоматического изменения lock-файла.

---

## Практика

### Задание 1

Создайте package:

```bash
cargo new geometry
```

Добавьте библиотечный target и реализуйте:

```rust
pub fn area_of_circle(radius: f64) -> f64 {
    std::f64::consts::PI * radius * radius
}

pub fn area_of_rectangle(width: f64, height: f64) -> f64 {
    width * height
}

pub fn area_of_triangle(base: f64, height: f64) -> f64 {
    base * height / 2.0
}
```

### Задание 2

Добавьте в тот же package binary target, который использует библиотеку:

```text
src/
├── lib.rs
└── main.rs
```

Запустите:

```bash
cargo run
```

### Задание 3

Добавьте второй binary target:

```text
src/bin/
└── inspect.rs
```

Запустите его:

```bash
cargo run --bin inspect
```

### Задание 4

Создайте workspace из двух packages:

```text
workspace/
├── Cargo.toml
├── geometry/
└── app/
```

Сделайте так, чтобы `app` зависел от `geometry` через:

```toml
[dependencies]
geometry = { path = "../geometry" }
```

### Задание 5

Добавьте `serde` через:

```bash
cargo add serde --features derive
```

Изучите изменения в:

```text
Cargo.toml
Cargo.lock
```

Затем выполните:

```bash
cargo tree
```

и найдите `serde` и его транзитивные зависимости.

### Задание 6

Создайте feature:

```toml
[features]
default = []
fast = []
```

и условно включайте код:

```rust
#[cfg(feature = "fast")]
fn mode() {
    println!("Fast mode");
}

#[cfg(not(feature = "fast"))]
fn mode() {
    println!("Normal mode");
}

fn main() {
    mode();
}
```

Запустите:

```bash
cargo run
cargo run --features fast
```

**Открыть пример в Rust Playground:**
[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bcfg%28feature%20%3D%20%22fast%22%29%5D%0Afn%20mode%28%29%20%7B%0A%20%20%20%20println%21%28%22Fast%20mode%22%29%3B%0A%7D%0A%0A%23%5Bcfg%28not%28feature%20%3D%20%22fast%22%29%29%5D%0Afn%20mode%28%29%20%7B%0A%20%20%20%20println%21%28%22Normal%20mode%22%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20mode%28%29%3B%0A%7D)

> В Playground feature `fast` не включён через `Cargo.toml`, поэтому этот пример демонстрирует сам механизм `#[cfg]`, а полноценный эксперимент с Cargo feature следует выполнять локально.

### Задание 7

🔨 **Эксперимент с компилятором.**

Добавьте:

```bash
cargo check --locked
```

и измените `Cargo.toml` так, чтобы Cargo потребовал пересчитать зависимости.

Наблюдайте, почему `--locked` запрещает автоматическое изменение `Cargo.lock`.

### Задание 8

🔨 **Эксперимент с зависимостями.**

Выполните:

```bash
cargo tree
```

затем:

```bash
cargo tree -d
```

Найдите зависимости, которые присутствуют в нескольких версиях.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Cargo** — система сборки и управления Rust packages.
- **Package** — единица, описываемая `Cargo.toml`.
- **Target** — конкретная цель сборки package.
- **Crate** — единица компиляции, создаваемая target.
- **`Cargo.toml`** — манифест package.
- **`Cargo.lock`** — точный результат разрешения зависимостей.
- **Dependencies** — внешние и локальные зависимости.
- **Features** — механизм условной функциональности.
- **Profiles** — настройки компиляции для `dev`, `release`, `test` и `bench`.
- **Workspace** — объединение нескольких packages.
- **`cargo tree`** — инструмент анализа зависимостей.
- **`cargo metadata`** — машинно-читаемое описание package и workspace.
- **`.cargo/config.toml`** — конфигурация поведения Cargo.
- **`--locked` и `--frozen`** — инструменты воспроизводимых сборок.
- **`--target`** — механизм кросс-компиляции.

**Самая важная идея:**

> Cargo — это слой, который связывает исходный код Rust с реальным проектом: targets, crates, зависимостями, сборкой, тестами, документацией и публикацией. Понимание разницы между **package → target → crate** позволяет правильно понимать архитектуру любого Cargo-проекта — от маленькой программы до большого workspace.

[1]: https://doc.rust-lang.org/cargo/appendix/glossary.html 'Appendix: Glossary - The Cargo Book'
[2]: https://doc.rust-lang.org/nightly/cargo/guide/project-layout.html 'Package Layout - The Cargo Book'
[3]: https://doc.rust-lang.org/cargo/reference/manifest.html 'The Manifest Format - The Cargo Book'
[4]: https://doc.rust-lang.org/nightly/cargo/guide/cargo-toml-vs-cargo-lock.html 'Cargo.toml vs Cargo.lock - The Cargo Book'
[5]: https://doc.rust-lang.org/nightly/cargo/reference/profiles.html 'Profiles - The Cargo Book'
[6]: https://doc.rust-lang.org/cargo/reference/build-cache.html 'Build Cache - The Cargo Book'
[7]: https://doc.rust-lang.org/cargo/reference/cargo-targets.html 'Cargo Targets - The Cargo Book'
[8]: https://doc.rust-lang.org/edition-guide/rust-2024/cargo-resolver.html 'Cargo: Rust-version aware resolver - The Rust Edition Guide'
[9]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html 'Specifying Dependencies - The Cargo Book'
[10]: https://doc.rust-lang.org/cargo/reference/semver.html 'SemVer Compatibility - The Cargo Book'
[11]: https://doc.rust-lang.org/cargo/commands/cargo-clean.html 'cargo clean - The Cargo Book'
[12]: https://doc.rust-lang.org/stable/cargo/appendix/glossary.html?highlight=Workspace 'Appendix: Glossary - The Cargo Book'
[13]: https://doc.rust-lang.org/cargo/reference/config.html 'Configuration - The Cargo Book'
