### Список внесенных изменений и почему

1. **Доработан раздел F.4 (Features)**:
* В примере кода условной компиляции для фич отсутствовал работающий макрос `main`, из-за чего примеры выглядели незаконченными черновиками. Добавлена компилируемая функция `main` и интерактивная кнопка **"Открыть пример в Rust Playground"** для демонстрации работы features на практике.


2. **Доработан раздел F.7 (Lints)**:
* Добавлен пример использования секции `[lints]` на уровне кода в файле `main.rs` с функцией `main`, чтобы читатель сразу видел, как именно переопределяются правила безопасности и предупреждения компилятора.



---

# Приложение F. Cargo Quick Reference

Краткий справочник по Cargo — системе сборки и управления пакетами в Rust. Все основные команды, настройки и концепции в одном месте.

---

## F.1. Основные команды

| Команда | Описание |
| --- | --- |
| `cargo new <name>` | Создать новый проект |
| `cargo init` | Инициализировать проект в существующей папке |
| `cargo build` | Собрать проект (debug режим) |
| `cargo build --release` | Собрать проект (release режим) |
| `cargo run` | Собрать и запустить |
| `cargo run -- <args>` | Запустить с аргументами |
| `cargo check` | Проверить код без сборки (быстро) |
| `cargo test` | Запустить тесты |
| `cargo test -- --nocapture` | Запустить тесты с выводом `println!` |
| `cargo test <name>` | Запустить конкретный тест |
| `cargo doc` | Сгенерировать документацию |
| `cargo doc --open` | Сгенерировать и открыть документацию |
| `cargo fmt` | Отформатировать код |
| `cargo fmt --check` | Проверить форматирование |
| `cargo clippy` | Запустить линтер |
| `cargo clippy -- -D warnings` | Линтинг с ошибками |
| `cargo add <crate>` | Добавить зависимость |
| `cargo add <crate> --features` | Добавить зависимость с фичами |
| `cargo remove <crate>` | Удалить зависимость |
| `cargo update` | Обновить `Cargo.lock` |
| `cargo update <crate>` | Обновить конкретную зависимость |
| `cargo upgrade` | Обновить зависимости до последних версий (cargo-upgrade) |
| `cargo tree` | Показать дерево зависимостей |
| `cargo tree --depth <n>` | Показать дерево с ограничением глубины |
| `cargo clean` | Очистить артефакты сборки |
| `cargo publish` | Опубликовать на crates.io |
| `cargo publish --dry-run` | Проверить перед публикацией |
| `cargo install <crate>` | Установить бинарный крейт |
| `cargo install --list` | Список установленных крейтов |
| `cargo outdated` | Проверить устаревшие зависимости (cargo-outdated) |
| `cargo audit` | Проверить уязвимости (cargo-audit) |
| `cargo deny` | Проверить лицензии и уязвимости (cargo-deny) |
| `cargo semver-checks` | Проверить SemVer (cargo-semver-checks) |

---

## F.2. `Cargo.toml` — манифест

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2024"
authors = ["Your Name <email@example.com>"]
description = "A brief description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/my_app"
documentation = "https://docs.rs/my_app"
homepage = "https://my_app.dev"
readme = "README.md"
keywords = ["keyword1", "keyword2"]
categories = ["development-tools", "science"]
rust-version = "1.85.0"
publish = ["crates-io"]

```

| Поле | Описание |
| --- | --- |
| `name` | Имя пакета |
| `version` | Версия (SemVer) |
| `edition` | Rust Edition (2018, 2021, 2024) |
| `authors` | Авторы |
| `description` | Краткое описание (до 100 символов) |
| `license` | Лицензия |
| `repository` | URL репозитория |
| `documentation` | URL документации |
| `readme` | Путь к README |
| `keywords` | Ключевые слова (до 5) |
| `categories` | Категории на crates.io |
| `rust-version` | Минимальная версия Rust (MSRV) |
| `publish` | Куда публиковать (crates-io) |

---

## F.3. Зависимости (Dependencies)

### Синтаксис версий

```toml
[dependencies]
serde = "1.0.0"         # >= 1.0.0 и < 2.0.0
serde = "=1.0.0"        # Только 1.0.0
serde = "^1.0.0"        # >= 1.0.0 и < 2.0.0 (как "1.0.0")
serde = "~1.0.0"        # >= 1.0.0 и < 1.1.0 (только патчи)
serde = "*"             # Любая версия (не рекомендуется)
serde = "1"             # >= 1.0.0 и < 2.0.0
serde = "1.0"           # >= 1.0.0 и < 1.1.0

