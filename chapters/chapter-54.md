# Глава 54. Command-Line Applications

В предыдущей главе мы научились работать с файлами. Теперь пришло время создать полноценное приложение командной строки (CLI). CLI-приложения — это основа системного программирования: утилиты, инструменты разработки, скрипты и сервисы.

В этой главе мы разберёмся, как создавать удобные CLI-приложения на Rust: обрабатывать аргументы, работать с переменными окружения, управлять конфигурацией и обрабатывать ошибки так, чтобы пользователь понимал, что пошло не так.

Все примеры этой главы используют **Rust Edition 2024** и библиотеку **clap** — стандарт для CLI-приложений в Rust.

---

## 54.1. Архитектура CLI-приложения

Типичное CLI-приложение лучше разделять на несколько уровней:

```text
┌─────────────────────────────────────────────────────────────┐
│                     CLI Application                         │
├─────────────────────────────────────────────────────────────┤
│  1. Парсинг аргументов и опций                              │
│                         ↓                                   │
│  2. Чтение конфигурации                                     │
│     CLI → environment → config file → defaults              │
│                         ↓                                   │
│  3. Выполнение бизнес-логики                                │
│                         ↓                                   │
│  4. Вывод результата                                        │
│                         ↓                                   │
│  5. Обработка ошибок → код выхода                           │
└─────────────────────────────────────────────────────────────┘
```

Хороший CLI должен отделять **разбор командной строки** от самой логики программы. Тогда бизнес-логику можно тестировать независимо от CLI.

Простейшее приложение можно написать вообще без зависимостей:

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    if args.len() != 2 {
        eprintln!("Usage: {} <name>", args[0]);
        std::process::exit(2);
    }

    let name = &args[1];

    println!("Hello, {name}!");
}
```

Запуск:

```bash
cargo run -- Alice
```

Результат:

```text
Hello, Alice!
```

При неправильном количестве аргументов:

```bash
cargo run
```

получим сообщение об использовании программы и ненулевой код выхода.

Для реальных приложений ручной разбор `env::args()` быстро становится неудобным. Поэтому обычно используют `clap`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Aenv%3B%0A%0Afn+main%28%29+%7B%0A++++let+args%3A+Vec%3CString%3E+%3D+env%3A%3Aargs%28%29.collect%28%29%3B%0A%0A++++if+args.len%28%29+%21%3D+2+%7B%0A++++++++eprintln%21%28%22Usage%3A+%7B%7D+%3Cname%3E%22%2C+args%5B0%5D%29%3B%0A++++++++std%3A%3Aprocess%3A%3Aexit%282%29%3B%0A++++%7D%0A%0A++++println%21%28%22Hello%2C+%7B%7D%21%22%2C+args%5B1%5D%29%3B%0A%7D)

---

## 54.2. `clap` — парсинг аргументов

`clap` — основная библиотека экосистемы Rust для создания CLI. Она умеет автоматически разбирать аргументы, генерировать `--help`, проверять типы и сообщать пользователю об ошибках.

Для `derive` API достаточно:

```toml
[dependencies]
clap = { version = "4.6", features = ["derive"] }
```

Актуальная документация `clap` указывает, что feature `derive` включает процедурные derive-макросы вроде `#[derive(Parser)]`. ([Docs.rs][2])

Вместо строки для операции лучше использовать перечисление:

```rust
use clap::{Parser, ValueEnum};

#[derive(Debug, Clone, Copy, ValueEnum)]
enum Operation {
    Add,
    Sub,
    Mul,
    Div,
}

#[derive(Debug, Parser)]
#[command(name = "calc", version, about = "Простой CLI-калькулятор")]
struct Cli {
    /// Первое число
    a: f64,

    /// Второе число
    b: f64,

    /// Операция
    #[arg(short, long, value_enum, default_value_t = Operation::Add)]
    operation: Operation,
}

fn main() {
    let cli = Cli::parse();

    let result = match cli.operation {
        Operation::Add => cli.a + cli.b,
        Operation::Sub => cli.a - cli.b,
        Operation::Mul => cli.a * cli.b,
        Operation::Div => {
            if cli.b == 0.0 {
                eprintln!("Error: division by zero");
                std::process::exit(1);
            }

            cli.a / cli.b
        }
    };

    println!("Result: {result}");
}
```

