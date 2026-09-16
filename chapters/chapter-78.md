# Глава 78. Добавляем async

В предыдущей главе мы выделили библиотеку с логикой обработки файлов. Но вся обработка оставалась синхронной.

Теперь представим реальное приложение. У нас есть сотни файлов, каждый файл может быть большим, а операция чтения с диска может занимать заметное время. Если обрабатывать файлы строго последовательно, следующая операция начинается только после завершения предыдущей.

В этой главе мы добавим асинхронный API:

* будем запускать обработку нескольких файлов конкурентно;
* ограничим количество одновременно обрабатываемых файлов;
* добавим timeout;
* добавим отмену операций;
* научимся правильно обрабатывать ошибки отдельных задач;
* реализуем потоковую выдачу результатов;
* разберёмся, что именно даёт `async` при работе с файловой системой.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 78.1. Почему async для файловой обработки?

Начнём с важного уточнения.

Асинхронность не означает:

> «Операция выполняется быстрее».

Она означает:

> «Поток выполнения не обязан простаивать, пока операция ввода-вывода находится в ожидании».

Например, у нас есть три файла:

```text
file-1.txt
file-2.txt
file-3.txt
```

Синхронная программа может работать так:

```text
read file-1
      │
      ▼
process file-1
      │
      ▼
read file-2
      │
      ▼
process file-2
      │
      ▼
read file-3
      │
      ▼
process file-3
```

Асинхронная программа может организовать работу иначе:

```text
             ┌── read file-1 ──┐
             │                 │
             ├── read file-2 ──┼──► результаты
             │                 │
             └── read file-3 ──┘
```

При этом важно понимать ещё одну вещь.

### Async не равно parallel

**Concurrency** — конкурентность — означает, что несколько операций находятся в работе одновременно.

**Parallelism** — параллелизм — означает, что работа действительно выполняется одновременно на нескольких ядрах CPU.

Для файловой обработки нам в первую очередь интересна конкурентность операций I/O.

---

### Что происходит с `tokio::fs`?

В Tokio файловые операции, такие как:

```rust
tokio::fs::read_to_string(...)
```

не превращают файловую систему ОС в настоящий async I/O API.

Tokio использует специальный blocking thread pool для файловых операций.

Поэтому правильнее говорить:

> `tokio::fs` позволяет встроить файловые операции в асинхронную архитектуру Tokio и не блокировать worker thread обычным синхронным `std::fs`.

Это особенно удобно, когда приложение одновременно занимается другими асинхронными задачами: HTTP-запросами, сетевыми соединениями, таймерами, очередями и т. д.

---

### Когда async действительно полезен?

Async особенно полезен, когда приложение большую часть времени ждёт:

* сетевые операции;
* файловые операции;
* базу данных;
* внешние API;
* очереди сообщений;
* другие I/O-операции.

Если же основная работа выглядит так:

```rust
for _ in 0..1_000_000_000 {
    calculate_something();
}
```

то `async` сам по себе проблему не решит. Это CPU-bound работа, и для неё чаще нужны обычные потоки, `rayon` или специализированные worker pools.

---

## 78.2. Обновляем зависимости

Для асинхронной версии нам достаточно Tokio и `tokio-util` для `CancellationToken`.

Корневой `Cargo.toml` workspace можно расширить:

```toml
[workspace.dependencies]

tokio = { version = "1", features = [
    "fs",
    "macros",
    "rt-multi-thread",
    "sync",
    "time"
] }

tokio-util = { version = "0.7", features = ["rt"] }

futures = "0.3"
```

В `file_processor_core/Cargo.toml`:

```toml
[dependencies]

serde = { workspace = true }
serde_json = { workspace = true }
toml = { workspace = true }
regex = { workspace = true }
thiserror = { workspace = true }

tokio = { workspace = true }
tokio-util = { workspace = true }
futures = { workspace = true }
```

Нам больше не нужны `async-stream`, если мы реализуем поток через `futures::stream::FuturesUnordered`.

Это хороший пример архитектурного принципа:

> Не добавляйте зависимость только потому, что она позволяет сделать уже решаемую задачу немного короче.

---

## 78.3. Первый async-код

Самая простая асинхронная функция выглядит так:

```rust
async fn message() -> String {
    "Hello from async Rust!".to_string()
}
```

