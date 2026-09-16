# Глава 77. Превращаем приложение в library

В предыдущей главе мы создали CLI-приложение. Но хороший код не должен быть заперт внутри одной программы.

Представим, что наше приложение умеет обрабатывать текстовые файлы:

- читать файлы;
- фильтровать строки;
- выполнять трансформации;
- сохранять результат;
- работать с конфигурацией.

Сегодня этим занимается CLI. Но завтра ту же функциональность может потребоваться:

- веб-сервису;
- GUI-приложению;
- другой CLI-утилите;
- серверу;
- тестам;
- WebAssembly-приложению.

Если вся логика находится в `main.rs`, повторно использовать её становится неудобно.

Поэтому мы разделим приложение на два крейта:

```text
                    file_processor
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
      file_processor_core     file_processor_cli
           library                  binary
              │                       │
              │                 CLI / arguments
              │                       │
              ▼                       ▼
       бизнес-логика              main.rs
```

**Библиотека содержит логику.**

**Бинарный крейт содержит интерфейс командной строки.**

Это одно из важных архитектурных разделений в реальных Rust-проектах.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 77.1. Почему library?

Выделение библиотеки даёт несколько преимуществ.

### 1. Переиспользуемость

Библиотеку можно использовать из разных приложений.

Например:

```text
                 file_processor_core
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
         CLI            Web            GUI
```

При этом бизнес-логика не дублируется.

### 2. Тестируемость

Основную функциональность можно тестировать независимо от CLI.

Например, тесту не нужно запускать программу и передавать ей аргументы:

```text
file_processor --config config.toml input.txt
```

Тест может напрямую вызвать:

```rust
let processor = FileProcessor::new(config);
let result = processor.process_files(&files)?;
```

### 3. Чёткая архитектура

CLI отвечает за:

- аргументы;
- переменные окружения;
- вывод сообщений;
- код завершения программы.

Library отвечает за:

- конфигурацию;
- обработку файлов;
- фильтрацию;
- трансформации;
- ошибки.

### 4. Независимость от интерфейса

Сегодня интерфейсом является CLI.

Завтра это может быть HTTP API:

```text
POST /process
```

Но библиотека при этом может остаться прежней.

### 5. Публикация

Библиотеку можно опубликовать как отдельный crate на crates.io и использовать в других проектах.

---

## 77.2. Новая структура проекта

Для небольшого проекта достаточно следующей структуры:

```text
file_processor/
├── Cargo.toml
├── Cargo.lock
│
└── crates/
    ├── file_processor_core/
    │   ├── Cargo.toml
    │   ├── src/
    │   │   ├── lib.rs
    │   │   ├── config.rs
    │   │   ├── error.rs
    │   │   ├── processor.rs
    │   │   └── types.rs
    │   └── tests/
    │       └── integration_tests.rs
    │
    └── file_processor_cli/
        ├── Cargo.toml
        └── src/
            ├── main.rs
            └── cli.rs
```

Здесь:

- `file_processor_core` — библиотечный crate;
- `file_processor_cli` — бинарный crate;
- корневой `Cargo.toml` — workspace.

Обратите внимание: **корневой workspace не обязан сам быть crate**.

Это называется **virtual workspace**.

---

## 77.3. Корневой `Cargo.toml`

```toml
[workspace]
members = [
    "crates/file_processor_core",
    "crates/file_processor_cli",
]

resolver = "3"

[workspace.dependencies]
clap = { version = "4.5", features = ["derive", "env"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.8"
regex = "1.12"
rayon = "1.11"
thiserror = "2.0"
pretty_assertions = "1.4"
tempfile = "3.23"

[profile.release]
lto = true
codegen-units = 1
```

Здесь стоит обратить внимание на:

```toml
[workspace.dependencies]
```

Зависимости объявляются один раз на уровне workspace.

После этого отдельный crate может написать:

```toml
serde = { workspace = true }
```

и получить ту же зависимость.

### Зачем нужен `resolver = "3"`?

