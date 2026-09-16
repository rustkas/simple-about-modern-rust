# Глава 42. `syn` и `quote`

В предыдущей главе мы познакомились с процедурными макросами и увидели, как они работают на высоком уровне. Теперь пришло время погрузиться в детали двух библиотек, которые являются сердцем процедурных макросов: **`syn`** и **`quote`**.

`syn` превращает поток токенов в структурированное синтаксическое дерево (AST), а `quote` превращает синтаксическое дерево обратно в поток токенов. Вместе они образуют фундамент для создания мощных и безопасных процедурных макросов.

В этой главе мы изучим эти библиотеки в деталях и создадим полноценный процедурный макрос.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 42.1. Что такое `syn`?

**`syn`** — библиотека для разбора Rust-кода и представления его в виде типизированных структур синтаксического дерева.

Важно понимать границу ответственности:

```text
proc_macro::TokenStream
        │
        ▼
      syn
        │
        ▼
типизированные структуры
DeriveInput / ItemFn / Expr / Type / Field / ...
        │
        ▼
анализ и преобразование
        │
        ▼
      quote!
        │
        ▼
proc_macro2::TokenStream
        │
        ▼
proc_macro::TokenStream
```

`TokenStream` содержит токены Rust-кода. `syn` позволяет работать с ними как с понятными программными структурами: именем типа, полями, типами полей, generics, атрибутами и т. д.

Например:

```rust
use syn::{parse_str, ItemStruct};

fn main() {
    let item: ItemStruct = parse_str(
        r#"
        struct Person {
            name: String,
            age: u32,
        }
        "#,
    )
    .unwrap();

    println!("name = {}", item.ident);
    println!("fields = {}", item.fields.len());
}
```

Здесь мы не работаем со строкой `"struct Person { ... }"` напрямую. `syn` разбирает её и создаёт `ItemStruct`.

**Открыть пример в Rust Playground:**
[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

Для procedural macro вход обычно выглядит так:

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait)]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    let name = &ast.ident;
    let generics = &ast.generics;
    let data = &ast.data;

    // Анализируем ast...

    todo!()
}
```

Основные структуры `syn`:

| Структура     | Что представляет                                         |
| ------------- | -------------------------------------------------------- |
| `DeriveInput` | `struct`, `enum` или `union`, полученные derive-макросом |
| `ItemFn`      | функцию                                                  |
| `ItemStruct`  | объявление структуры                                     |
| `ItemEnum`    | объявление перечисления                                  |
| `ItemImpl`    | блок `impl`                                              |
| `Fields`      | набор полей                                              |
| `Field`       | отдельное поле                                           |
| `Type`        | тип Rust                                                 |
| `Expr`        | выражение                                                |
| `Stmt`        | оператор                                                 |
| `Pat`         | pattern                                                  |
| `Attribute`   | атрибут                                                  |

Не все типы `syn` доступны безусловно: часть функциональности включается feature-флагами. Например, `full` предоставляет полное синтаксическое дерево Rust, а `visit` — API для обхода дерева. ([Docs.rs][3])

---

## 42.2. Парсинг с `syn`

### Парсинг `DeriveInput`

Для derive-макроса наиболее часто используется `parse_macro_input!`:

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait)]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    let name = &ast.ident;
    let generics = &ast.generics;
    let data = &ast.data;

    // Анализируем структуру...

    todo!()
}
```

`parse_macro_input!` удобен тем, что при ошибке парсинга автоматически превращает ошибку `syn` в compile-time error для пользователя макроса. ([Docs.rs][3])

После разбора можно проверить, что именно передал пользователь:

```rust
match &ast.data {
    syn::Data::Struct(data) => {
        println!("Это структура");
        println!("Количество полей: {}", data.fields.len());
    }

    syn::Data::Enum(data) => {
        println!("Это enum");
        println!("Количество вариантов: {}", data.variants.len());
    }

    syn::Data::Union(_) => {
        println!("Это union");
    }
}
```

### Парсинг функции

Для attribute-макроса можно разобрать вход как `ItemFn`:

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn my_attribute(
    _attr: TokenStream,
    item: TokenStream,
) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);

    let fn_name = &input_fn.sig.ident;
    let fn_vis = &input_fn.vis;
    let fn_sig = &input_fn.sig;
    let fn_block = &input_fn.block;

    // Анализируем функцию...

    todo!()
}
```

Теперь макрос имеет доступ не только к имени функции, но и к её аргументам, возвращаемому типу, generics, атрибутам и телу.

---

## 42.3. Обход полей структуры

Здесь важно учитывать, что Rust поддерживает **три формы структуры**:

```rust
struct Person {
    name: String,
    age: u32,
}

