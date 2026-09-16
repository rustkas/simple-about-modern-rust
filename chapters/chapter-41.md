# Глава 41. Procedural Macros

Декларативные макросы (`macro_rules!`), которые мы изучили в предыдущей главе, — это мощный инструмент. Но у них есть ограничения: они работают на уровне сопоставления с образцом и не могут выполнять сложные вычисления или анализ кода. Для более сложных задач в Rust существуют **процедурные макросы (procedural macros)**.

Процедурные макросы — это настоящие программы на Rust, которые выполняются во время компиляции и могут анализировать и генерировать код на основе синтаксических деревьев. Они открывают безграничные возможности для метапрограммирования: от автоматической реализации трейтов до создания DSL.

В этой главе мы разберёмся, что такое процедурные макросы, как они работают, и научимся создавать свои собственные.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 41.1. Что такое процедурные макросы?

**Процедурный макрос** — это функция, которую компилятор вызывает во время компиляции. Она получает входной `TokenStream`, анализирует его и возвращает другой `TokenStream`.

Важно не смешивать два понятия:

- procedural macro **получает и возвращает токены**;
- синтаксическое дерево (AST) — это более удобное представление этих токенов, которое мы обычно получаем с помощью библиотеки `syn`.

Упрощённо процесс выглядит так:

```text
Rust source code
       │
       ▼
   TokenStream
       │
       ▼
 procedural macro
       │
       ├── syn → структурированный AST
       │
       ├── анализ / вычисления
       │
       └── quote → новый TokenStream
       │
       ▼
   Rust compiler
       │
       ▼
 compiled code
```

Процедурные макросы являются частью компиляции: они не выполняются во время работы программы. Их задача — преобразовать исходный Rust-код в другой Rust-код. ([Rust Documentation][1])

Существует три вида procedural macros:

| Вид                     | Синтаксис использования | Назначение                            |
| ----------------------- | ----------------------- | ------------------------------------- |
| **Derive macro**        | `#[derive(MyTrait)]`    | Генерация реализаций трейтов          |
| **Attribute macro**     | `#[my_attribute]`       | Преобразование элемента Rust-кода     |
| **Function-like macro** | `my_macro!(...)`        | Генерация кода из произвольного входа |

Например:

```rust
#[derive(Debug)]
struct User {
    name: String,
}
```

`Debug` здесь — derive macro.

```rust
#[some_attribute]
fn process() {
    // ...
}
```

`some_attribute` может быть attribute procedural macro.

```rust
make_function!(hello);
```

`make_function!` может быть function-like procedural macro.

Не следует считать процедурными макросами все конструкции с `!` или `#[...]`. Например, `println!` — макрос стандартной библиотеки, а `#[test]` — встроенный атрибут языка; это не примеры пользовательских procedural macros.

---

## 41.2. Структура `proc-macro` crate

Процедурный макрос должен находиться в отдельном crate с типом:

```toml
[lib]
proc-macro = true
```

Причём crate, содержащий макрос, **не может использовать этот макрос непосредственно внутри себя**. Макрос определяется в одном crate, а используется другим crate. ([Rust Documentation][1])

Поэтому на практике удобно создавать workspace:

```text
hello-workspace/
├── Cargo.toml
├── hello-macro/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
└── app/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

Корневой `Cargo.toml`:

```toml
[workspace]
members = ["hello-macro", "app"]
resolver = "3"
```

`hello-macro/Cargo.toml`:

```toml
[package]
name = "hello-macro"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
quote = "1"
syn = { version = "2", features = ["full"] }
```

`app/Cargo.toml`:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2024"

[dependencies]
hello-macro = { path = "../hello-macro" }
```

Минимальный procedural macro:

```rust
use proc_macro::TokenStream;

#[proc_macro]
pub fn answer(_input: TokenStream) -> TokenStream {
    "42".parse().unwrap()
}
```

А в `app/src/main.rs`:

```rust
use hello_macro::answer;

fn main() {
    let value = answer!();

    println!("{value}");
}
```

Результат:

```text
42
```

Такой пример нужно запускать из Cargo workspace:

```bash
cargo run -p app
```

**Примечание о Rust Playground:** пользовательский procedural macro crate требует отдельного crate, поэтому такой workspace-пример нельзя корректно представить как один самостоятельный файл Rust Playground. Для него лучше использовать локальный Cargo-проект.

---

## 41.3. `TokenStream` — вход и выход procedural macro

`TokenStream` — это последовательность токенов Rust. Именно этот тип используется в интерфейсе всех трёх видов procedural macros. ([Rust Documentation][2])

Например, для function-like macro:

```rust
#[proc_macro]
pub fn answer(input: TokenStream) -> TokenStream {
    // input содержит то, что находится внутри answer!(...)
    //
    // Например:
    //
    // answer!(1 + 2)
    //
    // input представляет токены:
    // 1 + 2

    "42".parse().unwrap()
}
```

Для derive macro компилятор передаёт токены элемента, к которому применяется `derive`:

```rust
#[derive(MyTrait)]
struct User {
    name: String,
}
```

Вход procedural macro концептуально соответствует:

```rust
struct User {
    name: String,
}
```

Для attribute macro передаются два потока:

```rust
#[log]
fn hello() {}
```

Функция procedural macro имеет сигнатуру:

```rust
pub fn log(
    attr: TokenStream,
    item: TokenStream,
) -> TokenStream
```

Здесь:

- `attr` — содержимое атрибута;
- `item` — элемент, к которому применён атрибут;
- возвращаемый `TokenStream` заменяет исходный элемент.

Именно поэтому procedural macro может анализировать и преобразовывать практически любой допустимый Rust-синтаксис. ([Rust Documentation][1])

---

## 41.4. `syn` — превращаем токены в AST

Работать непосредственно с `TokenStream` можно, но это быстро становится неудобным.

Например, вместо ручного разбора:

```text
struct User {
    name: String
}
```

можно использовать `syn`:

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait)]
pub fn my_trait(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    let name = &ast.ident;

    // name — идентификатор структуры, например User
    // Далее можно обойти поля, варианты enum и т.д.

    // ...
}
```

`DeriveInput` содержит структурированное представление входного элемента.

Основные типы `syn`:

```
| Тип           | Назначение                                            |
| ------------- | ----------------------------------------------------- |
| `DeriveInput` | `struct`, `enum` или `union`, переданные derive macro |
| `ItemFn`      | функция                                               |
| `ItemStruct`  | структура                                             |
| `ItemEnum`    | перечисление                                          |
| `Field`       | поле структуры                                        |
| `Variant`     | вариант enum                                          |
| `Type`        | тип                                                   |
| `Expr`        | выражение                                             |
| `Pat`         | pattern                                               |
```

Пример обхода полей структуры:

```rust
use syn::{Data, Fields};

if let Data::Struct(data_struct) = &ast.data {
    if let Fields::Named(fields) = &data_struct.fields {
        for field in &fields.named {
            let field_name = field.ident.as_ref();
            let field_type = &field.ty;

            // Анализируем поле
        }
    }
}
```

Именно поэтому типичная архитектура procedural macro выглядит так:

```text
TokenStream
│
▼
syn
│
▼
AST
│
▼
analysis
│
▼
quote!
│
▼
TokenStream
```

---

## 41.5. `quote` — генерация Rust-кода

`quote` позволяет записывать генерируемый Rust-код почти так же, как обычный Rust-код.

```rust
let generated = quote! {
    fn hello() {
        println!("Hello!");
    }
};
```

Если нужно вставить значение из Rust-программы, используется `#`:

```rust
let name = &ast.ident;

let generated = quote! {
    impl MyTrait for #name {
        // ...
    }
};
```

Если `name` содержит `User`, результат будет концептуально таким:

```rust
impl MyTrait for User {
    // ...
}
```