Для современного Rust workspace используется современный алгоритм разрешения зависимостей:

```toml
resolver = "3"
```

Он особенно важен для workspace, где несколько crate могут иметь разные зависимости и feature flags.

---

## 77.4. Library core: `Cargo.toml`

```toml
[package]
name = "file_processor_core"
version = "0.1.0"
edition = "2024"
description = "Core library for processing text files"
license = "MIT"
repository = "https://github.com/yourname/file_processor"

[dependencies]
serde = { workspace = true }
serde_json = { workspace = true }
toml = { workspace = true }
regex = { workspace = true }
rayon = { workspace = true }
thiserror = { workspace = true }

[dev-dependencies]
pretty_assertions = { workspace = true }
tempfile = { workspace = true }
```

Главное отличие от CLI:

```toml
[package]
name = "file_processor_core"
```

и отсутствие:

```toml
[[bin]]
```

По умолчанию `src/lib.rs` означает, что это библиотечный crate.

---

## 77.5. Публичный API библиотеки: `src/lib.rs`

`lib.rs` является границей между внутренней реализацией библиотеки и её пользователем.

````rust
//! # File Processor Core
//!
//! Library for processing text files with filters and transformations.
//!
//! ## Example
//!
//! ```
//! use file_processor_core::{Config, FileProcessor};
//!
//! let processor = FileProcessor::new(Config::default());
//!
//! // Обработка файлов выполняется через FileProcessor.
//! // Здесь мы только показываем создание API.
//! let _ = processor;
//! ```

mod config;
mod error;
mod processor;
mod types;

pub use config::Config;
pub use error::{Error, Result};
pub use processor::{FileProcessor, ProcessingResult};
pub use types::{
    FilterConfig,
    OutputFormat,
    TransformConfig,
    TransformOperation,
};

pub mod prelude {
    pub use super::{
        Config,
        FileProcessor,
        ProcessingResult,
        Result,
    };
}
````

Обратите внимание на разницу:

```rust
mod processor;
```

означает:

> модуль существует внутри библиотеки, но его реализация не является частью публичного API.

А:

```rust
pub use processor::FileProcessor;
```

означает:

> пользователь библиотеки может использовать `FileProcessor`.

Поэтому внешний код пишет:

```rust
use file_processor_core::FileProcessor;
```

а не:

```rust
use file_processor_core::processor::FileProcessor;
```

Это позволяет менять внутреннюю структуру библиотеки, не меняя публичный API.

---

## 77.6. Доменные типы: `types.rs`

