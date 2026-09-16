# Глава 81. Создаём workspace

Наш проект вырос из простого CLI-инструмента в полноценную систему с библиотекой, HTTP API и базой данных. Теперь пришло время структурировать его как профессиональный **workspace** с чёткими границами между крейтами.

Workspace позволяет управлять несколькими пакетами в одном репозитории: у них общий `Cargo.lock`, общий каталог `target`, единое разрешение зависимостей и общие настройки. При этом каждый пакет остаётся самостоятельным крейтом со своим `Cargo.toml`.

В этой главе мы:

- создадим workspace;
- разделим проект на Domain, Application и Infrastructure;
- определим направления зависимостей;
- перенесём модели и repository trait в Domain;
- перенесём use cases в Application;
- реализуем PostgreSQL repository в Infrastructure;
- добавим HTTP API;
- создадим composition root;
- настроим dependency injection;
- организуем тесты;
- научимся проверять весь workspace одной командой.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 81.1. Зачем нам workspace?

Пока проект состоял из одного крейта, структура могла выглядеть просто:

```text
file_processor/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── processor.rs
    ├── http.rs
    └── db.rs
```

По мере роста проекта такой подход начинает создавать проблему: технические детали оказываются перемешаны с бизнес-логикой.

Например, если доменная логика напрямую использует `sqlx::PgPool`, то теперь она знает о PostgreSQL. Если она импортирует `axum`, она знает о HTTP. Если use case создаёт `PostgresProcessingRepository` самостоятельно, его уже нельзя нормально протестировать без PostgreSQL.

Workspace позволяет физически разделить эти части.

Мы построим следующую архитектуру:

```text
file_processor/
│
├── crates/
│   │
│   ├── domain/
│   │   └── file_processor_domain
│   │
│   ├── application/
│   │   └── file_processor_application
│   │
│   └── infrastructure/
│       └── file_processor_infrastructure
│
└── Cargo.toml
```

Главная идея:

```text
Domain
   ↑
Application
   ↑
Infrastructure
```

Но стрелка здесь означает **зависимость снизу вверх**:

```text
Application → Domain
Infrastructure → Application
Infrastructure → Domain
```

То есть Domain ничего не знает о внешних слоях.

---

## 81.2. Чистая архитектура

Мы будем использовать упрощённый вариант **Clean Architecture**.

### Domain Layer

Domain содержит то, что относится к предметной области:

- модели;
- правила;
- бизнес-ошибки;
- интерфейсы repository;
- чистую бизнес-логику.

Domain не должен знать:

- PostgreSQL;
- SQLx;
- Axum;
- HTTP;
- CLI;
- конкретную файловую систему.

### Application Layer

Application содержит сценарии использования системы:

- обработать файл;
- получить результат;
- получить список результатов;
- удалить результат;
- получить статистику.

Application знает Domain, но не знает конкретную реализацию инфраструктуры.

### Infrastructure Layer

Infrastructure содержит технические детали:

- PostgreSQL;
- SQLx;
- HTTP;
- Axum;
- CLI;
- конфигурацию;
- логирование.

Именно Infrastructure соединяет приложение с внешним миром.

Архитектура:

```text
┌─────────────────────────────────────────────────────────────┐
│                    Infrastructure                           │
│                                                             │
│   PostgreSQL     HTTP/Axum       CLI        Config          │
│       │              │             │           │            │
└───────┼──────────────┼─────────────┼───────────┼────────────┘
        │              │             │
        ▼              ▼             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application                              │
│                                                             │
│   ProcessFile    ListResults    GetResult    DeleteResult   │
│        │              │             │            │          │
└────────┼──────────────┼─────────────┼────────────┼──────────┘
         │              │             │            │
         ▼              ▼             ▼            ▼
┌─────────────────────────────────────────────────────────────┐
│                       Domain                                │
│                                                             │
│   Models     Business Rules     Repository Traits           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Это важнее структуры каталогов.

**Архитектуру определяют не папки, а зависимости.**

---

## 81.3. Направления зависимостей

Наше правило:

```text
Domain
   ↑
Application
   ↑
Infrastructure
```

В терминах Cargo:

```text
application
    └── depends on → domain

infrastructure
    ├── depends on → domain
    └── depends on → application
```

При этом:

```text
domain
    ├── НЕ зависит от application
    └── НЕ зависит от infrastructure

application
    └── НЕ зависит от infrastructure
```

Получается:

```text
                    ┌─────────────┐
                    │   Domain    │
                    └──────▲──────┘
                           │
                    ┌──────┴──────┐
                    │ Application │
                    └──────▲──────┘
                           │
                    ┌──────┴──────┐
                    │Infrastructure│
                    └─────────────┘