`quote!` возвращает `proc_macro2::TokenStream`, который можно преобразовать в `proc_macro::TokenStream`:

```rust
generated.into()
```

Полный типичный фрагмент procedural macro выглядит так:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait)]
pub fn my_trait(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;

    let generated = quote! {
        impl MyTrait for #name {
            fn hello(&self) {
                println!("Hello from {}", stringify!(#name));
            }
        }
    };

    generated.into()
}
```

`quote` отвечает за генерацию токенов, а `syn` — за их анализ. ([Docs.rs][3])

---

## 41.6. Пример: derive macro

Рассмотрим полноценный пример.

### Задача

Хотим написать:

```rust
#[derive(Hello)]
struct Person;
```

и автоматически получить:

```rust
impl Hello for Person {
    fn say_hello(&self) {
        println!("Hello from Person");
    }
}
```

### Основной crate

В `app/src/main.rs`:

```rust
use hello_macro::Hello;

pub trait Hello {
    fn say_hello(&self);
}

#[derive(Hello)]
struct Person;

fn main() {
    let person = Person;

    person.say_hello();
}
```

### `proc-macro` crate

В `hello-macro/src/lib.rs`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Hello)]
pub fn hello_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;

    let generated = quote! {
        impl Hello for #name {
            fn say_hello(&self) {
                println!("Hello from {}", stringify!(#name));
            }
        }
    };

    generated.into()
}
```

После расширения:

```rust
#[derive(Hello)]
struct Person;
```

становится примерно:

```rust
struct Person;

impl Hello for Person {
    fn say_hello(&self) {
        println!("Hello from {}", stringify!(Person));
    }
}
```

Здесь procedural macro не реализует `Hello` во время выполнения программы. Он **генерирует обычный Rust-код**, который затем компилируется как часть программы.

**Открыть пример в Rust Playground:** для полного примера с собственным derive macro требуется Cargo workspace из двух crates, поэтому его нельзя корректно запустить одной ссылкой Playground. Для проверки используйте структуру проекта из раздела 41.2.

---

## 41.7. Пример: Attribute macro

Attribute macro получает два `TokenStream`:

```rust
#[log_execution]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Первый содержит аргументы атрибута:

```text
log_execution(...)
       │
       ▼
     attr
```

Второй содержит сам элемент:

```text
fn add(a: i32, b: i32) -> i32 {
    a + b
}
       │
       ▼
     item
```

### Реализация

`hello-macro/src/lib.rs`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log_execution(
    _attr: TokenStream,
    item: TokenStream,
) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);

    let fn_name = &input_fn.sig.ident;
    let fn_sig = &input_fn.sig;
    let fn_vis = &input_fn.vis;
    let fn_block = &input_fn.block;

    let generated = quote! {
        #fn_vis #fn_sig {
            println!(
                "Executing function: {}",
                stringify!(#fn_name)
            );

            let result = #fn_block;

            println!(
                "Function completed: {}",
                stringify!(#fn_name)
            );

            result
        }
    };

    generated.into()
}
```

Использование:

```rust
use hello_macro::log_execution;

#[log_execution]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let result = add(5, 3);

    println!("Result: {result}");
}
```

Результат:

```text
Executing function: add
Function completed: add
Result: 8
```

Концептуально макрос преобразует:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

в:

```rust
fn add(a: i32, b: i32) -> i32 {
    println!("Executing function: add");

    let result = {
        a + b
    };

    println!("Function completed: add");

    result
}
```

**Важно:** этот простой пример специально демонстрирует механизм. В production-макросе нужно отдельно учитывать `async fn`, `unsafe fn`, `extern fn`, generics, `where` clauses, другие атрибуты и особенности возвращаемого значения.

**Открыть пример в Rust Playground:** собственный attribute macro требует отдельного `proc-macro` crate, поэтому полноценный вариант запускается через Cargo workspace, а не через один файл Playground.

---

## 41.8. Пример: Function-like procedural macro

Function-like procedural macro выглядит как обычный макрос:

```rust
my_format!("Hello", " ", "World", "!");
```

но реализуется функцией с атрибутом:

```rust
#[proc_macro]
pub fn my_format(input: TokenStream) -> TokenStream {
    // ...
}
```

Рассмотрим простой макрос, который принимает несколько строковых литералов и объединяет их **во время компиляции**.

```rust
use proc_macro::TokenStream;
use syn::{
    parse_macro_input,
    punctuated::Punctuated,
    LitStr,
    Token,
};
use quote::quote;

#[proc_macro]
pub fn my_format(input: TokenStream) -> TokenStream {
    let parts = parse_macro_input!(
        input with Punctuated::<LitStr, Token![,]>::parse_terminated
    );

    let mut result = String::new();

    for part in parts {
        result.push_str(&part.value());
    }

    let result = LitStr::new(
        &result,
        proc_macro2::Span::call_site(),
    );

    quote! {
        #result
    }
    .into()
}
```

Использование:

```rust
use hello_macro::my_format;

fn main() {
    let text = my_format!("Hello", " ", "World", "!");

    println!("{text}");
}
```

Результат:

```text
Hello World!
```

Важный момент: здесь макрос не генерирует вызов функции объединения строк. Он вычисляет результат во время компиляции и генерирует строковый литерал:

```rust
"Hello World!"
```

Поэтому это хороший пример того, чем procedural macro отличается от обычной функции.

**Открыть пример в Rust Playground:** реализация собственного function-like procedural macro требует отдельного `proc-macro` crate, поэтому этот пример запускается как Cargo workspace.

---

## 41.9. Диагностика ошибок

Procedural macro не должен использовать `panic!` для обычных ошибок пользователя.

Например, если derive macro предназначен только для структур, лучше сообщить об ошибке непосредственно на исходном элементе:

```rust
use proc_macro::TokenStream;
use syn::{
    parse_macro_input,
    Data,
    DeriveInput,
};

#[proc_macro_derive(MyTrait)]
pub fn my_trait(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    if !matches!(ast.data, Data::Struct(_)) {
        return syn::Error::new_spanned(
            &ast.ident,
            "MyTrait can only be derived for structs",
        )
        .to_compile_error()
        .into();
    }

    // Генерация кода...

    TokenStream::new()
}
```

Теперь:

```rust
#[derive(MyTrait)]
enum Color {
    Red,
    Green,
    Blue,
}
```

получит диагностическое сообщение:

```text
error: MyTrait can only be derived for structs
```

Это значительно лучше, чем:

```rust
panic!("Only structs are supported");
```

Потому что ошибка procedural macro становится частью нормальной диагностики компилятора.

Если нужно накопить несколько ошибок, их можно объединять и вернуть одним `TokenStream`. Это особенно полезно для derive-макросов, проверяющих множество полей.

---

## 41.10. Helper attributes на полях

Derive macro может объявить собственные **helper attributes**.

Например:

```rust
#[derive(MySerialize)]
struct User {
    name: String,

    #[my_serialize(skip)]
    password: String,
}
```

Чтобы атрибут `#[my_serialize(...)]` был разрешён для использования вместе с derive macro, его нужно объявить:

```rust
#[proc_macro_derive(MySerialize, attributes(my_serialize))]
pub fn my_serialize(input: TokenStream) -> TokenStream {
    // ...
}
```

Теперь procedural macro может анализировать атрибуты полей:

```rust
use syn::{Data, Fields};

if let Data::Struct(data_struct) = &ast.data {
    if let Fields::Named(fields) = &data_struct.fields {
        for field in &fields.named {
            for attr in &field.attrs {
                if attr.path().is_ident("my_serialize") {
                    // Здесь можно разобрать аргументы атрибута
                }
            }
        }
    }
}
```

Для конкретного формата атрибута удобно использовать `syn`. Например, для

```rust
#[my_serialize(skip)]
```

можно написать вспомогательную функцию:

```rust
fn is_skip(attr: &syn::Attribute) -> bool {
    if !attr.path().is_ident("my_serialize") {
        return false;
    }

    // Простейший вариант: проверяем, что аргумент — идентификатор `skip`
    matches!(
        attr.parse_args::<syn::Ident>(),
        Ok(ident) if ident == "skip"
    )
}
```

На практике лучше сразу собирать структурированную конфигурацию:

```rust
struct FieldOptions {
    skip: bool,
}

fn parse_field_options(field: &syn::Field) -> FieldOptions {
    let mut options = FieldOptions { skip: false };

    for attr in &field.attrs {
        if is_skip(attr) {
            options.skip = true;
        }
    }

    options
}
```

Тогда основная логика генерации кода не будет зависеть от синтаксиса атрибута и останется читаемой.

Helper attributes являются частью механизма derive macros и объявляются через `attributes(...)` в `proc_macro_derive`.

---

## 41.11. `proc_macro`, `proc_macro2`, `syn` и `quote`

В современном procedural macro crate часто встречаются четыре связанных типа:

```text
proc_macro::TokenStream
│
│ API компилятора
▼
proc_macro2::TokenStream
│
┌────┴────┐
▼         ▼
syn       quote
│         │
parse     generate
```

### `proc_macro`

Это стандартный crate Rust, предоставляемый компилятором.

Он используется в публичном интерфейсе procedural macro:

```rust
use proc_macro::TokenStream;

#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
    // ...
}
```

### `proc_macro2`

`proc_macro2` предоставляет совместимый тип `TokenStream`, предназначенный для использования обычным Rust-кодом и библиотеками procedural macro.

Поэтому типичная реализация выглядит так:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::parse_macro_input;

#[proc_macro_derive(MyTrait)]
pub fn derive_my_trait(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as syn::DeriveInput);

    let name = &ast.ident;

    let output = quote! {
        impl MyTrait for #name {}
    };

    output.into()
}
```

Здесь:

- `proc_macro::TokenStream` — интерфейс между компилятором и макросом;
- `syn` — разбирает вход;
- `quote` — создаёт выход;
- `.into()` — преобразует `proc_macro2::TokenStream` в `proc_macro::TokenStream`.

### `parse_quote!`

Для небольших фрагментов AST можно использовать `syn::parse_quote!`:

```rust
use syn::{parse_quote, Expr, ItemFn};