Начнём с типов, которые описывают предметную область.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub enum OutputFormat {
    #[default]
    Text,
    Json,
    Csv,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilterConfig {
    pub name: String,
    pub pattern: String,
    #[serde(default)]
    pub invert: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransformConfig {
    pub name: String,
    pub operation: TransformOperation,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TransformOperation {
    Uppercase,
    Lowercase,
    Replace {
        from: String,
        to: String,
    },
    RemoveEmpty,
    Trim,
}

impl TransformOperation {
    pub fn apply(&self, lines: Vec<String>) -> Vec<String> {
        match self {
            Self::Uppercase => lines
                .into_iter()
                .map(|s| s.to_uppercase())
                .collect(),

            Self::Lowercase => lines
                .into_iter()
                .map(|s| s.to_lowercase())
                .collect(),

            Self::Replace { from, to } => lines
                .into_iter()
                .map(|s| s.replace(from, to))
                .collect(),

            Self::RemoveEmpty => lines
                .into_iter()
                .filter(|s| !s.trim().is_empty())
                .collect(),

            Self::Trim => lines
                .into_iter()
                .map(|s| s.trim().to_string())
                .collect(),
        }
    }
}
```

Здесь хорошо видно одну из сильных сторон Rust: поведение можно выразить непосредственно через enum.

```rust
match self {
    Self::Uppercase => ...,
    Self::Lowercase => ...,
    Self::Replace { from, to } => ...,
    Self::RemoveEmpty => ...,
    Self::Trim => ...,
}
```

Если позже мы добавим новый вариант:

```rust
NormalizeWhitespace
```

компилятор поможет найти все `match`, которые необходимо обновить.

### Небольшой самостоятельный пример

Этот пример не требует файловой системы и зависимостей, поэтому его удобно запускать в Rust Playground.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+lines+%3D+vec%21%5B%0A++++++++%22++Hello+Rust++%22.to_string%28%29%2C%0A++++++++%22++World++%22.to_string%28%29%2C%0A++++%5D%3B%0A%0A++++let+result%3A+Vec%3C_%3E+%3D+lines%0A++++++++.into_iter%28%29%0A++++++++.map%28%7Bs%7C+s.trim%28%29.to_uppercase%28%29%7D%29%0A++++++++.collect%28%29%3B%0A%0A++++println%21%28%22%7Bresult%3A%3F%7D%22%2C+result%29%3B%0A%7D)

---

## 77.7. Ошибки библиотеки: `error.rs`

Библиотека не должна печатать ошибки вместо своего пользователя.

Она должна **возвращать ошибку**.

Для этого создадим собственный тип:

```rust
use std::io;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("file not found: {0}")]
    FileNotFound(String),

    #[error("file is too large: {size} bytes (maximum {max} bytes)")]
    FileTooLarge {
        size: u64,
        max: u64,
    },

    #[error("configuration error: {0}")]
    Config(String),

    #[error("processing error: {0}")]
    Processing(String),

    #[error("I/O error: {0}")]
    Io(#[from] io::Error),

    #[error("invalid regular expression: {0}")]
    Regex(#[from] regex::Error),
}

pub type Result<T> = std::result::Result<T, Error>;
```

Теперь библиотека может написать:

```rust
return Err(Error::FileNotFound(path.display().to_string()));
```

или:

```rust
let content = std::fs::read_to_string(path)?;
```

Оператор `?` автоматически преобразует `io::Error` в наш `Error` благодаря:

```rust
#[from] io::Error
```

Это важный принцип архитектуры:

> **Library сообщает об ошибке. Application решает, как её показать пользователю.**

CLI может вывести ошибку в терминал.

HTTP-сервис может превратить её в HTTP response.

GUI может показать dialog.

Библиотека не должна знать ни о CLI, ни о HTTP, ни о GUI.

---

## 77.8. Конфигурация: `config.rs`

Теперь создадим конфигурацию.

```rust
use serde::{Deserialize, Serialize};
use std::fs;
use std::path::Path;

use crate::error::{Error, Result};
use crate::types::{
    FilterConfig,
    OutputFormat,
    TransformConfig,
};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    pub max_file_size: Option<u64>,
    pub output_format: OutputFormat,
    pub filters: Vec<FilterConfig>,
    pub transforms: Vec<TransformConfig>,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            max_file_size: Some(10 * 1024 * 1024),
            output_format: OutputFormat::Text,
            filters: Vec::new(),
            transforms: Vec::new(),
        }
    }
}

impl Config {
    pub fn load(path: Option<&Path>) -> Result<Self> {
        let Some(path) = path else {
            return Ok(Self::default());
        };

        if !path.exists() {
            return Err(Error::FileNotFound(path.display().to_string()));
        }

        let content = fs::read_to_string(path)?;

        match path.extension().and_then(|ext| ext.to_str()) {
            Some("toml") => {
                toml::from_str(&content)
                    .map_err(|e| Error::Config(e.to_string()))
            }

            Some("json") => {
                serde_json::from_str(&content)
                    .map_err(|e| Error::Config(e.to_string()))
            }

            Some(extension) => Err(Error::Config(format!(
                "unsupported configuration format: .{extension}"
            ))),

            None => Err(Error::Config(
                "configuration file must have .toml or .json extension"
                    .to_string(),
            )),
        }
    }

