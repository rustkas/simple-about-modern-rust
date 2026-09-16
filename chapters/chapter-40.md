# Часть X. Macros

# Глава 40. `macro_rules!`

Макросы в Rust — это мощный инструмент для **метапрограммирования**. Они позволяют писать код, который генерирует другой код. Это может быть полезно для устранения повторяющихся паттернов, создания DSL (domain-specific languages) и реализации функций, которые невозможно выразить через обычные функции или трейты.

В этой главе мы изучим декларативные макросы — самый простой и распространённый вид макросов в Rust (`macro_rules!`). Мы разберёмся, как они работают, как их писать и где они применяются.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 40.1. Зачем нужны макросы?

Макросы решают несколько важных задач:

1. **Устранение дублирования.** Макрос может генерировать повторяющийся код.

2. **Создание DSL.** Макрос может предоставить специальный удобный синтаксис поверх обычного Rust-кода.

3. **Работа с синтаксисом Rust.** `macro_rules!` может принимать не только значения, но и выражения, типы, паттерны, элементы, идентификаторы и другие фрагменты синтаксиса.

4. **Вариативные конструкции.** Макросы могут принимать переменное количество аргументов:

```rust
let numbers = vec![1, 2, 3, 4, 5];
```

5. **Генерация большого количества однотипного кода.** Например, макрос может сгенерировать несколько функций или реализаций трейта из одного компактного описания.

Важно понимать, **чего `macro_rules!` не делает**.

Декларативный макрос не является полноценной системой интроспекции исходного кода. Он сопоставляет входные токены с шаблонами (`matcher`) и заменяет совпавший шаблон на соответствующее расширение (`transcriber`). ([Rust Documentation][1])

Например:

```rust
macro_rules! answer {
    () => {
        42
    };
}

fn main() {
    let value = answer!();

    println!("{value}");
}
```

После расширения макроса компилятор фактически получает код, эквивалентный:

```rust
fn main() {
    let value = 42;

    println!("{value}");
}
```

То есть макрос **генерирует исходный Rust-код**, после чего этот код проходит обычные этапы компиляции, включая проверку типов и borrow checker.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+answer+%7B%0A++++%28%29+%3D%3E+%7B%0A++++++++42%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+answer%21%28%29%3B%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)

---

## 40.2. `macro_rules!` — декларативные макросы

**Декларативные макросы** определяются с помощью `macro_rules!`.

Основная идея проста:

> **Сопоставить входной синтаксис с одним из шаблонов и развернуть соответствующую ветку.**

Например:

```rust
macro_rules! answer {
    () => {
        42
    };
}

fn main() {
    let x = answer!();

    println!("The answer is: {x}");
}
```

Здесь:

```rust
() => { 42 };
```

означает:

- если макрос вызывается без аргументов;
- заменить вызов результатом `42`.

Вызов:

```rust
answer!()
```

становится:

```rust
42
```

Макрос может иметь несколько правил:

```rust
macro_rules! describe_number {
    ($value:literal) => {
        println!("Number: {}", $value);
    };

    ($value:expr, $name:expr) => {
        println!("{} = {}", $name, $value);
    };
}

fn main() {
    describe_number!(42);
    describe_number!(10 + 20, "result");
}
```

При вызове `describe_number!(42)` будет выбрано первое правило, а при вызове `describe_number!(10 + 20, "result")` — второе.

Порядок правил имеет значение: `macro_rules!` пытается сопоставить вызов с правилами макроса последовательно.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+describe_number+%7B%0A++++%28%24value%3Aliteral%29+%3D%3E+%7B%0A++++++++println%21%28%22Number%3A+%7B%7D%22%2C+%24value%29%3B%0A++++%7D%3B%0A%0A++++%28%24value%3Aexpr%2C+%24name%3Aexpr%29+%3D%3E+%7B%0A++++++++println%21%28%22%7B%7D+%3D+%7B%7D%22%2C+%24name%2C+%24value%29%3B%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++describe_number%21%2842%29%3B%0A++++describe_number%21%2810+%2B+20%2C+%22result%22%29%3B%0A%7D)

---

## 40.3. Макросы с параметрами

Макрос может захватывать части входного кода в **метапеременные**:

```rust
macro_rules! create_struct {
    ($name:ident, $field:ident: $type:ty) => {
        struct $name {
            $field: $type,
        }
    };
}

create_struct!(User, name: String);

fn main() {
    let user = User {
        name: String::from("Alice"),
    };

    println!("User name: {}", user.name);
}
```

Вызов:

```rust
create_struct!(User, name: String);
```

сопоставляется с:

```text
$name:ident
$field:ident
$type:ty
```

и генерирует:

```rust
struct User {
    name: String,
}
```

Здесь:

- `ident` означает идентификатор;
- `ty` означает тип.

Важно: макрос не проверяет, является ли `String` действительно корректным типом с точки зрения семантики программы. Он только захватывает соответствующий синтаксический фрагмент. Обычная проверка Rust произойдёт после расширения макроса.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+create_struct+%7B%0A++++%28%24name%3Aident%2C+%24field%3Aident%3A+%24type%3Aty%29+%3D%3E+%7B%0A++++++++struct+%24name+%7B%0A++++++++++++%24field%3A+%24type%2C%0A++++++++%7D%0A++++%7D%3B%0A%7D%0A%0Acreate_struct%21%28User%2C+name%3A+String%29%3B%0A%0Afn+main%28%29+%7B%0A++++let+user+%3D+User+%7B%0A++++++++name%3A+String%3A%3Afrom%28%22Alice%22%29%2C%0A++++%7D%3B%0A++++println%21%28%22User+name%3A+%7B%7D%22%2C+user.name%29%3B%0A%7D)

---

## 40.4. Фрагменты (Fragments) в макросах

`macro_rules!` не работает с произвольными строками. Метапеременная должна иметь **fragment specifier**, который определяет, какой синтаксический фрагмент Rust она может захватить. ([Rust Documentation][2])

Основные fragment specifiers:

| Фрагмент    | Что захватывает                    | Пример                      |
| ----------- | ---------------------------------- | --------------------------- |
| `ident`     | идентификатор                      | `foo`, `User`               |
| `expr`      | выражение                          | `5 + 3`, `foo()`            |
| `expr_2021` | выражение с правилами Edition 2021 | `foo()`                     |
| `ty`        | тип                                | `i32`, `String`             |
| `literal`   | литерал                            | `42`, `"hello"`             |
| `path`      | путь                               | `std::io::Read`             |
| `stmt`      | оператор                           | `let x = 5;`                |
| `pat`       | паттерн                            | `Some(x)`                   |
| `pat_param` | паттерн без top-level or-pattern   | `Some(x)`                   |
| `item`      | элемент Rust                       | `fn foo() {}`               |
| `block`     | блок выражения                     | `{ println!("Hi"); }`       |
| `meta`      | содержимое атрибута                | `derive(Debug)`             |
| `vis`       | модификатор видимости              | `pub`, `pub(crate)`         |
| `lifetime`  | время жизни                        | `'a`                        |
| `tt`        | token tree                         | произвольное дерево токенов |

### Особенность Edition 2024

В Edition 2024 fragment specifier `expr` стал принимать более широкий набор выражений, включая top-level `const` и `_` expressions.

Например:

```rust
macro_rules! show {
    ($value:expr) => {
        println!("{}", stringify!($value));
    };
}

fn main() {
    show!(const { 42 });
    show!(_);
}
```

Для макросов, которым требуется сохранить старое поведение, существует `expr_2021`. Как и другие edition-dependent правила `macro_rules!`, соответствующая edition определяется по месту определения макроса. ([Rust Documentation][3])

### Пример использования нескольких fragment specifiers

```rust
macro_rules! print_info {
    ($name:ident, $value:expr, $type:ty) => {
        println!(
            "{} (type {}) = {:?}",
            stringify!($name),
            stringify!($type),
            $value
        );
    };
}

fn main() {
    let x = 42;

    print_info!(x, x, i32);

    let message = String::from("Hello");

    print_info!(message, message, String);
}
```

