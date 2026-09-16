# Глава 83. Создаём procedural macro

В предыдущих главах мы создавали трейты, сервисы и репозитории. Теперь сделаем следующий шаг: научимся **генерировать реализацию трейта автоматически**.

Для этого используются **процедурные макросы** (`procedural macros`).

Процедурный макрос получает Rust-код в виде токенов, анализирует его и генерирует новый Rust-код. Компилятор затем компилирует уже сгенерированный код вместе с остальной программой. В Rust существуют три разновидности procedural macros:

- function-like macros;
- derive macros;
- attribute macros.

В этой главе мы сосредоточимся на **derive macro** — макросе, который вызывается через `#[derive(...)]`.

Мы создадим:

```rust
#[derive(ProcessingService)]
struct DefaultFileProcessingService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

и заставим `ProcessingService` автоматически генерировать реализацию:

```rust
impl FileProcessingService for DefaultFileProcessingService {
    // ...
}
```

Таким образом, вместо большого количества повторяющегося кода мы описываем **структуру объекта и роли его полей**, а procedural macro генерирует необходимый boilerplate.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 83.1. Что такое procedural macro?

Процедурный макрос можно представить как функцию:

```text
Rust source code
       │
       ▼
┌──────────────────┐
│ Procedural macro │
│                  │
│ parse → analyze  │
│       → generate │
└────────┬─────────┘
         │
         ▼
Generated Rust code
         │
         ▼
      rustc
         │
         ▼
    executable
```

То есть макрос работает **во время компиляции**.

Например, мы пишем:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Макрос получает описание структуры и генерирует примерно такой код:

```rust
impl FileProcessingService for MyService {
    async fn process_text(...) -> Result<..., ...> {
        // generated code
    }

    async fn get_result(...) -> Result<..., ...> {
        // generated code
    }

    // ...
}
```

Важно понимать: procedural macro **не изменяет Rust-код во время выполнения программы**. Он работает при компиляции и добавляет в программу новый исходный код в виде токенов.

### Зачем это нужно?

Procedural macros особенно полезны там, где один и тот же шаблон кода повторяется для большого количества типов.

Например:

```rust
#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}
```

`serde` может сгенерировать реализацию `Serialize`.

Или:

```rust
#[derive(Debug)]
struct User {
    id: u64,
    name: String,
}
```

Компилятор генерирует реализацию `Debug`.

Наш пример будет похожим:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Мы делаем свой derive macro.

---

## 83.2. Что именно мы будем автоматизировать?

Наш сервис имеет две зависимости:

```text
DefaultFileProcessingService
│
├── repository
│       │
│       └── хранение результатов
│
└── processor
        │
        └── обработка текста
```

Без procedural macro реализация выглядит примерно так:

```rust
impl FileProcessingService for DefaultFileProcessingService {
    type Error = DomainError;

