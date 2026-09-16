# Часть IX. Modules, Crates и архитектура

# Глава 35. Modules

До этой главы мы писали все наши программы либо в одном файле `main.rs`, либо в Rust Playground. Это удобно для экспериментов, но в реальных проектах код быстро разрастается. Чтобы управлять сложностью, нужно уметь организовывать код в логические блоки — модули.

Модули в Rust — это система организации кода, которая позволяет:

- Группировать связанные функции, структуры и трейты.
- Контролировать видимость (что доступно снаружи).
- Скрывать детали реализации.
- Управлять пространствами имён.

В этой главе мы научимся создавать модули, управлять видимостью и организовывать код в проектах.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 35.1. Что такое модуль?

**Модуль (`mod`)** — это именованная область внутри дерева модулей crate. Он позволяет организовать код, создать границы видимости и скрыть детали реализации.

Модуль может содержать:

- функции;
- структуры и перечисления;
- трейты;
- константы и `static`;
- другие модули;
- реализации (`impl`);
- объявления типов;
- другие элементы Rust.

Простейший модуль можно определить непосредственно внутри файла:

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    fn multiply(a: i32, b: i32) -> i32 {
        a * b
    }
}

fn main() {
    let sum = math::add(5, 3);

    println!("5 + 3 = {sum}");

    // multiply() приватна для внешнего кода:
    // let result = math::multiply(2, 3);
}
```

Здесь `math` — модуль, а `add` и `multiply` — элементы этого модуля.

`add` объявлена как `pub`, поэтому внешний по отношению к `math` код может обратиться к ней через `math::add`.

`multiply` не имеет `pub`, поэтому код родительского модуля не может обратиться к ней напрямую.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+math+%7B%0A++++pub+fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++++++a+%2B+b%0A++++%7D%0A%0A++++fn+multiply%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++++++a+%2A+b%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+sum+%3D+math%3A%3Aadd%285%2C+3%29%3B%0A++++println%21%28%22%7Bsum%7D%22%29%3B%0A%7D)

### Приватность — это не просто «видно или не видно»

В Rust элемент без модификатора `pub` является приватным. Но важно понимать направление доступа.

Например:

```rust
mod parent {
    fn private_function() {
        println!("private");
    }

    mod child {
        pub fn call_parent() {
            // Потомок может обратиться к приватному
            // элементу родительского модуля.
            super::private_function();
        }
    }

    pub fn call_child() {
        child::call_parent();
    }
}

fn main() {
    parent::call_child();

    // Нельзя:
    // parent::private_function();
}
```

То есть приватность в Rust не означает «видно только непосредственно внутри этого модуля».

**Правило можно сформулировать так:**

> Приватный элемент доступен внутри модуля, в котором он объявлен, и его потомкам, но не доступен из родительского модуля через обычный путь.

Это важное отличие от модели «private = только этот класс/файл».

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+parent+%7B%0A++++fn+private_function%28%29+%7B%0A++++++++println%21%28%22private%22%29%3B%0A++++%7D%0A%0A++++mod+child+%7B%0A++++++++pub+fn+call_parent%28%29+%7B%0A++++++++++++super%3A%3Aprivate_function%28%29%3B%0A++++++++%7D%0A++++%7D%0A%0A++++pub+fn+call_child%28%29+%7B%0A++++++++child%3A%3Acall_parent%28%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++parent%3A%3Acall_child%28%29%3B%0A%7D)

**Ключевой принцип Rust:**

> Элементы приватны по умолчанию. Публичность нужно явно предоставить с помощью `pub` или одного из scoped-вариантов `pub(...)`.

---

## 35.2. Дерево модулей

Модули образуют **дерево модулей**.

Корнем этого дерева является **crate root** — корневой модуль crate.

Для бинарного приложения обычно crate root находится в:

```text
src/main.rs
```

Для библиотеки:

```text
src/lib.rs
```

Содержимое crate root образует корень дерева модулей. ([Rust Documentation][1])

Например:

```rust
mod math {
    pub mod geometry {
        pub fn area() {
            println!("geometry::area");
        }
    }