Теперь `clap` сам проверяет значение операции:

```bash
cargo run -- 10 5 --operation add
```

```text
Result: 15
```

```bash
cargo run -- 10 5 --operation mul
```

```text
Result: 50
```

Можно использовать короткую форму:

```bash
cargo run -- 10 5 -o mul
```

Получим:

```text
Result: 50
```

А `--help` автоматически показывает интерфейс программы:

```bash
cargo run -- --help
```

`clap` также автоматически сообщит об ошибке:

```bash
cargo run -- 10 5 --operation unknown
```

Поскольку `Operation` реализует `ValueEnum`, допустимые значения будут перечислены пользователю.

**Важная идея:** типы Rust можно использовать непосредственно как часть CLI-интерфейса. Если параметр имеет тип `u32`, `bool`, `PathBuf` или собственный `ValueEnum`, `clap` выполняет соответствующий разбор и валидацию.

---

## 54.3. Типы аргументов в `clap`

Основные формы аргументов можно представить следующим образом:

```rust
use clap::Parser;

#[derive(Debug, Parser)]
#[command(name = "app")]
struct Cli {
    /// Обязательный позиционный аргумент
    input: String,

    /// Необязательный аргумент
    #[arg(short, long)]
    output: Option<String>,

    /// Флаг
    #[arg(short, long)]
    verbose: bool,

    /// Необязательная опция с числовым значением
    #[arg(short, long)]
    count: Option<u32>,

    /// Опция со значением по умолчанию
    #[arg(long, default_value_t = 1)]
    threads: u32,

    /// Несколько значений через запятую
    #[arg(short, long, value_delimiter = ',')]
    files: Vec<String>,
}

fn main() {
    let cli = Cli::parse();

    println!("{cli:#?}");
}
```

Здесь:

| Объявление       | Пример                | Значение                          |
| ---------------- | --------------------- | --------------------------------- |
| `input: String`  | `app input.txt`       | обязательный позиционный аргумент |
| `Option<String>` | `--output result.txt` | необязательная опция              |
| `bool`           | `--verbose`           | флаг                              |
| `Option<u32>`    | `--count 10`          | необязательное значение           |
| `u32` + default  | `--threads 4`         | значение с default                |
| `Vec<String>`    | `--files a.txt,b.txt` | несколько значений                |

Обратите внимание: для обычного `String` без `Option` и без значения по умолчанию `clap` уже рассматривает аргумент как обязательный. Поэтому писать:

```rust
#[arg(long, required = true)]
config: String,
```

обычно не требуется. Достаточно:

```rust
#[arg(long)]
config: String,
```

`required = true` имеет смысл, например, когда обязательность определяется более сложной конфигурацией CLI.

---

## 54.4. Subcommands (подкоманды)

Subcommands позволяют строить CLI, похожие на `git` и `cargo`:

```rust
use clap::{Parser, Subcommand};

#[derive(Debug, Parser)]
#[command(name = "todo")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Debug, Subcommand)]
enum Commands {
    /// Добавить задачу
    Add {
        #[arg(short, long)]
        name: String,

        #[arg(short, long)]
        priority: u8,
    },

    /// Удалить задачу
    Remove {
        #[arg(short, long)]
        name: String,

        /// Удалить без дополнительного подтверждения
        #[arg(short, long)]
        force: bool,
    },

    /// Показать задачи
    List {
        /// Показать также завершённые задачи
        #[arg(long)]
        all: bool,
    },
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Add { name, priority } => {
            println!("Adding task: {name} (priority {priority})");
        }

        Commands::Remove { name, force } => {
            if force {
                println!("Force removing task: {name}");
            } else {
                println!("Removing task: {name}");
            }
        }

        Commands::List { all } => {
            if all {
                println!("Listing all tasks");
            } else {
                println!("Listing active tasks");
            }
        }
    }
}
```

Использование:

```bash
cargo run -- add --name "Learn Rust" --priority 1
cargo run -- remove --name "Learn Rust"
cargo run -- remove --name "Learn Rust" --force
cargo run -- list
cargo run -- list --all
```

`bool` здесь является именно флагом. Его значение:

- `false` — если флаг отсутствует;
- `true` — если пользователь указал `--force` или `--all`.

Поэтому писать `default_value = "false"` для `bool` не нужно.

---

## 54.5. Переменные окружения

Переменные окружения часто используются для параметров, которые не хочется передавать в командной строке:

```text
CLI argument
     ↓
environment variable
     ↓
configuration file
     ↓
default value
```

Прочитать переменную можно через `std::env::var`:

```rust
use std::env;

fn main() {
    let log_level =
        env::var("LOG_LEVEL").unwrap_or_else(|_| "info".to_owned());

    println!("Log level: {log_level}");
}
```

Запуск:

```bash
LOG_LEVEL=debug cargo run
```

Результат:

```text
Log level: debug
```

### Важное изменение в Rust 2024

В Rust Edition 2024 функции:

```rust
std::env::set_var
std::env::remove_var
```

стали `unsafe`. Причина связана с тем, что изменение окружения процесса может быть небезопасным в многопоточном Unix-приложении. ([Rust Documentation][1])

Поэтому следующий старый пример:

```rust
std::env::set_var("APP_VERSION", "1.0.0");
```

нельзя просто переносить в современный Rust-код.

Если необходимо задать переменную окружения **для дочернего процесса**, лучше использовать `Command::env`:

```rust
use std::process::Command;

fn main() -> std::io::Result<()> {
    let status = Command::new("my_program")
        .env("APP_VERSION", "1.0.0")
        .status()?;

    println!("Exit status: {status}");

    Ok(())
}
```

Для самого CLI обычно вообще не требуется изменять окружение процесса: достаточно **читать** переменные через `env::var`.

### Интеграция с `clap`

`clap` умеет использовать environment variables непосредственно при разборе аргументов. Для этого нужен feature `env`. ([Docs.rs][3])

```toml
[dependencies]
clap = { version = "4.6", features = ["derive", "env"] }
```

```rust
use clap::Parser;

#[derive(Debug, Parser)]
#[command(name = "app")]
struct Cli {
    /// Файл конфигурации
    #[arg(long, env = "APP_CONFIG")]
    config: String,

    /// Подробный вывод
    #[arg(long, env = "APP_VERBOSE", default_value_t = false)]
    verbose: bool,
}

fn main() {
    let cli = Cli::parse();

    println!("Config: {}", cli.config);
    println!("Verbose: {}", cli.verbose);
}
```

Теперь можно использовать либо CLI:

```bash
cargo run -- --config config.toml --verbose
```

либо окружение:

```bash
APP_CONFIG=config.toml APP_VERBOSE=true cargo run
```

Это особенно удобно для контейнеров и CI/CD.

---

## 54.6. Коды выхода

CLI-программа должна сообщать операционной системе, завершилась ли она успешно.

Главное правило:

| Код       | Значение |
| --------- | -------- |
| `0`       | Успех    |
| ненулевой | Ошибка   |

Конкретные ненулевые значения зависят от приложения и используемых соглашений.

Например:

```rust
use std::process;

fn main() {
    if let Err(error) = run() {
        eprintln!("Error: {error}");
        process::exit(1);
    }
}

fn run() -> Result<(), Box<dyn std::error::Error>> {
    // Логика приложения.
    Ok(())
}
```

Для проверки кода выхода в Unix-подобных системах:

```bash
cargo run
echo $?
```

На Windows в `cmd.exe`:

```cmd
cargo run
echo %ERRORLEVEL%
```

### Код `2`

Код `2` традиционно используется для ошибок использования CLI, например неверных аргументов. `clap` использует ненулевые коды выхода для ошибок разбора, включая стандартный сценарий с кодом `2`.

### Коды `64–78`

Значения вроде:

```text
64  EX_USAGE
65  EX_DATAERR
66  EX_NOINPUT
70  EX_SOFTWARE
```

относятся к Unix-соглашению `sysexits`. Это **не стандарт Rust** и не обязательное требование для любого CLI.

Для собственного приложения можно определить простую и понятную политику:

```text
0 — успех
1 — ошибка выполнения
2 — ошибка использования CLI
```

Главное — быть последовательным.

---

## 54.7. Конфигурация

В реальном CLI конфигурация часто поступает из нескольких источников:

```text
             ┌──────────────┐
             │ CLI arguments│
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │ Environment  │
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │ Config file  │
             └──────┬───────┘
                    │
             ┌──────▼───────┐
             │   Defaults   │
             └──────────────┘
```

Конкретный порядок приоритетов выбирает приложение. Хорошей практикой является документировать его явно.

Для TOML можно использовать `serde` и `toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
toml = "0.9"
```

Структура конфигурации:

```rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
struct Config {
    host: String,
    port: u16,
    logging: LoggingConfig,
}

#[derive(Debug, Deserialize)]
struct LoggingConfig {
    level: String,
    file: String,
}

fn read_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let config = toml::from_str(&content)?;

    Ok(config)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = read_config("config.toml")?;

    println!("Server: {}:{}", config.host, config.port);
    println!("Log level: {}", config.logging.level);
    println!("Log file: {}", config.logging.file);

    Ok(())
}
```

`config.toml`:

```toml
host = "127.0.0.1"
port = 8080

[logging]
level = "debug"
file = "app.log"
```

Запуск:

```bash
cargo run
```

Результат:

```text
Server: 127.0.0.1:8080
Log level: debug
Log file: app.log
```

В production-приложении важно также различать:

- отсутствие конфигурационного файла;
- синтаксически неправильный TOML;
- отсутствующее обязательное поле;
- недопустимое значение;
- невозможность прочитать файл.

Эти ситуации должны превращаться в понятные сообщения для пользователя.

---

## 54.8. Удобные ошибки для пользователей

Вместо `Box<dyn Error>` на границе CLI полезно иметь собственный тип ошибки.

Для небольшого приложения это можно сделать вручную:

```rust
use std::fmt;
use std::io;

#[derive(Debug)]
enum AppError {
    ConfigNotFound,
    InvalidConfig(String),
    Io(io::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::ConfigNotFound => {
                write!(f, "configuration file not found")
            }

            Self::InvalidConfig(message) => {
                write!(f, "invalid configuration: {message}")
            }

            Self::Io(error) => {
                write!(f, "I/O error: {error}")
            }
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::Io(error) => Some(error),
            _ => None,
        }
    }
}

fn run() -> Result<(), AppError> {
    // Логика приложения.
    Ok(())
}

fn main() {
    if let Err(error) = run() {
        eprintln!("Error: {error}");

        if let Some(source) = std::error::Error::source(&error) {
            eprintln!("Caused by: {source}");
        }

        std::process::exit(1);
    }
}
```

Здесь важно различать **сообщение для пользователя** и **причину ошибки**.

Например:

```text
Error: I/O error: No such file or directory
```

может быть гораздо полезнее для пользователя, чем:

```text
thread 'main' panicked at ...
```

Для крупных приложений ручная реализация `Display` и `Error` быстро становится многословной. Поэтому на практике часто используют `thiserror` для типов ошибок библиотеки/приложения и `anyhow` для удобной обработки ошибок на границе приложения.

---

## 54.9. Прогресс-бары и интерактивность

Для длительных операций CLI часто должен показывать пользователю прогресс.

Например, библиотека `indicatif` предоставляет progress bar:

```toml
[dependencies]
indicatif = "0.18"
```

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn main() {
    let pb = ProgressBar::new(100);

    pb.set_style(
        ProgressStyle::with_template(
            "[{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} {eta}",
        )
        .expect("valid progress-bar template")
        .progress_chars("#>-"),
    );

    for _ in 0..100 {
        std::thread::sleep(Duration::from_millis(20));
        pb.inc(1);
    }

    pb.finish_with_message("Done!");
}
```

Здесь progress bar обновляется по мере выполнения операции, а после завершения заменяется сообщением `Done!`.

Для CLI также важно различать:

- **stdout** — обычный результат программы;
- **stderr** — сообщения об ошибках, предупреждения и диагностический вывод.

Это позволяет корректно использовать CLI в shell pipelines:

```bash
myapp input.txt > result.txt
```

При этом диагностические сообщения не попадут в `result.txt`.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Неверный аргумент

Для программы с `clap` выполните:

```bash
cargo run -- --unknown
```

`clap` обнаружит неизвестный аргумент и завершит программу с ошибкой.

### Эксперимент 2: Отсутствие обязательного аргумента

Для CLI:

```rust
#[derive(clap::Parser)]
struct Cli {
    input: String,
}
```

выполните:

```bash
cargo run
```

`clap` сообщит, что обязательный аргумент `input` не был передан.

### Эксперимент 3: Неверный тип

Для:

```rust
#[derive(clap::Parser)]
struct Cli {
    #[arg(long)]
    count: u32,
}
```

выполните:

```bash
cargo run -- --count hello
```

`clap` не сможет преобразовать `hello` в `u32` и выдаст диагностическое сообщение вместо того, чтобы передать некорректное значение в программу.

---

## Практика

### Задание 1

Создайте CLI-приложение, которое принимает путь к файлу и выводит его содержимое.

Требования:

- использовать `clap`;
- путь должен быть обязательным аргументом;
- при отсутствии файла вывести понятную ошибку;
- вернуть ненулевой код выхода.

### Задание 2

Добавьте опцию:

```text
--lines N
```

которая выводит только первые `N` строк файла.

Например:

```bash
myapp README.md --lines 10
```

### Задание 3

Реализуйте CLI с подкомандами:

```text
todo add
todo remove
todo list
```

Используйте `#[derive(Subcommand)]`.

### Задание 4

Добавьте конфигурацию из TOML-файла и переменной окружения.

Реализуйте и явно задокументируйте приоритет:

```text
CLI argument > environment variable > config file > default
```

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если передать:

```bash
--count hello
```

для аргумента типа `u32`?

Объясните, где происходит преобразование строки в число.

### Задание 6

🔨 **Эксперимент с CLI.**

Что произойдёт, если не указать обязательный аргумент?

Проверьте:

1. сообщение `clap`;
2. код выхода;
3. содержимое `stdout`;
4. содержимое `stderr`.

### Задание 7

Добавьте `--verbose` и сделайте так, чтобы диагностические сообщения выводились в `stderr`, а результат — в `stdout`.

Проверьте работу:

```bash
myapp input.txt > result.txt
```

Убедитесь, что сообщения диагностики не попали в `result.txt`.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Архитектура CLI** — аргументы → конфигурация → логика → вывод → ошибки.
- **`clap`** — типобезопасный разбор аргументов и генерация `--help`.
- **Типы аргументов** — позиционные, опции, флаги, списки и пользовательские `ValueEnum`.
- **Subcommands** — построение многоуровневых CLI.
- **Переменные окружения** — конфигурация через environment variables.
- **Rust 2024** — `env::set_var` и `env::remove_var` требуют особого внимания и стали `unsafe`.
- **Коды выхода** — `0` означает успех, ненулевые значения — различные виды ошибок.
- **Конфигурация** — объединение CLI, environment, файлов и значений по умолчанию.
- **Ошибки** — пользователь должен получать понятное сообщение, а не внутренний stack trace.
- **stdout/stderr** — разделение результата и диагностики.
- **Progress bars** — обратная связь во время длительных операций.

**Самая важная идея:**

> CLI — это не просто программа, которая читает аргументы из `argv`. Это публичный интерфейс вашей программы. Хороший CLI должен иметь предсказуемые аргументы, понятный `--help`, типобезопасную валидацию, конфигурацию с ясным приоритетом, корректные коды выхода и сообщения об ошибках, ориентированные на пользователя.
>
> В Rust `clap` позволяет большую часть этой инфраструктуры описать декларативно. Но архитектура остаётся за разработчиком: парсинг CLI не должен смешиваться с бизнес-логикой. Хорошая граница выглядит так:
>
> ```text
> CLI
>  ↓
> Parsed configuration
>  ↓
> Application logic
>  ↓
> Result<T, E>
>  ↓
> User-facing output + exit code
> ```
>
> Такой подход делает CLI не только удобным для человека, но и предсказуемым для shell-скриптов, CI/CD и других программ.

[1]: https://doc.rust-lang.org/stable/edition-guide/rust-2024/newly-unsafe-functions.html 'Newly unsafe functions - The Rust Edition Guide'
[2]: https://docs.rs/clap/latest/clap/_derive/ 'clap::_derive - Rust'
[3]: https://docs.rs/clap/latest/clap/_features/index.html 'clap::_features - Rust'