Но вызов:

```rust
let result = message();
```

не возвращает `String`.

Он возвращает `Future`.

Чтобы получить результат, внутри async-контекста используется `.await`:

```rust
async fn message() -> String {
    "Hello from async Rust!".to_string()
}

#[tokio::main]
async fn main() {
    let result = message().await;

    println!("{result}");
}
```

### Важная идея

`async fn` фактически описывает операцию, которую можно выполнить асинхронно.

Вызов:

```rust
message()
```

создаёт future.

А:

```rust
message().await
```

ожидает её результат.

Открыть пример в Rust Playground:

[https://play.rust-lang.org/?version=stable&mode=debug&edition=2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

> Для этого примера не нужны сторонние crates. В Playground достаточно вставить код и запустить его.

---

## 78.4. Делаем `Config` пригодным для async-кода

В предыдущей главе `Config` используется одновременно несколькими задачами.

Поэтому конфигурация должна быть клонируемой:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    pub default_encoding: Option<String>,
    pub max_file_size: Option<u64>,
    pub output_format: OutputFormat,
    pub filters: Vec<FilterConfig>,
    pub transforms: Vec<TransformConfig>,
}
```

Теперь мы можем безопасно передавать копию конфигурации в асинхронные задачи.

В нашем случае это не означает, что каждая задача обязательно должна создавать независимую копию конфигурации. Если конфигурация большая, лучше использовать `Arc<Config>`.

---

## 78.5. Асинхронный процессор

Теперь создадим отдельный `AsyncFileProcessor`.

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use regex::Regex;
use std::path::{Path, PathBuf};
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Semaphore;
use tokio::time::timeout;

use crate::config::Config;
use crate::error::{Error, Result};

pub struct AsyncFileProcessor {
    config: Arc<Config>,
    timeout: Duration,
    max_concurrent: usize,
}
```

Конструктор:

```rust
impl AsyncFileProcessor {
    pub fn new(config: Config) -> Self {
        Self {
            config: Arc::new(config),
            timeout: Duration::from_secs(30),
            max_concurrent: 10,
        }
    }

    pub fn with_timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn with_max_concurrent(mut self, max: usize) -> Self {
        assert!(max > 0, "max_concurrent must be greater than zero");

        self.max_concurrent = max;
        self
    }
}
```

Почему мы используем `Arc<Config>`?

Потому что конфигурация является общей для всех задач:

```text
                  Config
                    │
          ┌─────────┼─────────┐
          │         │         │
        task 1    task 2    task 3
```

Нам не нужно создавать отдельную полную копию `Config` для каждого файла.

---

## 78.6. Асинхронная обработка одного файла

Вынесем обработку одного файла в отдельную функцию:

```rust
async fn process_single_file_async(
    config: &Config,
    path: &Path,
) -> Result<FileOutput> {
    if !path.exists() {
        return Err(Error::FileNotFound(path.display().to_string()));
    }

    let metadata = tokio::fs::metadata(path).await?;

    if let Some(max_size) = config.max_file_size {
        if metadata.len() > max_size {
            return Err(Error::Processing(format!(
                "File too large: {} bytes (max {})",
                metadata.len(),
                max_size
            )));
        }
    }

    let bytes_processed = metadata.len();

    let content = tokio::fs::read_to_string(path).await?;

    let mut lines: Vec<String> =
        content.lines().map(str::to_owned).collect();

    lines = apply_filters(config, lines)?;
    lines = apply_transforms(config, lines);

    Ok(FileOutput {
        lines,
        bytes_processed,
    })
}
```

Добавим внутренний тип результата:

```rust
struct FileOutput {
    lines: Vec<String>,
    bytes_processed: u64,
}
```

Обратите внимание на важную деталь.

Раньше мы вычисляли:

```rust
lines.iter().map(|l| l.len() as u64)
```

Это уже **не размер исходного файла**.

После трансформации:

```text
hello
```

может превратиться в:

```text
HELLO!!!
```

Поэтому теперь размер файла берётся из:

```rust
metadata.len()
```

---

## 78.7. Фильтры и трансформации

Эти функции пока остаются обычными.

