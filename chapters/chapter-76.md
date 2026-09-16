# Часть XVIII. Практический Rust-проект

До этого момента мы изучали Rust по отдельным темам: типы, ownership, borrowing, traits, обработку ошибок, работу с файлами, конфигурацию, тестирование и параллелизм.

Теперь эти знания нужно объединить.

Мы создадим полноценное CLI-приложение **File Processor** — утилиту для обработки текстовых файлов.

Приложение будет уметь:

- читать текстовые файлы;
- фильтровать строки по регулярному выражению;
- преобразовывать строки;
- показывать статистику;
- работать с конфигурационным файлом;
- обрабатывать несколько файлов параллельно;
- ограничивать размер обрабатываемого файла;
- возвращать понятные ошибки;
- иметь unit- и integration-тесты.

Главная цель этой главы — не количество функций.

Мы хотим увидеть, **как отдельные возможности Rust соединяются в реальном проекте**.

Все примеры используют **Rust Edition 2024**.

---

## Глава 76. Создаём CLI-приложение

### 76.1. Что мы создаём

Наше приложение будет запускаться примерно так:

```bash
file_processor input.txt
```

Можно применить фильтр:

```bash
file_processor filter --pattern "Rust" input.txt
```

Преобразовать строки:

```bash
file_processor transform --uppercase input.txt
```

Получить статистику:

```bash
file_processor stats input.txt
```

Или использовать конфигурационный файл:

```bash
file_processor --config config.toml input.txt
```

Архитектура будет выглядеть следующим образом:

```text
                    ┌─────────────────────┐
                    │     Command Line    │
                    │        clap         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Config        │
                    │ TOML / JSON / CLI   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   FileProcessor     │
                    ├─────────────────────┤
                    │ read files          │
                    │ filter lines        │
                    │ transform lines     │
                    │ calculate statistics│
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
      ┌───────────────┐                 ┌───────────────┐
      │ Error handling│                 │   Reporting   │
      │   thiserror   │                 │ text / json   │
      └───────────────┘                 └───────────────┘
```

---

## 76.2. Создание проекта

Создадим новый Cargo project:

```bash
cargo new file_processor
cd file_processor
```

Проверим его:

```bash
cargo run
```

Получим:

```text
Hello, world!
```

Теперь заменим стандартное приложение нашей программой.

---

## 76.3. Структура проекта

Используем следующую структуру:

```text
file_processor/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── cli.rs
│   ├── config.rs
│   ├── error.rs
│   └── processor.rs
└── tests/
    └── integration_test.rs
```

Здесь важно разделить **CLI** и **бизнес-логику**.

`main.rs` не должен содержать всю программу.

Он должен быть тонким слоем:

```text
CLI
 ↓
Config
 ↓
Processor
 ↓
Result
```

Это делает программу проще для тестирования и повторного использования.

---

## 76.4. Cargo.toml

Используем следующие зависимости:

```toml
[package]
name = "file_processor"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = "1"
clap = { version = "4", features = ["derive", "env"] }
rayon = "1"
regex = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
toml = "0.8"

[dev-dependencies]
tempfile = "3"
```

Каждая библиотека имеет свою ответственность:

| Crate        | Назначение                    |
| ------------ | ----------------------------- |
| `clap`       | CLI                           |
| `thiserror`  | типизированные ошибки         |
| `anyhow`     | ошибки на границе приложения  |
| `serde`      | сериализация и десериализация |
| `serde_json` | JSON                          |
| `toml`       | TOML                          |
| `regex`      | регулярные выражения          |
| `rayon`      | параллельная обработка        |
| `tempfile`   | временные файлы в тестах      |

Обратите внимание: `anyhow` здесь используется на границе приложения, а библиотечный код использует собственный `Error`.

---

## 76.5. Модуль `error.rs`

Начнём с ошибок.

```rust
use std::io;
use std::path::PathBuf;

use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("I/O error while accessing {path}: {source}")]
    Io {
        path: PathBuf,
        #[source]
        source: io::Error,
    },

    #[error("file not found: {0}")]
    FileNotFound(PathBuf),

    #[error("file is too large: {path} ({size} bytes, limit {limit} bytes)")]
    FileTooLarge {
        path: PathBuf,
        size: u64,
        limit: u64,
    },

    #[error("invalid regular expression: {0}")]
    InvalidPattern(#[from] regex::Error),

    #[error("configuration error: {0}")]
    Config(String),

    #[error("invalid argument: {0}")]
    InvalidArgument(String),

    #[error("serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("TOML error: {0}")]
    Toml(#[from] toml::de::Error),
}

pub type Result<T> = std::result::Result<T, Error>;
```