```

### Источники зависимостей

```toml
[dependencies]
# crates.io
serde = "1.0"

# Git
my_lib = { git = "https://github.com/user/my_lib.git", branch = "main" }
my_lib = { git = "https://github.com/user/my_lib.git", tag = "v0.1.0" }
my_lib = { git = "https://github.com/user/my_lib.git", rev = "abc123" }

# Локальный путь
my_lib = { path = "../my_lib" }

# Опциональная зависимость
serde = { version = "1.0", optional = true }

```

### Другие типы зависимостей

```toml
[dev-dependencies]      # Только для тестов
pretty_assertions = "1.0"
tempfile = "3.0"

[build-dependencies]    # Только для build.rs
cc = "1.0"

[target.'cfg(windows)'.dependencies]  # Платформозависимые
winapi = "0.3"

```

---

## F.4. Features

```toml
[features]
# Включены по умолчанию
default = ["serde", "tokio"]

# Отдельные фичи
serde = ["dep:serde", "serde/derive"]
tokio = ["dep:tokio", "tokio/full"]
json = ["serde", "dep:serde_json"]

# Фича без зависимостей
no_std = []
full = ["serde", "tokio", "json"]

# Включение фич зависимостей
serde = { version = "1.0", features = ["derive"], optional = true }

```

### Использование в коде

```rust
// Условная компиляция с проверкой фич
fn main() {
    #[cfg(feature = "serde")]
    {
        println!("Фича 'serde' включена!");
    }

    #[cfg(not(feature = "serde"))]
    {
        println!("Фича 'serde' выключена.");
    }
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520main()%2520%257B%250A%2520%2520%2520%2520%2523%255Bcfg(feature%2520%253D%2520%2522serde%2522)%255D%250A%2520%2520%2520%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520println!(%2522%25D0%25A4%25D0%25B8%25D1%2587%25D0%25B0%2520%27serde%27%2520%25D0%25B2%25D0%25BA%25D0%25BB%25D1%258E%25D1%2587%25D0%25B5%25D0%25BD%25D0%25B0!%2522)%253B%250A%2520%2520%2520%2520%257D%250A%250A%2520%2520%2520%2520%2523%255Bcfg(not(feature%2520%253D%2520%2522serde%2522))%255D%250A%2520%2520%2520%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520println!(%2522%25D0%25A4%25D0%25B8%25D1%2587%25D0%25B0%2520%27serde%27%2520%25D0%25B2%25D1%258B%25D0%25BA%25D0%25BB%25D1%258E%25D1%2587%25D0%25B5%25D0%25BD%25D0%25B0.%2522)%253B%250A%2520%2520%2520%2520%257D%250A%257D)

### Включение фич

```bash
# Сборка с фичами
cargo build --features "full,json"
cargo build --features json
cargo build --no-default-features

```

---

## F.5. Профили (Profiles)

```toml
[profile.dev]              # cargo build
opt-level = 0
debug = true
split-debuginfo = '...'
debug-assertions = true
overflow-checks = true
incremental = true

[profile.release]          # cargo build --release
opt-level = 3
debug = false
split-debuginfo = '...'
debug-assertions = false
overflow-checks = false
incremental = false
lto = true                 # Link Time Optimization
codegen-units = 1          # Оптимизация

[profile.test]             # cargo test
opt-level = 0
debug = true

[profile.dev]              # cargo bench
opt-level = 3
debug = false

```

**Настройка opt-level:**

| opt-level | Описание |
| --- | --- |
| 0 | Без оптимизации (быстрая компиляция) |
| 1 | Базовая оптимизация |
| 2 | Средняя оптимизация |
| 3 | Максимальная оптимизация |
| "s" | Оптимизация по размеру |
| "z" | Минимальный размер |

---

## F.6. Workspace

```toml
[workspace]
members = [
    "crates/core",
    "crates/cli",
    "crates/server",
]

# Исключить крейты
exclude = ["crates/legacy"]

# Настройка резолвера
resolver = "2"

[workspace.dependencies]    # Общие зависимости
serde = "1.0"
tokio = { version = "1.0", features = ["full"] }
anyhow = "1.0"

[workspace.package]         # Наследуемые настройки
edition = "2024"
authors = ["Your Name <email@example.com>"]
license = "MIT"

[profile.release]           # Общие настройки профилей
lto = true
codegen-units = 1

```

