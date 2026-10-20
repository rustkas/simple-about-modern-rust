# Глава 42. `syn` и `quote`

В предыдущей главе мы познакомились с процедурными макросами и увидели, как они работают на высоком уровне. Теперь пришло время погрузиться в детали двух библиотек, которые являются сердцем процедурных макросов: **`syn`** и **`quote`**.

`syn` превращает поток токенов в структурированное синтаксическое дерево (AST), а `quote` превращает синтаксическое дерево обратно в поток токенов. Вместе они образуют фундамент для создания мощных и безопасных процедурных макросов.

В этой главе мы изучим эти библиотеки в деталях и создадим полноценные процедурные макросы.

Все примеры этой главы используют **Rust Edition 2024**.

**Как запускать примеры.** Примеры с `syn` и `quote` запускаются как обычные Cargo-проекты, а не в Rust Playground. Для экспериментов удобно завести workspace из трёх packages:

```text
syn-lab/
├── Cargo.toml          # [workspace]
├── lab/                # обычный binary: syn и quote можно использовать и вне макросов
├── macros/             # proc-macro crate (proc-macro = true)
└── app/                # пользователь макросов
```

Зависимости (для `lab` и для `macros`):

```toml
[dependencies]
syn = { version = "2", features = ["full", "visit", "extra-traits"] }
quote = "1"
proc-macro2 = "1"
```

Эти features нужны не всем примерам, но с ними не придётся гадать, почему тот или иной тип отсутствует:

| Feature        | Что даёт                                                           |
| -------------- | ------------------------------------------------------------------ |
| `full`         | полное синтаксическое дерево Rust (`ItemFn`, `ItemImpl`, `Stmt`…)  |
| `visit`        | API обхода дерева (`syn::visit::Visit`)                            |
| `extra-traits` | реализации `Debug`, `Eq`, `Hash` для типов `syn` (по умолчанию нет) |

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

Интересно, что `syn` можно использовать и **вне** procedural macro — в обычной программе. Это удобный способ изучать его API:

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

Вывод:

```text
name = Person
fields = 2
```

Здесь мы не работаем со строкой `"struct Person { ... }"` напрямую. `syn` разбирает её и создаёт `ItemStruct`.

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

    TokenStream::new()
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

Не все типы `syn` доступны безусловно: часть функциональности включается feature-флагами (см. таблицу в начале главы). ([Docs.rs][3])

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

    TokenStream::new()
}
```

`parse_macro_input!` удобен тем, что при ошибке парсинга автоматически превращает ошибку `syn` в compile-time error для пользователя макроса. ([Docs.rs][3])

После разбора можно определить, что именно передал пользователь:

```rust
match &ast.data {
    syn::Data::Struct(data) => {
        // Это структура
        let count = data.fields.len();
    }

    syn::Data::Enum(data) => {
        // Это enum
        let count = data.variants.len();
    }

    syn::Data::Union(_) => {
        // Это union
    }
}
```

Обратите внимание: `println!` внутри процедурного макроса выводит текст **в консоль компилятора во время сборки**, а не во время работы программы. Для отладки это иногда полезно, но результат появится при `cargo build`, а не при `cargo run`.

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

    TokenStream::new()
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

Поэтому код вида:

```rust
field.ident.as_ref().unwrap()
```

**не является универсальным**. Он работает только для структур с именованными полями: у полей кортежной структуры `ident` равен `None`, и `unwrap()` приведёт к panic внутри процедурного макроса.

Если макрос действительно предназначен только для структур с именованными полями, это нужно явно проверить:

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

    TokenStream::new()
}
```

Теперь `unwrap()` безопасен: ветка с неименованными полями уже отсеяна. Пользователь получает понятную ошибку компиляции вместо panic внутри procedural macro.

---

## 42.4. Работа с атрибутами

В `syn` 2.x атрибуты обычно разбираются двумя способами:

- `parse_args()` — когда содержимое атрибута имеет собственную грамматику;
- `parse_nested_meta()` — для привычного синтаксиса вроде `#[my_trait(skip, rename = "...")]`. ([Docs.rs][1])