    async fn get_result(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>, Self::Error> {
        self.repo.find_by_id(id).await
    }

    async fn list_results(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>, Self::Error> {
        self.repo.find_all(filters).await
    }

    async fn get_stats(
        &self,
    ) -> Result<UsageStats, Self::Error> {
        self.repo.get_stats().await
    }

    async fn delete_result(
        &self,
        id: Uuid,
    ) -> Result<bool, Self::Error> {
        self.repo.delete(id).await
    }

    async fn cleanup_old(
        &self,
        days: i32,
    ) -> Result<u64, Self::Error> {
        self.repo.delete_older_than(days).await
    }

    // ...
}
```

Здесь большая часть кода — механическое делегирование:

```rust
self.repo.find_by_id(id).await
```

```rust
self.repo.find_all(filters).await
```

```rust
self.repo.delete(id).await
```

```rust
self.repo.delete_older_than(days).await
```

Именно такой boilerplate хорошо подходит для автоматической генерации.

### Наша цель

Мы хотим написать:

```rust
#[derive(ProcessingService)]
struct DefaultFileProcessingService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

а макрос должен самостоятельно найти:

```rust
#[repository]
repo: ...
```

и:

```rust
#[processor]
processor: ...
```

и использовать соответствующие поля в сгенерированной реализации.

---

## 83.3. Ограничение procedural macros: отдельный crate

Это один из самых важных моментов.

Procedural macro нельзя определить и использовать в том же crate.

Поэтому архитектура должна выглядеть примерно так:

```text
file-processor/
│
├── Cargo.toml
│
└── crates/
    │
    ├── file_processor_core/
    │   └── src/
    │       └── lib.rs
    │
    ├── file_processor_macros/
    │   ├── Cargo.toml
    │   └── src/
    │       ├── lib.rs
    │       ├── processing_service.rs
    │       ├── validation.rs
    │       └── attrs.rs
    │
    └── file_processor_infrastructure/
        └── src/
            └── services/
```

Связи:

```text
file_processor_core
        ▲
        │
        │ uses domain types
        │
file_processor_macros
        │
        │ generates implementation
        ▼
file_processor_infrastructure
```

Сам procedural macro crate имеет:

```toml
[lib]
proc-macro = true
```

Это специальный тип Cargo library target.

---

## 83.4. Создаём macro crate

Создадим:

```text
crates/file_processor_macros/
```

### `Cargo.toml`

```toml
[package]
name = "file_processor_macros"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

Здесь используются три основных инструмента.

### `proc_macro`

Это API самого Rust compiler для procedural macros.

Он предоставляет `TokenStream` — поток токенов, с которым работает procedural macro.

### `syn`

`syn` преобразует поток токенов в удобное представление Rust-кода.

Например, вместо анализа отдельных токенов мы можем получить:

```rust
DeriveInput
```

и затем обратиться к:

```rust
input.ident
input.data
input.attrs
```

### `quote`

`quote` позволяет удобно генерировать Rust-код:

```rust
quote! {
    impl MyTrait for MyType {
        // ...
    }
}
```

---

### Важное замечание о Rust Playground

Обычный Rust Playground отлично подходит для демонстрации Rust-кода, но полноценный procedural macro требует отдельного `proc-macro` crate.

Поэтому основной macro workspace этой главы нужно запускать через Cargo.

Для небольших примеров самого trait API можно использовать:

**Открыть пример в Rust Playground:**
[Rust Playground — Edition 2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## 83.5. Первый procedural macro

Создадим:

```text
crates/file_processor_macros/src/lib.rs
```

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

mod attrs;
mod processing_service;
mod validation;

#[proc_macro_derive(
    ProcessingService,
    attributes(repository, processor)
)]
pub fn processing_service_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    processing_service::expand(input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

Разберём его.

### `#[proc_macro_derive(...)]`

```rust
#[proc_macro_derive(
    ProcessingService,
    attributes(repository, processor)
)]
```

означает:

> Создать derive macro с именем `ProcessingService`.

После этого другой crate может написать:

```rust
#[derive(ProcessingService)]
struct MyService {
    // ...
}
```

Часть:

```rust
attributes(repository, processor)
```

объявляет helper attributes этого derive macro.

Поэтому после:

```rust
#[derive(ProcessingService)]
```

мы можем использовать:

```rust
#[repository]
```

и:

```rust
#[processor]
```

на полях структуры. Такие helper attributes должны быть объявлены самим derive macro.

---

### Получаем AST

```rust
let input = parse_macro_input!(input as DeriveInput);
```

На входе macro получает `TokenStream`.

`syn` преобразует его в:

```rust
DeriveInput
```

Для структуры:

```rust
struct MyService {
    repo: Repository,
}
```

можно получить:

```text
DeriveInput
│
├── ident = MyService
│
└── data
    └── struct
        └── fields
```

---

### Генерируем ошибку компиляции

Вместо:

```rust
panic!("something went wrong")
```

лучше вернуть:

```rust
syn::Error
```

и преобразовать его:

```rust
syn::Error::into_compile_error
```

Тогда пользователь получает обычную ошибку Rust-компилятора.

Это особенно важно для procedural macros: ошибка должна указывать пользователю, что именно он сделал неправильно.

---

## 83.6. Парсим атрибуты полей

Создадим:

```text
crates/file_processor_macros/src/attrs.rs
```

```rust
use syn::{Field, Result};

#[derive(Debug, Default)]
pub struct FieldAttrs {
    pub repository: bool,
    pub processor: bool,
    pub skip: bool,
}

impl FieldAttrs {
    pub fn parse(field: &Field) -> Result<Self> {
        let mut result = Self::default();

        for attr in &field.attrs {
            if attr.path().is_ident("repository") {
                if result.repository {
                    return Err(syn::Error::new_spanned(
                        attr,
                        "duplicate #[repository] attribute",
                    ));
                }

                result.repository = true;
            } else if attr.path().is_ident("processor") {
                if result.processor {
                    return Err(syn::Error::new_spanned(
                        attr,
                        "duplicate #[processor] attribute",
                    ));
                }

                result.processor = true;
            } else if attr.path().is_ident("skip") {
                result.skip = true;
            }
        }

        Ok(result)
    }
}
```

Теперь вместо того, чтобы в нескольких местах самостоятельно анализировать атрибуты, мы имеем единый parser.

Например:

```rust
#[repository]
repo: Box<dyn ProcessingRepository>,
```

даёт:

```rust
FieldAttrs {
    repository: true,
    processor: false,
    skip: false,
}
```

А:

```rust
#[processor]
processor: Processor,
```

даёт:

```rust
FieldAttrs {
    repository: false,
    processor: true,
    skip: false,
}
```

---

## 83.7. Реализуем генератор `ProcessingService`

Теперь создадим:

```text
crates/file_processor_macros/src/processing_service.rs
```

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::{
    Data, DeriveInput, Error, Fields, Ident, Result,
};

use crate::attrs::FieldAttrs;

fn named_field_ident(
    field: &syn::Field,
) -> Result<&Ident> {
    field.ident.as_ref().ok_or_else(|| {
        Error::new_spanned(
            field,
            "ProcessingService requires named fields",
        )
    })
}

fn find_field(
    fields: &Fields,
    kind: &str,
) -> Result<Ident> {
    let mut found = None;

    for field in fields {
        let attrs = FieldAttrs::parse(field)?;

        let matched = match kind {
            "repository" => attrs.repository,
            "processor" => attrs.processor,
            _ => false,
        };

        if matched {
            if found.is_some() {
                return Err(Error::new_spanned(
                    field,
                    format!(
                        "only one #[{kind}] field is allowed"
                    ),
                ));
            }

            found = Some(named_field_ident(field)?.clone());
        }
    }

    found.ok_or_else(|| {
        Error::new_spanned(
            fields,
            format!(
                "no field marked with #[{kind}] found"
            ),
        )
    })
}

pub fn expand(input: DeriveInput) -> Result<TokenStream> {
    let name = &input.ident;

    let fields = match &input.data {
        Data::Struct(data) => &data.fields,

        _ => {
            return Err(Error::new_spanned(
                &input,
                "ProcessingService can only be derived for structs",
            ));
        }
    };

    let repository = find_field(fields, "repository")?;
    let processor = find_field(fields, "processor")?;

    let expanded = quote! {
        #[automatically_derived]
        impl file_processor_core::FileProcessingService
            for #name
        {
            type Error =
                file_processor_core::DomainError;

            async fn process_text(
                &self,
                content: &str,
                filename: Option<String>,
            ) -> Result<
                file_processor_core::ProcessingResult,
                Self::Error,
            > {
                use std::time::Instant;

                let start = Instant::now();

                let lines: Vec<String> = content
                    .lines()
                    .map(str::to_owned)
                    .collect();

                let processed =
                    self.#processor.apply_transforms(lines);

                let elapsed = start.elapsed();

                let new_result =
                    file_processor_core::NewProcessingResult {
                        filename,
                        input_content: content.to_owned(),
                        output_content: processed.join("\n"),
                        lines_processed: processed.len() as i32,
                        bytes_processed: processed
                            .iter()
                            .map(|line| line.len() as i64)
                            .sum(),
                        processing_time_ms:
                            elapsed.as_millis() as i64,
                    };

                self.#repository.save(new_result).await
            }

            async fn get_result(
                &self,
                id: uuid::Uuid,
            ) -> Result<
                Option<file_processor_core::ProcessingResult>,
                Self::Error,
            > {
                self.#repository.find_by_id(id).await
            }

            async fn list_results(
                &self,
                filters: file_processor_core::ResultFilters,
            ) -> Result<
                Vec<file_processor_core::ProcessingResult>,
                Self::Error,
            > {
                self.#repository.find_all(filters).await
            }

            async fn get_stats(
                &self,
            ) -> Result<
                file_processor_core::UsageStats,
                Self::Error,
            > {
                self.#repository.get_stats().await
            }

            async fn delete_result(
                &self,
                id: uuid::Uuid,
            ) -> Result<bool, Self::Error> {
                self.#repository.delete(id).await
            }

            async fn cleanup_old(
                &self,
                days: i32,
            ) -> Result<u64, Self::Error> {
                self.#repository
                    .delete_older_than(days)
                    .await
            }
        }
    };

    Ok(expanded)
}
```

Это уже настоящий procedural macro.

Он выполняет четыре основных действия:

```text
1. Получает структуру
        ↓
2. Проверяет структуру
        ↓
3. Находит #[repository] и #[processor]
        ↓
4. Генерирует impl FileProcessingService
```

---

## 83.8. Почему generated code должен быть простым

Очень важно не превращать procedural macro в «второй Rust-компилятор».

Хороший macro должен делать примерно следующее:

```text
Input
  │
  ├── parse
  ├── validate
  └── generate
        │
        ▼
      Rust
```

Плохая архитектура выглядит так:

```text
Input
  │
  ▼
macro
 ├── бизнес-логика
 ├── database logic
 ├── HTTP logic
 ├── validation
 ├── configuration
 ├── dependency injection
 └── generation
```

Procedural macro должен автоматизировать **структурный boilerplate**, а не становиться местом размещения основной бизнес-логики.

В нашем случае хорошей границей является:

```text
Macro:
    "Как делегировать метод этому полю?"

Application:
    "Что означает обработка файла?"
```

---

## 83.9. Используем macro в приложении

Теперь в infrastructure crate:

```rust
use file_processor_core::{
    ProcessingRepository,
    Processor,
};

use file_processor_macros::ProcessingService;

#[derive(ProcessingService)]
pub struct DefaultFileProcessingService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

И всё.

Мы больше не пишем:

```rust
impl FileProcessingService
    for DefaultFileProcessingService
{
    // сотни строк boilerplate
}
```

Macro создаёт этот `impl` автоматически.

---

### Конструктор

Обычный код остаётся обычным Rust-кодом:

```rust
impl DefaultFileProcessingService {
    pub fn new(
        repo: Box<dyn ProcessingRepository>,
        processor: Processor,
    ) -> Self {
        Self {
            repo,
            processor,
        }
    }
}
```

Получаем:

```text
#[derive(ProcessingService)]
        │
        ▼
┌───────────────────────────────┐
│ generated impl                │
│                               │
│ process_text()                │
│ get_result()                  │
│ list_results()                │
│ get_stats()                   │
│ delete_result()               │
│ cleanup_old()                 │
└───────────────────────────────┘

+ обычный impl
        │
        └── new()
```

Это хороший баланс: macro генерирует повторяющийся код, а конструктор остаётся явно написанным.

---

## 83.10. Важная проблема: `async fn` и `dyn Trait`

В современных версиях Rust `async fn` в traits поддерживается самим языком. Поэтому сам trait можно написать без `async-trait`:

```rust
pub trait FileProcessingService {
    async fn process_text(
        &self,
        content: &str,
    ) -> Result<ProcessingResult, DomainError>;
}
```

Это отличается от старого Rust, где для `async fn` в trait обычно использовали `async-trait`.

Однако есть важное ограничение.

Такой trait нельзя просто превратить в:

```rust
Box<dyn FileProcessingService>
```

если он содержит `async fn`.

`async-trait` по-прежнему полезен именно в ситуации, когда нам нужен **trait object**. Документация `async-trait` прямо указывает, что native `async fn` в traits не сделал такие traits `dyn`-совместимыми.

Поэтому существуют два разных сценария.

### Статический dispatch

```rust
async fn run<S>(service: &S)
where
    S: FileProcessingService,
{
    // ...
}
```

Здесь `async-trait` не нужен.

### Dynamic dispatch

```rust
type Service =
    Box<dyn FileProcessingService>;
```

Если API требует такой подход, можно использовать:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait FileProcessingService {
    async fn process_text(
        &self,
        content: &str,
    ) -> Result<ProcessingResult, DomainError>;
}
```

и тот же `#[async_trait]` должен применяться к реализациям.

`async-trait` преобразует `async fn` в возвращающий boxed future метод, что позволяет использовать trait object.

**Открыть пример в Rust Playground:**
[Rust Playground — async trait example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## 83.11. Добавляем второй derive macro: `Validatable`

Procedural macros особенно интересны тогда, когда один и тот же шаблон применяется к большому количеству типов.

Например, можно создать:

```rust
#[derive(Validatable)]
struct User {
    name: String,
    email: String,

    #[skip]
    id: u64,
}
```

и автоматически получить проверку:

```rust
impl Validatable for User {
    fn validate(&self) -> Result<(), String> {
        if self.name.is_empty() {
            return Err("Field 'name' is empty".to_string());
        }

        if self.email.is_empty() {
            return Err("Field 'email' is empty".to_string());
        }

        Ok(())
    }
}
```

Сначала определим сам trait в core crate:

```rust
pub trait Validatable {
    fn validate(&self) -> Result<(), String>;
}
```

---

## 83.12. Реализация `Validatable`

В `lib.rs` добавим:

```rust
#[proc_macro_derive(
    Validatable,
    attributes(skip)
)]
pub fn validatable_derive(
    input: TokenStream,
) -> TokenStream {
    let input =
        parse_macro_input!(input as DeriveInput);

    validation::expand(input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

Теперь создадим:

```text
src/validation.rs
```

```rust
use proc_macro2::TokenStream;
use quote::quote;
use syn::{
    Data,
    DeriveInput,
    Error,
    Fields,
    Result,
};

pub fn expand(input: DeriveInput) -> Result<TokenStream> {
    let name = &input.ident;

    let fields = match &input.data {
        Data::Struct(data) => &data.fields,

        _ => {
            return Err(Error::new_spanned(
                &input,
                "Validatable can only be derived for structs",
            ));
        }
    };

    let fields = match fields {
        Fields::Named(fields) => &fields.named,

        _ => {
            return Err(Error::new_spanned(
                fields,
                "Validatable requires named fields",
            ));
        }
    };

    let validations = fields.iter().filter_map(|field| {
        let ident = field.ident.as_ref()?;

        let skipped = field.attrs.iter().any(|attr| {
            attr.path().is_ident("skip")
        });

        if skipped {
            return None;
        }

        Some(quote! {
            if self.#ident.is_empty() {
                return Err(
                    format!(
                        "Field '{}' is empty",
                        stringify!(#ident)
                    )
                );
            }
        })
    });

    Ok(quote! {
        impl file_processor_core::Validatable for #name {
            fn validate(&self) -> Result<(), String> {
                #(#validations)*

                Ok(())
            }
        }
    })
}
```

Здесь есть важная проблема.

Мы предполагаем, что каждое поле поддерживает:

```rust
.is_empty()
```

Но это справедливо не для всех Rust-типов.

Например:

```rust
String
```

имеет:

```rust
is_empty()
```

а:

```rust
u64
```

не имеет.

Поэтому универсальный `Validatable` для произвольных типов нельзя строить таким примитивным способом.

Это хороший пример того, **где procedural macro должен иметь более строгий контракт**.

---

## 83.13. Делаем `Validatable` действительно законченным

Вместо предположения, что любое поле имеет `is_empty()`, сделаем validation attribute явным.

Например:

```rust
#[derive(Validatable)]
struct User {
    #[validate(non_empty)]
    name: String,

    #[validate(non_empty)]
    email: String,

    id: u64,
}
```

Теперь macro знает, какие поля нужно проверять.

Для этого объявим helper attributes:

```rust
#[proc_macro_derive(
    Validatable,
    attributes(skip, validate)
)]
```

И структура становится декларативной:

```rust
#[derive(Validatable)]
struct User {
    #[validate(non_empty)]
    name: String,