let expr: Expr = parse_quote! {
    1 + 2
};

let function: ItemFn = parse_quote! {
    fn answer() -> i32 {
        42
    }
};
```

Это особенно удобно в тестах procedural macros и при построении небольших частей AST.

**Примечание:** примеры с `syn` и `quote` требуют добавления соответствующих зависимостей. Полноценный procedural macro всё равно нуждается в отдельном crate с `proc-macro = true` и обычно разрабатывается как часть Cargo workspace.

---

## 41.12. Hygiene процедурных макросов

Здесь есть принципиальное отличие от `macro_rules!`.

`macro_rules!` является гигиеничным механизмом макросов.

**Procedural macros, напротив, являются unhygienic.** Их результат рассматривается практически так, как будто сгенерированный код был написан непосредственно в месте вызова. Поэтому имена и пути в сгенерированном коде могут конфликтовать с кодом пользователя. ([Rust Documentation][1])

Например, генерировать:

```rust
fn helper() {
    // ...
}
```

может быть опасно: у пользователя уже может существовать функция `helper`.

Поэтому генераторы кода часто используют имена, маловероятные для конфликта:

```rust
fn __my_macro_helper() {
    // ...
}
```

Ещё важнее — абсолютные пути.

Вместо:

```rust
Option<String>
```

в сгенерированном коде иногда безопаснее использовать:

```rust
::std::option::Option<String>
```

поскольку пользовательский код может иметь собственный `Option` или изменить область видимости.

Поэтому procedural macro должен рассматривать сгенерированный код как обычный код пользователя и тщательно контролировать:

- имена;
- пути;
- видимость;
- helper functions;
- генерируемые типы;
- возможные конфликты.

---

## 41.13. Когда использовать procedural macros

Procedural macro оправдан, когда требуется анализировать структуру Rust-кода и на основе этого генерировать другой код.

### Хорошие кандидаты

**Derive macros:**

```rust
#[derive(MySerialize)]
struct User {
    name: String,
    age: u32,
}
```

Особенно хорошо подходят для:

- сериализации;
- десериализации;
- реализации трейтов;
- генерации builder API;
- регистрации типов;
- генерации boilerplate.

**Attribute macros:**

```rust
#[route("/users")]
fn users() {
    // ...
}
```

Подходят для:

- web frameworks;
- RPC;
- тестовых framework'ов;
- instrumentation;
- декларативной конфигурации.

**Function-like macros:**

```rust
sql!("SELECT * FROM users");
```

или:

```rust
html! {
    <div>Hello</div>
}
```

Подходят для:

- DSL;
- SQL;
- HTML;
- конфигурационных языков;
- compile-time validation.

### Когда procedural macro не нужен

Если задача решается обычной функцией:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

не следует превращать её в procedural macro.

Если достаточно простого сопоставления шаблонов:

```rust
macro_rules! double {
    ($x:expr) => {
        $x * 2
    };
}
```

лучше использовать `macro_rules!`.

Главный критерий:

> Используйте procedural macro тогда, когда вам действительно необходимо **анализировать структуру Rust-кода или генерировать код на её основе**.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: procedural macro нельзя определить в обычном crate

Попробуйте:

```rust
use proc_macro::TokenStream;