```

Почему это полезно?

Предположим, repository определён в Domain:

```rust
#[async_trait]
pub trait ProcessingRepository: Send + Sync {
    async fn save(
        &self,
        result: NewProcessingResult,
    ) -> DomainResult<ProcessingResult>;
}
```

Domain знает только об **абстракции**.

Infrastructure может реализовать её через PostgreSQL:

```rust
pub struct PostgresProcessingRepository {
    pool: PgPool,
}
```

В будущем можно добавить:

```text
PostgresProcessingRepository
InMemoryProcessingRepository
FileProcessingRepository
MockProcessingRepository
```

и Application не придётся менять.

Это и есть Dependency Inversion Principle:

> Высокоуровневый код зависит от абстракции, а не от конкретной технической реализации.

---

## 81.4. Структура workspace

В результате получим:

```text
file_processor/
│
├── Cargo.toml
├── Cargo.lock
│
├── crates/
│   │
│   ├── domain/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── error.rs
│   │       ├── models/
│   │       │   ├── mod.rs
│   │       │   └── processing.rs
│   │       └── repositories/
│   │           ├── mod.rs
│   │           └── processing_repository.rs
│   │
│   ├── application/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       └── use_cases/
│   │           ├── mod.rs
│   │           └── process_file.rs
│   │
│   └── infrastructure/
│       ├── Cargo.toml
│       ├── migrations/
│       │   └── 20260831000000_create_processing_results.sql
│       ├── src/
│       │   ├── lib.rs
│       │   ├── composition.rs
│       │   ├── db/
│       │   │   ├── mod.rs
│       │   │   ├── connection.rs
│       │   │   └── processing_repository.rs
│       │   └── http/
│       │       ├── mod.rs
│       │       ├── state.rs
│       │       ├── routes.rs
│       │       └── handlers.rs
│       │
│       └── tests/
│           └── repository_test.rs
│
├── .env.example
├── .gitignore
└── README.md
```

Обратите внимание на важное изменение по сравнению с предыдущей архитектурой.

Мы **не создаём отдельный `shared` crate**.

Если создать `shared` слишком рано, туда начинают попадать совершенно разные вещи:

```text
config
logging
time
helpers
DTO
database helpers
HTTP helpers
...
```

Через некоторое время `shared` становится зависимостью почти всех крейтов и фактически разрушает границы архитектуры.

Поэтому правило проще:

> Создавайте отдельный crate только тогда, когда у него действительно есть самостоятельная ответственность.

---

## 81.5. Корневой `Cargo.toml`

Создадим виртуальный workspace:

```toml
[workspace]
members = [
    "crates/domain",
    "crates/application",
    "crates/infrastructure",
]
resolver = "3"

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"

[workspace.dependencies]

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Domain types
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde", "clock"] }

# Async
tokio = { version = "1.0", features = ["full"] }
async-trait = "0.1"

# Errors
thiserror = "2.0"
anyhow = "1.0"

# Database
sqlx = { version = "0.8", features = [
    "runtime-tokio-native-tls",
    "postgres",
    "uuid",
    "chrono",
    "migrate",
] }

# HTTP
axum = "0.7"
tower = "0.5"
tower-http = "0.6"

# CLI
clap = { version = "4.0", features = ["derive", "env"] }

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Testing
tempfile = "3.0"

[profile.release]
lto = true
codegen-units = 1
```

Здесь используется **virtual workspace**: в корневом `Cargo.toml` нет секции `[package]`.

Это означает, что сам корень не является crate. Он только управляет пакетами workspace. Для виртуального workspace `resolver` задаётся явно.

### Зачем нужен `[workspace.dependencies]`?

Без него каждый crate мог бы содержать:

```toml
serde = "1.0"
uuid = "1.0"
tokio = "1.0"
```

Теперь версии централизованы.

В member crate достаточно написать:

```toml
serde = { workspace = true }
```

Это уменьшает вероятность того, что разные части проекта начнут использовать несовместимые версии одной библиотеки.

---

## 81.6. Domain crate

Domain — самый независимый crate проекта.

### `crates/domain/Cargo.toml`

```toml
[package]
name = "file_processor_domain"
version.workspace = true
edition.workspace = true
license.workspace = true
description = "Domain models and business rules"

[dependencies]
serde = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
async-trait = { workspace = true }
```

Обратите внимание:

```toml
sqlx = ...
```

здесь отсутствует.

Это принципиально.

Domain не знает, где хранятся данные.

---

## 81.7. Domain models

### `crates/domain/src/models/processing.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Результат обработки файла.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessingResult {
    pub id: Uuid,
    pub filename: Option<String>,
    pub input_content: String,
    pub output_content: String,
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub processing_time_ms: u64,
    pub created_at: DateTime<Utc>,
}

/// Данные, необходимые для создания результата.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NewProcessingResult {
    pub filename: Option<String>,
    pub input_content: String,
    pub output_content: String,
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub processing_time_ms: u64,
}

/// Фильтры поиска результатов.
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ResultFilters {
    pub limit: Option<usize>,
    pub offset: Option<usize>,
    pub filename_contains: Option<String>,
    pub from_date: Option<DateTime<Utc>>,
    pub to_date: Option<DateTime<Utc>>,
}

/// Статистика использования.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UsageStats {
    pub total_requests: u64,
    pub total_lines_processed: u64,
    pub total_bytes_processed: u64,
    pub avg_processing_time_ms: f64,
}
```

### `crates/domain/src/models/mod.rs`

```rust
mod processing;

pub use processing::*;
```

### `crates/domain/src/repositories/processing_repository.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;

use crate::{
    error::DomainResult,
    models::{
        NewProcessingResult,
        ProcessingResult,
        ResultFilters,
        UsageStats,
    },
};

#[async_trait]
pub trait ProcessingRepository: Send + Sync {
    async fn save(
        &self,
        result: NewProcessingResult,
    ) -> DomainResult<ProcessingResult>;

    async fn find_by_id(
        &self,
        id: Uuid,
    ) -> DomainResult<Option<ProcessingResult>>;

    async fn find_all(
        &self,
        filters: ResultFilters,
    ) -> DomainResult<Vec<ProcessingResult>>;

    async fn delete(&self, id: Uuid) -> DomainResult<bool>;

    async fn get_stats(&self) -> DomainResult<UsageStats>;