    #[validate(non_empty)]
    email: String,

    #[skip]
    id: u64,
}
```

Это гораздо лучше:

```text
❌ Macro угадывает тип поля

String → is_empty()
u64   → ???

```

против:

```text
✅ Автор явно сообщает macro,
   какое правило применить

#[validate(non_empty)]
name: String
```

Это один из важнейших принципов проектирования procedural macros:

> **Не заставляйте macro угадывать семантику, которую можно выразить атрибутом.**

---

## 83.14. Ошибки procedural macro

Procedural macro должен проверять неправильное использование как можно раньше.

Например:

```rust
#[derive(ProcessingService)]
struct MyService {
    repo: Box<dyn ProcessingRepository>,
}
```

Здесь отсутствует:

```rust
#[repository]
```

Macro должен сообщить:

```text
error: no field marked with #[repository] found
```

---

### Несколько `#[repository]`

Нельзя разрешать:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo1: Box<dyn ProcessingRepository>,

    #[repository]
    repo2: Box<dyn ProcessingRepository>,
}
```

В этом случае macro сообщает:

```text
error: only one #[repository] field is allowed
```

---

### Неподдерживаемый тип

Также:

```rust
#[derive(ProcessingService)]
enum Service {
    A,
    B,
}
```

должен привести к понятной ошибке:

```text
error:
ProcessingService can only be derived for structs
```

Procedural macros получают на вход структуру, enum или union, поэтому macro сам должен решить, какие из этих форм он поддерживает.

---

## 83.15. Тестируем generated implementation

Обычный тест может проверить результат работы сгенерированного кода.

Например:

```rust
#[tokio::test]
async fn process_text_uses_generated_impl() {
    let service =
        DefaultFileProcessingService::new(
            Box::new(MockRepository::default()),
            Processor::new(Config::default()),
        );

    let result = service
        .process_text(
            "hello\nworld",
            Some("test.txt".to_string()),
        )
        .await
        .unwrap();

    assert_eq!(
        result.lines_processed,
        2
    );
}
```

Здесь важно понимать:

```text
test
 │
 ▼