    pub mod algebra {
        pub fn add(a: i32, b: i32) -> i32 {
            a + b
        }
    }
}

mod utils {
    pub fn print_version() {
        println!("version 1");
    }
}

fn main() {
    math::geometry::area();

    let result = math::algebra::add(2, 3);
    println!("result = {result}");

    utils::print_version();
}
```

Дерево модулей здесь выглядит так:

```text
crate
├── math
│   ├── geometry
│   │   └── area
│   └── algebra
│       └── add
├── utils
│   └── print_version
└── main
```

Важное отличие:

```text
crate
└── math
    └── geometry
```

— это **дерево модулей**, а не обязательно дерево файлов.

Модули могут находиться:

1. непосредственно внутри другого модуля;
2. в отдельных файлах;
3. в комбинации этих вариантов.

Поэтому файл и модуль — не одно и то же.

---

## 35.3. Видимость: `pub` и приватность

В Rust существует несколько вариантов видимости:

| Объявление            | Видимость                                          |
| --------------------- | -------------------------------------------------- |
| `fn foo()`            | приватная                                          |
| `pub fn foo()`        | публичная                                          |
| `pub(crate) fn foo()` | в пределах текущего crate                          |
| `pub(super) fn foo()` | в родительском модуле и его доступном пространстве |
| `pub(self) fn foo()`  | текущий модуль; эквивалентно обычной приватности   |
| `pub(in crate::path)` | указанная область видимости                        |

На практике чаще всего используются:

```rust
pub
```

и

```rust
pub(crate)
```

### `pub`

`pub` предоставляет элемент внешним модулям, включая другие crate, если весь путь к элементу также доступен.

```rust
mod api {
    pub fn hello() {
        println!("Hello!");
    }
}

fn main() {
    api::hello();
}
```

### `pub(crate)`

`pub(crate)` делает элемент доступным в пределах текущего crate, но не экспортирует его как публичный API для других crate.

Это особенно полезно для внутреннего API библиотеки:

```rust
mod internal {
    pub(crate) fn helper() {
        println!("internal helper");
    }
}

fn main() {
    internal::helper();
}
```

Если этот код находится в библиотеке, другой crate не сможет вызвать:

```rust
// my_library::internal::helper();
```

### `pub(super)`

`pub(super)` ограничивает видимость родительским модулем:

```rust
mod parent {
    mod child {
        pub(super) fn helper() {
            println!("helper");
        }
    }

    pub fn run() {
        child::helper();
    }
}
```

### `pub(in ...)`

### `pub(in ...)`

Можно указать конкретную область видимости:

```rust
mod application {
    mod internal {
        pub(in crate::application) fn helper() {
            println!("helper");
        }
    }

    pub fn run() {
        internal::helper();
    }
}

fn main() {
    application::run();
}
```

`helper` доступна внутри `crate::application`, но не всему crate.

В Edition 2024 scoped visibility использует пути, начинающиеся с `crate`, `self` или `super`.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20application%20%7B%0A%20%20%20%20mod%20internal%20%7B%0A%20%20%20%20%20%20%20%20pub%28in%20crate%3A%3Aapplication%29%20fn%20helper%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22helper%22%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20run%28%29%20%7B%0A%20%20%20%20%20%20%20%20internal%3A%3Ahelper%28%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20application%3A%3Arun%28%29%3B%0A%7D)

### Важное замечание о `pub`

`pub` не «пробивает» приватность родительских модулей.

Например:

```rust
mod private_module {
    pub fn hello() {
        println!("Hello");
    }
}

fn main() {
    // Ошибка:
    // private_module::hello();
}
```

Хотя `hello` объявлена как `pub`, сам модуль `private_module` остаётся приватным.

Чтобы функция стала частью внешнего API, нужно открыть весь необходимый путь:

```rust
pub mod public_module {
    pub fn hello() {
        println!("Hello");
    }
}