### Наследование из workspace

```toml
# В дочернем крейте
[package]
name = "core"
version = "0.1.0"
edition = { workspace = true }
authors = { workspace = true }
license = { workspace = true }

[dependencies]
serde = { workspace = true }
tokio = { workspace = true }

```

---

## F.7. Lints

```toml
[lints]                     # Работает с Rust 1.74+
unused = "warn"
dead_code = "warn"
deprecated = "warn"

[lints.clippy]
pedantic = "warn"
nursery = "warn"

[lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

```

**Или в коде:**

```rust
#![warn(unused)]
#![warn(dead_code)]
#![deny(unsafe_code)]

fn main() {
    let unused_var = 42; // Вызовет предупреждение lints
    println!("Проверка настройки линтов в коде");
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3D%2523%255Bwarn(unused)%255D%250A%2523%255Bwarn(dead_code)%255D%250A%2523%255Bdeny(unsafe_code)%255D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520unused_var%2520%253D%252042%253B%250A%2520%2520%2520%2520println!(%2522%25D0%259F%25D1%2580%25D0%25BE%25D0%25B2%25D0%25B5%25D1%2580%25D0%25BA%25D0%25B0%2520%25D0%25BD%25D0%25B0%25D1%2581%25D1%2582%25D1%2580%25D0%25BE%25D0%25B9%25D0%25BA%25D0%25B8%2520%25D0%25BB%25D0%25B8%25D0%25BD%25D1%2582%25D0%25BE%25D0%25B2%2520%25D0%25B2%2520%25D0%25BA%25D0%25BE%25D0%25B4%25D0%25B5%2522)%253B%250A%257D)

---

## F.8. Основные расширения

```bash
# Установка расширений
cargo install cargo-outdated
cargo install cargo-audit
cargo install cargo-deny
cargo install cargo-semver-checks
cargo install cargo-expand
cargo install cargo-msrv
cargo install cargo-edit         # add/remove/upgrade

# Использование
cargo outdated
cargo audit
cargo deny check
cargo semver-checks
cargo expand
cargo msrv

```

---

## F.9. Переменные окружения

```bash
# Пути
CARGO_HOME=~/.cargo           # Домашняя директория Cargo
CARGO_TARGET_DIR=target      # Директория сборки

# Настройки сборки
RUSTFLAGS="-C target-cpu=native"  # Флаги компилятора
RUST_BACKTRACE=1              # Бэктрейс при панике
RUST_LOG=debug                # Уровень логирования

# Регистр
CARGO_REGISTRY_TOKEN=...      # Токен для crates.io
CARGO_NET_GIT_FETCH_WITH_CLI=true  # Использовать git для fetch

# Зависимости
CARGO_PROFILE_RELEASE_OPT_LEVEL=3  # Уровень оптимизации

```

---

## F.10. Шпаргалка по командам

| Действие | Команда |
| --- | --- |
| **Новый проект** | `cargo new my_project` |
| **Собрать debug** | `cargo build` |
| **Собрать release** | `cargo build --release` |
| **Запустить** | `cargo run` |
| **Проверить** | `cargo check` |
| **Тесты** | `cargo test` |
| **Документация** | `cargo doc --open` |
| **Формат** | `cargo fmt` |
| **Линтинг** | `cargo clippy` |
| **Добавить зависимость** | `cargo add serde` |
| **Удалить зависимость** | `cargo remove serde` |
| **Обновить зависимости** | `cargo update` |
| **Очистить** | `cargo clean` |
| **Публикация** | `cargo publish` |

---

### Главное из этого приложения

После этого приложения мы:

* **Знаем** все основные команды Cargo.
* **Умеем** настраивать `Cargo.toml`.
* **Понимаем** как управлять зависимостями.
* **Используем** features и profiles.
* **Работаем** с workspace.
* **Знаем** как управлять линтами.

**Самая важная идея:**

> Cargo — это не просто менеджер пакетов. Это всё, что нужно для управления проектом: сборка, тесты, документация, зависимости и публикация. Знание команд и настроек Cargo — ключ к эффективной разработке на Rust.