    pub fn save(&self, path: &Path) -> Result<()> {
        let content = match path.extension().and_then(|ext| ext.to_str()) {
            Some("toml") => {
                toml::to_string_pretty(self)
                    .map_err(|e| Error::Config(e.to_string()))?
            }

            Some("json") => {
                serde_json::to_string_pretty(self)
                    .map_err(|e| Error::Config(e.to_string()))?
            }

            Some(extension) => {
                return Err(Error::Config(format!(
                    "unsupported configuration format: .{extension}"
                )));
            }

            None => {
                return Err(Error::Config(
                    "configuration file must have .toml or .json extension"
                        .to_string(),
                ));
            }
        };

        fs::write(path, content)?;
        Ok(())
    }
}
```

Важное изменение по сравнению с исходным вариантом: мы не храним настройку `default_encoding`.

Причина проста:

```rust
fs::read_to_string(path)
```

работает с UTF-8.

Если приложению действительно потребуется поддержка других кодировок, это должна быть отдельная функциональность с соответствующим декодером, а не настройка, которая существует только в структуре `Config`.

---

## 77.9. Основная логика: `processor.rs`

Теперь создадим главный объект библиотеки.

```rust
use rayon::prelude::*;
use regex::Regex;
use std::fs;
use std::path::{Path, PathBuf};

use crate::config::Config;
use crate::error::{Error, Result};

pub struct FileProcessor {
    config: Config,
}

#[derive(Debug, Default)]
pub struct ProcessingResult {
    pub files_processed: usize,
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub output: Vec<String>,
}

impl FileProcessor {
    pub fn new(config: Config) -> Self {
        Self { config }
    }

    pub fn process_files(
        &self,
        files: &[PathBuf],
    ) -> Result<ProcessingResult> {
        let results: Vec<Result<(u64, Vec<String>)>> = files
            .par_iter()
            .map(|path| self.process_single_file(path))
            .collect();

        let mut result = ProcessingResult::default();

        for processed in results {
            let (bytes, lines) = processed?;

            result.files_processed += 1;
            result.lines_processed += lines.len();
            result.bytes_processed += bytes;
            result.output.extend(lines);
        }

        Ok(result)
    }

    fn process_single_file(
        &self,
        path: &Path,
    ) -> Result<(u64, Vec<String>)> {
        if !path.exists() {
            return Err(Error::FileNotFound(
                path.display().to_string(),
            ));
        }

        let metadata = fs::metadata(path)?;
        let bytes = metadata.len();

        if let Some(max_size) = self.config.max_file_size
            && bytes > max_size
        {
            return Err(Error::FileTooLarge {
                size: bytes,
                max: max_size,
            });
        }

        let content = fs::read_to_string(path)?;

        let mut lines: Vec<String> = content
            .lines()
            .map(str::to_string)
            .collect();

        lines = self.apply_filters(lines)?;
        lines = self.apply_transforms(lines);

        Ok((bytes, lines))
    }

    fn apply_filters(
        &self,
        lines: Vec<String>,
    ) -> Result<Vec<String>> {
        let mut result = lines;

        for filter in &self.config.filters {
            let pattern = Regex::new(&filter.pattern)?;

            result = result
                .into_iter()
                .filter(|line| {
                    let matches = pattern.is_match(line);

                    if filter.invert {
                        !matches
                    } else {
                        matches
                    }
                })
                .collect();
        }

        Ok(result)
    }

    fn apply_transforms(
        &self,
        mut lines: Vec<String>,
    ) -> Vec<String> {
        for transform in &self.config.transforms {
            lines = transform.operation.apply(lines);
        }

        lines
    }

    pub fn save_output(
        &self,
        output: &[String],
        path: &Path,
    ) -> Result<()> {
        let content = output.join("\n");
        fs::write(path, content)?;
        Ok(())
    }
}
```

Здесь есть важный архитектурный момент.

### Ошибка больше не теряется

В исходном варианте было:

```rust
Err(e) => {
    eprintln!("Error processing file: {}", e);
}
```

То есть библиотека сама печатала ошибку и продолжала работу.

Это плохо.

Теперь:

```rust
let (bytes, lines) = processed?;
```

Если хотя бы один файл завершился ошибкой, ошибка возвращается вызывающему коду.

Например:

```rust
let result = processor.process_files(&files)?;