```rust
fn apply_filters(
    config: &Config,
    lines: Vec<String>,
) -> Result<Vec<String>> {
    let mut result = lines;

    for filter in &config.filters {
        let pattern = Regex::new(&filter.pattern)?;
        let invert = filter.invert.unwrap_or(false);

        result = result
            .into_iter()
            .filter(|line| {
                let matches = pattern.is_match(line);

                if invert {
                    !matches
                } else {
                    matches
                }
            })
            .collect();
    }

    Ok(result)
}
```

Трансформации:

```rust
fn apply_transforms(
    config: &Config,
    lines: Vec<String>,
) -> Vec<String> {
    let mut result = lines;

    for transform in &config.transforms {
        result = transform.operation.apply(result);
    }

    result
}
```

Почему эти функции не `async`?

Потому что внутри них нет операции ожидания.

Например:

```rust
Regex::new(...)
```

не является асинхронной операцией.

И:

```rust
line.to_uppercase()
```

тоже не является асинхронной операцией.

Это обычная CPU-работа.

Не следует делать функцию `async` только потому, что её вызывают из async-кода.

---

## 78.8. Результат обработки

Определим публичный результат:

```rust
#[derive(Debug, Default)]
pub struct ProcessingResult {
    pub files_processed: usize,
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub output: Vec<String>,
    pub errors: Vec<ProcessingError>,
}
```

А ошибку конкретного файла представим отдельным типом:

```rust
#[derive(Debug)]
pub struct ProcessingError {
    pub path: PathBuf,
    pub error: Error,
}
```

Это лучше, чем:

```rust
errors: Vec<String>
```

потому что библиотека не теряет структурированную информацию.

Пользователь библиотеки может самостоятельно решить, как представить ошибку:

```rust
for error in &result.errors {
    eprintln!("{}: {}", error.path.display(), error.error);
}
```

---

## 78.9. Обработка нескольких файлов конкурентно

Теперь самое интересное.

Нам нужно:

1. запустить обработку нескольких файлов;
2. ограничить количество одновременно работающих задач;
3. дождаться всех результатов;
4. не потерять ошибки отдельных файлов.

Используем `FuturesUnordered`.

```rust
impl AsyncFileProcessor {
    pub async fn process_files(
        &self,
        files: &[PathBuf],
    ) -> Result<ProcessingResult> {
        let semaphore = Arc::new(
            Semaphore::new(self.max_concurrent)
        );

        let mut tasks = FuturesUnordered::new();

        for path in files {
            let path = path.clone();
            let config = Arc::clone(&self.config);
            let semaphore = Arc::clone(&semaphore);
            let timeout_duration = self.timeout;

            tasks.push(async move {
                let permit = semaphore
                    .acquire_owned()
                    .await
                    .map_err(|_| Error::Cancelled)?;

                let result = timeout(
                    timeout_duration,
                    process_single_file_async(&config, &path),
                )
                .await;

                drop(permit);

                let result = match result {
                    Ok(result) => result,
                    Err(_) => Err(Error::Timeout),
                };

                (path, result)
            });
        }

        let mut output = ProcessingResult::default();

        while let Some((path, result)) = tasks.next().await {
            match result {
                Ok(file) => {
                    output.files_processed += 1;
                    output.lines_processed += file.lines.len();
                    output.bytes_processed += file.bytes_processed;
                    output.output.extend(file.lines);
                }

                Err(error) => {
                    output.errors.push(ProcessingError {
                        path,
                        error,
                    });
                }
            }
        }

        Ok(output)
    }
}
```

Здесь есть важная архитектурная деталь.

Мы **не используем `tokio::spawn`**.

Почему?

Потому что нам не обязательно создавать отдельные Tokio tasks для каждой операции.

`FuturesUnordered` позволяет одновременно продвигать множество futures и получать результат по мере завершения.

---

## 78.10. Зачем нужен `Semaphore`?

Предположим:

```text
10 000 файлов
```

Если мы без ограничений запустим 10 000 операций одновременно, это может оказаться плохой идеей.

Нам не обязательно ограничивать количество файлов в самом входном массиве.

Мы ограничиваем количество одновременно выполняемых операций:

```rust
let semaphore = Arc::new(Semaphore::new(10));
```

Это означает:

```text
                 Semaphore(10)
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
     task 1         task 2         task 3
       ...
     task 10

     task 11 → ждёт permit
```

Когда одна операция завершается:

```rust
drop(permit);
```