`stringify!` здесь превращает захваченные токены в строку. При этом `$value:expr` остаётся настоящим Rust-выражением и вычисляется обычным образом.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+print_info+%7B%0A++++%28%24name%3Aident%2C+%24value%3Aexpr%2C+%24type%3Aty%29+%3D%3E+%7B%0A++++++++println%21%28%22%7B%7D+%28type+%7B%7D%29+%3D+%7B%3A%3F%7D%22%2C+stringify%21%28%24name%29%2C+stringify%21%28%24type%29%2C+%24value%29%3B%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+x+%3D+42%3B%0A++++print_info%21%28x%2C+x%2C+i32%29%3B%0A%0A++++let+message+%3D+String%3A%3Afrom%28%22Hello%22%29%3B%0A++++print_info%21%28message%2C+message%2C+String%29%3B%0A%7D)

---

## 40.5. Повторения (Repetition)

Одна из наиболее важных возможностей `macro_rules!` — обработка переменного количества фрагментов.

Синтаксис:

```text
$( ... )*
$( ... )+
$( ... )?
```

означает:

```
| Оператор | Значение                  |
| -------- | ------------------------- |
| `*`      | ноль или более повторений |
| `+`      | одно или более повторений |
| `?`      | ноль или одно повторение  |
```

Также можно указать разделитель:

```rust
$( $arg:expr ),*
```

означает:

> захватывать любое количество выражений, разделённых запятыми.

Например:

```rust
macro_rules! print_all {
    ($($arg:expr),*) => {
        $(
            println!("{}", $arg);
        )*
    };
}

fn main() {
    print_all!();
    print_all!(42);
    print_all!(1, 2, 3, 4, 5);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+print_all+%7B%0A++++%28%24%28%24arg%3Aexpr%29%2C%2A%29+%3D%3E+%7B%0A++++++++%24%28%0A++++++++++++println%21%28%22%7B%7D%22%2C+%24arg%29%3B%0A++++++++%29%2A%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++print_all%21%28%29%3B%0A++++print_all%21%2842%29%3B%0A++++print_all%21%281%2C+2%2C+3%2C+4%2C+5%29%3B%0A%7D)

### Trailing comma

Если нужно разрешить завершающую запятую, часто используют `?`:

```rust
macro_rules! print_all {
    ($($arg:expr),* $(,)?) => {
        $(
            println!("{}", $arg);
        )*
    };
}

fn main() {
    print_all!(1, 2, 3,);
}
```

Здесь:

```text
$(,)?
```

означает:

> после списка может присутствовать одна необязательная запятая.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+print_all+%7B%0A++++%28%24%28%24arg%3Aexpr%29%2C%2A+%28%2C%29%3F%29+%3D%3E+%7B%0A++++++++%24%28%0A++++++++++++println%21%28%22%7B%7D%22%2C+%24arg%29%3B%0A++++++++%29%2A%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++print_all%21%281%2C+2%2C+3%2C%29%3B%0A%7D)

Повторения могут быть вложенными. Это позволяет создавать макросы для более сложных структур, например таблиц, списков полей или пар `key => value`.

Пример вложенного повторения — упрощённый макрос для создания «матрицы» (вектора векторов):

```rust
macro_rules! matrix {
    ( $( $( $x:expr ),* );* $(;)? ) => {
        {
            let mut result = Vec::new();
            $(
                let mut row = Vec::new();
                $(
                    row.push($x);
                )*
                result.push(row);
            )*
            result
        }
    };
}

fn main() {
    let m = matrix![
        1, 2, 3;
        4, 5, 6;
        7, 8, 9;
    ];

    println!("{m:?}");
}
```

Здесь внешнее повторение `$( ... );*` захватывает строки, а внутреннее `$( $x:expr ),*` — элементы внутри каждой строки. После расширения компилятор получает обычный Rust-код с несколькими `push`.

Правила повторений определяют не только то, сколько раз фрагмент захватывается, но и то, сколько раз соответствующая часть expansion будет сгенерирована.

---

## 40.6. Пример: упрощённая реализация `vec!`

Макрос `vec!` хорошо демонстрирует сразу несколько возможностей `macro_rules!`.

Упрощённый вариант для списка элементов можно написать так:

```rust
macro_rules! make_vec {
    ($($element:expr),* $(,)?) => {{
        let mut result = Vec::new();

        $(
            result.push($element);
        )*

        result
    }};
}