    async fn delete_older_than(
        &self,
        days: i32,
    ) -> DomainResult<u64>;
}
```

Repository — это **контракт**, а не PostgreSQL-код.

Domain говорит:

> Мне нужен объект, который умеет сохранять и получать `ProcessingResult`.

Но Domain не говорит:

> Используй PostgreSQL.

---

## 81.8. Ошибки Domain

### `crates/domain/src/error.rs`

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("validation error: {0}")]
    Validation(String),

    #[error("processing error: {0}")]
    Processing(String),

    #[error("repository error: {0}")]
    Repository(String),
}

pub type DomainResult<T> = Result<T, DomainError>;
```

Обратите внимание: здесь больше нет:

```rust
Sqlx(#[from] sqlx::Error)
```

Это ещё один важный архитектурный момент.

Если Domain содержит `sqlx::Error`, PostgreSQL уже проник в Domain.

Infrastructure преобразует свои технические ошибки:

```rust
.map_err(|error| DomainError::Repository(error.to_string()))?
```

---

## 81.9. `lib.rs` Domain

### `crates/domain/src/lib.rs`

```rust
pub mod error;
pub mod models;
pub mod repositories;

pub use error::{DomainError, DomainResult};
pub use models::*;
pub use repositories::*;
```

Теперь внешний код может писать:

```rust
use file_processor_domain::{
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
};
```

---

## 81.10. Application crate

Application реализует **use cases**.

### `crates/application/Cargo.toml`

```toml
[package]
name = "file_processor_application"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
file_processor_domain = {
    path = "../domain"
}

serde = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
async-trait = { workspace = true }
tracing = { workspace = true }
thiserror = { workspace = true }
```

Application зависит от Domain:

```text
application
     │
     ▼
   domain
```

Но не зависит от Infrastructure.

---

## 81.11. Use case `ProcessFile`

### `crates/application/src/use_cases/process_file.rs`

```rust
use std::{
    sync::Arc,
    time::Instant,
};

use file_processor_domain::{
    Config,
    DomainResult,
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
    Processor,
};

use tracing::info;

pub struct ProcessFileUseCase {
    repository: Arc<dyn ProcessingRepository>,
    processor: Processor,
}

impl ProcessFileUseCase {
    pub fn new(
        repository: Arc<dyn ProcessingRepository>,
        processor: Processor,
    ) -> Self {
        Self {
            repository,
            processor,
        }
    }

    pub async fn execute(
        &self,
        input: &str,
        filename: Option<String>,
    ) -> DomainResult<ProcessingResult> {
        info!("Processing file: {:?}", filename);

        let start = Instant::now();

        let lines: Vec<String> =
            input.lines().map(str::to_owned).collect();

        let processed = self.processor.apply_transforms(lines);

        let elapsed = start.elapsed();

        let output_content = processed.join("\n");

        let new_result = NewProcessingResult {
            filename,
            input_content: input.to_owned(),
            output_content,
            lines_processed: processed.len(),
            bytes_processed: input.len() as u64,
            processing_time_ms: elapsed.as_millis() as u64,
        };

        let result = self.repository.save(new_result).await?;

        info!(
            "Processed {} lines in {} ms",
            result.lines_processed,
            result.processing_time_ms
        );

        Ok(result)
    }
}
```

Здесь есть важная деталь:

```rust
Arc<dyn ProcessingRepository>
```

Use case не знает конкретный тип repository.

Он знает только интерфейс:

```rust
ProcessingRepository
```

Это позволяет передать:

```text
PostgresProcessingRepository
InMemoryProcessingRepository
MockProcessingRepository
```

без изменения use case.

---

## 81.12. Модули Application

### `crates/application/src/use_cases/mod.rs`

```rust
mod process_file;

pub use process_file::ProcessFileUseCase;
```

### `crates/application/src/lib.rs`

```rust
pub mod use_cases;

pub use use_cases::ProcessFileUseCase;
```

---

## 81.13. Infrastructure crate

Теперь можно перейти к техническим деталям.

### `crates/infrastructure/Cargo.toml`

```toml
[package]
name = "file_processor_infrastructure"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
file_processor_domain = {
    path = "../domain"
}

file_processor_application = {
    path = "../application"
}

tokio = { workspace = true }
sqlx = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
async-trait = { workspace = true }
axum = { workspace = true }
tracing = { workspace = true }
anyhow = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
```

Теперь граф зависимостей выглядит так:

```text
file_processor_infrastructure
        │
        ├──────────────► file_processor_application
        │                       │
        │                       ▼
        └──────────────────► file_processor_domain
```

---

## 81.14. Подключение PostgreSQL

### `crates/infrastructure/src/db/connection.rs`

```rust
use sqlx::{
    postgres::PgPoolOptions,
    PgPool,
};

pub async fn create_pool(
    database_url: &str,
) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(10)
        .connect(database_url)
        .await
}

pub async fn run_migrations(
    pool: &PgPool,
) -> Result<(), sqlx::migrate::MigrateError> {
    sqlx::migrate!("./migrations")
        .run(pool)
        .await
}
```

Миграции находятся рядом с Infrastructure:

```text
crates/infrastructure/
├── migrations/
└── src/
```

Это важно для:

```rust
sqlx::migrate!("./migrations")
```

Путь привязан к crate, в котором находится макрос.

---

## 81.15. Миграция

### `crates/infrastructure/migrations/20260831000000_create_processing_results.sql`