#[proc_macro]
pub fn answer(_input: TokenStream) -> TokenStream {
    "42".parse().unwrap()
}

fn main() {}
```

без:

```toml
[lib]
proc-macro = true
```

Компилятор сообщит, что атрибут `#[proc_macro]` можно использовать только в crate типа `proc-macro`.

Это показывает, что procedural macro — специальный тип crate, а не обычная функция с дополнительным атрибутом. ([Rust Documentation][4])

---

### Эксперимент 2: macro нельзя использовать в том же crate

Создайте `proc-macro` crate и попробуйте вызвать собственный макрос внутри `src/lib.rs`.

Это запрещено:

```rust
#[proc_macro]
pub fn answer(_input: TokenStream) -> TokenStream {
    "42".parse().unwrap()
}

answer!();
```

Procedural macro должен быть импортирован и использован из **другого crate**. ([Rust Documentation][1])

---

### Эксперимент 3: макрос возвращает невалидный Rust-код

Создайте:

```rust
#[proc_macro]
pub fn broken(_input: TokenStream) -> TokenStream {
    "this is not valid Rust".parse().unwrap()
}
```

Затем вызовите:

```rust
broken!();
```

Procedural macro успешно выполнил свою функцию и вернул токены, но эти токены не являются допустимым Rust-кодом.