DefaultFileProcessingService
 │
 ▼
generated impl
 │
 ▼
process_text()
 │
 ├── Processor
 │
 └── Repository
```

То есть мы тестируем не сам procedural macro как парсер, а **результат его работы**.

---

## 83.16. Тестируем сам procedural macro

Для procedural macros полезны два уровня тестирования.

### 1. Runtime/integration tests

Проверяют:

```text
macro
 ↓
generated code
 ↓
program behavior
```

### 2. Compile-fail tests

Проверяют:

```text
invalid input
      ↓
procedural macro
      ↓
expected compiler error
```

Например:

```rust
#[derive(ProcessingService)]
struct BrokenService {
    repo: Box<dyn ProcessingRepository>,
}
```

Мы ожидаем:

```text
no field marked with #[repository] found
```

Для production-quality procedural macro compile-fail tests особенно важны, потому что значительная часть качества macro — это **качество сообщений об ошибках**.

На практике для таких тестов часто используют отдельные compile-test инструменты, например `trybuild`.

---

## 83.17. Что происходит с `#[derive(...)]` внутри компилятора?

Рассмотрим:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Упрощённо процесс выглядит так:

```text
                 source code
                      │
                      ▼
              #[derive(...)]
                      │
                      ▼
          procedural macro invoked
                      │
                      ▼
               proc_macro
                      │
                      ▼
                  syn parse
                      │
                      ▼
             DeriveInput / AST
                      │
                      ▼
                 validation
                      │
                      ▼
                   quote!
                      │
                      ▼
             generated tokens
                      │
                      ▼
                    rustc
                      │
                      ▼
                  binary
```