```sql
CREATE TABLE processing_results (
    id UUID PRIMARY KEY,
    filename TEXT,
    input_content TEXT NOT NULL,
    output_content TEXT NOT NULL,
    lines_processed BIGINT NOT NULL DEFAULT 0,
    bytes_processed BIGINT NOT NULL DEFAULT 0,
    processing_time_ms BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_processing_results_created_at
    ON processing_results (created_at DESC);

CREATE INDEX idx_processing_results_filename
    ON processing_results (filename);
```

Мы намеренно не используем `uuid-ossp`.

UUID генерируется Rust:

```rust
let id = Uuid::new_v4();
```

Поэтому PostgreSQL не требуется дополнительное расширение.

---

## 81.16. PostgreSQL repository

Теперь реализуем интерфейс Domain:

```rust
ProcessingRepository
```

через PostgreSQL.

### `crates/infrastructure/src/db/processing_repository.rs`

```rust
use async_trait::async_trait;
use chrono::{Duration, Utc};
use sqlx::PgPool;
use uuid::Uuid;

use file_processor_domain::{
    DomainError,
    DomainResult,
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
    ResultFilters,
    UsageStats,
};

pub struct PostgresProcessingRepository {
    pool: PgPool,
}

impl PostgresProcessingRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    fn map_row(row: ProcessingRow) -> ProcessingResult {
        ProcessingResult {
            id: row.id,
            filename: row.filename,
            input_content: row.input_content,
            output_content: row.output_content,
            lines_processed: row.lines_processed as usize,
            bytes_processed: row.bytes_processed as u64,
            processing_time_ms: row.processing_time_ms as u64,
            created_at: row.created_at,
        }
    }
}

#[derive(sqlx::FromRow)]
struct ProcessingRow {
    id: Uuid,
    filename: Option<String>,
    input_content: String,
    output_content: String,
    lines_processed: i64,
    bytes_processed: i64,
    processing_time_ms: i64,
    created_at: chrono::DateTime<Utc>,
}

#[async_trait]
impl ProcessingRepository for PostgresProcessingRepository {
    async fn save(
        &self,
        result: NewProcessingResult,
    ) -> DomainResult<ProcessingResult> {
        let id = Uuid::new_v4();
        let created_at = Utc::now();

        sqlx::query(
            r#"
            INSERT INTO processing_results (
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            "#,
        )
        .bind(id)
        .bind(&result.filename)
        .bind(&result.input_content)
        .bind(&result.output_content)
        .bind(result.lines_processed as i64)
        .bind(result.bytes_processed as i64)
        .bind(result.processing_time_ms as i64)
        .bind(created_at)
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(ProcessingResult {
            id,
            filename: result.filename,
            input_content: result.input_content,
            output_content: result.output_content,
            lines_processed: result.lines_processed,
            bytes_processed: result.bytes_processed,
            processing_time_ms: result.processing_time_ms,
            created_at,
        })
    }

    async fn find_by_id(
        &self,
        id: Uuid,
    ) -> DomainResult<Option<ProcessingResult>> {
        let row = sqlx::query_as::<_, ProcessingRow>(
            r#"
            SELECT
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at
            FROM processing_results
            WHERE id = $1
            "#,
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(row.map(Self::map_row))
    }

    async fn find_all(
        &self,
        filters: ResultFilters,
    ) -> DomainResult<Vec<ProcessingResult>> {
        let limit = filters
            .limit
            .unwrap_or(50)
            .min(100) as i64;

        let offset = filters
            .offset
            .unwrap_or(0) as i64;

        let rows = sqlx::query_as::<_, ProcessingRow>(
            r#"
            SELECT
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at
            FROM processing_results
            WHERE
                ($1::text IS NULL
                 OR filename ILIKE '%' || $1 || '%')
            AND
                ($2::timestamptz IS NULL
                 OR created_at >= $2)
            AND
                ($3::timestamptz IS NULL
                 OR created_at <= $3)
            ORDER BY created_at DESC
            LIMIT $4
            OFFSET $5
            "#,
        )
        .bind(filters.filename_contains)
        .bind(filters.from_date)
        .bind(filters.to_date)
        .bind(limit)
        .bind(offset)
        .fetch_all(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(rows
            .into_iter()
            .map(Self::map_row)
            .collect())
    }

    async fn delete(
        &self,
        id: Uuid,
    ) -> DomainResult<bool> {
        let result = sqlx::query(
            "DELETE FROM processing_results WHERE id = $1",
        )
        .bind(id)
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(result.rows_affected() > 0)
    }

    async fn get_stats(&self) -> DomainResult<UsageStats> {
        let row = sqlx::query(
            r#"
            SELECT
                COUNT(*) AS total_requests,
                COALESCE(SUM(lines_processed), 0)
                    AS total_lines,
                COALESCE(SUM(bytes_processed), 0)
                    AS total_bytes,
                COALESCE(AVG(processing_time_ms), 0.0)
                    AS avg_time
            FROM processing_results
            "#,
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        use sqlx::Row;

        let total_requests: i64 =
            row.try_get("total_requests")
                .map_err(|e| DomainError::Repository(e.to_string()))?;

        let total_lines: i64 =
            row.try_get("total_lines")
                .map_err(|e| DomainError::Repository(e.to_string()))?;

        let total_bytes: i64 =
            row.try_get("total_bytes")
                .map_err(|e| DomainError::Repository(e.to_string()))?;

        let avg_time: f64 =
            row.try_get("avg_time")
                .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(UsageStats {
            total_requests: total_requests as u64,
            total_lines_processed: total_lines as u64,
            total_bytes_processed: total_bytes as u64,
            avg_processing_time_ms: avg_time,
        })
    }

    async fn delete_older_than(
        &self,
        days: i32,
    ) -> DomainResult<u64> {
        if days < 0 {
            return Err(DomainError::Validation(
                "days must be non-negative".into(),
            ));
        }

        let cutoff =
            Utc::now() - Duration::days(days as i64);

        let result = sqlx::query(
            r#"
            DELETE FROM processing_results
            WHERE created_at < $1
            "#,
        )
        .bind(cutoff)
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::Repository(e.to_string()))?;

        Ok(result.rows_affected())
    }
}
```