fn main() {
    let values = make_vec![10, 20, 30];

    println!("{values:?}");
}
```

Здесь:

```rust
$($element:expr),*
```

захватывает:

```text
10, 20, 30
```

а затем:

```rust
$(
    result.push($element);
)*
```

генерирует:

```rust
result.push(10);
result.push(20);
result.push(30);
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+make_vec+%7B%0A++++%28%24%28%24element%3Aexpr%29%2C%2A+%28%2C%29%3F%29+%3D%3E+%7B%7B%0A++++++++let+mut+result+%3D+Vec%3A%3Anew%28%29%3B%0A%0A++++++++%24%28%0A++++++++++++result.push%28%24element%29%3B%0A++++++++%29%2A%0A%0A++++++++result%0A++++%7D%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+values+%3D+make_vec%21%5B10%2C+20%2C+30%5D%3B%0A++++println%21%28%22%7Bvalues%3A%3F%7D%22%2C+values%29%3B%0A%7D)

### Почему `vec![value; count]` — отдельный случай

У настоящего `vec!` есть ещё форма:

```rust
vec![value; count]
```

Например:

```rust
let values = vec![String::from("hello"); 3];
```

Здесь нельзя просто сделать:

```rust
for _ in 0..count {
    result.push(value);
}
```

потому что `value` может быть некопируемым значением, которое после первого `push` уже перемещено.

Кроме того, семантика стандартного `vec![value; count]` заключается не в том, чтобы заново вычислять выражение `value` на каждой итерации. Для этой формы требуется возможность клонирования элемента.

Поэтому учебная реализация должна либо явно ограничить тип:

```rust
macro_rules! repeat_vec {
    ($value:expr; $count:expr) => {{
        let value = $value;
        let mut result = Vec::with_capacity($count);

        for _ in 0..$count {
            result.push(value.clone());
        }

        result
    }};
}

fn main() {
    let values = repeat_vec![String::from("hello"); 3];

    println!("{values:?}");
}
```

либо честно назвать реализацию упрощённой.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+repeat_vec+%7B%0A++++%28%24value%3Aexpr%3B+%24count%3Aexpr%29+%3D%3E+%7B%7B%0A++++++++let+value+%3D+%24value%3B%0A++++++++let+mut+result+%3D+Vec%3A%3Awith_capacity%28%24count%29%3B%0A%0A++++++++for+_+in+0..%24count+%7B%0A++++++++++++result.push%28value.clone%28%29%29%3B%0A++++++++%7D%0A%0A++++++++result%0A++++%7D%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+values+%3D+repeat_vec%21%5BString%3A%3Afrom%28%22hello%22%29%3B+3%5D%3B%0A++++println%21%28%22%7Bvalues%3A%3F%7D%22%2C+values%29%3B%0A%7D)

Таким образом, `macro_rules!` позволяет воспроизвести **синтаксис** `vec!`, но полностью воспроизвести поведение стандартной библиотеки — уже отдельная задача.

---

## 40.7. Макросы в разных модулях

У `macro_rules!` есть важное правило видимости.

Макрос, объявленный внутри модуля, не следует автоматически считать обычным `pub`-элементом модуля. Для экспорта `macro_rules!` из crate используется `#[macro_export]`.

Например:

```rust
mod macros {
    #[macro_export]
    macro_rules! hello {
        () => {
            println!("Hello!");
        };
    }
}

fn main() {
    hello!();
}
```

Несмотря на то что макрос определён внутри `mod macros`, после `#[macro_export]` он экспортируется в **корень crate**:

```rust
hello!();
```

а не как:

```rust
macros::hello!();
```

Это важное отличие от обычных функций и типов.

### `$crate`

При создании библиотечных макросов часто нужно обратиться из expansion к элементам того crate, где макрос определён.

Для этого существует специальная метапеременная:

```rust
$crate
```

Полный пример:

```rust
// Предположим, этот код находится в библиотечном crate
pub fn helper() {
    println!("helper from the crate");
}

#[macro_export]
macro_rules! call_helper {
    () => {
        $crate::helper();
    };
}

// В другом месте (в том же или во внешнем crate) можно написать:
// call_helper!();
```

`$crate` всегда указывает на crate, в котором определён макрос. Благодаря этому макрос корректно обращается к собственным элементам даже тогда, когда вызывается из другого crate. Это особенно важно для публичных библиотечных макросов: без `$crate` пришлось бы жёстко прописывать имя crate, которое пользователь может изменить.

---

