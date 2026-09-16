# Глава 82. Создаём собственный trait API

В предыдущих главах мы использовали трейты для абстракции репозиториев и других компонентов. Теперь пришло время создать **собственный trait API** — публичный контракт, через который приложение будет обращаться к сервису обработки файлов.

Это важный архитектурный шаг.

До этого мы абстрагировали отдельные технические компоненты:

```text
Application
    │
    ▼
ProcessingRepository
    │
    ├── PostgreSQL
    ├── In-memory
    └── Test implementation
```

Теперь абстрагируем уже **сам сервис**:

```text
                    FileProcessingService
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        Production       Mock       Alternative
        implementation implementation implementation
              │
              ▼
         PostgreSQL
```

HTTP API, CLI, тесты и другие клиенты будут знать только о `FileProcessingService`. Они не должны знать, как именно сервис обрабатывает данные и где он хранит результаты.

Это позволяет:

- менять реализацию сервиса;
- писать тесты без PostgreSQL;
- использовать один сервис из HTTP и CLI;
- отделять бизнес-контракт от инфраструктуры;
- использовать статическую или динамическую диспетчеризацию;
- расширять API, не раскрывая внутреннее устройство сервиса.

Все основные примеры этой главы используют **Rust Edition 2024**.

---

## 82.1. Зачем нужен собственный trait API?

**Trait API — это контракт между потребителем сервиса и его реализацией.**

Потребителю не нужно знать конкретный тип:

```rust
FileProcessingServiceImpl
```

Ему достаточно знать:

```rust
dyn FileProcessingService
```

Архитектура становится такой:

```text
┌─────────────────────────────────────────────────────────────┐
│                     Application / HTTP                      │
│                                                             │
│                 FileProcessingService                       │
│                          ▲                                  │
│                          │                                  │
├──────────────────────────┼──────────────────────────────────┤
│                          │                                  │
│             ┌────────────┴────────────┐                     │
│             │                         │                     │
│             ▼                         ▼                     │
│   FileProcessingServiceImpl      MockFileProcessingService  │
│             │                         │                     │
│             ▼                         ▼                     │
│        PostgreSQL                  In-memory                │
└─────────────────────────────────────────────────────────────┘
```

### Основные преимущества

### Абстракция

Клиент работает с контрактом:

```rust
service.process_text(...).await?;
```

а не с конкретной базой данных или алгоритмом обработки.

### Тестируемость

В тесте можно передать:

```rust
MockFileProcessingService
```

вместо настоящего сервиса.

### Заменяемость

Можно создать несколько реализаций:

```text
FileProcessingServiceImpl
CachedFileProcessingService
RemoteFileProcessingService
MockFileProcessingService
```

при этом код HTTP API останется прежним.

### Динамическая диспетчеризация

Если нам необходимо хранить разные реализации за одним интерфейсом, можно использовать:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

Но здесь появляется важный нюанс, связанный с асинхронными методами.

---

## 82.2. `async fn` в trait: современный Rust и `dyn Trait`

Начиная с Rust 1.75, асинхронные функции непосредственно в traits являются стабильной возможностью языка:

```rust
trait Service {
    async fn execute(&self);
}
```

То есть больше не требуется `async-trait` просто для того, чтобы **объявить** асинхронный trait.

Однако есть важное ограничение.

Trait с `async fn` не является `dyn`-compatible:

```rust
trait Service {
    async fn execute(&self);
}

// ❌ Такой trait нельзя использовать как dyn Service
let service: Box<dyn Service>;
```

Причина заключается в том, что `async fn` возвращает скрытый тип `Future`, а `dyn Trait` требует возможности построить vtable для методов trait. Нативный `async fn` в trait этого сейчас не обеспечивает.

Поэтому существуют два разных сценария.

### Статическая диспетчеризация

Если конкретный тип известен во время компиляции:

```rust
async fn run<S: Service>(service: &S) {
    service.execute().await;
}
```

можно использовать обычный современный:

```rust
trait Service {
    async fn execute(&self);
}
```

### Динамическая диспетчеризация