struct Color(u8, u8, u8);

struct Empty;
```

Для них `syn::Fields` содержит соответственно:

```rust
syn::Fields::Named(...)
syn::Fields::Unnamed(...)
syn::Fields::Unit
```

Поэтому следующий код из чернового варианта:

```rust
field.ident.as_ref().unwrap()
```

**не является универсальным**. Он работает только для структур с именованными полями.

Если макрос действительно предназначен только для таких структур, это нужно явно проверить:

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(MyTrait)]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => &fields.named,

            _ => {
                return syn::Error::new_spanned(
                    &ast,
                    "MyTrait requires a struct with named fields",
                )
                .to_compile_error()
                .into();
            }
        },

        _ => {
            return syn::Error::new_spanned(
                &ast,
                "MyTrait can only be derived for structs",
            )
            .to_compile_error()
            .into();
        }
    };

    for field in fields {
        let name = field.ident.as_ref().unwrap();
        let ty = &field.ty;

        // Анализируем name и ty...
    }

    todo!()
}
```

Такой вариант лучше, чем `unwrap()`: пользователь получает понятную ошибку компиляции вместо panic внутри procedural macro.

---

## 42.4. Работа с атрибутами

В `syn` 2.x атрибуты обычно разбираются двумя способами:

- `parse_args()` — когда содержимое атрибута имеет собственную грамматику;
- `parse_nested_meta()` — для привычного синтаксиса вроде `#[my_trait(skip, rename = "...")]`. ([Docs.rs][1])

Например, допустим, мы хотим поддерживать:

```rust
#[derive(MyTrait)]
#[my_trait(skip_all)]
struct Person {
    #[my_trait(skip)]
    password: String,

    name: String,
}
```

Разбор `skip_all`:

```rust
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait, attributes(my_trait))]
pub fn my_trait_derive(input: proc_macro::TokenStream)
    -> proc_macro::TokenStream
{
    let ast = parse_macro_input!(input as DeriveInput);

    let mut skip_all = false;

    for attr in &ast.attrs {
        if attr.path().is_ident("my_trait") {
            let result = attr.parse_nested_meta(|meta| {
                if meta.path.is_ident("skip_all") {
                    skip_all = true;
                    Ok(())
                } else {
                    Err(meta.error("expected `skip_all`"))
                }
            });

            if let Err(error) = result {
                return error.to_compile_error().into();
            }
        }
    }

    // ...
    todo!()
}
```

Аналогично можно разбирать атрибуты полей:

```rust
for field in fields {
    let mut skip = false;

    for attr in &field.attrs {
        if attr.path().is_ident("my_trait") {
            let result = attr.parse_nested_meta(|meta| {
                if meta.path.is_ident("skip") {
                    skip = true;
                    Ok(())
                } else {
                    Err(meta.error("expected `skip`"))
                }
            });

            if let Err(error) = result {
                return error.to_compile_error().into();
            }
        }
    }

    if skip {
        continue;
    }

    // Обрабатываем поле.
}
```

Для аргумента вида:

```rust
#[my_trait(rename = "user_name")]
```

можно использовать:

```rust
use syn::LitStr;

attr.parse_nested_meta(|meta| {
    if meta.path.is_ident("rename") {
        let value: LitStr = meta.value()?.parse()?;
        println!("Новое имя: {}", value.value());
        Ok(())
    } else {
        Err(meta.error("unknown option"))
    }
})?;
```

Если же атрибут содержит произвольную грамматику, можно использовать `attr.parse_args::<T>()`. Например, `#[precondition(value < 5)]` можно разобрать непосредственно в `syn::Expr`. ([Docs.rs][1])

---

## 42.5. Что такое `quote`?

**`quote`** — библиотека для генерации потока токенов.

Основной инструмент — макрос `quote!`:

```rust
use quote::{format_ident, quote};

fn main() {
    let name = format_ident!("Person");
    let method = format_ident!("hello");

    let tokens = quote! {
        impl #name {
            fn #method(&self) {
                println!("Hello!");
            }
        }
    };

    println!("{tokens}");
}
```

`#name` означает: **вставить в это место значение переменной `name`**.