То есть:

```rust
#[derive(ProcessingService)]
```

не является обычной функцией.

Это инструкция компилятору:

> «Во время компиляции передай эту структуру procedural macro с именем `ProcessingService`».

---

## 83.18. Что именно делает `quote!`?

Например:

```rust
let name = &input.ident;

let generated = quote! {
    impl MyTrait for #name {
        fn hello(&self) {
            println!("hello");
        }
    }
};
```

Если:

```rust
name == MyService
```

то результат будет концептуально таким:

```rust
impl MyTrait for MyService {
    fn hello(&self) {
        println!("hello");
    }
}
```

Особый синтаксис:

```rust
#name
```

означает:

> вставить значение Rust-переменной в генерируемый token stream.

А:

```rust
#(#items)*
```

означает повторить последовательность:

```rust
items
```

с генерацией каждого элемента.

Это особенно полезно для генерации нескольких методов.

---

## 83.19. Почему мы используем `syn::Error`

Рассмотрим:

```rust
return Err(Error::new_spanned(
    field,
    "only one #[repository] field is allowed",
));
```

`new_spanned` связывает ошибку с исходным фрагментом кода.

В результате ошибка компилятора может указывать непосредственно на проблемное поле.

Это намного лучше, чем:

```rust
panic!("bad input");
```