fn main() {
    public_module::hello();
}
```

Таким образом, публичность элемента и публичность пути к нему — связанные, но разные вещи.

---

## 35.4. `use` — импорт в область видимости

`use` создаёт локальное имя для элемента, доступного по указанному пути. Это позволяет не повторять длинные пути. ([Rust Documentation][3])

```rust
mod math {
    pub mod geometry {
        pub fn area() {
            println!("Calculating area");
        }
    }
}

use math::geometry::area;

fn main() {
    area();
}
```

Без `use` пришлось бы писать:

```rust
math::geometry::area();
```

### Импорт модуля

Можно импортировать сам модуль:

```rust
use math::geometry;

fn main() {
    geometry::area();
}
```

### Переименование с помощью `as`

Если имя конфликтует с другим именем, используется `as`:

```rust
mod first {
    pub fn print() {
        println!("first");
    }
}

mod second {
    pub fn print() {
        println!("second");
    }
}

use first::print as print_first;
use second::print as print_second;

fn main() {
    print_first();
    print_second();
}
```

### Группировка импортов

Несколько элементов с одним префиксом можно импортировать группой:

```rust
use std::collections::{HashMap, HashSet};
```

То же самое работает для собственных модулей:

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    pub fn sub(a: i32, b: i32) -> i32 {
        a - b
    }
}

use math::{add, sub};

fn main() {
    println!("{}", add(10, 3));
    println!("{}", sub(10, 3));
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+math+%7B%0A++++pub+fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B+a+%2B+b+%7D%0A++++pub+fn+sub%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B+a+-+b+%7D%0A%7D%0A%0Ause+math%3A%3A%7Badd%2C+sub%7D%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+add%2810%2C+3%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+sub%2810%2C+3%29%29%3B%0A%7D)

**Практическое правило:** используйте `use`, когда он делает код понятнее. Не стоит автоматически импортировать всё подряд через `*`.

---

## 35.5. `self`, `super`, `crate` в путях

Rust предоставляет специальные префиксы для навигации по дереву модулей:

| Путь      | Значение              |
| --------- | --------------------- |
| `self::`  | текущий модуль        |
| `super::` | родительский модуль   |
| `crate::` | корень текущего crate |

Например:

```rust
const ROOT_VALUE: i32 = 100;

mod parent {
    pub const PARENT_VALUE: i32 = 200;

    mod child {
        pub fn show() {
            println!("crate = {}", crate::ROOT_VALUE);
            println!("super = {}", super::PARENT_VALUE);
        }

        pub fn show_self() {
            self::local();
        }

        fn local() {
            println!("self = current module");
        }
    }

    pub fn run() {
        child::show();
        child::show_self();
    }
}

fn main() {
    parent::run();
}
```

Здесь:

```rust
crate::ROOT_VALUE
```

начинается от корня crate.

```rust
super::PARENT_VALUE
```

поднимается на один уровень вверх.

```rust
self::local()
```

обращается к элементу текущего модуля.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=const+ROOT_VALUE%3A+i32+%3D+100%3B%0A%0Amod+parent+%7B%0A++++pub+const+PARENT_VALUE%3A+i32+%3D+200%3B%0A%0A++++mod+child+%7B%0A++++++++pub+fn+show%28%29+%7B%0A++++++++++++println%21%28%22crate+%3D+%7B%7D%22%2C+crate%3A%3AROOT_VALUE%29%3B%0A++++++++++++println%21%28%22super+%3D+%7B%7D%22%2C+super%3A%3APARENT_VALUE%29%3B%0A++++++++%7D%0A%0A++++++++pub+fn+show_self%28%29+%7B%0A++++++++++++self%3A%3Alocal%28%29%3B%0A++++++++%7D%0A%0A++++++++fn+local%28%29+%7B%0A++++++++++++println%21%28%22self+%3D+current+module%22%29%3B%0A++++++++%7D%0A++++%7D%0A%0A++++pub+fn+run%28%29+%7B%0A++++++++child%3A%3Ashow%28%29%3B%0A++++++++child%3A%3Ashow_self%28%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++parent%3A%3Arun%28%29%3B%0A%7D)

### Почему часто полезен `crate::`

В большом проекте путь:

```rust
crate::database::postgres::connect()
```

однозначно начинается от корня текущего crate.

Это особенно удобно внутри глубоко вложенных модулей, где относительный путь вроде:

```rust
super::super::database
```

становится трудным для понимания.

---

## 35.6. Вложенные модули

Модули можно вкладывать друг в друга:

```rust
mod database {
    pub mod postgres {
        pub fn connect() {
            println!("Connecting to PostgreSQL...");
        }
    }