Ошибка возникнет уже на следующем этапе компиляции.

Это важная модель:

```text
procedural macro
       │
       │ генерирует TokenStream
       ▼
Rust compiler
       │
       │ парсит сгенерированный код
       ▼
compiler error
```

Procedural macro отвечает за генерацию корректного потока токенов, а компилятор затем проверяет получившийся Rust-код.

---

### Эксперимент 4: проверить результат через `cargo expand`

Для изучения procedural macros очень полезен `cargo expand`.

Установка:

```bash
cargo install cargo-expand
```

После этого:

```bash
cargo expand
```

показывает код после macro expansion.

Например, исходник:

```rust
#[derive(Hello)]
struct Person;
```

можно исследовать, чтобы увидеть сгенерированный:

```rust
impl Hello for Person {
    // ...
}
```

Это один из лучших способов понять, что именно делает procedural macro.

---

## Практика

### Задание 1 — `Describe`

Создайте derive macro:

```rust
#[derive(Describe)]
struct User {
    name: String,
    age: u32,
}
```

который генерирует метод:

```rust
user.describe();
```

с выводом:

```text
User
  name: String
  age: u32
```

Макрос должен получать имя типа и анализировать поля через `syn`.

---

### Задание 2 — `timed`

Создайте attribute macro:

```rust
#[timed]
fn calculate() {
    // ...
}
```