Потому что пользователь macro получает нормальное сообщение компилятора с указанием места ошибки.

Procedural macros могут сообщать ошибки через `compile_error!` либо через panic; использование `syn::Error` с последующим `into_compile_error()` позволяет генерировать обычную диагностическую ошибку Rust.

---

## 83.20. Интеграция с приложением

Теперь наше приложение может выглядеть следующим образом:

```rust
use file_processor_core::{
    Config,
    Processor,
    ProcessingRepository,
};

use file_processor_macros::ProcessingService;

#[derive(ProcessingService)]
pub struct DefaultFileProcessingService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}

impl DefaultFileProcessingService {
    pub fn new(
        repo: Box<dyn ProcessingRepository>,
        processor: Processor,
    ) -> Self {
        Self {
            repo,
            processor,
        }
    }
}
```

А composition root:

```rust
pub async fn create_service(
    database_url: &str,
) -> Result<
    DefaultFileProcessingService,
    Box<dyn std::error::Error>,
> {
    let pool =
        create_pool(database_url).await?;

    run_migrations(&pool).await?;

    let repository =
        PostgresProcessingRepository::new(pool);

    let processor =
        Processor::new(Config::default());

    Ok(
        DefaultFileProcessingService::new(
            Box::new(repository),
            processor,
        )
    )
}
```