    pub mod mysql {
        pub fn connect() {
            println!("Connecting to MySQL...");
        }
    }
}

fn main() {
    database::postgres::connect();
    database::mysql::connect();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20database%20%7B%0A%20%20%20%20pub%20mod%20postgres%20%7B%0A%20%20%20%20%20%20%20%20pub%20fn%20connect%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22Connecting%20to%20PostgreSQL...%22%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20mod%20mysql%20%7B%0A%20%20%20%20%20%20%20%20pub%20fn%20connect%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22Connecting%20to%20MySQL...%22%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20database%3A%3Apostgres%3A%3Aconnect%28%29%3B%0A%20%20%20%20database%3A%3Amysql%3A%3Aconnect%28%29%3B%0A%7D)

---

## 35.7. Модули в отдельных файлах

В реальном проекте не обязательно помещать весь модуль внутрь `main.rs` или `lib.rs`.

Например, можно создать:

```text
src/
├── main.rs
├── database.rs
└── database/
    ├── postgres.rs
    └── mysql.rs
```

В `main.rs` объявляем модуль:

```rust
mod database;

fn main() {
    database::postgres::connect();
    database::mysql::connect();
}
```

Файл `database.rs` содержит определение модуля:

```rust
pub mod postgres;
pub mod mysql;
```

Файл `database/postgres.rs`:

```rust
pub fn connect() {
    println!("Connecting to PostgreSQL...");
}
```

Файл `database/mysql.rs`:

```rust
pub fn connect() {
    println!("Connecting to MySQL...");
}
```

Получается такое дерево:

```text
crate
└── database
    ├── postgres
    │   └── connect
    └── mysql
        └── connect
```

### Почему здесь нет `mod.rs`?

В современном Rust для модуля `database` можно использовать:

```text
database.rs
```

вместо:

```text
database/mod.rs
```

Оба варианта поддерживаются, но для новых проектов обычно удобнее использовать форму:

```text
database.rs
database/
├── postgres.rs
└── mysql.rs
```

Это делает структуру проекта проще для чтения.

Схема:

```text
src/
├── main.rs
├── database.rs
└── database/
    ├── postgres.rs
    └── mysql.rs
```

соответствует объявлению:

```rust
mod database;
```

в `main.rs` и объявлениям:

```rust
pub mod postgres;
pub mod mysql;
```

в `database.rs`.

Rust также поддерживает старую форму:

```text
src/
├── main.rs
└── database/
    ├── mod.rs
    ├── postgres.rs
    └── mysql.rs
```

Но **`mod.rs` не является обязательным и не должен рассматриваться как современный стандарт организации модулей**. ([Rust Documentation][1])

### Библиотека и `lib.rs`

Если проект является библиотекой, crate root обычно:

```text
src/lib.rs
```

Например:

```text
src/
├── lib.rs
└── database.rs
```

`lib.rs`:

```rust
pub mod database;
```

`database.rs`:

```rust
pub fn connect() {
    println!("Connected");
}
```

Тогда другой crate сможет использовать библиотеку через её публичный API:

```rust
use my_library::database::connect;

fn main() {
    connect();
}
```

Здесь особенно хорошо видно назначение модулей:

- `lib.rs` определяет верхний уровень публичного API;
- `database.rs` организует внутреннюю реализацию;
- `pub` определяет, что именно разрешено использовать клиентскому коду.

---

## 35.8. Переэкспорт (`pub use`)