который генерирует измерение времени выполнения:

```text
calculate: 123 µs
```

Используйте:

```rust
std::time::Instant
```

Обратите внимание на сохранение исходной сигнатуры функции.

---

### Задание 3 — `stringify_values!`

Создайте function-like procedural macro:

```rust
stringify_values!(10, 20, 30)
```

который генерирует строку:

```text
10, 20, 30
```

Попробуйте сделать так, чтобы выражения:

```rust
stringify_values!(1 + 2, 10 * 3)
```

сохраняли именно исходный текст:

```text
1 + 2, 10 * 3
```

Для этого понадобится работа с `TokenStream`, а не только с вычисленными значениями.

---

### Задание 4 — ошибка derive macro

Создайте:

```rust
#[derive(Describe)]
enum Color {
    Red,
    Green,
    Blue,
}
```

и сделайте так, чтобы макрос выдавал понятную compile-time ошибку:

```text
Describe can only be derived for structs
```

Используйте:

```rust
syn::Error::new_spanned(...)
```

вместо `panic!`.

---

### Задание 5 — helper attribute

Расширьте `Describe`:

```rust
#[derive(Describe)]
struct User {
    name: String,

    #[describe(skip)]
    password: String,
}
```

Поле `password` не должно попадать в результат.

Для этого объявите helper attribute:

```rust
#[proc_macro_derive(Describe, attributes(describe))]
```

и разберите его с помощью `syn`.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Procedural macros** — это программы, которые компилятор запускает во время компиляции.
- Их интерфейс основан на `TokenStream`.
- Существует три вида procedural macros:
  - **derive** — `#[derive(MyTrait)]`;
  - **attribute** — `#[my_attribute]`;
  - **function-like** — `my_macro!(...)`.

- Procedural macros должны находиться в отдельном crate с `proc-macro = true`.
- `syn` позволяет превратить `TokenStream` в удобные структуры синтаксического дерева.
- `quote` позволяет удобно генерировать новый Rust-код.
- `proc_macro2` предоставляет удобный `TokenStream` для экосистемы `syn`/`quote`.
- `syn::Error` позволяет создавать нормальные compile-time diagnostics.
- Derive macros могут объявлять helper attributes.
- В отличие от `macro_rules!`, procedural macros **не являются hygienic**.
- `cargo expand` позволяет увидеть результат macro expansion.
- Полноценные procedural macros обычно разрабатываются как Cargo workspace с отдельным crate для макроса и отдельным crate для его использования.

**Самая важная идея:**

> `macro_rules!` позволяет сопоставлять синтаксические шаблоны и заменять их другим кодом. Procedural macro идёт дальше: он получает поток токенов, может разобрать его в синтаксическое дерево, выполнить произвольную логику анализа и сгенерировать новый Rust-код.
>
> Поэтому procedural macros — это основа многих сложных библиотек Rust: они позволяют превратить описание структуры программы в автоматически сгенерированный, проверяемый компилятором Rust-код.

[1]: https://doc.rust-lang.org/beta/reference/procedural-macros.html 'Procedural macros - The Rust Reference'
[2]: https://doc.rust-lang.org/proc_macro/struct.TokenStream.html 'TokenStream in proc_macro - Rust'
[3]: https://docs.rs/crate/quote/latest/source/README.md 'quote 1.0.47 - Docs.rs'
[4]: https://doc.rust-lang.org/beta/core/attribute.proc_macro.html 'proc_macro - Rust'