## 40.8. Hygiene — гигиена макросов

Утверждение «макросы полностью изолированы от окружающих имён» было бы слишком сильным.

`macro_rules!` использует **mixed-site hygiene**. В частности, локальные переменные и некоторые другие локальные имена разрешаются с учётом места определения макроса, тогда как многие внешние символы разрешаются в контексте вызова макроса.

Рассмотрим пример:

```rust
macro_rules! with_x {
    ($value:expr) => {{
        let x = $value;
        println!("macro x = {x}");
    }};
}

fn main() {
    let x = 10;

    with_x!(42);

    println!("outer x = {x}");
}
```

Результат:

```text
macro x = 42
outer x = 10
```

Переменная `x`, созданная внутри expansion, не становится обычной переменной `x` вызывающего кода.

Именно поэтому следующий код не работает так, как можно было бы ожидать:

```rust
macro_rules! define_x {
    () => {
        let x = 42;
    };
}

macro_rules! print_x {
    () => {
        println!("{x}");
    };
}

fn main() {
    define_x!();
    print_x!();
}
```

Переменная `x`, созданная одним расширением, не становится автоматически доступной другому макросу. Компилятор выдаст ошибку о неразрешённом имени `x`.

[Открыть пример с hygiene в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+with_x+%7B%0A++++%28%24value%3Aexpr%29+%3D%3E+%7B%7B%0A++++++++let+x+%3D+%24value%3B%0A++++++++println%21%28%22macro+x+%3D+%7Bx%7D%22%29%3B%0A++++%7D%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+x+%3D+10%3B%0A++++with_x%21%2842%29%3B%0A++++println%21%28%22outer+x+%3D+%7Bx%7D%22%29%3B%0A%7D)

### Практический вывод

Не следует рассчитывать на имена внутренних переменных макроса как на часть API.

Хороший макрос обычно:

- минимизирует количество внутренних имён;
- использует блок `{ ... }`, чтобы ограничить область видимости;
- явно принимает необходимые значения через metavariables;
- не рассчитывает на существование переменных в месте вызова.

---

## 40.9. Макро-расширение и отладка

При сложных макросах бывает трудно понять, какой код реально получился после expansion.

Для этого удобно использовать `cargo expand`.

Установка:

```bash
cargo install cargo-expand
```

После этого можно посмотреть расширение библиотеки:

```bash
cargo expand --lib
```

или бинарного приложения:

```bash
cargo expand --bin my_app
```

Если в проекте есть несколько targets, можно использовать соответствующий target:

```bash
cargo expand --example demo
cargo expand --test integration
```

Это особенно полезно при разработке больших `macro_rules!`, потому что позволяет увидеть не исходный шаблон макроса, а код, который после расширения анализирует обычный Rust-компилятор.

Например, если есть:

```rust
macro_rules! double {
    ($value:expr) => {
        $value * 2
    };
}

fn main() {
    let x = double!(21);
}
```

`cargo expand` покажет примерно следующий результат:

```rust
fn main() {
    let x = 21 * 2;
}
```

и позволит убедиться, что макрос действительно генерирует ожидаемое выражение.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `expr` и Edition 2024

Создайте макрос:

```rust
macro_rules! show {
    ($value:expr) => {
        println!("{}", stringify!($value));
    };
}

fn main() {
    show!(const { 42 });
    show!(_);
}
```

В Edition 2024 `expr` допускает эти формы выражений. Это одно из изменений fragment matching в Edition 2024. ([Rust Documentation][3])

Попробуйте заменить:

```rust
edition = "2024"
```

на более старую edition и сравнить поведение.

### Эксперимент 2: отсутствующее правило

Что произойдёт?

```rust
macro_rules! only_one {
    ($value:expr) => {
        println!("{value}", value = $value);
    };
}

fn main() {
    only_one!(10, 20);
}
```

Макрос не найдёт подходящего правила, и компиляция завершится ошибкой.

Это важно: `macro_rules!` не пытается «догадаться», что хотел сделать программист. Вход должен соответствовать одному из matcher'ов.

### Эксперимент 3: fragment specifier имеет значение

Сравните:

```rust
macro_rules! literal_only {
    ($value:literal) => {
        println!("{}", $value);
    };
}
```

и:

```rust
macro_rules! expression {
    ($value:expr) => {
        println!("{}", $value);
    };
}
```

Теперь попробуйте:

```rust
literal_only!(10 + 20);
expression!(10 + 20);
```

Первый вызов не соответствует `literal`, а второй соответствует `expr`.

### Эксперимент 4: hygiene

Попробуйте сделать так:

```rust
macro_rules! define {
    () => {
        let x = 42;
    };
}

macro_rules! use_x {
    () => {
        println!("{x}");
    };
}

fn main() {
    define!();
    use_x!();
}
```

Компилятор не позволит использовать `x` таким способом.

Это хороший практический эксперимент для понимания того, что локальное имя, созданное expansion одного макроса, не превращается автоматически в переменную, доступную другому макросу. ([Rust Documentation][1])

---

## Практика

### Задание 1

Напишите макрос `say_hello!`, который выводит:

```text
Hello, world!
```

### Задание 2

Напишите макрос `assert_eq_custom!`, который принимает два выражения:

```rust
assert_eq_custom!(left, right);
```

и паникует, если значения не равны.

Постарайтесь принимать именно `expr`, а не `literal`.

### Задание 3

Напишите макрос `map!`, создающий `HashMap`:

```rust
let values = map! {
    "name" => "Alice",
    "city" => "Bangkok",
};
```

Используйте повторение:

```text
$( ... ),*
```

и разрешите завершающую запятую.

### Задание 4

Напишите макрос:

```rust
repeat! {
    3 => println!("Hello");
}
```

который выполняет переданный оператор указанное количество раз.

### Задание 5

Создайте макрос с двумя правилами:

```rust
describe!(42);
describe!(42, "answer");
```

Первый вариант должен печатать только значение, второй — имя и значение.

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если макрос использует переменную, которой нет в месте его определения?

Попробуйте намеренно получить ошибку и объясните её с точки зрения hygiene.

### Задание 7

🔨 **Эксперимент с compiler expansion.**

Создайте макрос с повторением и посмотрите его expansion с помощью:

```bash
cargo expand
```

Определите, какой обычный Rust-код получает компилятор после раскрытия макроса.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`macro_rules!`** — механизм декларативных макросов;
- **matcher** — шаблон, с которым сопоставляется вызов макроса;
- **transcriber** — код, который генерируется после успешного сопоставления;
- **fragment specifiers** — `expr`, `ident`, `ty`, `pat`, `item`, `tt` и другие способы захвата синтаксических фрагментов;
- **Edition 2024** изменяет поведение `expr`, а `expr_2021` сохраняет старые правила сопоставления; ([Rust Documentation][3])
- **repetition** — `*`, `+` и `?` позволяют работать с переменным количеством элементов;
- **`stringify!` и `concat!`** — примеры встроенных декларативных макросов;
- **hygiene** защищает локальные имена макросов, но `macro_rules!` использует именно _mixed-site hygiene_, а не полную изоляцию всех имён; ([Rust Documentation][1])
- **`#[macro_export]`** позволяет экспортировать `macro_rules!` из crate;
- **`$crate`** позволяет макросу надёжно обращаться к элементам crate, в котором он определён;
- **`cargo expand`** помогает увидеть результат раскрытия макроса.

**Самая важная идея:**

> `macro_rules!` — это не «функция, которая работает с текстом», а механизм сопоставления Rust-токенов с шаблонами и генерации нового Rust-кода. Макрос позволяет выразить повторяющийся синтаксический шаблон компактно, после чего сгенерированный код проходит обычную проверку Rust. Именно поэтому декларативные макросы особенно полезны там, где обычной функции или трейта недостаточно: при генерации синтаксических конструкций, работе с переменным количеством аргументов и создании небольших DSL.

[1]: https://doc.rust-lang.org/stable/reference/macros-by-example.html 'Macros by example - The Rust Reference'
[2]: https://doc.rust-lang.org/reference/macros-by-example.html?highlight=macro 'Macros by example - The Rust Reference'
[3]: https://doc.rust-lang.org/edition-guide/rust-2024/macro-fragment-specifiers.html 'Macro fragment specifiers - The Rust Edition Guide'
[4]: https://doc.rust-lang.org/stable/reference/macros-by-example.html?highlight=hygiene 'Macros by example - The Rust Reference'