следующая ожидающая задача получает возможность продолжить.

Это называется **backpressure** — управление нагрузкой.

---

## 78.11. Почему `FuturesUnordered` лучше простого `Vec<JoinHandle>`

Можно было бы написать:

```rust
let mut handles = Vec::new();

for file in files {
    handles.push(tokio::spawn(...));
}
```

А затем:

```rust
for handle in handles {
    handle.await?;
}
```

Но здесь есть важный недостаток.

Если первая задача работает долго:

```text
task 1 ────────────────────────────────► result

task 2 ───────► result
task 3 ───► result
```

при последовательном ожидании `handle 1` мы не обработаем результат `task 2` и `task 3`, пока не завершится первая задача.

`FuturesUnordered` работает иначе:

```text
task 1 ────────────────────────────────►
task 2 ───────► result
task 3 ───► result
```

Мы получаем результаты **по мере их готовности**.

Это особенно важно для потоковой обработки.

---

## 78.12. `tokio::spawn` всё-таки нужен

`tokio::spawn` остаётся полезным инструментом.

Например:

```rust
let handle = tokio::spawn(async {
    expensive_operation().await
});

let result = handle.await?;
```

`spawn` создаёт отдельную Tokio task.

Но не каждая future должна становиться отдельной spawned task.

Полезно различать:

```text
Future
  │
  ├── может выполняться внутри другой future
  │
  └── может быть запущена через tokio::spawn
```

Для нашей задачи `FuturesUnordered` достаточно.

---

## 78.13. Таймаут отдельного файла

В нашем процессоре timeout устанавливается для каждого файла:

```rust
let result = timeout(
    timeout_duration,
    process_single_file_async(&config, &path),
)
.await;
```

Если операция завершилась вовремя:

```rust
Ok(result)
```

Если timeout истёк:

```rust
Err(_) => Err(Error::Timeout)
```

Таким образом, один слишком медленный файл не заставляет ждать его бесконечно.

Например:

```text
file-1 ───► OK
file-2 ───► OK
file-3 ─────────────────► TIMEOUT
file-4 ───► OK
```

Результат всё равно может содержать успешные файлы:

```text
Processed: 3
Errors:    1
```

---

## 78.14. Глобальный timeout

Иногда нужен другой уровень timeout.

Нас может интересовать не время одного файла, а время **всей операции**.

Например:

```rust
use std::time::Duration;
use tokio::time::timeout;

pub async fn process_with_global_timeout(
    processor: &AsyncFileProcessor,
    files: &[PathBuf],
    duration: Duration,
) -> Result<ProcessingResult> {
    match timeout(
        duration,
        processor.process_files(files),
    )
    .await
    {
        Ok(result) => result,
        Err(_) => Err(Error::Timeout),
    }
}
```

Теперь есть два разных ограничения:

```text
                   whole operation
        ┌─────────────────────────────────┐
        │                                 │
        │ file 1 ───►                     │
        │ file 2 ───────►                 │
        │ file 3 ───────────► TIMEOUT     │
        │                                 │
        └─────────────────────────────────┘
```

и:

```text
file timeout:

file 1 ───► OK
file 2 ─────────────────► TIMEOUT
file 3 ───► OK
```

Это разные уровни политики.

---

## 78.15. Обновляем ошибки

Добавим timeout и cancellation в `error.rs`:

```rust
use std::io;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("I/O error: {0}")]
    Io(#[from] io::Error),

    #[error("Configuration error: {0}")]
    Config(String),

    #[error("Invalid pattern: {0}")]
    InvalidPattern(#[from] regex::Error),

    #[error("File not found: {0}")]
    FileNotFound(String),

    #[error("Processing error: {0}")]
    Processing(String),

    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("TOML parsing error: {0}")]
    Toml(#[from] toml::de::Error),

    #[error("Operation timed out")]
    Timeout,

    #[error("Task cancelled")]
    Cancelled,
}

pub type Result<T> = std::result::Result<T, Error>;
```

В реальном production-коде можно пойти ещё дальше и хранить в ошибке контекст:

```rust
pub struct ProcessingError {
    pub path: PathBuf,
    pub error: Error,
}
```

Это намного полезнее, чем:

```text
"Operation timed out"
```

потому что пользователь сразу знает:

```text
large-file.txt: Operation timed out
```

---