Для интерполяции значение должно реализовывать `quote::ToTokens`; большинство типов синтаксического дерева `syn` это уже умеют. ([Docs.rs][4])

Например, `syn::Ident` можно вставить непосредственно:

```rust
let name: syn::Ident = syn::parse_quote!(Person);

let tokens = quote! {
    struct #name;
};
```

А для создания идентификатора из строки удобно использовать `format_ident!`:

```rust
let field_name = format_ident!("name");

let tokens = quote! {
    self.#field_name
};
```

Это существенно важнее, чем кажется: обычная строка

```rust
let name = "Person";
```

не является Rust-идентификатором. Поэтому нельзя бездумно подставлять строки туда, где синтаксис Rust требует имя типа, поля или функции.

**Открыть пример в Rust Playground:**
[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## 42.6. Повторения в `quote!`

Одно из самых важных возможностей `quote!` — повторение:

```rust
#(#items)*
```

или с разделителем:

```rust
#(#items),*
```

Например:

```rust
use quote::{format_ident, quote};

fn main() {
    let fields = ["name", "age", "email"]
        .into_iter()
        .map(format_ident)
        .collect::<Vec<_>>();

    let tokens = quote! {
        #(
            println!("{}", stringify!(#fields));
        )*
    };

    println!("{tokens}");
}
```

Результат будет примерно таким:

```rust
println!("{}", stringify!(name));
println!("{}", stringify!(age));
println!("{}", stringify!(email));
```

Для разделителей:

```rust
let fields = vec![
    format_ident!("name"),
    format_ident!("age"),
    format_ident!("email"),
];

let tokens = quote! {
    (#(#fields),*)
};
```

получится:

```rust
(name, age, email)
```

Основные варианты:

```rust
#(#items)*
```

Без разделителя.

```rust
#(#items),*
```

С запятой.

```rust
#(#items);*
```

С точкой с запятой.

Особенно полезно то, что повторяться может не просто переменная:

```rust
#(
    fn #methods(&self) {}
)*
```

Здесь `methods` определяет количество повторений, а весь фрагмент внутри `#(...)` повторяется для каждого элемента.

Важно: внутри повторения должна присутствовать переменная, по которой `quote!` может определить количество итераций. Именно поэтому конструкции вроде:

```rust
#(
    println!("hello");
)*
```

недостаточно — `quote!` не знает, сколько раз её повторять. ([Docs.rs][4])

---

## 42.7. Практический пример: полноценный derive-макрос

Вместо собственного `ToString` лучше создать отдельный трейт:

```rust
trait Describe {
    fn describe(&self) -> String;
}
```

Почему?

Потому что `std::string::ToString` уже является частью стандартного Rust API и связан с `Display`. Для учебного procedural macro собственный трейт `Describe` намного понятнее.

### Макрос

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{
    parse_macro_input,
    Data,
    DeriveInput,
    Fields,
};

#[proc_macro_derive(Describe)]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    let name = &ast.ident;
    let generics = &ast.generics;

    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => &fields.named,

            _ => {
                return syn::Error::new_spanned(
                    &ast,
                    "Describe requires a struct with named fields",
                )
                .to_compile_error()
                .into();
            }
        },

        _ => {
            return syn::Error::new_spanned(
                &ast,
                "Describe can only be derived for structs",
            )
            .to_compile_error()
            .into();
        }
    };

    let field_names = fields.iter().map(|field| {
        let ident = field.ident.as_ref().unwrap();
        quote! {
            stringify!(#ident)
        }
    });

    let field_values = fields.iter().map(|field| {
        let ident = field.ident.as_ref().unwrap();
        quote! {
            &self.#ident
        }
    });

    let (impl_generics, ty_generics, where_clause) =
        generics.split_for_impl();

    let expanded = quote! {
        impl #impl_generics Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String {
                let mut parts = Vec::new();

                #(
                    parts.push(format!(
                        "{}={:?}",
                        #field_names,
                        #field_values
                    ));
                )*

                parts.join(", ")
            }
        }
    };

    expanded.into()
}
```

Здесь появилась важная конструкция:

```rust
let (impl_generics, ty_generics, where_clause) =
    generics.split_for_impl();