Теперь ошибка содержит не только сообщение, но и структурированную информацию.

Например:

```rust
Error::FileTooLarge {
    path,
    size,
    limit,
}
```

Это гораздо полезнее, чем просто:

```rust
Error::Processing("file too large".to_string())
```

Типизированная ошибка позволяет программе принимать решения на основании типа ошибки.

---

## 76.6. Модуль `config.rs`

Конфигурация описывает правила обработки файлов.

```rust
use serde::{Deserialize, Serialize};
use std::fs;
use std::path::Path;

use crate::error::{Error, Result};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    #[serde(default = "default_max_file_size")]
    pub max_file_size: u64,

    #[serde(default)]
    pub output_format: OutputFormat,

    #[serde(default)]
    pub filters: Vec<FilterConfig>,

    #[serde(default)]
    pub transforms: Vec<TransformConfig>,
}

fn default_max_file_size() -> u64 {
    10 * 1024 * 1024
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
#[serde(rename_all = "lowercase")]
pub enum OutputFormat {
    #[default]
    Text,
    Json,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilterConfig {
    pub pattern: String,

    #[serde(default)]
    pub invert: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum TransformConfig {
    Uppercase,
    Lowercase,
    Replace {
        from: String,
        to: String,
    },
    RemoveEmpty,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            max_file_size: default_max_file_size(),
            output_format: OutputFormat::default(),
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
            return Err(Error::FileNotFound(path.to_path_buf()));
        }

        let content = fs::read_to_string(path).map_err(|source| Error::Io {
            path: path.to_path_buf(),
            source,
        })?;

        match path.extension().and_then(|ext| ext.to_str()) {
            Some("toml") => {
                toml::from_str(&content)
                    .map_err(|error| Error::Config(error.to_string()))
            }

            Some("json") => {
                serde_json::from_str(&content)
                    .map_err(|error| Error::Config(error.to_string()))
            }

            _ => Err(Error::Config(
                "configuration file must have .toml or .json extension"
                    .to_string(),
            )),
        }
    }
}
```

Здесь мы используем несколько важных возможностей Rust:

- `Option`;
- `Result`;
- `derive`;
- enum;
- `serde`;
- pattern matching;
- `let ... else`;
- собственный тип ошибки.

---

## 76.7. Пример конфигурационного файла

Например, `config.toml`:

```toml
max_file_size = 10485760
output_format = "text"

[[filters]]
pattern = "Rust"
invert = false

[[transforms]]
type = "uppercase"
```

Теперь программа может описывать обработку **данными**, а не только аргументами командной строки.

Это важный архитектурный принцип:

> Конфигурация должна описывать правила работы программы, а не содержать саму реализацию.

---

## 76.8. Модуль `processor.rs`

Теперь создадим основной компонент приложения.