## 78.16. CancellationToken

Timeout и cancellation — не одно и то же.

**Timeout** означает:

> «Если операция занимает больше N секунд — прекращаем её ожидание».

**Cancellation** означает:

> «Кто-то явно попросил прекратить работу».

Для cancellation используем:

```rust
tokio_util::sync::CancellationToken
```

Например:

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();

let child = token.child_token();
```

Можно передать дочерний token отдельной операции.

---

## 78.17. Ожидание отмены через `select!`

Простейший пример:

```rust
use tokio_util::sync::CancellationToken;

async fn wait_for_shutdown(
    token: CancellationToken,
) {
    tokio::select! {
        _ = do_work() => {
            println!("Work completed");
        }

        _ = token.cancelled() => {
            println!("Work cancelled");
        }
    }
}
```

`tokio::select!` ждёт несколько futures одновременно.

В нашем случае:

```text
              ┌── do_work()
select! ──────┤
              └── token.cancelled()
```

Побеждает тот future, который завершился первым.

---

## 78.18. Добавляем cancellation в процессор

Создадим метод:

```rust
pub async fn process_files_with_cancellation(
    &self,
    files: &[PathBuf],
    token: CancellationToken,
) -> Result<ProcessingResult> {
    tokio::select! {
        result = self.process_files(files) => {
            result
        }

        _ = token.cancelled() => {
            Err(Error::Cancelled)
        }
    }
}
```

Это уже рабочая модель отмены на уровне всей операции.

Но здесь есть важное ограничение.

Если `process_files()` уже запустил множество задач, простая отмена внешнего `select!` не обязательно означает, что каждая внутренняя операция немедленно прекратится.

Поэтому для полноценной cooperative cancellation token должен передаваться **внутрь самих операций**.

---

## 78.19. Cooperative cancellation

Более правильная архитектура:

```rust
async fn process_single_file_async(
    config: &Config,
    path: &Path,
    token: &CancellationToken,
) -> Result<FileOutput> {
    if token.is_cancelled() {
        return Err(Error::Cancelled);
    }

    let content = tokio::fs::read_to_string(path).await?;

    if token.is_cancelled() {
        return Err(Error::Cancelled);
    }

    // дальнейшая обработка...

    Ok(...)
}
```

Теперь функция периодически проверяет:

```rust
token.is_cancelled()
```

и прекращает работу.

Но следует понимать важную вещь:

> Cancellation в async Rust обычно является кооперативной.

Нельзя гарантировать, что произвольная операция мгновенно прекратится в любой точке.

Если future находится внутри операции, которая не умеет реагировать на cancellation, она может продолжать работу до своего завершения.

---

## 78.20. CLI с async

Теперь CLI может использовать асинхронный процессор.

В `Cargo.toml` CLI добавим Tokio:

```toml
[dependencies]

clap = { workspace = true }
anyhow = { workspace = true }

tokio = { workspace = true }

file_processor_core = {
    path = "../file_processor_core"
}
```

`main.rs`:

```rust
use clap::Parser;
use file_processor_core::{AsyncFileProcessor, Config, Result};
use std::time::Duration;

mod cli;

use cli::Cli;

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();

    let config = Config::load(cli.config.as_deref())?;

    let processor = AsyncFileProcessor::new(config)
        .with_timeout(Duration::from_secs(60))
        .with_max_concurrent(cli.threads.max(1));

    println!(
        "Processing {} files concurrently...",
        cli.files.len()
    );

    let result = processor
        .process_files(&cli.files)
        .await?;

    println!(
        "Processed {} files, {} lines, {} bytes",
        result.files_processed,
        result.lines_processed,
        result.bytes_processed
    );

    if !result.errors.is_empty() {
        eprintln!("\n--- Errors ---");

        for error in &result.errors {
            eprintln!(
                "{}: {}",
                error.path.display(),
                error.error
            );
        }
    }

    if !result.output.is_empty() {
        println!("\n--- Output ---");

        for line in result.output {
            println!("{line}");
        }
    }

    Ok(())
}
```

Теперь архитектура выглядит так:

```text
                  CLI
                   │
                   ▼
          AsyncFileProcessor
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
        file 1   file 2   file 3
          │        │        │
          └────────┼────────┘
                   ▼
               Result