В этой реализации мы используем `query_as` и `FromRow`, а не `query!`.

Причина — архитектурная и практическая.

Макросы `sqlx::query!` и `sqlx::query_as!` выполняют дополнительную проверку SQL во время компиляции и поэтому требуют доступной схеме базы данных либо настроенного offline mode. Для учебного workspace, который должен собираться без запущенного PostgreSQL, вариант с `FromRow` проще.

В production-проекте compile-time проверка SQL через SQLx также является отличным вариантом.

---

## 81.17. HTTP слой

HTTP должен обращаться к Application, а не непосредственно к PostgreSQL.

Создадим состояние приложения.

### `crates/infrastructure/src/http/state.rs`

```rust
use std::sync::Arc;

use file_processor_application::ProcessFileUseCase;
use file_processor_domain::ProcessingRepository;

#[derive(Clone)]
pub struct AppState {
    pub repository: Arc<dyn ProcessingRepository>,
    pub process_file: Arc<ProcessFileUseCase>,
}
```

Теперь handler может получать `AppState` через Axum `State`.

---

## 81.18. HTTP handler

### `crates/infrastructure/src/http/handlers.rs`

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};

use serde_json::json;
use uuid::Uuid;

use crate::http::state::AppState;

pub async fn get_result(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> impl IntoResponse {
    match state.repository.find_by_id(id).await {
        Ok(Some(result)) => (
            StatusCode::OK,
            Json(result),
        )
            .into_response(),

        Ok(None) => (
            StatusCode::NOT_FOUND,
            Json(json!({
                "error": "Result not found"
            })),
        )
            .into_response(),

        Err(error) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({
                "error": error.to_string()
            })),
        )
            .into_response(),
    }
}

pub async fn delete_result(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> impl IntoResponse {
    match state.repository.delete(id).await {
        Ok(true) => StatusCode::NO_CONTENT.into_response(),

        Ok(false) => (
            StatusCode::NOT_FOUND,
            Json(json!({
                "error": "Result not found"
            })),
        )
            .into_response(),

        Err(error) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({
                "error": error.to_string()
            })),
        )
            .into_response(),
    }
}
```

Обратите внимание на архитектурную границу:

```text
HTTP handler
     │
     ▼
AppState
     │
     ▼
ProcessingRepository trait
```

Handler не содержит:

```rust
PgPool
sqlx::query(...)
```

Это техническая деталь Infrastructure.

---

## 81.19. HTTP routes

### `crates/infrastructure/src/http/routes.rs`

```rust
use axum::{
    routing::{delete, get},
    Router,
};

use crate::http::{
    handlers::{delete_result, get_result},
    state::AppState,
};

pub fn create_router(
    state: AppState,
) -> Router {
    Router::new()
        .route(
            "/api/results/:id",
            get(get_result).delete(delete_result),
        )
        .with_state(state)
}
```

В современных версиях Axum состояние передаётся через `Router::with_state`, а handler получает его через `State`.

---

## 81.20. Composition Root

Теперь возникает важный вопрос:

> Кто создаёт конкретный `PostgresProcessingRepository`?

Не Domain.

Не Application.

Это делает **composition root**.

Composition root — место, где конкретные реализации связываются с абстракциями.

### `crates/infrastructure/src/composition.rs`

```rust
use std::sync::Arc;

use anyhow::Result;

use file_processor_application::ProcessFileUseCase;
use file_processor_domain::{
    Config,
    ProcessingRepository,
    Processor,
};

use crate::{
    db::{
        connection::{
            create_pool,
            run_migrations,
        },
        processing_repository::PostgresProcessingRepository,
    },
    http::state::AppState,
};

pub struct AppContainer {
    pub state: AppState,
}

impl AppContainer {
    pub async fn new(
        database_url: &str,
    ) -> Result<Self> {
        let pool =
            create_pool(database_url).await?;

        run_migrations(&pool).await?;

        let repository: Arc<
            dyn ProcessingRepository
        > = Arc::new(
            PostgresProcessingRepository::new(pool)
        );

        let processor =
            Processor::new(Config::default());

        let process_file =
            Arc::new(
                ProcessFileUseCase::new(
                    repository.clone(),
                    processor,
                )
            );

        let state = AppState {
            repository,
            process_file,
        };

        Ok(Self { state })
    }
}
```

Теперь вся сборка приложения находится в одном месте:

```text
DATABASE_URL
      │
      ▼
   PgPool
      │
      ▼
PostgresProcessingRepository
      │
      │ implements
      ▼
ProcessingRepository
      │
      ▼
ProcessFileUseCase
      │
      ▼
   AppState
      │
      ▼
   Axum Router
```

Это значительно лучше, чем создавать repository внутри каждого handler.

---

## 81.21. `lib.rs` Infrastructure

### `crates/infrastructure/src/db/mod.rs`

```rust
pub mod connection;
pub mod processing_repository;
```

### `crates/infrastructure/src/http/mod.rs`

```rust
pub mod handlers;
pub mod routes;
pub mod state;
```

### `crates/infrastructure/src/lib.rs`

```rust
pub mod composition;
pub mod db;
pub mod http;