```rust
use rayon::prelude::*;
use regex::Regex;
use std::fs;
use std::fmt;
use std::path::{Path, PathBuf};

use crate::config::{Config, TransformConfig};
use crate::error::{Error, Result};

#[derive(Debug)]
pub struct ProcessingResult {
    pub files_processed: usize,
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub output: Vec<String>,
}

impl fmt::Display for ProcessingResult {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        writeln!(f, "=== Processing Summary ===")?;
        writeln!(f, "Files processed: {}", self.files_processed)?;
        writeln!(f, "Lines processed: {}", self.lines_processed)?;
        writeln!(f, "Bytes processed: {}", self.bytes_processed)?;
        writeln!(f, "--- Output ---")?;

        for line in &self.output {
            writeln!(f, "{line}")?;
        }

        Ok(())
    }
}

pub struct FileProcessor {
    config: Config,
}

impl FileProcessor {
    pub fn new(config: Config) -> Self {
        Self { config }
    }

    pub fn process_files(
        &self,
        files: &[PathBuf],
    ) -> Result<ProcessingResult> {
        let results: Vec<Result<Vec<String>>> = files
            .par_iter()
            .map(|path| self.process_single_file(path))
            .collect();

        let mut output = Vec::new();
        let mut bytes_processed = 0;

        for result in results {
            let lines = result?;

            bytes_processed += lines
                .iter()
                .map(|line| line.len() as u64)
                .sum::<u64>();

            output.extend(lines);
        }

        Ok(ProcessingResult {
            files_processed: files.len(),
            lines_processed: output.len(),
            bytes_processed,
            output,
        })
    }

    fn process_single_file(&self, path: &Path) -> Result<Vec<String>> {
        if !path.exists() {
            return Err(Error::FileNotFound(path.to_path_buf()));
        }

        let metadata = fs::metadata(path).map_err(|source| Error::Io {
            path: path.to_path_buf(),
            source,
        })?;

        if metadata.len() > self.config.max_file_size {
            return Err(Error::FileTooLarge {
                path: path.to_path_buf(),
                size: metadata.len(),
                limit: self.config.max_file_size,
            });
        }

        let content =
            fs::read_to_string(path).map_err(|source| Error::Io {
                path: path.to_path_buf(),
                source,
            })?;

        let lines: Vec<String> = content
            .lines()
            .map(str::to_owned)
            .collect();

        let lines = self.apply_filters(lines)?;
        let lines = self.apply_transforms(lines)?;

        Ok(lines)
    }

    fn apply_filters(
        &self,
        mut lines: Vec<String>,
    ) -> Result<Vec<String>> {
        for filter in &self.config.filters {
            let pattern = Regex::new(&filter.pattern)?;

            lines.retain(|line| {
                let matched = pattern.is_match(line);

                if filter.invert {
                    !matched
                } else {
                    matched
                }
            });
        }

        Ok(lines)
    }

    fn apply_transforms(
        &self,
        mut lines: Vec<String>,
    ) -> Result<Vec<String>> {
        for transform in &self.config.transforms {
            lines = match transform {
                TransformConfig::Uppercase => {
                    lines
                        .into_iter()
                        .map(|line| line.to_uppercase())
                        .collect()
                }

                TransformConfig::Lowercase => {
                    lines
                        .into_iter()
                        .map(|line| line.to_lowercase())
                        .collect()
                }

                TransformConfig::Replace { from, to } => {
                    lines
                        .into_iter()
                        .map(|line| line.replace(from, to))
                        .collect()
                }

                TransformConfig::RemoveEmpty => {
                    lines
                        .into_iter()
                        .filter(|line| !line.trim().is_empty())
                        .collect()
                }
            };
        }

        Ok(lines)
    }
}
```

Здесь особенно хорошо видно преимущество `enum`.

`TransformConfig` может иметь только четыре варианта:

```rust
Uppercase
Lowercase
Replace
RemoveEmpty
```

Поэтому `match` заставляет нас обработать каждый из них.

Если позже мы добавим:

```rust
Sort
```

компилятор заставит нас подумать о том, как этот вариант должен обрабатываться.

---

## 76.9. Небольшой пример `processor`

Основную идею обработки строк можно проверить независимо от файлов:

```rust
fn uppercase(lines: Vec<String>) -> Vec<String> {
    lines
        .into_iter()
        .map(|line| line.to_uppercase())
        .collect()
}

fn main() {
    let lines = vec![
        "hello rust".to_string(),
        "type driven design".to_string(),
    ];

    let result = uppercase(lines);

    println!("{result:?}");
}
```

Результат:

```text
["HELLO RUST", "TYPE DRIVEN DESIGN"]
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+uppercase%28lines%3A+Vec%3CString%3E%29+-%3E+Vec%3CString%3E+%7B%0A++++lines.into_iter%28%29.map%28%7Cline%7C+line.to_uppercase%28%29%29.collect%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+lines+%3D+vec%21%5B%0A++++++++%22hello+rust%22.to_string%28%29%2C%0A++++++++%22type+driven+design%22.to_string%28%29%2C%0A++++%5D%3B%0A%0A++++let+result+%3D+uppercase%28lines%29%3B%0A++++println%21%28%22%7Bresult%3A%3F%7D%22%29%3B%0A%7D)

---

## 76.10. Параллельная обработка

В `process_files` мы использовали:

```rust
files
    .par_iter()
    .map(...)
    .collect()
```

Это `rayon`.

Важно понимать разницу:

```rust
files.iter()
```

обрабатывает элементы последовательно.

```rust
files.par_iter()
```

позволяет Rayon распределить работу между потоками.

При этом сама функция обработки одного файла остаётся обычной:

```rust
fn process_single_file(...)
```

Это хороший пример абстракции.

Мы не пишем вручную:

```rust
thread::spawn(...)
```

для каждого файла.

Параллелизм становится свойством реализации коллекции обработки.

---

## 76.11. Настройка количества потоков

Для CLI можно предоставить пользователю параметр:

```bash
file_processor --threads 8 file1.txt file2.txt
```

Создадим thread pool:

```rust
rayon::ThreadPoolBuilder::new()
    .num_threads(threads)
    .build_global()
    .expect("failed to initialize thread pool");