Теперь procedural macro отвечает только за boilerplate:

```text
             Application
                  │
                  ▼
      DefaultFileProcessingService
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
  Repository             Processor
       │                     │
       ▼                     ▼
   Database              transforms
```

А composition root отвечает за создание конкретных реализаций.

Это хорошее разделение ответственности.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. Удаляем `#[repository]`

Было:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Сделайте:

```rust
#[derive(ProcessingService)]
struct MyService {
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Ожидаемая ошибка:

```text
no field marked with #[repository] found
```

**Открыть пример в Rust Playground:**
[Rust Playground — Edition 2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

В Playground сам procedural macro не будет доступен как локальный workspace, поэтому этот эксперимент выполняйте в Cargo-проекте главы.

---

### Эксперимент 2. Два `#[repository]`

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo1: Box<dyn ProcessingRepository>,

    #[repository]
    repo2: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

Macro должен завершиться ошибкой:

```text
only one #[repository] field is allowed
```

---

### Эксперимент 3. Используем macro для enum

```rust
#[derive(ProcessingService)]
enum MyService {
    A,
    B,
}
```

Ожидаем:

```text
ProcessingService can only be derived for structs
```

---

### Эксперимент 4. Убираем `#[processor]`

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,
}
```

Ожидаем:

```text
no field marked with #[processor] found
```

---

### Эксперимент 5. Посмотрим на generated code

Попробуйте мысленно раскрыть:

```rust
#[derive(ProcessingService)]
struct MyService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