pub use composition::AppContainer;
```

---

## 81.22. Запуск сервера

Поскольку Infrastructure является конечным слоем, здесь же может находиться binary.

Добавим в `crates/infrastructure/Cargo.toml`:

```toml
[[bin]]
name = "file_processor_server"
path = "src/main.rs"
```

### `crates/infrastructure/src/main.rs`

```rust
use std::net::SocketAddr;

use anyhow::Result;
use file_processor_infrastructure::{
    AppContainer,
    http::routes::create_router,
};

#[tokio::main]
async fn main() -> Result<()> {
    let database_url = std::env::var(
        "DATABASE_URL"
    )?;

    let container =
        AppContainer::new(&database_url).await?;

    let app =
        create_router(container.state);

    let address =
        SocketAddr::from(([127, 0, 0, 1], 8080));

    let listener =
        tokio::net::TcpListener::bind(address)
            .await?;

    println!(
        "Server listening on http://{}",
        address
    );

    axum::serve(listener, app).await?;

    Ok(())
}
```

Запуск:

```bash
DATABASE_URL=postgres://postgres:password@localhost/file_processor \
cargo run -p file_processor_infrastructure --bin file_processor_server
```

На Windows PowerShell:

```powershell
$env:DATABASE_URL="postgres://postgres:password@localhost/file_processor"
cargo run -p file_processor_infrastructure --bin file_processor_server
```

---

## 81.23. Почему `Arc` нужен здесь?

Мы передаём repository одновременно в несколько компонентов:

```text
             ┌──────────────────────┐
             │ ProcessingRepository │
             └──────────┬───────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       ProcessFileUseCase      HTTP State
```

Поэтому нам нужен совместно владеемый объект:

```rust
Arc<dyn ProcessingRepository>
```

Это не просто «трюк для компилятора».

`Arc` означает:

> несколько владельцев одного объекта могут безопасно существовать одновременно.

При этом `PgPool` сам рассчитан на совместное использование, поэтому `PostgresProcessingRepository` удобно хранить за `Arc`.

---

## 81.24. Что произошло с `Box<dyn Trait>`?

В исходном варианте можно было написать:

```rust
Box<dyn ProcessingRepository>
```

Это означает:

> один владелец объекта.

Но затем код пытался передать:

```rust
Arc<PostgresProcessingRepository>
```

что является другим типом.

Нельзя автоматически передать:

```text
Arc<T>
```

туда, где ожидается:

```text
Box<dyn Trait>
```

Поэтому для нашего сценария используем единый тип:

```rust
Arc<dyn ProcessingRepository>
```

и передаём его через все слои.

---

## 81.25. Тестирование Domain и Application

Одна из главных выгод архитектуры проявляется при тестировании.

Нам не нужен PostgreSQL, чтобы проверить use case.

Можно создать простой in-memory repository.

### `crates/application/src/use_cases/process_file.rs`

В тесте можно реализовать:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use async_trait::async_trait;
    use std::sync::Mutex;

    use file_processor_domain::{
        DomainResult,
        NewProcessingResult,
        ProcessingRepository,
        ProcessingResult,
        ResultFilters,
        UsageStats,
    };
    use uuid::Uuid;

    struct InMemoryRepository {
        results: Mutex<Vec<ProcessingResult>>,
    }

    impl InMemoryRepository {
        fn new() -> Self {
            Self {
                results: Mutex::new(Vec::new()),
            }
        }
    }

    #[async_trait]
    impl ProcessingRepository for InMemoryRepository {
        async fn save(
            &self,
            input: NewProcessingResult,
        ) -> DomainResult<ProcessingResult> {
            let result = ProcessingResult {
                id: Uuid::new_v4(),
                filename: input.filename,
                input_content: input.input_content,
                output_content: input.output_content,
                lines_processed: input.lines_processed,
                bytes_processed: input.bytes_processed,
                processing_time_ms: input.processing_time_ms,
                created_at: chrono::Utc::now(),
            };

            self.results
                .lock()
                .unwrap()
                .push(result.clone());

            Ok(result)
        }

        async fn find_by_id(
            &self,
            id: Uuid,
        ) -> DomainResult<Option<ProcessingResult>> {
            Ok(self
                .results
                .lock()
                .unwrap()
                .iter()
                .find(|r| r.id == id)
                .cloned())
        }

        async fn find_all(
            &self,
            _filters: ResultFilters,
        ) -> DomainResult<Vec<ProcessingResult>> {
            Ok(self.results.lock().unwrap().clone())
        }

        async fn delete(
            &self,
            id: Uuid,
        ) -> DomainResult<bool> {
            let mut results =
                self.results.lock().unwrap();

            let old_len = results.len();

            results.retain(|r| r.id != id);

            Ok(results.len() != old_len)
        }

        async fn get_stats(
            &self,
        ) -> DomainResult<UsageStats> {
            Ok(UsageStats {
                total_requests: 0,
                total_lines_processed: 0,
                total_bytes_processed: 0,
                avg_processing_time_ms: 0.0,
            })
        }

        async fn delete_older_than(
            &self,
            _days: i32,
        ) -> DomainResult<u64> {
            Ok(0)
        }
    }
}
```