```

Однако есть важный нюанс: глобальный Rayon pool можно установить только один раз.

Поэтому это нужно сделать **один раз при запуске приложения**, до первой параллельной операции.

Если количество потоков не указано, можно использовать значение по умолчанию:

```rust
let threads = std::thread::available_parallelism()
    .map(|n| n.get())
    .unwrap_or(1);
```

---

## 76.12. Модуль `cli.rs`

Теперь опишем интерфейс программы.

```rust
use clap::{Args, Parser, Subcommand};
use std::path::PathBuf;

#[derive(Debug, Parser)]
#[command(
    name = "file_processor",
    version,
    about = "Process text files"
)]
pub struct Cli {
    /// Configuration file (.toml or .json)
    #[arg(short, long, env = "FP_CONFIG")]
    pub config: Option<PathBuf>,

    /// Number of worker threads
    #[arg(short, long)]
    pub threads: Option<usize>,

    /// Print additional information
    #[arg(short, long)]
    pub verbose: bool,

    #[command(subcommand)]
    pub command: Option<Commands>,

    /// Files to process
    #[arg(value_name = "FILE")]
    pub files: Vec<PathBuf>,
}

#[derive(Debug, Subcommand)]
pub enum Commands {
    /// Filter lines
    Filter(FilterArgs),

    /// Transform lines
    Transform(TransformArgs),

    /// Show file statistics
    Stats,
}

#[derive(Debug, Args)]
pub struct FilterArgs {
    /// Regular expression
    #[arg(short, long)]
    pub pattern: String,

    /// Invert the filter
    #[arg(short, long)]
    pub invert: bool,

    /// Files to process
    #[arg(value_name = "FILE")]
    pub files: Vec<PathBuf>,
}

#[derive(Debug, Args)]
pub struct TransformArgs {
    /// Convert to uppercase
    #[arg(long, conflicts_with = "lowercase")]
    pub uppercase: bool,

    /// Convert to lowercase
    #[arg(long, conflicts_with = "uppercase")]
    pub lowercase: bool,

    /// Remove empty lines
    #[arg(long)]
    pub remove_empty: bool,

    /// Replace text: FROM=TO
    #[arg(long, value_parser = parse_replacement)]
    pub replace: Option<(String, String)>,

    /// Files to process
    #[arg(value_name = "FILE")]
    pub files: Vec<PathBuf>,
}

fn parse_replacement(value: &str) -> Result<(String, String), String> {
    let (from, to) = value
        .split_once('=')
        .ok_or("replacement must have the form FROM=TO")?;

    if from.is_empty() {
        return Err("FROM cannot be empty".to_string());
    }

    Ok((from.to_string(), to.to_string()))
}
```

Обратите внимание на:

```rust
conflicts_with = "lowercase"
```

Теперь CLI сам запрещает противоречивые аргументы.

Например:

```bash
file_processor transform --uppercase --lowercase input.txt
```

закончится ошибкой ещё до запуска бизнес-логики.

Это хороший пример принципа:

> Проверяйте ограничения как можно раньше.

---

## 76.13. Почему CLI не должен содержать бизнес-логику

Плохая архитектура выглядела бы так:

```rust
fn main() {
    // parse arguments
    // read files
    // regex
    // transformations
    // error handling
    // output
}
```

В результате `main.rs` превращается в огромную функцию.

Лучше:

```text
main.rs
   │
   ├── Cli
   │
   ├── Config
   │
   └── FileProcessor
          │
          ├── read
          ├── filter
          └── transform
```

Теперь `FileProcessor` можно тестировать независимо от `clap`.

---

## 76.14. `lib.rs`

Корнем библиотеки будет:

```rust
pub mod cli;
pub mod config;
pub mod error;
pub mod processor;

pub use config::{
    Config,
    FilterConfig,
    OutputFormat,
    TransformConfig,
};

pub use error::{Error, Result};

pub use processor::{
    FileProcessor,
    ProcessingResult,
};
```

Это позволяет пользователю библиотеки писать:

```rust
use file_processor::{
    Config,
    FileProcessor,
};
```

вместо:

```rust
use file_processor::config::Config;
use file_processor::processor::FileProcessor;
```

Переэкспорт формирует более удобный публичный API.

---

## 76.15. `main.rs`

Теперь соберём всё вместе.

```rust
use anyhow::Result;
use clap::Parser;

use file_processor::{
    cli::{Cli, Commands},
    Config,
    FileProcessor,
};