Допустим, мы хотим поддерживать такой интерфейс:

```rust
#[derive(MyTrait)]
#[my_trait(skip_all)]
struct Person {
    #[my_trait(skip)]
    password: String,

    #[my_trait(rename = "mail")]
    email: String,

    name: String,
}
```

Здесь три опции: `skip_all` на самой структуре, `skip` и `rename = "..."` на полях.

Важно, что все эти атрибуты нужно **объявить** в `proc_macro_derive`, иначе компилятор не разрешит их использовать:

```rust
#[proc_macro_derive(MyTrait, attributes(my_trait))]
```

### Полный пример

Макрос будет генерировать метод `names()`, возвращающий список имён полей, которые не пропущены, с учётом `rename`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields, LitStr};

#[proc_macro_derive(MyTrait, attributes(my_trait))]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    // 1. Атрибуты самой структуры.
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

    // 2. Поля.
    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => &fields.named,
            _ => return TokenStream::new(),
        },
        _ => return TokenStream::new(),
    };

    let mut kept = Vec::new();

    for field in fields {
        let mut skip = false;
        let mut rename: Option<String> = None;

        for attr in &field.attrs {
            if attr.path().is_ident("my_trait") {
                let result = attr.parse_nested_meta(|meta| {
                    if meta.path.is_ident("skip") {
                        skip = true;
                        Ok(())
                    } else if meta.path.is_ident("rename") {
                        let value: LitStr = meta.value()?.parse()?;
                        rename = Some(value.value());
                        Ok(())
                    } else {
                        Err(meta.error("unknown option"))
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

        let name = rename.unwrap_or_else(|| {
            field.ident.as_ref().unwrap().to_string()
        });

        kept.push(name);
    }

    let kept = if skip_all { Vec::new() } else { kept };

    let name = &ast.ident;

    quote! {
        impl #name {
            fn names() -> &'static [&'static str] {
                &[#(#kept),*]
            }
        }
    }
    .into()
}
```

Использование:

```rust
#[derive(MyTrait)]
struct Account {
    name: String,

    #[my_trait(skip)]
    password: String,

    #[my_trait(rename = "mail")]
    email: String,
}

fn main() {
    println!("{:?}", Account::names());
}
```

Результат:

```text
["name", "mail"]
```

Поле `password` пропущено, а `email` переименовано в `mail`.

Если пользователь ошибся в названии опции:

```rust
#[derive(MyTrait)]
#[my_trait(unknown)]
struct Account {
    name: String,
}
```

компилятор укажет именно на неправильный токен:

```text
error: expected `skip_all`
 --> src/main.rs:2:12
  |
2 | #[my_trait(unknown)]
  |            ^^^^^^^
```

Такую диагностику даёт `meta.error(...)`: он привязывает сообщение к месту в исходнике, где находится проблемная опция.

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

Вывод:

```text
impl Person { fn hello (& self) { println ! ("Hello!") ; } }
```

Обратите внимание на формат вывода. `TokenStream` при печати через `Display` вставляет пробелы между токенами (`fn hello (& self)`, `println ! (...)`). Это нормально: компилятору важны токены, а не их форматирование. Для красивого вывода сгенерированного кода используют `cargo expand` или `prettyplease`.

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

`format_ident!` принимает формат-строку, как `format!`: `format_ident!("get_{}", field)` создаст идентификатор `get_name`.

Это существенно важнее, чем кажется: обычная строка

```rust
let name = "Person";
```

не является Rust-идентификатором. Поэтому нельзя бездумно подставлять строки туда, где синтаксис Rust требует имя типа, поля или функции. Подстановка `#name` для `&str` даст не идентификатор, а **строковый литерал** `"Person"`.

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
        .map(|n| format_ident!("{}", n))
        .collect::<Vec<_>>();

    let tokens = quote! {
        #(
            println!("{}", stringify!(#fields));
        )*
    };

    println!("{tokens}");
}
```

Обратите внимание на `.map(|n| format_ident!("{}", n))`. Записать `.map(format_ident)` нельзя: `format_ident!` — **макрос**, а не функция, и передать его как значение не получится (`error[E0423]: expected value, found macro`).

Результат будет таким (с пробелами между токенами, как в предыдущем разделе):

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

Если внутри одного повторения встречаются **несколько** переменных, `quote!` обходит их параллельно (как `zip`). Именно так работает вставка пар «имя — значение» в разделе 42.7.

Важно: внутри повторения должна присутствовать переменная, по которой `quote!` может определить количество итераций. Именно поэтому конструкция вроде:

```rust
let t = quote! {
    #(
        println!("hello");
    )*
};
```

**не компилируется** — `quote!` не знает, сколько раз её повторять. Компилятор сообщит об этом не очень понятной ошибкой:

```text
error[E0277]: the trait bound `HasIterator<false>: CheckHasIterator<true>` is not satisfied
```

Если вы видите такое сообщение внутри `quote!`, значит, в `#( ... )*` нет ни одной итерируемой переменной. ([Docs.rs][4])

---

## 42.7. Практический пример: полноценный derive-макрос

Мы напишем derive-макрос для **собственного** трейта `Describe`:

```rust
trait Describe {
    fn describe(&self) -> String;
}
```

Почему не использовать уже существующий `ToString`? Потому что `std::string::ToString` связан с `Display` и уже имеет собственную семантику. Для учебного procedural macro отдельный трейт `Describe` намного понятнее.

**Где что лежит.** Из crate с `proc-macro = true` нельзя экспортировать ничего, кроме самих макросов. Поэтому трейт `Describe` объявляется в пользовательском crate (или в отдельной библиотеке), а в `proc-macro` crate находится только макрос с тем же именем. Так устроен и `serde`: трейт `Serialize` лежит в одном crate, а `#[derive(Serialize)]` — в `serde_derive`.

### Первая версия макроса

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

Она разбирает generics типа на три части, которые нужны, чтобы **синтаксически** корректно написать `impl`:

```rust
impl<T> Describe for Wrapper<T> where ... { ... }
//  ^^^                     ^^^   ^^^^^
//  impl_generics      ty_generics where_clause
```

### Использование

В пользовательском crate:

```rust
use macros::Describe;

pub trait Describe {
    fn describe(&self) -> String;
}

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

### Ловушка: `split_for_impl()` не добавляет ограничения на параметры

Попробуем применить макрос к обобщённой структуре:

```rust
#[derive(Describe)]
struct Wrapper<T> {
    value: T,
}
```

Этот код **не компилируется**:

```text
error[E0277]: `T` doesn't implement `Debug`
 --> src/main.rs:3:10
  |
3 | #[derive(Describe)]
  |          ^^^^^^^^ `T` cannot be formatted using `{:?}` because it doesn't implement `Debug`
```

Почему? `split_for_impl()` лишь аккуратно переносит в `impl` те параметры и ограничения, которые **уже были** у структуры. Наш сгенерированный код использует `{:?}` для значения поля типа `T`, а структура не требует `T: Debug`. Значит, в `impl` это ограничение нужно добавить самому.

Так поступают и стандартные derive-макросы: `#[derive(Debug)]` на `Wrapper<T>` генерирует `impl<T: Debug> Debug for Wrapper<T>`. Исправим наш макрос — добавим `Debug` ко всем параметрам типа:

```rust
let name = &ast.ident;

let mut generics = ast.generics.clone();

for param in generics.type_params_mut() {
    param.bounds.push(syn::parse_quote!(::core::fmt::Debug));
}

// ... далее без изменений:
let (impl_generics, ty_generics, where_clause) =
    generics.split_for_impl();
```

Теперь сгенерированный код выглядит так:

```rust
impl<T: ::core::fmt::Debug> Describe for Wrapper<T> {
    // ...
}
```

и вызов `Wrapper { value: 5 }.describe()` выводит:

```text
value=5
```

Обратите внимание на `::core::fmt::Debug`: мы пишем **абсолютный путь**, как советовали в разделе про hygiene в предыдущей главе. Пользователь мог бы определить собственный тип `Debug`, и короткое имя разрешилось бы не туда.

Урок для авторов derive-макросов: `split_for_impl()` решает вопрос **синтаксиса** generics, а вопрос **ограничений** (`T: Debug`, `T: Clone`, ...) решает автор макроса.

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

### Реализация

Как мы выяснили в предыдущей главе, самый очевидный способ — вставить тело функции в `let result = #fn_block;` — ломается при раннем `return`: `return` внутри вставленного блока выходит из внешней функции и пропускает код после блока. Поэтому «завершающее» действие выносим в `Drop`-guard:

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
            struct __ExitGuard(&'static str);

            impl Drop for __ExitGuard {
                fn drop(&mut self) {
                    println!("Exiting: {}", self.0);
                }
            }

            println!("Entering: {}", stringify!(#fn_name));
            let __guard = __ExitGuard(stringify!(#fn_name));

            #fn_block
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
Exiting: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Exiting: factorial
Result: 120
```

Пять `Entering` и пять `Exiting`: функция вызывается для `n = 5, 4, 3, 2, 1` — именно пять раз. Сначала входим в каждый уровень рекурсии, затем в обратном порядке выходим.

### Проверка раннего `return`

```rust
#[log]
fn early(x: i32) -> i32 {
    if x > 0 {
        return 1;
    }
    0
}

fn main() {
    early(5);
}
```

Вывод:

```text
Entering: early
Exiting: early
```

Сообщение о выходе печатается и при раннем `return`, потому что `Drop` срабатывает при любом способе выхода: нормальное завершение, `return`, оператор `?` и даже panic. Если бы мы использовали наивную схему `let result = #fn_block;`, строки `Exiting: early` в выводе **не было бы**.

Главный принцип здесь: **не нужно вручную реконструировать всю функцию**. Мы разбираем её через `syn`, меняем только необходимую часть и генерируем новую версию через `quote!`.

Однако даже этот вариант остаётся учебным. Для production-quality attribute macro необходимо аккуратно работать с `async fn` (guard сработает при уничтожении future, а не по завершении вычисления), `unsafe`, `extern`, `const` и другими элементами сигнатуры.

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
use proc_macro::TokenStream;
use syn::{parse_macro_input, spanned::Spanned, Data, DeriveInput, Fields};

#[proc_macro_derive(NoPassword)]
pub fn no_password(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);

    if let Data::Struct(data) = &ast.data {
        if let Fields::Named(fields) = &data.fields {
            let found = fields.named.iter().find(|field| {
                field
                    .ident
                    .as_ref()
                    .is_some_and(|ident| ident == "password")
            });

            if let Some(field) = found {
                return syn::Error::new(
                    field.span(),
                    "the `password` field is not allowed",
                )
                .to_compile_error()
                .into();
            }
        }
    }

    TokenStream::new()
}
```

Для

```rust
#[derive(NoPassword)]
struct User {
    name: String,
    password: String,
}
```

компилятор покажет ошибку непосредственно под полем:

```text
error: the `password` field is not allowed
 --> src/main.rs:4:26
  |
4 | struct User { name: String, password: String }
  |                             ^^^^^^^^
```

Это одна из сильных сторон procedural macros: токены сохраняют информацию о `Span`, поэтому диагностические сообщения могут быть привязаны к исходному коду пользователя. ([Docs.rs][5])

Обратите внимание на `ident == "password"`: тип `proc_macro2::Ident` реализует сравнение со строками, так что отдельно вызывать `.to_string()` не нужно.

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
use quote::quote;
use syn::{parse_quote, Type};

let ty: Type = parse_quote!(Option<String>);

println!("{}", quote!(#ty));
```

Вывод:

```text
Option < String >
```

Здесь мы печатаем тип через `quote!(#ty)`, а не через `{ty:?}`: типы `syn` по умолчанию **не реализуют `Debug`**. Печать через `{:?}` заработает, только если включить feature `extra-traits`:

```toml
syn = { version = "2", features = ["full", "extra-traits"] }
```

Эта feature заметно увеличивает время компиляции, поэтому в production-макросах её обычно включают только на время разработки.

### Обход дерева с `Visit`

Если нужно найти определённые элементы внутри большого синтаксического дерева, можно использовать visitor API:

```toml
[dependencies]
syn = { version = "2", features = ["full", "visit"] }
```

Полный пример:

```rust
use syn::visit::{self, Visit};
use syn::{parse_str, ItemStruct};

struct FieldCollector {
    fields: Vec<String>,
}

impl<'ast> Visit<'ast> for FieldCollector {
    fn visit_field(&mut self, field: &'ast syn::Field) {
        if let Some(ident) = &field.ident {
            self.fields.push(ident.to_string());
        }

        // Продолжаем обход вложенных узлов.
        visit::visit_field(self, field);
    }
}

fn main() {
    let item: ItemStruct = parse_str(
        "struct Person { name: String, age: u32 }"
    )
    .unwrap();

    let mut collector = FieldCollector { fields: Vec::new() };
    collector.visit_item_struct(&item);

    println!("{:?}", collector.fields);
}
```

Вывод:

```text
["name", "age"]
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

`proc_macro` тесно связан с самим компилятором и доступен только внутри procedural macro. `proc_macro2` предоставляет совместимый token API, который можно использовать также в обычных Rust-крейтах, тестах и `build.rs`. Именно поэтому примеры из этой главы с `syn::parse_str` и `quote!` запускаются в обычной программе.

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

`compile_fail` проверяет противоположное, причём сравнивает текст ошибки с эталоном в файле `.stderr`:

```rust
tests.compile_fail("tests/ui/invalid.rs");
```

Например:

```rust
// tests/ui/invalid.rs

use my_macros::Describe;

pub trait Describe {
    fn describe(&self) -> String;
}

#[derive(Describe)]
enum Color {
    Red,
    Green,
    Blue,
}

fn main() {}
```

Если `Describe` разрешён только для структур, этот файл должен завершаться ошибкой. Файл `invalid.stderr` с ожидаемым текстом ошибки создаётся автоматически при первом запуске: `trybuild` запишет результат в каталог `wip/`, и вы перенесёте его в `tests/ui/`, если он вас устраивает.

Обратите внимание, что каждый UI-файл — это самостоятельная программа: в нём нужно объявить трейт `Describe` и подключить макрос, как в обычном пользовательском crate.

Такой подход особенно ценен для procedural macros, потому что тестировать нужно не только успешную генерацию кода, но и **качество диагностических сообщений**.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Что происходит при ошибке парсинга?

Попробуйте передать `syn` заведомо некорректный Rust-код:

```rust
use syn::{parse_str, ItemStruct};

fn main() {
    let result = parse_str::<ItemStruct>(
        "struct Person { name String }"
    );

    match result {
        Ok(_) => println!("Успешно"),
        Err(error) => println!("Ошибка: {error}"),
    }
}
```

Вывод:

```text
Ошибка: expected `:`
```

Здесь ошибка обнаруживается **на этапе синтаксического анализа**: все токены корректны, но они не складываются в допустимое объявление структуры (пропущено двоеточие).

Теперь замените строку на `"struct Person { name: String "` (без закрывающей скобки). Сообщение будет другим:

```text
Ошибка: cannot parse string into token stream
```

Это ошибка более раннего этапа: строка не превращается даже в корректный поток токенов, потому что скобка не закрыта, и до работы `syn` дело не доходит. Различие полезно знать: ошибки «до токенов» и «после токенов» возникают в разных местах.

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

### Эксперимент 4: Обобщённая структура и `Debug`

Возьмите макрос `Describe` из раздела 42.7 в его **первой** версии (без добавления `Debug` к параметрам) и примените его к:

```rust
#[derive(Describe)]
struct Wrapper<T> {
    value: T,
}
```

Прочитайте сообщение компилятора и объясните, почему ошибка указывает на `derive`, хотя сам код `Wrapper` выглядит корректным. Затем добавьте ограничения на параметры типа и убедитесь, что ошибка исчезла.

---

### Эксперимент 5: Повторение без переменной

Напишите:

```rust
let t = quote! {
    #(
        println!("hello");
    )*
};
```

и изучите ошибку `HasIterator<false>: CheckHasIterator<true>`. Что нужно изменить, чтобы `quote!` знал, сколько раз повторять фрагмент?

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

Подумайте: нужны ли здесь дополнительные ограничения на `T`, как в `Describe`? (Подсказка: геттер возвращает ссылку и ничего не форматирует.)

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

Не забудьте объявить атрибут: `#[proc_macro_derive(Describe, attributes(describe))]`. Для разбора используйте `parse_nested_meta` по образцу раздела 42.4.

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

Постарайтесь сохранить исходную сигнатуру функции. Время должно печататься и при раннем `return`: используйте `Drop`-guard, как в `log`.

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

- **`syn`** — типизированное представление и парсер Rust-синтаксиса; его можно использовать и вне procedural macro;
- **features `syn`** (`full`, `visit`, `extra-traits`) определяют, какие типы и API доступны;
- **`DeriveInput`** — основной вход для derive-макросов;
- **`ItemFn`** — удобное представление функции для attribute-макросов;
- **`Fields`** — абстракция над named, unnamed и unit-полями; `field.ident.unwrap()` безопасен только после проверки на именованные поля;
- **`parse_macro_input!`** — стандартный способ разобрать вход procedural macro;
- **`parse_quote!`** — удобный способ создавать AST-фрагменты;
- **`Attribute::parse_args()`** — разбор произвольных аргументов атрибута;
- **`parse_nested_meta()`** — разбор стандартного формата аргументов атрибутов (`skip`, `rename = "..."`);
- **`quote!`** — генерация Rust-токенов;
- **`#value`** — интерполяция значения;
- **`#(#items)*`** — повторение; внутри обязательно должна быть итерируемая переменная;
- **`#(#items),*`** — повторение с разделителем;
- **`format_ident!`** — создание Rust-идентификаторов (это макрос, а не функция);
- **`split_for_impl()`** решает вопрос синтаксиса generics, а ограничения на параметры (`T: Debug`) нужно добавлять самому;
- **`proc_macro2::TokenStream`** — промежуточное представление, используемое экосистемой `syn`/`quote`;
- **`syn::Error`** — создание диагностических сообщений, привязанных к `Span`;
- **`Visit`** — обход синтаксического дерева;
- **`Drop`-guard** — надёжный способ выполнять «завершающие» действия в сгенерированной функции при любом способе выхода;
- **`trybuild`** — тестирование успешных и ошибочных случаев.

**Самая важная идея:**

> `syn` и `quote` — не просто две библиотеки для procedural macros. Вместе они образуют практически стандартный конвейер работы с Rust-кодом: `syn` превращает токены в структурированные данные, мы анализируем эти данные обычным Rust-кодом, а `quote` превращает результат обратно в токены. При этом важно понимать границы ответственности: `syn` отвечает за разбор, наша программа — за анализ и принятие решений (включая ограничения на generics и корректность при раннем `return`), `quote` — за генерацию токенов, а `rustc` — за окончательную проверку и компиляцию сгенерированного Rust-кода.

[1]: https://docs.rs/syn/latest/syn/struct.Attribute.html 'Attribute in syn - Rust'
[2]: https://docs.rs/crate/quote/latest 'quote 1.0.47 - Docs.rs'
[3]: https://docs.rs/syn/latest/syn/index.html 'syn - Rust'
[4]: https://docs.rs/crate/quote/latest/source/README.md 'quote 1.0.47 - Docs.rs'
[5]: https://docs.rs/crate/syn/2.0.114 'syn 2.0.114 - Docs.rs'

Присылайте следующую главу. В этой серии проверка на `rustc`/`cargo` нашла четыре ошибки, которые чтение глазами пропускало: `.map(format_ident)`, `{ty:?}` без `extra-traits`, `Describe` для `Wrapper<T>` и неверное число строк вывода у `factorial(5)`.