Сам принцип можно проверить в Rust Playground.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3AMutex%3B%0A%0Atrait+Repository%3A+Send+%2B+Sync+%7B%0A++++fn+save%28%26self%2C+value%3A+String%29%3B%0A%7D%0A%0Astruct+MemoryRepository+%7B%0A++++values%3A+Mutex%3CVec%3CString%3E%3E%2C%0A%7D%0A%0Aimpl+Repository+for+MemoryRepository+%7B%0A++++fn+save%28%26self%2C+value%3A+String%29+%7B%0A++++++++self.values.lock%28%29.unwrap%28%29.push%28value%29%3B%0A++++%7D%0A%7D%0A%0Afn+use_repository%28repo%3A+%26dyn+Repository%29+%7B%0A++++repo.save%28%22hello%22.to_owned%28%29%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+repo+%3D+MemoryRepository+%7B%0A++++++++values%3A+Mutex%3A%3Anew%28Vec%3A%3Anew%28%29%29%2C%0A++++%7D%3B%0A++++use_repository%28%26repo%29%3B%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+repo.values.lock%28%29.unwrap%28%29%29%3B%0A%7D)

---

## 81.26. Интеграционные тесты PostgreSQL

Тесты, которым действительно нужна PostgreSQL, должны находиться в Infrastructure.

Например:

```text
crates/infrastructure/
└── tests/
    └── repository_test.rs
```

А не:

```text
tests/
└── integration_test.rs
```

в корне виртуального workspace.

Почему?

Потому что корневой виртуальный workspace не является package. Интеграционные тесты Cargo относятся к конкретному package и используют его public API.

Для базы данных можно использовать:

```rust
#[sqlx::test]
async fn save_and_find(pool: sqlx::PgPool) {
    // ...
}
```

или отдельный тестовый PostgreSQL через контейнер.

Главное архитектурное правило:

```text
Domain/Application tests
        │
        └── без PostgreSQL

Infrastructure tests
        │
        └── PostgreSQL допустима
```

---

## 81.27. Проверка всего workspace

Теперь workspace позволяет выполнять команды сразу для всех пакетов.

Проверка:

```bash
cargo check --workspace
```

Сборка:

```bash
cargo build --workspace
```

Тесты:

```bash
cargo test --workspace
```

Форматирование:

```bash
cargo fmt --all
```

Clippy:

```bash
cargo clippy --workspace --all-targets --all-features
```

`cargo test --workspace` запускает тесты всех workspace members.

Это одно из главных преимуществ workspace:

```text
                 cargo test --workspace
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
       domain        application    infrastructure
```

---

## 81.28. Проверяем границы архитектуры

Попробуем нарушить архитектуру.

В `domain/Cargo.toml` добавим:

```toml
sqlx = { workspace = true }
```

а затем:

```rust
use sqlx::PgPool;
```

Технически Rust это позволит.

Но архитектурно это ошибка.

Теперь:

```text
Domain → PostgreSQL
```

и Domain перестал быть независимым.

Гораздо опаснее другое нарушение:

```text
Application → Infrastructure
```

Например:

```rust
use file_processor_infrastructure::
    PostgresProcessingRepository;
```

Теперь Application знает конкретную реализацию.

В результате:

```text
Application
     │
     ▼
Postgres
```

и заменить PostgreSQL становится значительно сложнее.

---

## 81.29. Почему workspace — это больше, чем папки?

Можно иметь такую структуру:

```text
domain/
application/
infrastructure/
```

и при этом иметь ужасную архитектуру.

Например:

```text
domain → infrastructure
application → infrastructure
domain → database
```

Физические каталоги ничего не гарантируют.

Workspace даёт нам **механизм компилятора**, который помогает контролировать эти границы.

Если `application` не содержит зависимости:

```toml
file_processor_infrastructure = ...
```

то следующий код просто невозможно написать:

```rust
use file_processor_infrastructure::PostgresProcessingRepository;
```

Компилятор сообщит, что crate не является зависимостью.

Именно поэтому архитектурные границы лучше закреплять не только соглашением команды, но и структурой зависимостей Cargo.

---

## 81.30. Эксперименты с компилятором

### Эксперимент 1. Циклическая зависимость

Попробуйте сделать:

```text
domain → application
application → domain
```

Cargo обнаружит циклическую зависимость и откажется строить граф.

Это хороший пример того, что dependency graph — реальная структура программы, а не документация.

### Эксперимент 2. Application зависит от Infrastructure

Добавьте в:

```text
crates/application/Cargo.toml
```

```toml
file_processor_infrastructure = {
    path = "../infrastructure"
}
```

А Infrastructure уже зависит от Application.

Получится:

```text
application
     │
     ▼
infrastructure
     │
     ▼
application
```

Cargo обнаружит цикл.

### Эксперимент 3. Domain зависит от SQLx

Добавьте:

```toml
sqlx = { workspace = true }
```

Теперь проект может скомпилироваться, но архитектурная граница разрушена.

Это важный урок:

> Компилятор может проверить допустимость зависимости, но не всегда может проверить её архитектурную желательность.

---

## 81.31. Rust Playground: простой пример dependency inversion

Полный workspace с PostgreSQL невозможно запустить в Rust Playground, потому что Playground не предоставляет нашу структуру Cargo workspace и работающую PostgreSQL.

Но сам принцип Dependency Inversion можно показать автономно:

```rust
trait Repository {
    fn save(&self, value: &str);
}

struct MemoryRepository;

impl Repository for MemoryRepository {
    fn save(&self, value: &str) {
        println!("Saving: {value}");
    }
}

struct Application<R: Repository> {
    repository: R,
}

impl<R: Repository> Application<R> {
    fn execute(&self) {
        self.repository.save("hello");
    }
}

fn main() {
    let app = Application {
        repository: MemoryRepository,
    };

    app.execute();
}
```