```

CLI знает о пользовательском интерфейсе.

Библиотека знает о бизнес-логике.

---

## 78.21. Потоковая обработка результатов

Иногда нам не нужно ждать обработки всех файлов.

Представим:

```text
100 000 файлов
```

Если собрать весь результат:

```rust
Vec<String>
```

может потребоваться много памяти.

Гораздо интереснее получать результаты постепенно:

```text
file 17 → result
file 4  → result
file 23 → result
file 8  → result
...
```

Для этого можно вернуть `Stream`.

---

## 78.22. `FuturesUnordered` как Stream

Создадим метод:

```rust
use futures::stream::FuturesUnordered;
use futures::Stream;
use futures::StreamExt;
```

И:

```rust
pub fn process_files_stream(
    &self,
    files: &[PathBuf],
) -> impl Stream<Item = Result<(PathBuf, Vec<String>)>> + '_ {
    let semaphore = Arc::new(
        Semaphore::new(self.max_concurrent)
    );

    let futures = files.iter().cloned().map(move |path| {
        let semaphore = Arc::clone(&semaphore);
        let config = Arc::clone(&self.config);
        let timeout_duration = self.timeout;

        async move {
            let permit = semaphore
                .acquire_owned()
                .await
                .map_err(|_| Error::Cancelled)?;

            let result = timeout(
                timeout_duration,
                process_single_file_async(
                    &config,
                    &path,
                ),
            )
            .await;

            drop(permit);

            let result = match result {
                Ok(result) => result,
                Err(_) => Err(Error::Timeout),
            }?;

            Ok((path, result.lines))
        }
    });

    futures.collect()
}
```

Однако здесь есть тонкость: `collect()` вернёт `Vec`, то есть мы снова потеряем потоковую семантику.

Поэтому лучше непосредственно использовать `FuturesUnordered`:

```rust
pub fn process_files_stream(
    &self,
    files: &[PathBuf],
) -> impl Stream<Item = (PathBuf, Result<FileOutput>)> + '_ {
    let semaphore = Arc::new(
        Semaphore::new(self.max_concurrent)
    );

    let futures = files.iter().cloned().map(move |path| {
        let semaphore = Arc::clone(&semaphore);
        let config = Arc::clone(&self.config);
        let timeout_duration = self.timeout;

        async move {
            let result = async {
                let _permit = semaphore
                    .acquire_owned()
                    .await
                    .map_err(|_| Error::Cancelled)?;

                timeout(
                    timeout_duration,
                    process_single_file_async(
                        &config,
                        &path,
                    ),
                )
                .await
                .map_err(|_| Error::Timeout)?
            }
            .await;

            (path, result)
        }
    });

    FuturesUnordered::from_iter(futures)
}
```

Теперь результаты появляются по мере завершения операций.

---

## 78.23. Использование Stream

Чтобы читать stream:

```rust
use futures::StreamExt;

let mut stream = processor.process_files_stream(&files);

while let Some((path, result)) = stream.next().await {
    match result {
        Ok(output) => {
            println!(
                "{}: {} lines",
                path.display(),
                output.lines.len()
            );
        }

        Err(error) => {
            eprintln!(
                "{}: {}",
                path.display(),
                error
            );
        }
    }
}
```

Вместо:

```text
wait for everything
       ↓
receive Vec
```

получаем:

```text
file 7  → result
file 2  → result
file 10 → result
file 1  → result
...
```

Это особенно полезно для:

* больших наборов файлов;
* CLI с прогрессом;
* серверных приложений;
* обработки потоков данных;
* долгих batch-задач.

---

## 78.24. Async не отменяет CPU-bound работу

Наш код содержит две принципиально разные части.

### I/O

```rust
tokio::fs::read_to_string(path).await
```

Здесь ожидание I/O можно встроить в async runtime.

### CPU

```rust
lines = apply_filters(config, lines)?;
lines = apply_transforms(config, lines);
```

Здесь выполняется обычная вычислительная работа.

Если трансформация становится очень тяжёлой:

```rust
for line in huge_file {
    perform_expensive_calculation(line);
}
```

она может надолго занять worker thread Tokio.

В таком случае можно использовать:

```rust
tokio::task::spawn_blocking(...)
```

Например:

```rust
let result = tokio::task::spawn_blocking(move || {
    expensive_cpu_operation(data)
})
.await
.map_err(|e| Error::Processing(e.to_string()))?;
```

Таким образом:

```text
async runtime
     │
     ├── I/O tasks
     │
     └── spawn_blocking
              │
              ▼
        CPU-heavy work