Если мы хотим:

```rust
Arc<dyn Service>
```

для асинхронного trait, нам необходимо преобразовать async-методы в object-safe форму.

Для этого широко используется crate `async-trait`. Он преобразует async-методы в boxed futures и тем самым позволяет использовать trait objects.

Именно этот вариант мы будем использовать в нашем приложении, потому что HTTP-приложению удобно хранить сервис как:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

---

## 82.3. Определяем trait API

Создадим файл:

```text
crates/domain/src/services/file_processing_service.rs
```

Сначала добавим `async-trait` в `Cargo.toml` domain-крейта:

```toml
[dependencies]
async-trait = { workspace = true }
```

В корневом `Cargo.toml`:

```toml
[workspace.dependencies]
async-trait = "0.1"
```

Теперь сам trait:

```rust
use async_trait::async_trait;
use uuid::Uuid;

use crate::error::DomainError;
use crate::models::{
    ProcessingResult,
    ResultFilters,
    UsageStats,
};

/// Публичный контракт сервиса обработки файлов.
///
/// Trait намеренно не знает:
/// - о PostgreSQL;
/// - о SQLx;
/// - о HTTP;
/// - о Tokio;
/// - о файловой системе.
#[async_trait]
pub trait FileProcessingService: Send + Sync {
    /// Ошибка конкретной реализации сервиса.
    type Error: std::error::Error + Send + Sync + 'static;

    /// Обработать текст.
    async fn process_text(
        &self,
        content: &str,
        filename: Option<String>,
    ) -> Result<ProcessingResult, Self::Error>;

    /// Получить результат по ID.
    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error>;

    /// Получить список результатов.
    async fn list_results(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error>;

    /// Получить статистику.
    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error>;

    /// Удалить результат.
    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error>;

    /// Удалить старые результаты.
    async fn cleanup_old(
        &self,
        days: i32,
    ) -> Result<u64, Self::Error>;
}
```

Обратите внимание на:

```rust
type Error: std::error::Error + Send + Sync + 'static;
```

Это **ассоциированный тип**.

Каждая реализация trait самостоятельно определяет свой тип ошибки:

```rust
impl FileProcessingService for MyService {
    type Error = MyError;

    // ...
}
```

Другая реализация может использовать:

```rust
type Error = DomainError;
```

а тестовая:

```rust
type Error = MockError;
```

При этом сам trait остаётся одинаковым.

### Почему здесь нужен `Send + Sync`?

Наш сервис предполагается использовать в многопоточном асинхронном приложении.

Поэтому мы хотим иметь возможность написать:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

и передавать этот объект между потоками.

`Send + Sync` на самом trait фиксирует это требование на уровне контракта.

---

### Полезный default-метод

Trait может содержать не только обязательные методы, но и методы с реализацией по умолчанию.

Например, обработка нескольких текстов может быть построена поверх `process_text`:

```rust
#[async_trait]
pub trait FileProcessingService: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;

    async fn process_text(
        &self,
        content: &str,
        filename: Option<String>,
    ) -> Result<ProcessingResult, Self::Error>;

    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error>;

    async fn list_results(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error>;

    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error>;

    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error>;

    async fn cleanup_old(
        &self,
        days: i32,
    ) -> Result<u64, Self::Error>;

    /// Обработать несколько текстов.
    async fn process_many(
        &self,
        items: Vec<(String, Option<String>)>,
    ) -> Result<Vec<ProcessingResult>, Self::Error> {
        let mut results = Vec::with_capacity(items.len());

        for (content, filename) in items {
            let result = self.process_text(&content, filename).await?;
            results.push(result);
        }

        Ok(results)
    }
}
```

Здесь `process_many()` не требует отдельной реализации от каждого сервиса.

Он использует уже существующий контракт:

```rust
process_text()
```

Это хороший пример повторного использования поведения через trait.

### Почему мы не добавляем сюда `tokio::fs`?

Можно было бы написать:

```rust
async fn process_file_path(...)
```

и читать файл прямо внутри trait.

