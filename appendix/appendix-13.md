# Приложение M. Rust Developer Toolchain

Полный набор инструментов для разработки на Rust: от установки и компиляции до форматирования, линтинга, тестирования и аудита безопасности.

---

## M.1. `rustup` — управление версиями Rust

**Назначение:** Установка и управление версиями Rust, целями и компонентами.

```bash
# Установка
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Основные команды
rustup update              # Обновить все компоненты
rustup update stable       # Обновить stable
rustup default stable      # Установить stable по умолчанию
rustup show                # Показать установленные версии
rustup toolchain list      # Список toolchain-ов

# Цели (targets)
rustup target list
rustup target add wasm32-unknown-unknown

# Компоненты
rustup component list
rustup component add rustfmt
rustup component add clippy
rustup component add rust-analyzer

# Удаление
rustup self uninstall
```

**Каналы:**

| Канал     | Описание                                      |
| --------- | --------------------------------------------- |
| `stable`  | Стабильная версия (по умолчанию)              |
| `beta`    | Бета-версия (перед релизом)                   |
| `nightly` | Ежедневная сборка (экспериментальные функции) |

---

## M.2. `rustc` — компилятор Rust

**Назначение:** Компиляция Rust-кода.

```bash
rustc --version
rustc --help
rustc main.rs
rustc -O main.rs
rustc -C target-cpu=native main.rs

# Объяснение ошибок
rustc --explain E0308

rustc --print target-cpus
rustc --print target-features
```

---

## M.3. `cargo` — система сборки и управления пакетами

**Назначение:** Сборка, тестирование, зависимости, публикация.

```bash
# Проект
cargo new my_project
cargo init

# Сборка
cargo build
cargo build --release
cargo build --target wasm32-unknown-unknown
cargo check

# Запуск
cargo run
cargo run -- arg1 arg2

# Тесты
cargo test
cargo test -- --nocapture
cargo test test_name

# Документация
cargo doc
cargo doc --open

# Зависимости
cargo add serde
cargo add serde --features derive
cargo remove serde
cargo update
cargo update serde

# Публикация
cargo publish
cargo publish --dry-run

# Другое
cargo clean
cargo tree
cargo tree --depth 2
cargo install cargo-outdated
```

---

## M.4. `rustfmt` — форматирование кода

**Назначение:** Автоматическое форматирование по стилю Rust.

```bash
cargo fmt
cargo fmt --check
rustfmt src/main.rs
rustfmt --check src/main.rs
```

**`rustfmt.toml` (пример):**

```toml
edition = "2024"
max_width = 100
hard_tabs = false
tab_spaces = 4
newline_style = "Unix"
```

**Игнорирование:**

```rust
#[rustfmt::skip]
fn unformatted_function() {
    // не форматировать
}
```

---

## M.5. `clippy` — линтер

**Назначение:** Поиск ошибок, неоптимальностей и неидиоматичного кода.

```bash
cargo clippy
cargo clippy -- -D warnings
cargo clippy -- -W clippy::pedantic
cargo clippy -- -A clippy::needless_pass_by_value
cargo clippy --fix
```

**Категории линтов:**

| Категория             | Описание                |
| --------------------- | ----------------------- |
| `clippy::style`       | Стиль и идиоматичность  |
| `clippy::complexity`  | Избыточная сложность    |
| `clippy::perf`        | Производительность      |
| `clippy::correctness` | Потенциальные ошибки    |
| `clippy::suspicious`  | Подозрительные паттерны |
| `clippy::pedantic`    | Строгие (полезные)      |
| `clippy::nursery`     | Экспериментальные       |

```rust
#[allow(clippy::needless_pass_by_value)]
fn process(data: String) {
    // ...
}
```

---

## M.6. `rust-analyzer` — LSP-сервер

**Назначение:** Интеграция с редакторами (VS Code, Neovim, Emacs и др.).

```bash
rustup component add rust-analyzer
# или
cargo install rust-analyzer --locked
```

**Возможности:**
- Go to definition
- Find references
- Peek definition
- Hover (типы, документация)
- Code completion
- Rename
- Inlay hints

**Пример настроек VS Code (`settings.json`):**

```json
{
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.procMacro.enable": true,
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.inlayHints.typeHints.enable": true
}
```

---

## M.7. `cargo-audit` — аудит безопасности

**Назначение:** Проверка зависимостей на известные уязвимости.

```bash
cargo install cargo-audit

cargo audit
cargo audit --ignore RUSTSEC-2024-0001
cargo audit --deny warnings
```

**`audit.toml` (пример):**

```toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
```

---

## M.8. `cargo-deny` — лицензии и безопасность

**Назначение:** Проверка лицензий, уязвимостей и источников зависимостей.

```bash
cargo install cargo-deny

cargo deny check
cargo deny check advisories
cargo deny check licenses
cargo deny check sources
cargo deny init
```

**`deny.toml` (фрагмент):**

```toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[licenses]
unlicensed = "deny"
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "ISC"]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

---

## M.9. `cargo-nextest` — быстрый тестовый раннер

**Назначение:** Быстрый параллельный запуск тестов.

```bash
cargo install cargo-nextest --locked

cargo nextest run
cargo nextest run --release
cargo nextest run --no-run
cargo nextest run -E 'test(test_name)'
```

**Преимущества:**
- Обычно быстрее `cargo test`
- Удобная параллельность и отчёты
- Гибкая фильтрация тестов

---

## M.10. Другие полезные инструменты

| Инструмент            | Назначение                       | Установка                           |
| --------------------- | -------------------------------- | ----------------------------------- |
| `cargo-outdated`      | Устаревшие зависимости           | `cargo install cargo-outdated`      |
| `cargo-semver-checks` | Проверка SemVer API              | `cargo install cargo-semver-checks` |
| `cargo-expand`        | Развёртывание макросов           | `cargo install cargo-expand`        |
| `cargo-msrv`          | Минимальная поддерживаемая версия| `cargo install cargo-msrv`          |
| `cargo-tarpaulin`     | Покрытие кода                    | `cargo install cargo-tarpaulin`     |
| `cargo-flamegraph`    | Flamegraph                       | `cargo install flamegraph`          |
| `probe-rs`            | Embedded-отладка                 | `cargo install probe-rs-tools`      |

---

## M.11. Чек-лист: настройка окружения

| Шаг | Действие                   |
| --- | -------------------------- |
| 1   | Установить `rustup`        |
| 2   | Установить `stable`        |
| 3   | Установить `rustfmt`       |
| 4   | Установить `clippy`        |
| 5   | Установить `rust-analyzer` |
| 6   | Установить `cargo-audit`   |
| 7   | Установить `cargo-deny`    |
| 8   | (Опционально) `cargo-nextest` |
| 9   | Настроить редактор         |
| 10  | Настроить CI               |

---

### Главное из этого приложения

После этого приложения мы:

- **Знаем** основные инструменты разработки на Rust.
- **Умеем** устанавливать и настраивать их.
- **Понимаем**, когда какой инструмент уместен.
- **Можем** собрать полноценное окружение для работы.

**Самая важная идея:**

> Инструменты делают разработку на Rust эффективной: `rustup` управляет версиями, `cargo` — проектами, `rustfmt` форматирует, `clippy` анализирует, `rust-analyzer` помогает в редакторе. Встройте их в ежедневный процесс — и качество кода вырастет.