```

Это важная граница.

> Async хорошо организует ожидание. Он не делает CPU-вычисления дешевле.

---

## 78.25. Тестирование async-кода

Для async-тестов Tokio предоставляет атрибут:

```rust
#[tokio::test]
async fn test_async_processing() {
    // ...
}
```

Например:

```rust
#[tokio::test]
async fn test_basic_async_processing() {
    let dir = tempfile::tempdir().unwrap();

    let file_path = dir.path().join("test.txt");

    tokio::fs::write(
        &file_path,
        "hello\nworld\nRust\n",
    )
    .await
    .unwrap();

    let processor =
        AsyncFileProcessor::new(Config::default());

    let result = processor
        .process_files(&[file_path])
        .await
        .unwrap();

    assert_eq!(result.files_processed, 1);
    assert_eq!(result.lines_processed, 3);
    assert_eq!(
        result.output,
        vec!["hello", "world", "Rust"]
    );
}
```

Теперь тест действительно проверяет async API библиотеки.

---

## 78.26. Тестирование timeout

Timeout трудно проверять, если операция зависит от реального диска.

Но сам механизм timeout можно протестировать на простой future:

```rust
#[tokio::test]
async fn test_timeout() {
    let result = tokio::time::timeout(
        std::time::Duration::from_millis(10),
        tokio::time::sleep(
            std::time::Duration::from_secs(1)
        ),
    )
    .await;

    assert!(result.is_err());
}
```

Этот тест не зависит от скорости компьютера.

Мы проверяем именно семантику timeout:

```text
sleep 1 second
      │
      │
      └──── timeout 10 ms
                ↓
             Err
```

---

## 78.27. Тестирование cancellation

Аналогично можно проверить cancellation:

```rust
#[tokio::test]
async fn test_cancellation() {
    use tokio_util::sync::CancellationToken;

    let token = CancellationToken::new();
    let child = token.child_token();

    let task = tokio::spawn(async move {
        child.cancelled().await;
        true
    });

    token.cancel();

    assert!(task.await.unwrap());
}
```

Здесь:

```rust
token.cancel();
```

сигнализирует всем связанным child tokens:

```text
CancellationToken
       │
       ├── child 1 → cancelled
       ├── child 2 → cancelled
       └── child 3 → cancelled
```

---

## 78.28. Эксперименты с компилятором

### Эксперимент 1: забытый `.await`

Создайте:

```rust
async fn process() -> Result<String, ()> {
    Ok("done".to_string())
}

fn main() {
    let result = process();

    println!("{result:?}");
}
```

Попробуйте скомпилировать программу.

Здесь `result` — не `String` и не `Result<String, ()>`.

Это future.

Компилятор покажет, что future нельзя использовать как обычный результат операции.

Исправление:

```rust
async fn main() {
    let result = process().await;
}
```

Но теперь появляется следующая проблема: обычный `main` не является async runtime.

В Tokio:

```rust
#[tokio::main]
async fn main() {
    let result = process().await;
    println!("{result:?}");
}
```

**Вывод:**

`.await` используется внутри async-контекста, а для запуска async-программы нужен runtime.

---

### Эксперимент 2: два permit одного семафора

Создайте семафор:

```rust
let semaphore =
    tokio::sync::Semaphore::new(1);
```

Первый permit:

```rust
let _permit =
    semaphore.acquire().await.unwrap();
```

Теперь попробуйте получить второй:

```rust
let _permit2 =
    semaphore.acquire().await.unwrap();
```

Вторая операция будет ждать.

Почему?

Потому что:

```text
Semaphore(1)

permit 1 → занят
permit 2 → ждёт
```

Если первый permit уничтожить:

```rust
drop(_permit);
```

второй сможет продолжить.

---

### Эксперимент 3: timeout

Попробуйте:

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

#[tokio::main]
async fn main() {
    let result = timeout(
        Duration::from_millis(100),
        sleep(Duration::from_secs(1)),
    )
    .await;

    println!("{result:?}");
}
```

Операция `sleep` рассчитана на одну секунду, а timeout — на 100 миллисекунд.

