# Приложение H. Rust Error Messages

Ошибки компилятора в Rust — это не просто сообщения о проблемах. Это **подсказки**, которые объясняют, что пошло не так и как это исправить. Rust известен своими понятными и полезными ошибками.

В этом приложении мы разберём, как читать и понимать ошибки компилятора, какие типы ошибок бывают и как интерпретировать их сообщения.

---

## H.1. Структура сообщения об ошибке

```text
error[E0596]: cannot borrow `*self` as mutable, as it is behind a `&` reference
  --> src/main.rs:10:9
   |
10 |     self.push(42);
   |     ^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable
   |
help: consider changing this to be a mutable reference
   |
 8 | fn push(&mut self, value: i32) {
   |         ~~~~~~~~~

```

| Часть | Описание |
| --- | --- |
| `error[E0596]` | Код ошибки (можно искать в документации) |
| `cannot borrow...` | Краткое описание проблемы |
| `--> src/main.rs:10:9` | Файл, строка, колонка |
| `10 | self.push(42);` |
| `^^^^` | Указатель на проблемное место |
| `help:` | Предложение по исправлению |

---

## H.2. Типы ошибок

### `error` — ошибка компиляции

Программа не скомпилируется, пока ошибка не исправлена.

```text
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let x: i32 = "hello";
  |            ---   ^^^^^^^ expected `i32`, found `&str`
  |            |
  |            expected due to this

```

### `warning` — предупреждение

Код компилируется, но есть потенциальная проблема.

```text
warning: unused variable: `x`
 --> src/main.rs:2:9
  |
2 |     let x = 5;
  |         ^ help: if this is intentional, prefix it with an underscore: `_x`
  |
  = note: `#[warn(unused_variables)]` on by default

```

### `help` — подсказка

Компилятор предлагает, как исправить проблему.

```text
help: consider changing this to be mutable
  |
2 | let mut x = 5;
  |     +++

```

### `note` — дополнительная информация

Объяснение, почему возникла ошибка.

```text
note: expected `i32` because of this type annotation
 --> src/main.rs:4:9
  |
4 |     let x: i32 = "hello";
  |         ^^^

```

---

## H.3. Самые частые ошибки

### E0308 — несоответствие типов

```rust
let x: i32 = "hello";

```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:17
  |
2 |     let x: i32 = "hello";
  |            ---   ^^^^^^^ expected `i32`, found `&str`
  |            |
  |            expected due to this

```

**Исправление:** Используйте правильный тип.

```rust
fn main() {
    let x: &str = "hello";
    println!("{}", x);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520x%253A%2520%2526str%2520%253D%2520%2522hello%2522%253B%250A%2520%2520%2520%2520println!(%2522%257B%257D%2522%252C%2520x)%253B%250A%257D)

---

### E0382 — использование перемещённого значения

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}", s1);

```

```text
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:4:20
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`
3 |     let s2 = s1;
  |         -- value moved here
4 |     println!("{}", s1);
  |                    ^^ value borrowed here after move

```

**Исправление:** Используйте `.clone()` для сохранения оригинального значения.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();
    println!("s1: {}, s2: {}", s1, s2);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520s1%2520%253D%2520String%253A%253Afrom(%2522hello%2522)%253B%250A%2520%2520%2520%2520let%2520s2%2520%253D%2520s1.clone()%253B%250A%2520%2520%2520%2520println!(%2522s1%253A%2520%257B%257D%252C%2520s2%253A%2520%257B%257D%2522%252C%2520s1%252C%2520s2)%253B%250A%257D)

---

### E0596 — заимствование как изменяемое через `&`

```rust
fn push(s: &String) {
    s.push('!');
}

```

```text
error[E0596]: cannot borrow `*s` as mutable, as it is behind a `&` reference
 --> src/main.rs:2:5
  |
2 |     s.push('!');
  |     ^^^^^^^^^^^ `s` is a `&` reference, so the data it refers to cannot be borrowed as mutable
  |
help: consider changing this to be a mutable reference
  |