Но это архитектурно связывает domain API с конкретной инфраструктурой:

```text
Domain
   │
   └── tokio::fs
```

В нашей архитектуре это нежелательно.

Domain должен работать с **данными**, а не с конкретным способом их получения.

Поэтому:

```text
HTTP ──────────────┐
CLI ───────────────┤
File system ───────┤──► Application ──► Domain trait
Database ──────────┘
```

Файловую систему будем подключать на инфраструктурном уровне.

---

### Минимальный пример ассоциированного типа

Для эксперимента можно использовать гораздо более простой trait:

```rust
trait Service {
    type Error: std::error::Error;

    fn execute(&self) -> Result<String, Self::Error>;
}

#[derive(Debug)]
struct MyError;

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "service error")
    }
}

impl std::error::Error for MyError {}

struct MyService;

impl Service for MyService {
    type Error = MyError;

    fn execute(&self) -> Result<String, Self::Error> {
        Ok("Hello, trait!".to_string())
    }
}

fn main() {
    let result = MyService.execute().unwrap();

    println!("{result}");
}
```

**Открыть пример в Rust Playground:**
[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

В этом примере хорошо видно главное свойство ассоциированного типа:

```rust
type Error = MyError;
```

Тип ошибки выбирает реализация, а не пользователь trait.

---

## 82.4. Обновляем `lib.rs`

Добавим новый модуль:

```rust
//! Domain layer.

mod error;
mod models;
mod repositories;
mod services;

pub use error::*;
pub use models::*;
pub use repositories::*;
pub use services::*;
```

Создадим:

```text
crates/domain/src/services/mod.rs
```

```rust
mod file_processing_service;

pub use file_processing_service::*;
```

В результате внешний код может написать:

```rust
use file_processor_domain::FileProcessingService;
```

вместо:

```rust
use file_processor_domain::services::file_processing_service::FileProcessingService;
```

Это позволяет контролировать публичный API crate.

---

## 82.5. Реализация сервиса

Теперь создадим инфраструктурную реализацию:

```text
crates/infrastructure/src/services/file_processing_service.rs
```

Используем `Arc`, поскольку репозиторий может совместно использоваться несколькими компонентами приложения.

```rust
use async_trait::async_trait;
use std::sync::Arc;
use std::time::Instant;
use uuid::Uuid;

use file_processor_domain::{
    Config,
    DomainError,
    FileProcessingService,
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
    Processor,
    ResultFilters,
    UsageStats,
};

pub struct FileProcessingServiceImpl {
    repository: Arc<dyn ProcessingRepository>,
    processor: Processor,
}

impl FileProcessingServiceImpl {
    pub fn new(
        repository: Arc<dyn ProcessingRepository>,
        config: Config,
    ) -> Self {
        Self {
            repository,
            processor: Processor::new(config),
        }
    }

    fn process_lines(&self, lines: Vec<String>) -> Vec<String> {
        self.processor.apply_transforms(lines)
    }
}

#[async_trait]
impl FileProcessingService for FileProcessingServiceImpl {
    type Error = DomainError;

    async fn process_text(
        &self,
        content: &str,
        filename: Option<String>,
    ) -> Result<ProcessingResult, Self::Error> {
        let start = Instant::now();

        let lines: Vec<String> =
            content.lines().map(str::to_owned).collect();

        let processed = self.process_lines(lines);

        let elapsed = start.elapsed();

        let new_result = NewProcessingResult {
            filename,
            input_content: content.to_owned(),
            output_content: processed.join("\n"),
            lines_processed: processed.len(),
            bytes_processed: processed
                .iter()
                .map(|line| line.len() as u64)
                .sum(),
            processing_time_ms: elapsed.as_millis() as u64,
        };

        let result = self.repository.save(new_result).await?;

        Ok(result)
    }

    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error> {
        self.repository.find_by_id(id).await
    }

    async fn list_results(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error> {
        self.repository.find_all(filters).await
    }

    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error> {
        self.repository.get_stats().await
    }

    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error> {
        self.repository.delete(id).await
    }

    async fn cleanup_old(
        &self,
        days: i32,
    ) -> Result<u64, Self::Error> {
        self.repository.delete_older_than(days).await
    }
}
```

Здесь особенно важно следующее:

```rust
impl FileProcessingService for FileProcessingServiceImpl {
    type Error = DomainError;
}
```

Мы связали конкретную реализацию с конкретной ошибкой.

После этого можно использовать:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

---

## 82.6. Mock-реализация

Mock должен быть максимально простым.

Ему не нужны:

- PostgreSQL;
- SQLx;
- миграции;
- файлы;
- HTTP.

Он хранит результаты в памяти.

Разместим его в application crate, потому что это тестовая реализация application-level API:

```text
crates/application/
└── src/
    └── test_support/
        └── mock_service.rs
```

```rust
use async_trait::async_trait;
use chrono::Utc;
use std::collections::HashMap;
use tokio::sync::Mutex;
use uuid::Uuid;

use file_processor_domain::{
    DomainError,
    FileProcessingService,
    ProcessingResult,
    ResultFilters,
    UsageStats,
};

pub struct MockFileProcessingService {
    storage: Mutex<HashMap<Uuid, ProcessingResult>>,
}

impl MockFileProcessingService {
    pub fn new() -> Self {
        Self {
            storage: Mutex::new(HashMap::new()),
        }
    }
}

impl Default for MockFileProcessingService {
    fn default() -> Self {
        Self::new()
    }
}

#[async_trait]
impl FileProcessingService for MockFileProcessingService {
    type Error = DomainError;

    async fn process_text(
        &self,
        content: &str,
        filename: Option<String>,
    ) -> Result<ProcessingResult, Self::Error> {
        let processed = content
            .lines()
            .map(str::to_uppercase)
            .collect::<Vec<_>>();

        let result = ProcessingResult {
            id: Uuid::new_v4(),
            filename,
            input_content: content.to_owned(),
            output_content: processed.join("\n"),
            lines_processed: processed.len(),
            bytes_processed: processed
                .iter()
                .map(|line| line.len() as u64)
                .sum(),
            processing_time_ms: 0,
            created_at: Utc::now(),
        };

        let mut storage = self.storage.lock().await;

        storage.insert(result.id, result.clone());

        Ok(result)
    }

    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error> {
        let storage = self.storage.lock().await;

        Ok(storage.get(&id).cloned())
    }

    async fn list_results(
        &self,
        _filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error> {
        let storage = self.storage.lock().await;

        Ok(storage.values().cloned().collect())
    }

    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error> {
        let storage = self.storage.lock().await;

        let total_requests = storage.len() as u64;

        let total_lines_processed = storage
            .values()
            .map(|result| result.lines_processed as u64)
            .sum();

        let total_bytes_processed = storage
            .values()
            .map(|result| result.bytes_processed)
            .sum();

        let avg_processing_time_ms =
            if total_requests == 0 {
                0.0
            } else {
                storage
                    .values()
                    .map(|result| result.processing_time_ms as f64)
                    .sum::<f64>()
                    / total_requests as f64
            };

        Ok(UsageStats {
            total_requests,
            total_lines_processed,
            total_bytes_processed,
            avg_processing_time_ms,
        })
    }

    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error> {
        let mut storage = self.storage.lock().await;

        Ok(storage.remove(&id).is_some())
    }

    async fn cleanup_old(
        &self,
        _days: i32,
    ) -> Result<u64, Self::Error> {
        let mut storage = self.storage.lock().await;

        let count = storage.len() as u64;

        storage.clear();

        Ok(count)
    }
}
```

Важное отличие от реальной базы данных:

```rust
storage: Mutex<HashMap<Uuid, ProcessingResult>>
```

Это не имитация PostgreSQL на уровне SQL.

Mock имитирует **контракт**:

```text
FileProcessingService
        │
        ├── process_text()
        ├── get_result()
        ├── list_results()
        ├── get_stats()
        ├── delete_result()
        └── cleanup_old()
```

И именно это нам необходимо для unit/integration-тестов.

---

## 82.7. Generic-реализация

Trait можно реализовать не только для конкретного типа:

```rust
impl FileProcessingService for FileProcessingServiceImpl
```

но и для обобщённого типа.

Это особенно полезно, если конкретный сервис должен работать с разными реализациями репозитория.

```rust
use async_trait::async_trait;
use std::sync::Arc;
use uuid::Uuid;

use file_processor_domain::{
    Config,
    DomainError,
    FileProcessingService,
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
    Processor,
    ResultFilters,
    UsageStats,
};

pub struct GenericFileProcessingService<R> {
    repository: Arc<R>,
    processor: Processor,
}

impl<R> GenericFileProcessingService<R> {
    pub fn new(
        repository: Arc<R>,
        config: Config,
    ) -> Self {
        Self {
            repository,
            processor: Processor::new(config),
        }
    }
}

#[async_trait]
impl<R> FileProcessingService
    for GenericFileProcessingService<R>
where
    R: ProcessingRepository + Send + Sync,
{
    type Error = DomainError;

    async fn process_text(
        &self,
        content: &str,
        filename: Option<String>,
    ) -> Result<ProcessingResult, Self::Error> {
        let lines: Vec<String> =
            content.lines().map(str::to_owned).collect();

        let processed =
            self.processor.apply_transforms(lines);

        let new_result = NewProcessingResult {
            filename,
            input_content: content.to_owned(),
            output_content: processed.join("\n"),
            lines_processed: processed.len(),
            bytes_processed: processed
                .iter()
                .map(|line| line.len() as u64)
                .sum(),
            processing_time_ms: 0,
        };

        self.repository
            .save(new_result)
            .await
            .map_err(|error| DomainError::Repository(error.to_string()))
    }

    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error> {
        self.repository
            .find_by_id(id)
            .await
    }

    async fn list_results(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error> {
        self.repository
            .find_all(filters)
            .await
    }

    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error> {
        self.repository
            .get_stats()
            .await
    }

    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error> {
        self.repository
            .delete(id)
            .await
    }

    async fn cleanup_old(
        &self,
        days: i32,
    ) -> Result<u64, Self::Error> {
        self.repository
            .delete_older_than(days)
            .await
    }
}
```

Здесь есть важное отличие от первоначального варианта главы.

`ProcessingRepository` в нашей архитектуре использует единый:

```rust
DomainResult<T>
```

поэтому у него **нет**:

```rust
R::Error
```

Следовательно, нельзя писать:

```rust
type Error = R::Error;
```

если такой ассоциированный тип действительно не объявлен в `ProcessingRepository`.

Вместо этого generic-реализация приводит ошибку репозитория к доменной ошибке:

```rust
.map_err(|error| {
    DomainError::Repository(error.to_string())
})
```

Это согласуется с архитектурой предыдущей главы.

---

## 82.8. Статическая и динамическая диспетчеризация

Теперь у нас есть две возможности.

### Статическая диспетчеризация

Можно принимать generic:

```rust
async fn run<S>(
    service: &S,
    content: &str,
) -> Result<(), S::Error>
where
    S: FileProcessingService,
{
    service
        .process_text(content, None)
        .await?;

    Ok(())
}
```

Конкретный тип `S` известен во время компиляции.

Компилятор может специализировать код под конкретную реализацию.

---

### Динамическая диспетчеризация

Если тип сервиса нужно скрыть:

```rust
use std::sync::Arc;

type DynService =
    Arc<dyn FileProcessingService<Error = DomainError>>;
```

Теперь можно передать:

```rust
let service: DynService =
    Arc::new(FileProcessingServiceImpl::new(
        repository,
        config,
    ));
```

А тест может использовать:

```rust
let service: DynService =
    Arc::new(MockFileProcessingService::new());
```

Потребителю совершенно не важно, какая конкретно структура находится внутри `Arc`.

Схематически:

```text
             FileProcessingService
                       ▲
                       │
                dyn FileProcessingService
                       │
                 ┌─────┴─────┐
                 │           │
                 ▼           ▼
              Real          Mock
              service       service
```

У динамической диспетчеризации есть небольшая стоимость вызова через vtable, зато она позволяет скрыть конкретный тип и менять реализации во время выполнения.

---

## 82.9. Dependency Injection через trait API

Теперь dependency injection становится очень простым.

Создадим:

```rust
use std::sync::Arc;

use file_processor_domain::{
    DomainError,
    FileProcessingService,
};

pub type DynFileProcessingService =
    Arc<dyn FileProcessingService<Error = DomainError>>;
```

Например:

```rust
pub struct AppState {
    pub service: DynFileProcessingService,
}
```

Production:

```rust
let service = Arc::new(
    FileProcessingServiceImpl::new(
        repository,
        config,
    )
);

let state = AppState {
    service,
};
```

Test:

```rust
let service = Arc::new(
    MockFileProcessingService::new()
);

let state = AppState {
    service,
};
```

Таким образом, HTTP-слой больше не знает, какой конкретно сервис используется.

---

## 82.10. Использование в HTTP handlers

Теперь HTTP handler работает только с trait API.

```rust
use std::sync::Arc;

use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde::Deserialize;
use serde_json::json;
use uuid::Uuid;

use file_processor_domain::{
    DomainError,
    FileProcessingService,
    ResultFilters,
};

pub type DynFileProcessingService =
    Arc<dyn FileProcessingService<Error = DomainError>>;

pub struct AppState {
    pub service: DynFileProcessingService,
}

#[derive(Debug, Deserialize)]
pub struct ProcessRequest {
    pub content: String,
    pub filename: Option<String>,
}

pub async fn process_text_handler(
    State(state): State<AppState>,
    Json(request): Json<ProcessRequest>,
) -> impl IntoResponse {
    match state
        .service
        .process_text(
            &request.content,
            request.filename,
        )
        .await
    {
        Ok(result) => {
            (StatusCode::OK, Json(result))
                .into_response()
        }

        Err(error) => {
            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(json!({
                    "error": error.to_string()
                })),
            )
                .into_response()
        }
    }
}
```

Обратите внимание:

handler ничего не знает о:

```text
PostgreSQL
SQLx
Processor
FileProcessingServiceImpl
MockFileProcessingService
```

Он знает только:

```rust
state.service.process_text(...)
```

Это и есть практическая ценность trait API.

---

## 82.11. Тестирование через mock

Теперь можем тестировать сервисный контракт без базы данных.

Интеграционный тест должен находиться внутри package, например:

```text
crates/application/
├── Cargo.toml
├── src/
└── tests/
    └── service_test.rs
```

Это важно для workspace.

Если корневой `Cargo.toml` является виртуальным manifest и содержит только:

```toml
[workspace]
```

то корень workspace сам по себе не является package. Поэтому корневой:

```text
tests/
```

не становится автоматически интеграционными тестами одного из crates. Интеграционный тест принадлежит конкретному package и должен находиться в его `tests/` либо быть явно объявлен как test target.

Пример:

```rust
use file_processor_application::MockFileProcessingService;
use file_processor_domain::FileProcessingService;
use uuid::Uuid;

#[tokio::test]
async fn process_text() {
    let service =
        MockFileProcessingService::new();

    let result = service
        .process_text(
            "hello\nworld",
            Some("test.txt".to_string()),
        )
        .await
        .unwrap();

    assert_eq!(result.lines_processed, 2);
    assert_eq!(
        result.output_content,
        "HELLO\nWORLD"
    );

    assert_ne!(result.id, Uuid::nil());
}

#[tokio::test]
async fn get_result() {
    let service =
        MockFileProcessingService::new();

    let created = service
        .process_text("test", None)
        .await
        .unwrap();

    let found = service
        .get_result(created.id)
        .await
        .unwrap();

    assert!(found.is_some());

    assert_eq!(
        found.unwrap().input_content,
        "test"
    );
}

#[tokio::test]
async fn delete_result() {
    let service =
        MockFileProcessingService::new();

    let created = service
        .process_text("test", None)
        .await
        .unwrap();

    let deleted = service
        .delete_result(created.id)
        .await
        .unwrap();

    assert!(deleted);

    let found = service
        .get_result(created.id)
        .await
        .unwrap();

    assert!(found.is_none());
}

#[tokio::test]
async fn get_stats() {
    let service =
        MockFileProcessingService::new();

    service
        .process_text("hello", None)
        .await
        .unwrap();

    service
        .process_text("world", None)
        .await
        .unwrap();

    service
        .process_text("rust", None)
        .await
        .unwrap();

    let stats =
        service.get_stats().await.unwrap();

    assert_eq!(stats.total_requests, 3);
    assert_eq!(stats.total_lines_processed, 3);
}
```

Теперь тесты проверяют именно публичный контракт:

```rust
FileProcessingService
```

а не внутреннее устройство mock.

Это очень важное различие.

---

## 82.12. Проверяем `dyn Trait`

Можно отдельно убедиться, что trait действительно можно использовать как trait object:

```rust
use std::sync::Arc;

use file_processor_domain::{
    DomainError,
    FileProcessingService,
};

fn accept_service(
    service: Arc<
        dyn FileProcessingService<
            Error = DomainError
        >
    >
) {
    let _ = service;
}
```

Если trait содержит:

```rust
type Error;
```

то при использовании `dyn Trait` необходимо указать конкретное значение ассоциированного типа:

```rust
dyn FileProcessingService<Error = DomainError>
```

Нельзя написать просто:

```rust
dyn FileProcessingService
```

потому что компилятор не знает, какой именно `Error` имеется в виду.

Это одна из важных особенностей ассоциированных типов.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Неполная реализация trait

```rust
trait Service {
    type Error;

    fn execute(&self) -> Result<(), Self::Error>;
    fn cleanup(&self);
}

struct MyService;

impl Service for MyService {
    type Error = ();

    fn execute(&self) -> Result<(), Self::Error> {
        Ok(())
    }

    // ❌ cleanup отсутствует
}
```

Компилятор сообщит, что реализация trait не завершена.

Trait — это контракт.

Если метод не имеет default-реализации, реализация должна его предоставить.

---

### Эксперимент 2: Не указан ассоциированный тип

```rust
trait Service {
    type Error;

    fn execute(&self) -> Result<(), Self::Error>;
}

struct MyService;

impl Service for MyService {
    // ❌ type Error отсутствует

    fn execute(&self) -> Result<(), Self::Error> {
        Ok(())
    }
}
```

Компилятор потребует определить:

```rust
type Error = ...;
```

---

### Эксперимент 3: Неверный ассоциированный тип

```rust
trait Service {
    type Error: std::error::Error;

    fn execute(&self) -> Result<(), Self::Error>;
}

struct MyService;

impl Service for MyService {
    type Error = String;

    fn execute(&self) -> Result<(), Self::Error> {
        Ok(())
    }
}
```

`String` не удовлетворяет bound:

```rust
std::error::Error
```

поэтому реализация не скомпилируется.

---

### Эксперимент 4: `async fn` и `dyn Trait`

Современный Rust позволяет:

```rust
trait Service {
    async fn execute(&self);
}
```

Но попытка:

```rust
let service: Box<dyn Service>;
```

приведёт к ошибке dyn compatibility.

Это принципиальное отличие:

```text
async fn in trait
        │
        ├── обычная generic/static dispatch
        │       ✓
        │
        └── dyn Trait
                ✗
```

Для `dyn Trait` в нашем случае используется:

```rust
#[async_trait]
```

`async-trait` как раз предназначен для type erasure асинхронных методов и позволяет использовать такие traits как trait objects.

**Открыть пример в Rust Playground:**
[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

### Эксперимент 5: Ассоциированный тип в `dyn Trait`

```rust
trait Service {
    type Error;

    fn execute(&self) -> Result<(), Self::Error>;
}

struct MyService;

impl Service for MyService {
    type Error = ();

    fn execute(&self) -> Result<(), Self::Error> {
        Ok(())
    }
}

// ❌ Какой Error?
fn use_service(
    service: &dyn Service
) {
    let _ = service;
}
```

Чтобы trait object был однозначным, необходимо указать:

```rust
fn use_service(
    service: &dyn Service<Error = ()>
) {
    let _ = service;
}
```

**Открыть пример в Rust Playground:**
[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## Практика

### Задание 1

Создайте trait:

```rust
FileProcessingService
```

со следующими операциями:

```text
process_text
get_result
list_results
get_stats
delete_result
cleanup_old
```

---

### Задание 2

Добавьте ассоциированный тип:

```rust
type Error;
```

и задайте ему ограничения:

```rust
type Error:
    std::error::Error
    + Send
    + Sync
    + 'static;
```

---

### Задание 3

Создайте production-реализацию:

```rust
FileProcessingServiceImpl
```

которая использует:

```rust
Arc<dyn ProcessingRepository>
```

---

### Задание 4

Создайте:

```rust
MockFileProcessingService
```

который хранит данные в:

```rust
HashMap<Uuid, ProcessingResult>
```

---

### Задание 5

Создайте функцию, принимающую generic service:

```rust
async fn run<S>(
    service: &S,
) -> Result<(), S::Error>
where
    S: FileProcessingService,
{
    // ...
}
```

Сравните этот вариант со:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Попробуйте использовать:

```rust
dyn FileProcessingService
```

без указания:

```rust
Error = ...
```

Объясните ошибку компилятора.

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Уберите:

```rust
#[async_trait]
```

и попробуйте создать:

```rust
Arc<dyn FileProcessingService<Error = DomainError>>
```

Объясните, почему trait с нативным `async fn` нельзя использовать как `dyn Trait`.

---

### Задание 8

Добавьте default-метод:

```rust
process_many()
```

который использует:

```rust
process_text()
```

и убедитесь, что production и mock реализации получают этот метод автоматически.

---

## Главное из этой главы

После этой главы мы:

- **создали** собственный trait API;
- **определили** публичный контракт сервиса;
- **использовали** ассоциированный тип `Error`;
- **разобрались** с `async fn` в traits;
- **поняли** разницу между static и dynamic dispatch;
- **реализовали** production-сервис;
- **создали** mock-реализацию;
- **использовали** `Arc<dyn Trait>`;
- **интегрировали** trait API с HTTP handlers;
- **написали** тесты без PostgreSQL;
- **добавили** default-метод к trait;
- **увидели**, почему domain API не должен зависеть от файловой системы или Tokio.

### Самая важная идея

> **Trait API — это контракт, а не реализация.**

Клиент должен зависеть от:

```rust
FileProcessingService
```

а не от:

```rust
FileProcessingServiceImpl
```

Архитектурно это выглядит так:

```text
                   CONTRACT
                       │
                       ▼
          FileProcessingService
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Production     Mock       Alternative
       implementation implementation implementation
```

Именно поэтому trait является одним из центральных инструментов проектирования Rust-приложений.

Ассоциированный тип позволяет каждой реализации определить собственный тип ошибки:

```rust
type Error = DomainError;
```

а generic implementation позволяет использовать один и тот же контракт с разными конкретными типами.

Наконец, современный Rust уже поддерживает `async fn` непосредственно в traits, но это не означает, что любой такой trait можно использовать как `dyn Trait`. Для статической диспетчеризации нативный `async fn` обычно является предпочтительным вариантом; когда архитектуре необходим динамический trait object, как в нашем `Arc<dyn FileProcessingService>`, `async-trait` остаётся практическим решением.

Таким образом, мы получили не просто «ещё один trait», а **стабильную границу между бизнес-контрактом и конкретной реализацией**:

```text
                 HTTP
                  │
                  ▼
        ┌───────────────────┐
        │ FileProcessing    │
        │     Service       │
        └─────────┬─────────┘
                  │
             trait API
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   Production              Mock
        │                   │
        ▼                   ▼
   PostgreSQL             Memory
```

Это уже тот уровень абстракции, на котором можно строить настоящий многокрейтовый Rust-сервис.