println!("{:?}", result);
```

Теперь приложение само решает, что делать с ошибкой.

### Почему `bytes_processed` считается до трансформаций?

Предположим, исходный файл содержит:

```text
hello
```

А трансформация делает:

```text
HELLO HELLO HELLO
```

Размер результата уже не равен размеру исходного файла.

Поэтому:

```rust
let bytes = metadata.len();
```

фиксирует именно размер исходного файла.

Это делает имя `bytes_processed` однозначным.

---

## 77.10. CLI: `Cargo.toml`

Теперь создаём бинарный crate.

```toml
[package]
name = "file_processor_cli"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "file_processor"
path = "src/main.rs"

[dependencies]
clap = { workspace = true }
file_processor_core = { path = "../file_processor_core" }
```

Обратите внимание: `anyhow` здесь больше не нужен.

Библиотека уже предоставляет собственный:

```rust
file_processor_core::Result
```

CLI может использовать его непосредственно.

---

## 77.11. CLI-модуль: `cli.rs`

```rust
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(
    name = "file_processor",
    about = "Process text files",
    version,
    author
)]
pub struct Cli {
    /// Путь к конфигурационному файлу
    #[arg(short, long, env = "FP_CONFIG")]
    pub config: Option<PathBuf>,

    /// Файлы для обработки
    #[arg(required = true)]
    pub files: Vec<PathBuf>,
}
```

Этот модуль занимается исключительно CLI.

Он не знает:

- как фильтруются строки;
- как работают трансформации;
- как читаются файлы;
- как реализована библиотека.

Это и есть разделение ответственности.

---

## 77.12. CLI: `main.rs`

```rust
use clap::Parser;

use file_processor_core::{
    Config,
    FileProcessor,
    Result,
};

mod cli;

use cli::Cli;

fn main() -> Result<()> {
    let cli = Cli::parse();

    let config = Config::load(cli.config.as_deref())?;

    let processor = FileProcessor::new(config);

    let result = processor.process_files(&cli.files)?;

    println!(
        "Processed {} files, {} lines, {} bytes",
        result.files_processed,
        result.lines_processed,
        result.bytes_processed,
    );

    if !result.output.is_empty() {
        println!("\n--- Output ---");

        for line in result.output {
            println!("{line}");
        }
    }

    Ok(())
}
```

Теперь `main.rs` действительно стал тонкой оболочкой.

Его задача:

```text
CLI arguments
     │
     ▼
Config
     │
     ▼
FileProcessor
     │
     ▼
ProcessingResult
     │
     ▼
terminal output
```

А вся предметная логика находится в library.

---

## 77.13. Проверяем workspace

Теперь можно запускать команды из корня проекта.

Проверка всех crate:

```bash
cargo check --workspace
```

Сборка:

```bash
cargo build --workspace
```

Тестирование:

```bash
cargo test --workspace
```

Сборка release:

```bash
cargo build --workspace --release
```

Это одна из причин использовать workspace: одна команда управляет всем проектом.

---

## 77.14. Интеграционные тесты библиотеки

Интеграционные тесты располагаются внутри библиотечного crate:

```text
crates/
└── file_processor_core/
    ├── src/
    └── tests/
        └── integration_tests.rs
```

Создадим тесты:

```rust
use file_processor_core::{
    Config,
    FileProcessor,
    FilterConfig,
    TransformConfig,
    TransformOperation,
};

use std::fs;

use tempfile::tempdir;

#[test]
fn test_basic_processing() {
    let dir = tempdir().unwrap();
    let file_path = dir.path().join("test.txt");

    fs::write(
        &file_path,
        "hello\nworld\nRust\n",
    )
    .unwrap();

    let processor = FileProcessor::new(Config::default());

    let result = processor
        .process_files(std::slice::from_ref(&file_path))
        .unwrap();

    assert_eq!(result.files_processed, 1);
    assert_eq!(result.lines_processed, 3);
    assert_eq!(result.output, vec![
        "hello",
        "world",
        "Rust",
    ]);
}