В:

```rust
impl FileProcessingService for MyService {
    // ...
}
```

Главный навык этой главы — научиться видеть за:

```rust
#[derive(ProcessingService)]
```

реальный generated Rust code.

---

## Практика

### Задание 1. Минимальный derive macro

Создайте:

```rust
#[derive(Hello)]
struct User;
```

который генерирует:

```rust
impl User {
    fn hello(&self) {
        println!("Hello!");
    }
}
```

---

### Задание 2. Используйте имя структуры

Сделайте так, чтобы:

```rust
#[derive(Hello)]
struct User;
```

генерировал:

```rust
impl User {
    fn hello(&self) {
        println!("Hello, User!");
    }
}
```

Для этого используйте:

```rust
let name = &input.ident;
```

и:

```rust
stringify!(#name)
```

---

### Задание 3. Создайте `#[derive(Validatable)]`

Реализуйте:

```rust
#[derive(Validatable)]
struct User {
    #[validate(non_empty)]
    name: String,

    #[validate(non_empty)]
    email: String,
}
```

Macro должен генерировать проверки.

---

### Задание 4. Добавьте `#[skip]`

Сделайте:

```rust
#[derive(Validatable)]
struct User {
    #[validate(non_empty)]
    name: String,

    #[skip]
    id: u64,
}
```

Поле `id` не должно участвовать в generated validation code.

---

### Задание 5. Обработка ошибок

Сделайте так, чтобы:

```rust
#[derive(ProcessingService)]
struct BrokenService {
    repo: Box<dyn ProcessingRepository>,
}
```

выдавал понятную ошибку:

```text
no field marked with #[repository] found
```

---

### Задание 6. Поддержка только named structs

Проверьте:

```rust
#[derive(Validatable)]
struct User(String);
```

Macro должен завершаться понятной ошибкой, а не падать на:

```rust
unwrap()
```

---

### Задание 7. Compile-fail тесты

Добавьте тесты для:

```text
missing #[repository]
missing #[processor]
duplicate #[repository]
duplicate #[processor]
enum
tuple struct
```

Проверяйте не только факт ошибки, но и содержание диагностического сообщения.

---

## Главное из этой главы

После этой главы мы:

- создали настоящий `proc-macro` crate;
- разобрались, почему procedural macro должен находиться отдельно от crate-потребителя;
- использовали `syn` для анализа Rust-кода;
- использовали `quote` для генерации Rust-кода;
- создали собственный `#[derive(ProcessingService)]`;
- использовали helper attributes `#[repository]` и `#[processor]`;
- реализовали проверку структуры входного типа;
- научились генерировать понятные ошибки компиляции;
- увидели, как generated code становится частью обычной Rust-программы;
- создали второй derive macro для валидации;
- разобрались с `#[skip]`;
- рассмотрели compile-time тестирование procedural macros;
- разобрались с различием между native `async fn` в traits и использованием `async-trait` для `dyn Trait`.

### Самая важная идея

> **Procedural macro — это генератор Rust-кода, работающий во время компиляции.**

Он получает токены:

```text
Rust code
    │
    ▼
procedural macro
    │
    ├── parse
    ├── validate
    └── generate
    │
    ▼
Rust code
    │
    ▼
rustc
```

Главная ценность procedural macros не в том, что они позволяют написать «магический» код.

Их ценность в другом:

```rust
#[derive(ProcessingService)]
struct DefaultFileProcessingService {
    #[repository]
    repo: Box<dyn ProcessingRepository>,

    #[processor]
    processor: Processor,
}
```

становится компактным **описанием структуры и намерения**, а повторяющийся технический код генерируется автоматически.

Хороший procedural macro должен:

1. иметь простой API;
2. явно описывать свои входные требования;
3. генерировать предсказуемый код;
4. выдавать понятные ошибки;
5. не скрывать бизнес-логику;
6. автоматизировать именно повторяющийся boilerplate.

Именно тогда procedural macro становится не «магией», а ещё одним инструментом проектирования Rust-приложения.