`pub use` — это не просто способ сократить длинный путь. Это инструмент **проектирования публичного API библиотеки**.

Предположим, реализация находится здесь:

```rust
mod database {
    pub mod postgres {
        pub fn connect() {
            println!("Connecting to PostgreSQL...");
        }
    }
}
```

Без переэкспорта пользователю библиотеки пришлось бы знать внутреннюю структуру:

```rust
database::postgres::connect();
```

Можно предоставить более удобный API:

```rust
mod database {
    pub mod postgres {
        pub fn connect() {
            println!("Connecting to PostgreSQL...");
        }
    }

    pub use postgres::connect;
}

fn main() {
    database::connect();
}
```

Теперь внешнему коду не нужно знать, что реализация находится именно в `postgres`.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+database+%7B%0A++++pub+mod+postgres+%7B%0A++++++++pub+fn+connect%28%29+%7B%0A++++++++++++println%21%28%22Connecting+to+PostgreSQL...%22%29%3B%0A++++++++%7D%0A++++%7D%0A%0A++++pub+use+postgres%3A%3Aconnect%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++database%3A%3Aconnect%28%29%3B%0A%7D)

Можно переименовать переэкспорт:

```rust
pub use postgres::connect as connect_postgres;
```

Тогда API будет:

```rust
database::connect_postgres();
```

### Переэкспорт на уровне crate

Это особенно полезно в библиотеке:

```rust
mod database {
    pub mod postgres {
        pub struct Connection;
    }
}

pub use database::postgres::Connection;
```

Теперь пользователь библиотеки может написать:

```rust
use my_library::Connection;
```

вместо:

```rust
use my_library::database::postgres::Connection;
```

Это позволяет скрыть внутреннюю структуру модулей.

**Хорошая архитектурная идея:**

> Внутренняя структура модулей может быть сложной, а публичный API библиотеки — простым.

`pub use` позволяет создать такой API.

---

## 35.9. Границы модулей

Модули — это границы, которые определяют, что видно снаружи:

```rust
mod module_a {
    pub struct PublicStruct {
        pub field: i32, // Публичное поле
        secret: i32,    // Приватное поле
    }

    impl PublicStruct {
        pub fn new(value: i32) -> Self {
            PublicStruct {
                field: value,
                secret: value * 2,
            }
        }

        pub fn get_secret(&self) -> i32 {
            self.secret
        }
    }
}

fn main() {
    let s = module_a::PublicStruct::new(10);
    println!("field: {}", s.field);

    // ❌ Ошибка! secret приватное поле
    // println!("secret: {}", s.secret);

    // Но можно через метод
    println!("secret: {}", s.get_secret());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20module_a%20%7B%0A%20%20%20%20pub%20struct%20PublicStruct%20%7B%0A%20%20%20%20%20%20%20%20pub%20field%3A%20i32%2C%0A%20%20%20%20%20%20%20%20secret%3A%20i32%2C%0A%20%20%20%20%7D%0A%0A%20%20%20%20impl%20PublicStruct%20%7B%0A%20%20%20%20%20%20%20%20pub%20fn%20new%28value%3A%20i32%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20PublicStruct%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20field%3A%20value%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20secret%3A%20value%20*%202%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20pub%20fn%20get_secret%28%26self%29%20-%3E%20i32%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.secret%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20s%20%3D%20module_a%3A%3APublicStruct%3A%3Anew%2810%29%3B%0A%20%20%20%20println%21%28%22field%3A%20%7B%7D%22%2C%20s.field%29%3B%0A%20%20%20%20println%21%28%22secret%3A%20%7B%7D%22%2C%20s.get_secret%28%29%29%3B%0A%7D)

---

## 35.10. Практика организации кода

Модульная архитектура нужна не для того, чтобы разбить большой файл на несколько маленьких файлов. Главная задача — создать **понятные границы ответственности и доступа**.

### Практические правила

**1. Организуйте модули вокруг ответственности.**

Например:

```text
crate
├── http
├── database
├── authentication
├── configuration
└── domain
```

лучше отражает архитектуру приложения, чем набор модулей:

```text
functions1
functions2
helpers
misc
```

**2. Не делайте всё `pub`.**

Если функция используется только внутри реализации, оставьте её приватной:

```rust
fn validate_token() {
    // internal implementation
}
```

Чем меньше публичный API, тем меньше связей между частями программы.

**3. Используйте `pub(crate)` для внутреннего API crate.**

Если элемент нужен нескольким модулям, но не должен становиться частью API библиотеки:

```rust
pub(crate) fn internal_helper() {
    // ...
}
```

**4. Используйте `pub use` для формирования удобного API.**

Пусть внутренняя структура:

```text
database
└── postgres
    └── Connection