#[test]
fn test_filter_and_transform() {
    let dir = tempdir().unwrap();
    let file_path = dir.path().join("test.txt");

    fs::write(
        &file_path,
        "hello\nworld\nRUST\nrust\n",
    )
    .unwrap();

    let mut config = Config::default();

    config.filters.push(FilterConfig {
        name: "contains_r".to_string(),
        pattern: "[rR]".to_string(),
        invert: false,
    });

    config.transforms.push(TransformConfig {
        name: "uppercase".to_string(),
        operation: TransformOperation::Uppercase,
    });

    let processor = FileProcessor::new(config);

    let result = processor
        .process_files(std::slice::from_ref(&file_path))
        .unwrap();

    assert_eq!(
        result.output,
        vec!["HELLO", "RUST", "RUST"]
    );
}

#[test]
fn test_file_not_found() {
    let processor = FileProcessor::new(Config::default());

    let result = processor.process_files(&[
        "nonexistent.txt".into(),
    ]);

    assert!(result.is_err());

    let error = result.unwrap_err();

    assert!(
        error.to_string().contains("file not found")
    );
}

#[test]
fn test_remove_empty_lines() {
    let dir = tempdir().unwrap();
    let file_path = dir.path().join("test.txt");

    fs::write(
        &file_path,
        "hello\n\nworld\n\nrust\n",
    )
    .unwrap();

    let mut config = Config::default();

    config.transforms.push(TransformConfig {
        name: "remove_empty".to_string(),
        operation: TransformOperation::RemoveEmpty,
    });

    let processor = FileProcessor::new(config);

    let result = processor
        .process_files(std::slice::from_ref(&file_path))
        .unwrap();

    assert_eq!(
        result.output,
        vec!["hello", "world", "rust"]
    );
}

#[test]
fn test_trim() {
    let dir = tempdir().unwrap();
    let file_path = dir.path().join("test.txt");

    fs::write(
        &file_path,
        "  hello  \n world \n",
    )
    .unwrap();

    let mut config = Config::default();

    config.transforms.push(TransformConfig {
        name: "trim".to_string(),
        operation: TransformOperation::Trim,
    });

    let processor = FileProcessor::new(config);

    let result = processor
        .process_files(std::slice::from_ref(&file_path))
        .unwrap();

    assert_eq!(
        result.output,
        vec!["hello", "world"]
    );
}
```

Запускаем:

```bash
cargo test --workspace
```

Теперь тестируется именно библиотека, а не CLI.

---

## 77.15. Unit-тесты и integration-тесты

В Rust важно понимать разницу.

**Unit-тесты** обычно находятся рядом с реализацией:

```text
src/
├── processor.rs
└── ...
```

и используют:

```rust
#[cfg(test)]
mod tests {
    ...
}
```

Они могут обращаться к приватным элементам модуля.

**Integration-тесты** находятся в:

```text
tests/
```

Они используют библиотеку так, как её использует внешний пользователь.

Например:

```rust
use file_processor_core::FileProcessor;
```

Именно поэтому интеграционные тесты особенно полезны для проверки публичного API.

---

## 77.16. Документация публичного API

Библиотека должна документировать не только `lib.rs`, но и публичные типы и методы.

Например:

```rust
/// Processes text files according to the supplied configuration.
pub struct FileProcessor {
    config: Config,
}
```

И:

```rust
impl FileProcessor {
    /// Creates a processor with the specified configuration.
    pub fn new(config: Config) -> Self {
        Self { config }
    }
}
```

Для результата:

```rust
/// Result of processing one or more files.
#[derive(Debug, Default)]
pub struct ProcessingResult {
    /// Number of successfully processed files.
    pub files_processed: usize,

    /// Number of lines after processing.
    pub lines_processed: usize,