fn main() -> Result<()> {
    let cli = Cli::parse();

    if let Some(threads) = cli.threads {
        rayon::ThreadPoolBuilder::new()
            .num_threads(threads)
            .build_global()?;
    }

    let mut config = Config::load(cli.config.as_deref())?;

    let (command, files) = match cli.command {
        Some(Commands::Filter(args)) => {
            config.filters.push(
                file_processor::FilterConfig {
                    pattern: args.pattern,
                    invert: args.invert,
                },
            );

            ("filter", args.files)
        }

        Some(Commands::Transform(args)) => {
            if args.uppercase {
                config
                    .transforms
                    .push(file_processor::TransformConfig::Uppercase);
            }

            if args.lowercase {
                config
                    .transforms
                    .push(file_processor::TransformConfig::Lowercase);
            }

            if args.remove_empty {
                config
                    .transforms
                    .push(file_processor::TransformConfig::RemoveEmpty);
            }

            if let Some((from, to)) = args.replace {
                config.transforms.push(
                    file_processor::TransformConfig::Replace {
                        from,
                        to,
                    },
                );
            }

            ("transform", args.files)
        }

        Some(Commands::Stats) => {
            println!("Stats command is handled separately.");
            return Ok(());
        }

        None => ("process", cli.files),
    };

    if files.is_empty() {
        anyhow::bail!(
            "no input files specified for command `{command}`"
        );
    }

    if cli.verbose {
        eprintln!("Command: {command}");
        eprintln!("Files: {}", files.len());
    }

    let processor = FileProcessor::new(config);
    let result = processor.process_files(&files)?;

    println!("{result}");

    Ok(())
}
```

Теперь весь путь данных выглядит следующим образом:

```text
command line
     │
     ▼
   clap
     │
     ▼
    Cli
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
  terminal
```

---

## 76.16. Команда `stats`

Статистика — отдельная операция, поэтому её лучше не смешивать с обычной обработкой.

Добавим в `FileProcessor`:

```rust
#[derive(Debug)]
pub struct FileStats {
    pub path: PathBuf,
    pub bytes: u64,
    pub lines: usize,
}

impl FileProcessor {
    pub fn stats(&self, path: &Path) -> Result<FileStats> {
        if !path.exists() {
            return Err(Error::FileNotFound(path.to_path_buf()));
        }

        let metadata =
            fs::metadata(path).map_err(|source| Error::Io {
                path: path.to_path_buf(),
                source,
            })?;

        if metadata.len() > self.config.max_file_size {
            return Err(Error::FileTooLarge {
                path: path.to_path_buf(),
                size: metadata.len(),
                limit: self.config.max_file_size,
            });
        }

        let content =
            fs::read_to_string(path).map_err(|source| Error::Io {
                path: path.to_path_buf(),
                source,
            })?;

        Ok(FileStats {
            path: path.to_path_buf(),
            bytes: metadata.len(),
            lines: content.lines().count(),
        })
    }
}
```

Теперь команда:

```bash
file_processor stats input.txt
```

может вывести:

```text
File: input.txt
Bytes: 1234
Lines: 42
```

---

## 76.17. Формирование JSON

Поскольку `FileStats` и `ProcessingResult` являются структурированными данными, их можно сериализовать.

Добавим:

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct FileStats {
    pub path: PathBuf,
    pub bytes: u64,
    pub lines: usize,
}
```

Теперь:

```rust
let json = serde_json::to_string_pretty(&stats)?;
println!("{json}");
```

может дать:

```json
{
  "path": "input.txt",
  "bytes": 1234,
  "lines": 42
}
```

Это показывает важную идею:

> Внутренняя модель программы не должна зависеть от способа отображения результата.

Один и тот же объект можно вывести как:

```text
human-readable text
```

или:

```json
machine-readable JSON
```

---

## 76.18. Работа с конфигурацией и CLI одновременно

Теперь рассмотрим важный вопрос.

Допустим, конфигурация содержит:

```toml
[[filters]]
pattern = "Rust"
```

А пользователь запускает:

```bash
file_processor filter --pattern "WebAssembly" input.txt
```

Какой фильтр должен применяться?

Нужно определить правило приоритетов.

Для нашей программы:

```text
CLI
 ↓
Configuration
 ↓
Default
```

То есть CLI имеет более высокий приоритет.

Это необходимо документировать.

Иначе поведение программы становится неожиданным.

---

## 76.19. Почему `Result` должен проходить вверх

Рассмотрим:

```rust
fn process_file(...) -> Result<Vec<String>>
```

Вместо:

```rust
fn process_file(...) -> Vec<String>
```

Почему?

Потому что ошибка — часть контракта функции.

Если файл не существует:

```text
FileNotFound
```

Если файл слишком большой:

```text
FileTooLarge
```

Если регулярное выражение неправильное:

```text
InvalidPattern
```

Функция не должна скрывать эти события.

Плохой вариант:

```rust
match process_file(path) {
    Ok(result) => result,
    Err(_) => Vec::new(),
}
```

Так мы превращаем ошибку в пустой результат.

Хороший вариант:

```rust
let result = process_file(path)?;
```

Ошибка передаётся дальше.

В конечном итоге её обработает граница приложения:

```rust
fn main() -> anyhow::Result<()> {
    // ...
}
```

---

## 76.20. Интеграционные тесты

Теперь протестируем настоящий `FileProcessor`.

Добавим:

```toml
[dev-dependencies]
tempfile = "3"
```

Создадим:

```text
tests/integration_test.rs
```

```rust
use std::fs;

use file_processor::{
    Config,
    FileProcessor,
    FilterConfig,
    TransformConfig,
};
use tempfile::tempdir;

#[test]
fn processes_file() {
    let dir = tempdir().unwrap();
    let path = dir.path().join("input.txt");

    fs::write(
        &path,
        "hello\nworld\nrust\n",
    )
    .unwrap();

    let processor =
        FileProcessor::new(Config::default());

    let result =
        processor.process_files(&[path]).unwrap();

    assert_eq!(result.files_processed, 1);
    assert_eq!(result.lines_processed, 3);
    assert_eq!(
        result.output,
        vec!["hello", "world", "rust"]
    );
}

#[test]
fn filters_lines() {
    let dir = tempdir().unwrap();
    let path = dir.path().join("input.txt");

    fs::write(
        &path,
        "Rust\nPython\nRust WebAssembly\nGo\n",
    )
    .unwrap();

    let mut config = Config::default();

    config.filters.push(FilterConfig {
        pattern: "Rust".to_string(),
        invert: false,
    });

    let processor = FileProcessor::new(config);

    let result =
        processor.process_files(&[path]).unwrap();

    assert_eq!(
        result.output,
        vec![
            "Rust",
            "Rust WebAssembly"
        ]
    );
}

#[test]
fn transforms_lines() {
    let dir = tempdir().unwrap();
    let path = dir.path().join("input.txt");

    fs::write(
        &path,
        "hello\nrust\n",
    )
    .unwrap();

    let mut config = Config::default();

    config
        .transforms
        .push(TransformConfig::Uppercase);

    let processor = FileProcessor::new(config);

    let result =
        processor.process_files(&[path]).unwrap();

    assert_eq!(
        result.output,
        vec!["HELLO", "RUST"]
    );
}
```

Запускаем:

```bash
cargo test
```

Если всё правильно:

```text
running 3 tests
test filters_lines ... ok
test processes_file ... ok
test transforms_lines ... ok
```

---

## 76.21. Тестирование ошибок

Тесты должны проверять не только успешный путь.

Например, файл не существует:

```rust
#[test]
fn reports_missing_file() {
    let processor =
        FileProcessor::new(Config::default());

    let result = processor.process_files(&[
        "does-not-exist.txt".into()
    ]);

    assert!(result.is_err());
}
```

Ещё лучше проверить конкретный тип ошибки:

```rust
#[test]
fn reports_missing_file_precisely() {
    use file_processor::Error;

    let processor =
        FileProcessor::new(Config::default());

    let result = processor.process_files(&[
        "does-not-exist.txt".into()
    ]);

    match result {
        Err(Error::FileNotFound(path)) => {
            assert_eq!(
                path.to_str(),
                Some("does-not-exist.txt")
            );
        }

        other => panic!("unexpected result: {other:?}"),
    }
}
```

Это особенно важно для проекта с собственным error type.

---

## 76.22. Ограничение размера файла

Мы специально добавили:

```rust
max_file_size
```

Например:

```rust
Config {
    max_file_size: 1024,
    ..Config::default()
}
```

Теперь файл размером больше 1024 байт будет отклонён.

Тест:

```rust
#[test]
fn rejects_large_file() {
    let dir = tempfile::tempdir().unwrap();
    let path = dir.path().join("large.txt");

    std::fs::write(&path, "0123456789")
        .unwrap();

    let config = Config {
        max_file_size: 5,
        ..Config::default()
    };

    let processor = FileProcessor::new(config);

    let result =
        processor.process_files(&[path]);

    assert!(result.is_err());
}
```