1 | fn push(s: &mut String) {
  |            ~~~~~~~~~

```

**Исправление:** Используйте изменяемую ссылку `&mut`.

```rust
fn push(s: &mut String) {
    s.push('!');
}

fn main() {
    let mut s = String::from("hello");
    push(&mut s);
    println!("{}", s);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520push(s%253A%2520%2526mut%2520String)%2520%257B%250A%2520%2520%2520%2520s.push(%27!%27)%253B%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520mut%2520s%2520%253D%2520String%253A%253Afrom(%2522hello%2522)%253B%250A%2520%2520%2520%2520push(%2526mut%2520s)%253B%250A%2520%2520%2520%2520println!(%2522%257B%257D%2522%252C%2520s)%253B%250A%257D)

---

### E0502 — конфликт заимствований

```rust
let mut v = vec![1, 2, 3];
let r1 = &v[0];
v.push(4);
println!("{}", r1);

```

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let r1 = &v[0];
  |              -- immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- immutable borrow later used here

```

**Исправление:** Копируйте значение вместо создания ссылки или завершайте время жизни неизменяемой ссылки раньше.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let r1 = v[0]; // Копирование примитива
    v.push(4);
    println!("r1: {}, v: {:?}", r1, v);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520mut%2520v%2520%253D%2520vec!%255B1%252C%25202%252C%25203%255D%253B%250A%2520%2520%2520%2520let%2520r1%2520%253D%2520v%255B0%255D%253B%250A%2520%2520%2520%2520v.push(4)%253B%250A%2520%2520%2520%2520println!(%2522r1%253A%2520%257B%257D%252C%2520v%253A%2520%253F%253A%253F%2522%252C%2520r1%252C%2520v)%253B%250A%257D)

---

### E0277 — трейт не реализован

```rust
fn print<T: std::fmt::Display>(x: T) {
    println!("{}", x);
}

print(vec![1, 2, 3]);

```

```text
error[E0277]: `Vec<i32>` doesn't implement `std::fmt::Display`
 --> src/main.rs:5:5
  |
5 |     print(vec![1, 2, 3]);
  |     ^^^^^ `Vec<i32>` cannot be formatted with the default formatter
  |
  = help: the trait `std::fmt::Display` is not implemented for `Vec<i32>`
  = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead

```

**Исправление:** Используйте `Debug` вместо `Display` для коллекций.

```rust
fn print<T: std::fmt::Debug>(x: T) {
    println!("{:?}", x);
}

fn main() {
    print(vec![1, 2, 3]);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520print%253CT%253A%2520std%253A%253Afmt%253A%253ADebug%253E(x%253A%2520T)%2520%257B%250A%2520%2520%2520%2520println!(%2522%253F%253A%253F%2522%252C%2520x)%253B%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520print(vec!%255B1%252C%25202%252C%25203%255D)%253B%250A%257D)

---

### E0599 — метод не найден

```rust
let x = 42;
x.push(5);

```

```text
error[E0599]: no method named `push` found for integer `{integer}` in the current scope
 --> src/main.rs:2:7
  |
2 |     x.push(5);
  |       ^^^^ method not found in `{integer}`

```

**Исправление:** Используйте правильный тип коллекции (например, `Vec`).

```rust
fn main() {
    let mut x = vec![42];
    x.push(5);
    println!("{:?}", x);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520mut%2520x%2520%253D%2520vec!%255B42%255D%253B%250A%2520%2520%2520%2520x.push(5)%253B%250A%2520%2520%2520%2520println!(%2522%253F%253A%253F%2522%252C%2520x)%253B%250A%257D)

---

### E0432 — импорт не найден

```rust
use nonexistent::module;

```

```text
error[E0432]: unresolved import `nonexistent::module`
 --> src/main.rs:1:5
  |
1 | use nonexistent::module;
  |     ^^^^^^^^^^^^^^^^^^^ no `module` in `nonexistent`

```

**Исправление:** Убедитесь, что внешний крейт добавлен в `Cargo.toml` и написан корректно.

```rust
// Пример правильного подключения стандартного модуля
use std::collections::HashMap;

fn main() {
    let _map: HashMap<String, i32> = HashMap::new();
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Duse%2520std%253A%253Acollections%253A%253AHashMap%253B%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520_map%253A%2520HashMap%253CString%252C%2520i32%253E%2520%253D%2520HashMap%253A%253Anew()%253B%250A%257D)

---

### E0609 — поле не найдено

```rust
struct User {
    name: String,
}

let user = User { name: "Alice".to_string() };
println!("{}", user.age);

```

```text
error[E0609]: no field `age` on type `User`
 --> src/main.rs:6:20
  |
6 |     println!("{}", user.age);
  |                    ^^^^^^^^ unknown field
  |
  = note: available fields are: `name`

```

**Исправление:** Используйте существующее поле структуры.

```rust
struct User {
    name: String,
}

fn main() {
    let user = User { name: "Alice".to_string() };
    println!("{}", user.name);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dstruct%2520User%2520%257B%250A%2520%2520%2520%2520name%253A%2520String%252C%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520user%2520%253D%2520User%2520%257B%2520name%253A%2520%2522Alice%2522.to_string()%2520%257D%253B%250A%2520%2520%2520%2520println!(%2522%257B%257D%2522%252C%2520user.name)%253B%250A%257D)

---

## H.4. Как читать ошибки

### Пошаговый алгоритм

1. **Прочитайте сообщение** — что говорит компилятор?
2. **Найдите строку с ошибкой** — где она произошла?
3. **Прочитайте `help**` — что предлагает компилятор?
4. **Понятно?** Если нет, ищите код ошибки (`E####`).
5. **Поищите в документации** — `rustc --explain E####` или онлайн.
6. **Исправьте** и проверьте снова.

```bash
# Получить полное объяснение ошибки
rustc --explain E0308

```

---

## H.5. Сложные ошибки

### Lifetime ошибки

```rust
fn get_ref() -> &String {
    let s = String::from("hello");
    &s
}

```

```text
error[E0515]: cannot return reference to local variable `s`
 --> src/main.rs:3:5
  |
3 |     &s
  |     ^^ returns a reference to data owned by the current function

```

**Исправление:** Верните владение строкой вместо ссылки.

```rust
fn get_ref() -> String {
    let s = String::from("hello");
    s
}

fn main() {
    let s = get_ref();
    println!("{}", s);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dfn%2520get_ref()%2520-%253E%2520String%2520%257B%250A%2520%2520%2520%2520let%2520s%2520%253D%2520String%253A%253Afrom(%2522hello%2522)%253B%250A%2520%2520%2520%2520s%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520s%2520%253D%2520get_ref()%253B%250A%2520%2520%2520%2520println!(%2522%257B%257D%2522%252C%2520s)%253B%250A%257D)

### Trait bound ошибки

```rust
fn process<T>(item: T) {
    println!("{}", item);
}

```

```text
error[E0277]: `T` doesn't implement `std::fmt::Display`
 --> src/main.rs:2:20
  |
2 |     println!("{}", item);
  |                    ^^^^ `T` cannot be formatted with the default formatter
  |
help: consider restricting type parameter `T`
  |
1 | fn process<T: std::fmt::Display>(item: T) {
  |             ++++++++++++++++++++

```

---

## H.6. Инструменты для отладки ошибок

```bash
# Подробное объяснение ошибки
rustc --explain E0308

# Больше информации о сборке
cargo build --verbose

# Показать расширенные макросы
cargo expand

# Проверка с более подробными сообщениями
cargo check --message-format=json

# Бэктрейс паники
RUST_BACKTRACE=1 cargo run

```

---

## H.7. Шпаргалка: ошибки и их решения

| Ошибка | Типичная причина | Решение |
| --- | --- | --- |
| `E0308` | Несоответствие типов | Исправьте тип или приведите значение |
| `E0382` | Использование после перемещения | Используйте `.clone()` или измените логику |
| `E0596` | Изменение через `&` | Используйте `&mut` |
| `E0502` | Конфликт заимствований | Разделите заимствования во времени |
| `E0277` | Трейт не реализован | Реализуйте трейт или используйте другой |
| `E0599` | Метод не найден | Используйте правильный тип |
| `E0432` | Импорт не найден | Проверьте имя и зависимости |
| `E0609` | Поле не найдено | Используйте существующее поле |
| `E0515` | Возврат локальной ссылки | Верните владение |

---

### Главное из этого приложения

После этого приложения мы:

* **Понимаем** структуру ошибок компилятора.
* **Умеем** читать сообщения об ошибках.
* **Знаем** как интерпретировать `error`, `warning`, `help`, `note`.
* **Понимаем** самые частые ошибки и их решения.
* **Знаем** как использовать `rustc --explain`.

**Самая важная идея:**

> Ошибки компилятора в Rust — это не просто преграды. Это подробные инструкции по исправлению кода. Учитесь читать их, обращайте внимание на `help` и `note` сообщения. Компилятор часто подсказывает правильное решение. Это одна из причин, почему Rust так удобен для разработки.