```

не заставляет пользователя библиотеки писать длинный путь:

```rust
database::postgres::Connection
```

если концептуально библиотека хочет предоставить:

```rust
Connection
```

**5. Не привязывайте архитектуру к `mod.rs`.**

Для новых проектов обычно достаточно:

```text
foo.rs
foo/
├── bar.rs
└── baz.rs
```

а не:

```text
foo/
├── mod.rs
├── bar.rs
└── baz.rs
```

Обе формы поддерживаются, но первая обычно проще.

**6. Не путайте физическую структуру файлов с логической структурой модулей.**

Файл — способ организовать исходный код.

Модуль — часть языка Rust, которая создаёт пространство имён и границу видимости.

**7. Проектируйте публичный API отдельно от внутренней реализации.**

Особенно важно для библиотек:

```text
Public API
    ↓
pub use
    ↓
internal modules
    ↓
implementation details
```

Пользователь библиотеки должен зависеть от стабильного публичного API, а не от ваших внутренних деталей.

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: Попытка использовать приватный элемент

```rust
mod secret {
    fn hidden() {
        println!("hidden");
    }
}

fn main() {
    secret::hidden();
}
```

Компилятор сообщит, что функция `hidden` является приватной.

Попробуйте добавить `pub`:

```rust
mod secret {
    pub fn hidden() {
        println!("hidden");
    }
}
```

Теперь код компилируется.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+secret+%7B%0A++++pub+fn+hidden%28%29+%7B%0A++++++++println%21%28%22hidden%22%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++secret%3A%3Ahidden%28%29%3B%0A%7D)

---

## Эксперимент 2: Неправильный путь

```rust
mod a {
    pub mod b {
        pub fn c() {
            println!("c");
        }
    }
}

fn main() {
    // Ошибка:
    // a::c();

    a::b::c();
}
```

Функция `c` находится внутри `b`, поэтому путь должен содержать весь путь до неё:

```text
a → b → c
```

то есть:

```rust
a::b::c();
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+a+%7B%0A++++pub+mod+b+%7B%0A++++++++pub+fn+c%28%29+%7B%0A++++++++++++println%21%28%22c%22%29%3B%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++a%3A%3Ab%3A%3Ac%28%29%3B%0A%7D)

---

## Эксперимент 3: `pub` не открывает приватный родительский модуль

Попробуйте:

```rust
mod internal {
    pub fn hello() {
        println!("Hello");
    }
}

fn main() {
    internal::hello();
}
```

Затем сделайте модуль публичным:

```rust
pub mod internal {
    pub fn hello() {
        println!("Hello");
    }
}
```

Разница показывает, что для доступа к элементу важна **вся цепочка пути**, а не только видимость последнего элемента.

---

## Эксперимент 4: `pub(crate)` и другой crate

Создайте библиотеку:

```rust
// lib.rs

pub(crate) fn internal_function() {
    println!("internal");
}

pub fn public_function() {
    println!("public");
}
```

Внутри этого crate:

```rust
internal_function();
```

работает.

Но другой crate не может написать:

```rust
my_library::internal_function();
```

потому что `pub(crate)` ограничивает видимость текущим crate.

Это одно из главных практических применений `pub(crate)`:

> `pub(crate)` — это внутренний API crate, а не публичный API библиотеки.

---

# Практика

### Задание 1