```

Она необходима, если макрос должен корректно работать не только с:

```rust
struct Person {
    name: String,
}
```

но и с:

```rust
struct Person<T>
where
    T: std::fmt::Debug,
{
    value: T,
}
```

Без `split_for_impl()` derive-макросы часто начинают ломаться на generics.

### Использование

```rust
#[derive(Describe)]
struct Person {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    println!("{}", person.describe());
}
```

Результат:

```text
name="Alice", age=30, active=true
```

### Что реально генерирует макрос

Концептуально:

```rust
impl Describe for Person {
    fn describe(&self) -> String {
        let mut parts = Vec::new();

        parts.push(format!("{}={:?}", "name", &self.name));
        parts.push(format!("{}={:?}", "age", &self.age));
        parts.push(format!("{}={:?}", "active", &self.active));

        parts.join(", ")
    }
}
```

Именно в этом состоит основная идея связки `syn` + `quote`:

```text
DeriveInput
    ↓
анализ полей
    ↓
создание коллекций токенов
    ↓
quote!
    ↓
готовый impl
```

---

## 42.8. Практический пример: Attribute-макрос `log`

Attribute-макрос получает **два** потока токенов:

```rust
#[log(level = "debug")]
fn calculate() {}
```

Первый:

```rust
level = "debug"
```

передаётся в `attr`.

Второй:

```rust
fn calculate() {}
```

передаётся в `item`.

Простейший вариант без аргументов:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log(
    _attr: TokenStream,
    item: TokenStream,
) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);

    let fn_name = &input_fn.sig.ident;
    let fn_vis = &input_fn.vis;
    let fn_sig = &input_fn.sig;
    let fn_block = &input_fn.block;

    let expanded = quote! {
        #fn_vis #fn_sig {
            println!("Entering: {}", stringify!(#fn_name));

            let result = #fn_block;

            println!("Exiting: {}", stringify!(#fn_name));

            result
        }
    };

    expanded.into()
}
```

Использование:

```rust
#[log]
fn factorial(n: u32) -> u32 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}

fn main() {
    let result = factorial(5);
    println!("Result: {result}");
}
```

Вывод:

```text
Entering: factorial
Entering: factorial
Entering: factorial
Entering: factorial
Entering: factorial
Entering: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Result: 120
```

Однако этот пример имеет важное ограничение: мы оборачиваем тело функции вручную и должны учитывать особенности исходной сигнатуры. Для production-quality attribute macro необходимо аккуратно работать с `async`, `unsafe`, `extern`, `const` и другими элементами сигнатуры.

Главный принцип здесь другой: **не нужно вручную реконструировать всю функцию**. Мы разбираем её через `syn`, изменяем только необходимую часть и генерируем новую версию через `quote!`.

---

## 42.9. Диагностика ошибок в `syn`

Хороший procedural macro должен сообщать об ошибках пользователю **в месте, где находится проблема**.

Например, если derive-макрос разрешён только для структур:

```rust
let error = syn::Error::new_spanned(
    &ast,
    "Describe can only be derived for structs",
);

return error.to_compile_error().into();
```

Но ещё лучше привязать ошибку непосредственно к проблемному полю:

```rust
use syn::spanned::Spanned;

if let Some(field) = fields.iter().find(|field| {
    field.ident.as_ref().is_some_and(|ident| ident == "password")
}) {
    let error = syn::Error::new(
        field.span(),
        "the `password` field is not allowed",
    );

    return error.to_compile_error().into();
}
```

Тогда компилятор сможет показать ошибку непосредственно возле соответствующего поля.

Это одна из сильных сторон procedural macros: токены сохраняют информацию о `Span`, поэтому диагностические сообщения могут быть привязаны к исходному коду пользователя. ([Docs.rs][5])

---

## 42.10. Продвинутые возможности `syn`

### `parse_quote!`

Если нужно создать небольшой фрагмент синтаксического дерева непосредственно в коде макроса, удобно использовать `parse_quote!`:

```rust
use syn::parse_quote;

let expr: syn::Expr = parse_quote! {
    self.value + 1
};

let function: syn::ItemFn = parse_quote! {
    fn example() -> i32 {
        42
    }
};
```

`parse_quote!` похож на `quote!`, но работает в обратную сторону: он **создаёт и парсит синтаксическое дерево**, причём тип результата определяется контекстом. ([Docs.rs][3])

Например:

```rust
use syn::{parse_quote, Type};

let ty: Type = parse_quote!(Option<String>);

println!("{ty:?}");
```

Это особенно удобно при создании новых AST-фрагментов.

---

### Обход дерева с `Visit`