Здесь Application зависит от абстракции:

```rust
Repository
```

а не от:

```rust
MemoryRepository
```

В реальном проекте вместо generic-параметра мы использовали:

```rust
Arc<dyn ProcessingRepository>
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Repository+%7B%0A++++fn+save%28%26self%2C+value%3A+%26str%29%3B%0A%7D%0A%0Astruct+MemoryRepository%3B%0A%0Aimpl+Repository+for+MemoryRepository+%7B%0A++++fn+save%28%26self%2C+value%3A+%26str%29+%7B%0A++++++++println%21%28%22Saving%3A+%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D%0A%0Astruct+Application%3CR%3A+Repository%3E+%7B%0A++++repository%3A+R%2C%0A%7D%0A%0Aimpl%3CR%3A+Repository%3E+Application%3CR%3E+%7B%0A++++fn+execute%28%26self%29+%7B%0A++++++++self.repository.save%28%22hello%22%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+app+%3D+Application+%7B%0A++++++++repository%3A+MemoryRepository%2C%0A++++%7D%3B%0A%0A++++app.execute%28%29%3B%0A%7D)

---

## 81.32. Практика

### Задание 1

Создайте virtual workspace:

```text
file_processor/
├── Cargo.toml
└── crates/
```

и добавьте:

```text
domain
application
infrastructure
```

### Задание 2

Перенесите модели `ProcessingResult` и `NewProcessingResult` в Domain.

### Задание 3

Перенесите `ProcessingRepository` в Domain.

Убедитесь, что Domain не зависит от SQLx.

### Задание 4

Создайте `ProcessFileUseCase` в Application.

Use case не должен импортировать:

```rust
sqlx
axum
PostgresProcessingRepository
```

### Задание 5

Реализуйте `PostgresProcessingRepository` в Infrastructure.

### Задание 6

Создайте composition root, который:

1. создаёт `PgPool`;
2. применяет миграции;
3. создаёт `PostgresProcessingRepository`;
4. превращает его в `Arc<dyn ProcessingRepository>`;
5. создаёт use cases;
6. создаёт `AppState`;
7. запускает HTTP server.

### Задание 7

Создайте `InMemoryProcessingRepository`.

Используйте его для тестирования `ProcessFileUseCase` без PostgreSQL.

### Задание 8

Попробуйте намеренно создать:

```text
domain → infrastructure
```

и:

```text
application → infrastructure
```

Посмотрите, что произойдёт с dependency graph.

### Задание 9

Запустите:

```bash
cargo check --workspace
cargo test --workspace
cargo clippy --workspace --all-targets
```

и добейтесь, чтобы весь workspace проходил проверку без ошибок.

---

## 81.33. Что мы получили

После этой главы проект имеет следующую архитектуру:

```text
┌──────────────────────────────────────────────────────────────┐
│                  Infrastructure                              │
│                                                              │
│  PostgreSQL     SQLx       Axum       CLI       Composition  │
│      │            │          │          │           │        │
└──────┼────────────┼──────────┼──────────┼───────────┼────────┘
       │            │          │          │           │
       └────────────┴──────────┴──────────┴───────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    Application                               │
│                                                              │
│             ProcessFileUseCase                               │
│             ListResultsUseCase                               │
│             GetStatsUseCase                                  │
│                                                              │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       Domain                                 │
│                                                              │
│ Models    Business Rules    Repository Traits    Errors      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Каждый слой имеет чёткую ответственность.

### Domain

Знает:

```text
что такое ProcessingResult
что такое обработка
что такое Repository
```

Не знает:

```text
PostgreSQL
HTTP
Axum
```

### Application

Знает:

```text
какие операции выполняет приложение
```

Не знает:

```text
как именно хранить данные
как принимать HTTP-запросы
```

### Infrastructure

Знает:

```text
как подключиться к PostgreSQL
как запустить HTTP
как применить миграции
```

и связывает всё вместе.

---

## Главное из этой главы

После этой главы мы:

- **создали Cargo workspace**;
- **разделили проект на независимые crates**;
- **выделили Domain Layer**;
- **выделили Application Layer**;
- **выделили Infrastructure Layer**;
- **определили направления зависимостей**;
- **вынесли repository в виде trait в Domain**;
- **реализовали PostgreSQL repository в Infrastructure**;
- **добавили composition root**;
- **использовали `Arc<dyn Trait>` для dependency injection**;
- **отделили HTTP от бизнес-логики**;
- **организовали тестирование без обязательной PostgreSQL**;
- **организовали интеграционные тесты на уровне Infrastructure**;
- **научились проверять весь workspace одной командой**.

**Самая важная идея:**

> Workspace — это не просто способ положить несколько Rust-проектов в один репозиторий. Это инструмент управления границами системы.
>
> Domain описывает предметную область и не знает о технических деталях. Application реализует сценарии использования и работает через абстракции. Infrastructure предоставляет конкретные реализации — PostgreSQL, HTTP, CLI и другие внешние системы.
>
> Самое важное происходит в composition root: именно там абстракции связываются с конкретными реализациями.
>
> Благодаря этому бизнес-логика остаётся независимой от PostgreSQL, HTTP-фреймворка и других технических деталей. А Cargo workspace превращает эту архитектуру из договорённости в реальную структуру зависимостей, которую способен контролировать компилятор.