Создайте модуль `calculator` с функциями:

```text
add
sub
mul
div
```

Все четыре функции должны быть публичными.

Продемонстрируйте использование как через полный путь:

```rust
calculator::add(...)
```

так и через `use`.

---

### Задание 2

Создайте модуль:

```text
geometry
├── circle
└── rectangle
```

В каждом подмодуле реализуйте функцию `area`.

Используйте:

```rust
geometry::circle::area(...)
geometry::rectangle::area(...)
```

---

### Задание 3

Используйте `use` для импорта нескольких функций:

```rust
use geometry::{...};
```

Затем попробуйте:

- импортировать функцию;
- импортировать модуль;
- переименовать функцию через `as`;
- использовать группировку импортов.

---

### Задание 4

Создайте структуру `Person` в отдельном модуле.

Сделайте поля приватными:

```rust
struct Person {
    name: String,
    age: u32,
}
```

Добавьте публичный конструктор:

```rust
pub fn new(...)
```

и публичные методы для чтения данных.

Проверьте, что прямой доступ:

```rust
person.name
```

из внешнего модуля невозможен.

---

### Задание 5

Создайте функцию:

```rust
pub(crate) fn internal_operation()
```

и вызовите её из другого модуля того же crate.

Затем представьте, что этот код находится в библиотеке, и попробуйте вызвать функцию из другого crate.

Объясните, почему второй вариант не работает.

---

### Задание 6

Создайте внутреннюю структуру:

```text
database
└── postgres
    └── Connection
```

Затем с помощью:

```rust
pub use
```

сделайте `Connection` доступным на более высоком уровне:

```rust
use crate::Connection;
```

Цель задания — увидеть, как `pub use` позволяет отделить публичный API от внутренней структуры модулей.

---

### Задание 7

Создайте проект со следующей структурой:

```text
src/
├── main.rs
├── database.rs
└── database/
    ├── postgres.rs
    └── mysql.rs
```

Объявите:

```rust
mod database;
```

в `main.rs`.

В `database.rs` объявите два подмодуля:

```rust
pub mod postgres;
pub mod mysql;
```

Реализуйте в каждом `connect()`.

Затем вызовите:

```rust
database::postgres::connect();
database::mysql::connect();
```

---

# Главное из этой главы

После этой главы мы понимаем:

- **`mod`** — объявляет модуль и создаёт узел в дереве модулей.
- **Модули образуют дерево**, корнем которого является crate root.
- **Элементы приватны по умолчанию.**
- Приватный элемент доступен внутри своего модуля и его потомков, но не родительскому модулю.
- **`pub`** открывает элемент для внешнего доступа при условии, что сам путь к нему доступен.
- **`pub(crate)`** ограничивает доступ текущим crate.
- **`pub(super)`** ограничивает доступ родительским модулем.
- **`pub(in ...)`** позволяет указать конкретную область видимости.
- **`use`** создаёт удобное имя для существующего пути.
- **`as`** позволяет переименовать импорт.
- **`self`** означает текущий модуль.
- **`super`** означает родительский модуль.
- **`crate`** означает корень текущего crate.
- Модули могут быть определены **inline** или размещены в отдельных файлах.
- Современная файловая схема обычно использует `foo.rs` и каталог `foo/`; `mod.rs` остаётся допустимым, но не обязателен.
- **`pub use`** позволяет формировать удобный публичный API, скрывая внутреннюю структуру реализации.

**Самая важная идея:**

> Модули в Rust — это не просто способ разложить код по файлам. Это механизм проектирования границ программы: они определяют пространство имён, видимость и зависимость одного компонента от другого. Хорошая архитектура стремится сделать внутреннюю реализацию максимально закрытой, а публичный API — небольшим, понятным и стабильным.

[1]: https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html 'Control Scope and Privacy with Modules - The Rust Programming Language'
[2]: https://doc.rust-lang.org/reference/items.html?search=edition 'Items - The Rust Reference'
[3]: https://doc.rust-lang.org/stable/std/keyword.use.html 'use - Rust'