    /// Number of bytes in the original files.
    pub bytes_processed: u64,

    /// Processed output lines.
    pub output: Vec<String>,
}
```

Теперь пользователь библиотеки может понять API непосредственно из документации.

---

## 77.17. Генерация документации

Из корня workspace:

```bash
cargo doc --workspace --open
```

Если нужно проверить документацию без предупреждений:

```bash
cargo doc --workspace
```

Можно также использовать:

```bash
cargo test --doc --workspace
```

Последняя команда запускает примеры из Rustdoc как тесты.

Это особенно полезно для такого блока:

````rust
/// ```
/// use file_processor_core::Config;
///
/// let config = Config::default();
/// assert!(config.max_file_size.is_some());
/// ```
````

Документационный пример становится одновременно **частью документации и тестом**.

---

## 77.18. Публикация библиотеки

Перед публикацией полезно проверить crate:

```bash
cargo package -p file_processor_core
```

Затем можно выполнить:

```bash
cargo publish -p file_processor_core
```

Но перед настоящей публикацией необходимо заменить:

```toml
repository = "https://github.com/yourname/file_processor"
```

на настоящий URL репозитория.

Кроме того, библиотека должна иметь корректные:

- `name`;
- `version`;
- `description`;
- `license`;
- `repository`;
- документацию;
- публичный API.

И самое главное: **не следует публиковать библиотеку только потому, что технически это возможно**.

Публикация на crates.io — это уже обещание другим разработчикам поддерживать определённый API.

---

# 🔨 Эксперименты с компилятором

Теперь посмотрим, что произойдёт, если нарушить архитектурные границы.

## Эксперимент 1. Циклическая зависимость

Попробуйте сделать так:

```text
file_processor_core
        │
        ▼
file_processor_cli
        │
        ▼
file_processor_core
```

Например, добавьте в `file_processor_core/Cargo.toml`:

```toml
file_processor_cli = {
    path = "../file_processor_cli"
}
```

При этом CLI уже зависит от:

```toml
file_processor_core = {
    path = "../file_processor_core"
}
```

Получается:

```text
core ──────► cli
 ▲           │
 └───────────┘
```

Cargo сообщит о циклической зависимости.

Это не просто ограничение Cargo.

Это архитектурный сигнал.

Правильное направление зависимостей:

```text
CLI ─────────► Library
```

а не:

```text
CLI ◄────────► Library
```

---

## Эксперимент 2. Несуществующий crate в workspace

Измените:

```toml
[workspace]
members = [
    "crates/file_processor_core",
    "crates/file_processor_cli",
    "nonexistent",
]
```

и выполните:

```bash
cargo check
```

Cargo не сможет найти указанный член workspace.

Здесь полезно увидеть принцип:

> `workspace.members` — это не список желаемых проектов. Это список реальных путей к crate.

---

## Эксперимент 3. Попытка использовать приватный элемент

В `processor.rs` оставьте:

```rust
pub struct FileProcessor {
    config: Config,
}
```

Поле `config` является приватным.

Теперь в CLI попробуйте:

```rust
let processor = FileProcessor::new(Config::default());

println!("{:?}", processor.config);
```

Компилятор сообщит, что поле `config` является приватным.

Это правильно.

CLI должен работать через публичный API:

```rust
FileProcessor::new(...)
```

а не знать внутреннее устройство `FileProcessor`.

---

## Эксперимент 4. Изменение публичного API

Пусть было:

```rust
pub fn new(config: Config) -> Self
```

А затем мы изменили его на:

```rust
pub fn new() -> Self
```

CLI, использующий:

```rust
FileProcessor::new(config)
```

перестанет компилироваться.

Это показывает важную вещь:

> Публичный API — это контракт.

Пока тип или функция приватны, мы можем менять их свободнее.

Когда мы делаем их `pub`, изменения начинают затрагивать пользователей библиотеки.

---

# Практика

## Задание 1

Создайте workspace со следующей структурой:

```text
file_processor/
├── Cargo.toml
└── crates/
    ├── file_processor_core/
    └── file_processor_cli/
```

Убедитесь, что:

```bash
cargo check --workspace
```

проходит успешно.

---

## Задание 2

Перенесите всю логику обработки файлов в:

```text
file_processor_core
```

В CLI оставьте только:

- разбор аргументов;
- загрузку конфигурации;
- вызов библиотеки;
- вывод результата.

---

## Задание 3

Добавьте новую трансформацию:

```rust
Replace {
    from: String,
    to: String,
}
```

и протестируйте её.

Например:

```text
Hello Rust
```

должно превращаться в:

```text
Hello WebAssembly
```

при конфигурации:

```rust
TransformOperation::Replace {
    from: "Rust".to_string(),
    to: "WebAssembly".to_string(),
}
```

---

## Задание 4

Добавьте тест для слишком большого файла.

Используйте:

```rust
max_file_size
```

и убедитесь, что библиотека возвращает:

```rust
Error::FileTooLarge { .. }
```

а не печатает ошибку самостоятельно.

---

## Задание 5

Добавьте документационный пример для:

```rust
FileProcessor::new()
```

и проверьте его:

```bash
cargo test --doc --workspace
```

---

## Задание 6 — архитектурный эксперимент

Создайте временную функцию в библиотеке:

```rust
fn internal_helper() {
    println!("internal");
}
```

Попробуйте вызвать её из CLI.

Затем измените её на:

```rust
pub fn internal_helper() {
    println!("internal");
}
```

Сравните поведение компилятора.

После эксперимента снова сделайте функцию приватной.

---

## Задание 7 — новый потребитель библиотеки

Создайте третий crate:

```text
crates/
├── file_processor_core/
├── file_processor_cli/
└── file_processor_demo/
```

`file_processor_demo` должен использовать:

```rust
file_processor_core
```

напрямую.

Его задача — показать, что библиотека действительно не зависит от CLI.

Получится:

```text
                 file_processor_core
                    ▲          ▲
                    │          │
                    │          │
          file_processor_cli   │
                               │
                    file_processor_demo
```

Это и есть главное преимущество выделения library.

---

# Главное из этой главы

После этой главы мы:

- **выделили** бизнес-логику в библиотечный crate;
- **оставили** CLI отдельным бинарным crate;
- **организовали** оба crate в workspace;
- **определили** публичную границу библиотеки через `lib.rs`;
- **создали** собственный тип ошибок;
- **перестали** скрывать ошибки внутри библиотеки;
- **написали** интеграционные тесты;
- **добавили** документацию публичного API;
- **научились** проверять весь workspace одной командой;
- **увидели**, как библиотека может использоваться независимо от CLI.

Самое важное изменение произошло не в структуре каталогов.

Мы изменили **границу ответственности**:

```text
┌─────────────────────────────────────────────┐
│              Application / CLI              │
│                                             │
│  arguments → configuration → output         │
└──────────────────────┬──────────────────────┘
                       │
                       │ public API
                       ▼
┌─────────────────────────────────────────────┐
│                Library                      │
│                                             │
│  domain types                               │
│  business logic                             │
│  file processing                            │
│  transformations                            │
│  errors                                     │
└─────────────────────────────────────────────┘
```

CLI теперь **не содержит бизнес-логику**.

Library теперь **не знает о CLI**.

Это означает, что завтра мы можем добавить:

```text
                 file_processor_core
                         ▲
             ┌───────────┼───────────┐
             │           │           │
             │           │           │
            CLI         Web          GUI
             │           │           │
          terminal      HTTP       desktop
```

и все эти приложения будут использовать одну и ту же библиотеку.

> **Library — это не просто `main.rs`, перемещённый в `lib.rs`.**
>
> Это архитектурная граница, которая отделяет **что приложение делает** от **того, как пользователь с ним взаимодействует**.

Именно поэтому выделение библиотеки — один из первых шагов от маленького учебного приложения к архитектуре реального Rust-проекта.