Если нужно найти определённые элементы внутри большого синтаксического дерева, можно использовать visitor API:

```toml
[dependencies]
syn = { version = "2.0", features = ["full", "visit"] }
```

Пример:

```rust
use syn::visit::{self, Visit};

struct FieldCollector {
    fields: Vec<String>,
}

impl<'ast> Visit<'ast> for FieldCollector {
    fn visit_field(&mut self, field: &'ast syn::Field) {
        if let Some(ident) = &field.ident {
            self.fields.push(ident.to_string());
        }

        visit::visit_field(self, field);
    }
}
```

Теперь visitor можно запустить на синтаксическом дереве:

```rust
visitor.visit_item_struct(&item);
```

`Visit` особенно полезен, когда интересующий узел может находиться глубоко внутри другого AST. Для простого derive-макроса часто достаточно обычного `.iter()`, и использовать visitor там не требуется. ([Docs.rs][3])

---

## 42.11. `proc_macro2` и почему он появляется рядом с `quote`

Это важная деталь архитектуры procedural macros.

В procedural macro entry point используется:

```rust
proc_macro::TokenStream
```

Но `quote!` возвращает:

```rust
proc_macro2::TokenStream
```

Поэтому типичный код заканчивается:

```rust
let expanded = quote! {
    // generated Rust code
};

expanded.into()
```

Здесь `.into()` преобразует `proc_macro2::TokenStream` в `proc_macro::TokenStream`. ([Docs.rs][2])

Почему существует `proc_macro2`?

`proc_macro` тесно связан с самим компилятором и доступен в контексте procedural macro. `proc_macro2` предоставляет совместимый token API, который можно использовать также в обычных Rust-крейтах, тестах и `build.rs`.

Поэтому современная архитектура procedural macro обычно выглядит так:

```text
proc_macro
    │
    │ вход/выход макроса
    ▼
syn
    │
    │ анализ
    ▼
AST / данные
    │
    ▼
quote
    │
    ▼
proc_macro2::TokenStream
    │
    │ .into()
    ▼
proc_macro::TokenStream
```

Это также делает внутреннюю логику макроса гораздо удобнее для unit-тестирования.

---

## 42.12. Тестирование procedural macros

Для procedural macros особенно полезен `trybuild`.

Структура проекта:

```text
my_macros/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    ├── compile.rs
    └── ui/
        ├── basic.rs
        ├── invalid.rs
        └── invalid.stderr
```

В `Cargo.toml`:

```toml
[dev-dependencies]
trybuild = "1"
```

Тест:

```rust
#[test]
fn ui() {
    let tests = trybuild::TestCases::new();

    tests.pass("tests/ui/basic.rs");
    tests.compile_fail("tests/ui/invalid.rs");
}
```

`pass` проверяет, что пример **должен успешно скомпилироваться**:

```rust
tests.pass("tests/ui/basic.rs");
```

`compile_fail` проверяет противоположное:

```rust
tests.compile_fail("tests/ui/invalid.rs");
```

Например:

```rust
// tests/ui/invalid.rs

#[derive(Describe)]
enum Color {
    Red,
    Green,
    Blue,
}

fn main() {}
```

Если `Describe` разрешён только для структур, этот файл должен завершаться ошибкой.

Такой подход особенно ценен для procedural macros, потому что тестировать нужно не только успешную генерацию кода, но и **качество диагностических сообщений**.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Что происходит при ошибке парсинга?

Попробуйте передать `syn` заведомо некорректный Rust-код:

```rust
use syn::{parse_str, ItemStruct};

fn main() {
    let result = parse_str::<ItemStruct>(
        "struct Person { name: String "
    );

    match result {
        Ok(_) => println!("Успешно"),
        Err(error) => println!("Ошибка: {error}"),
    }
}
```

Здесь ошибка обнаруживается **на этапе синтаксического анализа**, ещё до генерации кода.

---

### Эксперимент 2: Что произойдёт без `#`?

Сравните:

```rust
quote! {
    struct #name;
}
```

и:

```rust
quote! {
    struct name;
}
```

Во втором случае `name` не является переменной Rust-программы. Это просто токен `name`.

То есть `quote!` не выполняет текстовую подстановку наподобие:

```text
"замени строку name на значение переменной"
```

Вместо этого `#name` означает **интерполяцию значения, реализующего `ToTokens`**.

---

### Эксперимент 3: Некорректный сгенерированный Rust-код