Это хороший пример того, почему конфигурационные параметры должны действительно использоваться.

Если поле:

```rust
max_file_size
```

существует только в `Config`, но нигде не проверяется, оно является **мёртвым API**.

---

## 76.23. Документирование библиотеки

`src/lib.rs` можно дополнить документацией:

````rust
//! # File Processor
//!
//! A small command-line-oriented library for processing
//! text files.
//!
//! The library supports:
//!
//! - regular-expression filters;
//! - text transformations;
//! - file-size limits;
//! - structured errors;
//! - parallel processing.
//!
//! ## Example
//!
//! ```no_run
//! use file_processor::{Config, FileProcessor};
//!
//! let processor = FileProcessor::new(Config::default());
//!
//! let result = processor
//!     .process_files(&["input.txt".into()])
//!     .unwrap();
//!
//! println!("{result}");
//! ```
````

Теперь:

```bash
cargo doc --open
```

создаст документацию API.

---

## 76.24. Проверка проекта

Перед тем как считать работу законченной, нужно выполнить несколько команд.

### Проверка компиляции

```bash
cargo check
```

### Сборка

```bash
cargo build
```

### Тесты

```bash
cargo test
```

### Проверка форматирования

```bash
cargo fmt --check
```

### Проверка Clippy

```bash
cargo clippy -- -D warnings
```

Это уже похоже на минимальный CI pipeline:

```text
cargo fmt
      ↓
cargo check
      ↓
cargo clippy
      ↓
cargo test
      ↓
cargo build
```

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1. `enum` заставляет обработать все состояния

Попробуйте:

```rust
enum Operation {
    Uppercase,
    Lowercase,
    RemoveEmpty,
}

fn apply(operation: Operation) {
    match operation {
        Operation::Uppercase => println!("upper"),
        Operation::Lowercase => println!("lower"),
    }
}

fn main() {
    apply(Operation::Uppercase);
}
```

Компилятор сообщит, что:

```text
Operation::RemoveEmpty
```

не обработан.

Исправьте:

```rust
match operation {
    Operation::Uppercase => println!("upper"),
    Operation::Lowercase => println!("lower"),
    Operation::RemoveEmpty => println!("remove empty"),
}
```

Это один из фундаментальных механизмов type-driven design.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum+Operation+%7B%0A++++Uppercase%2C%0A++++Lowercase%2C%0A++++RemoveEmpty%2C%0A%7D%0A%0Afn+apply%28operation%3A+Operation%29+%7B%0A++++match+operation+%7B%0A++++++++Operation%3A%3AUppercase+%3D%3E+println%21%28%22upper%22%29%2C%0A++++++++Operation%3A%3ALowercase+%3D%3E+println%21%28%22lower%22%29%2C%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++apply%28Operation%3A%3AUppercase%29%3B%0A%7D)

---

## Эксперимент 2. Ошибка вместо `panic`

Создайте:

```rust
use std::fs;

fn read_file(path: &str) -> Result<String, std::io::Error> {
    fs::read_to_string(path)
}

fn main() {
    match read_file("missing.txt") {
        Ok(content) => println!("{content}"),
        Err(error) => eprintln!("Error: {error}"),
    }
}
```

Здесь отсутствие файла — нормальная ситуация, которую программа умеет обработать.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afs%3B%0A%0Afn+read_file%28path%3A+%26str%29+-%3E+Result%3CString%2C+std%3A%3Aio%3A%3AError%3E+%7B%0A++++fs%3A%3Aread_to_string%28path%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++match+read_file%28%22missing.txt%22%29+%7B%0A++++++++Ok%28content%29+%3D%3E+println%21%28%22%7Bcontent%7D%22%29%2C%0A++++++++Err%28error%29+%3D%3E+eprintln%21%28%22Error%3A+%7Berror%7D%22%29%2C%0A++++%7D%0A%7D)

---

## Эксперимент 3. `?` передаёт ошибку вверх

Сравните:

```rust
fn read(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?;
    Ok(content)
}
```

с:

```rust
fn read(path: &str) -> Result<String, std::io::Error> {
    match std::fs::read_to_string(path) {
        Ok(content) => Ok(content),
        Err(error) => Err(error),
    }
}
```

Это разные записи одной и той же идеи.

Оператор `?` делает поток ошибок компактным и хорошо читаемым.

---

## Эксперимент 4. `Vec` и `retain`