Поэтому future завершится через timeout.

---

### Эксперимент 4: concurrency против sequential

Создайте две задачи:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn work(id: u32) {
    sleep(Duration::from_millis(100)).await;
    println!("task {id}");
}

#[tokio::main]
async fn main() {
    work(1).await;
    work(2).await;
}
```

Затем измените программу:

```rust
#[tokio::main]
async fn main() {
    tokio::join!(
        work(1),
        work(2),
    );
}
```

В первом варианте операции выполняются последовательно.

Во втором они выполняются конкурентно.

Это один из самых важных экспериментов всей главы.

---

## Практика

### Задание 1

Добавьте в `AsyncFileProcessor` асинхронную обработку одного файла.

Проверьте:

* существование файла;
* максимальный размер;
* чтение;
* фильтры;
* трансформации.

---

### Задание 2

Добавьте настройку:

```rust
max_concurrent
```

и проверьте экспериментально, что при значении:

```rust
2
```

одновременно выполняется не более двух операций.

---

### Задание 3

Добавьте timeout для каждого файла.

Проверьте отдельно:

* успешную обработку;
* timeout;
* ошибку чтения;
* отсутствие файла.

---

### Задание 4

Добавьте глобальный timeout для всей операции:

```rust
process_with_global_timeout(...)
```

Сравните его с timeout отдельного файла.

Объясните, почему это две разные политики.

---

### Задание 5

Добавьте `CancellationToken`.

Реализуйте:

```rust
process_files_with_cancellation(...)
```

Затем передавайте token внутрь обработки отдельных файлов.

---

### Задание 6

Реализуйте потоковую обработку:

```rust
process_files_stream(...)
```

и выводите результат сразу после завершения каждого файла.

Обратите внимание:

> порядок результатов не обязан совпадать с порядком входных файлов.

---

### Задание 7

Проверьте поведение при:

```text
1000 файлов
max_concurrent = 1
```

и:

```text
1000 файлов
max_concurrent = 100
```

Объясните, почему увеличение конкурентности не гарантирует пропорционального увеличения производительности.

---

### Задание 8

Добавьте CPU-intensive трансформацию и сравните два варианта:

```rust
expensive_operation(...)
```

и:

```rust
tokio::task::spawn_blocking(...)
```

Объясните, почему CPU-bound работа отличается от I/O-bound работы.

---

## Главное из этой главы

После этой главы мы:

* **добавили** асинхронный API библиотеки;
* **использовали** `async` и `.await`;
* **использовали** Tokio runtime;
* **организовали** конкурентную обработку файлов;
* **ограничили** конкурентность через `Semaphore`;
* **использовали** `FuturesUnordered` для получения результатов по мере завершения;
* **добавили** timeout отдельных операций;
* **добавили** глобальный timeout;
* **реализовали** cooperative cancellation через `CancellationToken`;
* **научились** писать async-тесты;
* **разобрались**, чем I/O-bound работа отличается от CPU-bound;
* **увидели**, когда нужен `spawn_blocking`.

### Самая важная идея

Асинхронность — это не «ускоритель программы».

Это модель организации работы, при которой приложение может эффективно использовать время ожидания I/O и обслуживать множество конкурентных операций.

Для нашего `FileProcessor` это даёт следующую архитектуру:

```text
                    CLI
                     │
                     ▼
           AsyncFileProcessor
                     │
          ┌──────────┼──────────┐
          │          │          │
       file 1     file 2     file 3
          │          │          │
          └──────────┼──────────┘
                     │
              Semaphore
                     │
              timeout / cancel
                     │
                     ▼
                  Result
```

Но ещё важнее понимать границы этой модели:

```text
I/O-bound
   │
   └── async / Tokio

CPU-bound
   │
   └── threads / rayon / spawn_blocking
```

Именно поэтому хороший async-код — это не код, в котором везде написано `async`.

Это код, в котором правильно определено:

> **что мы ждём, что выполняем, сколько операций разрешаем выполнять одновременно и как приложение должно вести себя при timeout, ошибке или отмене.**

Этот вариант уже можно использовать как **завершённую главу**, а не как набор незаконченных заготовок: особенно важны исправления вокруг `FuturesUnordered`, `CancellationToken`, семафора и различия между I/O-bound и CPU-bound работой.