Попробуйте намеренно сгенерировать:

```rust
let tokens = quote! {
    impl MyTrait for Person {
        fn hello(&self) {
            let x = ;
        }
    }
};
```

`quote!` сам по себе не обязан проверять семантическую корректность всего сгенерированного Rust-кода.

Он формирует токены.

Затем эти токены передаются компилятору, и уже **компилятор Rust** обнаруживает синтаксическую или семантическую ошибку.

Это важное разделение ответственности:

```text
syn
  ↓
может проверить входной синтаксис

quote
  ↓
генерирует токены

rustc
  ↓
компилирует результат
```

---

## Практика

### Задание 1

Создайте derive-макрос `Getters`, который генерирует методы:

```rust
struct Person {
    name: String,
    age: u32,
}
```

в:

```rust
impl Person {
    fn name(&self) -> &String {
        &self.name
    }

    fn age(&self) -> &u32 {
        &self.age
    }
}
```

Обязательно используйте:

- `syn::Fields`;
- `quote!`;
- `format_ident!`;
- повторение `#(...)*`.

---

### Задание 2

Расширьте `Getters` поддержкой generics:

```rust
struct Person<T> {
    value: T,
}
```

Макрос должен использовать:

```rust
generics.split_for_impl()
```

и корректно генерировать `impl` для обобщённой структуры.

---

### Задание 3

Добавьте атрибут:

```rust
#[derive(Describe)]
struct User {
    #[describe(skip)]
    password: String,

    name: String,
}
```

Поле `password` не должно попадать в результат `describe()`.

Для разбора атрибута используйте `parse_nested_meta`.

---

### Задание 4

Создайте attribute-макрос:

```rust
#[measure]
fn calculate() {
    // ...
}
```

который измеряет время выполнения функции с помощью:

```rust
std::time::Instant
```

и выводит:

```text
calculate: 153 µs
```

Постарайтесь сохранить исходную сигнатуру функции.

---

### Задание 5

Создайте тесты `trybuild` для трёх ситуаций:

1. корректный `struct`;
2. попытка применить derive к `enum`;
3. структура с неподдерживаемым атрибутом.

Для каждого случая должен быть отдельный тест.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`syn`** — типизированное представление и парсер Rust-синтаксиса;
- **`DeriveInput`** — основной вход для derive-макросов;
- **`ItemFn`** — удобное представление функции для attribute-макросов;
- **`Fields`** — абстракция над named, unnamed и unit-полями;
- **`parse_macro_input!`** — стандартный способ разобрать вход procedural macro;
- **`parse_quote!`** — удобный способ создавать AST-фрагменты;
- **`Attribute::parse_args()`** — разбор произвольных аргументов атрибута;
- **`parse_nested_meta()`** — разбор стандартного формата аргументов атрибутов;
- **`quote!`** — генерация Rust-токенов;
- **`#value`** — интерполяция значения;
- **`#(#items)*`** — повторение;
- **`#(#items),*`** — повторение с разделителем;
- **`format_ident!`** — создание Rust-идентификаторов;
- **`proc_macro2::TokenStream`** — промежуточное представление, используемое экосистемой `syn`/`quote`;
- **`syn::Error`** — создание диагностических сообщений;
- **`Visit`** — обход синтаксического дерева;
- **`trybuild`** — тестирование успешных и ошибочных случаев.

**Самая важная идея:**

> `syn` и `quote` — не просто две библиотеки для procedural macros. Вместе они образуют практически стандартный конвейер работы с Rust-кодом: `syn` превращает токены в структурированные данные, мы анализируем эти данные обычным Rust-кодом, а `quote` превращает результат обратно в токены. При этом важно понимать границы ответственности: `syn` отвечает за разбор, наша программа — за анализ и принятие решений, `quote` — за генерацию токенов, а `rustc` — за окончательную проверку и компиляцию сгенерированного Rust-кода.

[1]: https://docs.rs/syn/latest/syn/struct.Attribute.html 'Attribute in syn - Rust'
[2]: https://docs.rs/crate/quote/latest 'quote 1.0.47 - Docs.rs'
[3]: https://docs.rs/syn/latest/syn/index.html 'syn - Rust'
[4]: https://docs.rs/crate/quote/latest/source/README.md 'quote 1.0.47 - Docs.rs'
[5]: https://docs.rs/crate/syn/2.0.114 'syn 2.0.114 - Docs.rs'