Рассмотрим:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];

    numbers.retain(|number| *number % 2 == 0);

    println!("{numbers:?}");
}
```

Результат:

```text
[2, 4]
```

В нашем проекте тот же принцип используется для фильтрации строк:

```rust
lines.retain(|line| pattern.is_match(line));
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+mut+numbers+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A++++numbers.retain%28%7Cnumber%7C+%2Anumber+%25+2+%3D%3D+0%29%3B%0A++++println%21%28%22%7Bnumbers%3F%7D%22%29%3B%0A%7D)

---

# Практика

## Задание 1. Запустите проект

Создайте тестовый файл:

```text
hello rust
rust is fast
python
rust webassembly
go
```

Запустите:

```bash
cargo run -- input.txt
```

---

## Задание 2. Добавьте фильтрацию

Запустите:

```bash
cargo run -- filter --pattern "rust" input.txt
```

Проверьте результат.

Затем попробуйте:

```bash
cargo run -- filter --pattern "rust" --invert input.txt
```

---

## Задание 3. Добавьте трансформации

Проверьте:

```bash
cargo run -- transform --uppercase input.txt
```

и:

```bash
cargo run -- transform --lowercase input.txt
```

---

## Задание 4. Добавьте `--unique`

Создайте новый вариант обработки:

```rust
Unique
```

Он должен удалять повторяющиеся строки.

Например:

```text
Rust
Go
Rust
Python
Go
```

должно превратиться в:

```text
Rust
Go
Python
```

Подсказка: здесь потребуется `HashSet`.

---

## Задание 5. Добавьте сортировку

Добавьте:

```rust
Sort
```

и сортируйте строки:

```rust
lines.sort();
```

После добавления нового варианта `enum` компилятор покажет все места, которые необходимо обновить.

---

## Задание 6. JSON output

Добавьте поддержку:

```bash
--output json
```

и сериализацию `ProcessingResult` через `serde_json`.

---

## Задание 7. Тест ограничения размера

Напишите тест, который:

1. создаёт временный файл;
2. записывает в него больше данных, чем разрешено;
3. вызывает `process_files`;
4. проверяет `Error::FileTooLarge`.

---

## Задание 8. Проверьте качество проекта

Выполните:

```bash
cargo fmt --check
cargo check
cargo clippy -- -D warnings
cargo test
```

Все четыре команды должны завершиться успешно.

---

# Что мы получили

В начале главы у нас был только набор отдельных механизмов:

```text
clap
serde
thiserror
regex
rayon
tempfile
```

Теперь они образуют единую систему:

```text
                    File Processor
                          │
             ┌────────────┴────────────┐
             │                         │
           CLI                      Config
          clap                    serde/toml
             │                         │
             └────────────┬────────────┘
                          │
                          ▼
                   FileProcessor
                          │
             ┌────────────┼────────────┐
             │            │            │
           Files        Filters     Transforms
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                   ProcessingResult
                          │
                 ┌────────┴────────┐
                 │                 │
                Text              JSON
```

При этом каждая часть имеет чёткую ответственность.

---

# Главное из этой главы

После этой главы мы:

- создали полноценный Cargo-проект;
- разделили приложение на модули;
- отделили CLI от бизнес-логики;
- использовали `clap` для командной строки;
- создали собственный тип `Error`;
- использовали `Result` для обработки ошибок;
- загрузили конфигурацию через `serde`;
- поддержали TOML и JSON;
- использовали `enum` для описания операций;
- реализовали фильтрацию через `regex`;
- реализовали преобразование текста;
- ограничили размер обрабатываемых файлов;
- добавили параллельную обработку через `rayon`;
- написали интеграционные тесты;
- протестировали ошибки;
- добавили документацию;
- проверили проект через `cargo check`, `cargo test`, `cargo fmt` и `cargo clippy`.

Но самое важное — мы увидели, **как проектировать настоящий Rust-проект целиком**.

Отдельный механизм сам по себе не делает программу хорошей.

Хорошая программа возникает тогда, когда:

```text
types
   +
ownership
   +
errors
   +
modules
   +
tests
   +
tooling
   +
domain logic
```

образуют согласованную систему.

> **Главная идея главы:** реальный Rust-проект — это не коллекция примеров языка. Это система, в которой типы определяют данные и состояния, модули разделяют ответственность, `Result` делает ошибки частью контракта, тесты проверяют поведение, а Cargo-инструменты контролируют качество проекта.

Именно с этого момента Rust начинает восприниматься не как набор отдельных возможностей языка, а как **инструмент создания законченных программ**.
