# Приложение B. Rust Syntax Quick Reference

Краткий справочник синтаксиса Rust. Предназначен для быстрого поиска нужной конструкции — без объяснений, только синтаксис и минимальный пример.

---

## B.1. Переменные и типы

```rust
// Переменные
let x = 5;                       // неизменяемая
let mut y = 10;                  // изменяемая
const MAX: u32 = 100;            // константа
static APP_NAME: &str = "MyApp"; // статическая переменная

// Приведение типа
let x = 5i32;
let y = x as i64;

// Shadowing
let x = 5;
let x = x + 1; // x = 6

// Типы
let _i: i32 = 42;                // целые: i8, i16, i32, i64, i128, isize
let _u: u32 = 42;                // беззнаковые: u8, u16, u32, u64, u128, usize
let _f: f64 = 3.14;              // дробные: f32, f64
let _b: bool = true;             // логический
let _c: char = 'a';              // символ (Unicode)
let _s: &str = "hello";          // строковый срез
let _t: (i32, f64) = (42, 3.14); // кортеж
let _a: [i32; 3] = [1, 2, 3];    // массив
let _v: Vec<i32> = vec![1, 2, 3]; // вектор
```

---

## B.2. Функции

```rust
// Базовая функция
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Без возвращаемого значения (-> ())
fn print_sum(a: i32, b: i32) {
    println!("{}", a + b);
}

// Возврат через return
fn max(a: i32, b: i32) -> i32 {
    if a > b {
        return a;
    }
    b
}

// Generic-функция
fn identity<T>(x: T) -> T {
    x
}

// impl Trait
fn to_string(x: impl std::fmt::Display) -> String {
    format!("{}", x)
}

// Методы структуры
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }

    fn distance(&self) -> f64 {
        ((self.x.pow(2) + self.y.pow(2)) as f64).sqrt()
    }
}
```

---

## B.3. Замыкания (Closures)

```rust
// Базовое замыкание
let add = |a, b| a + b;
let result = add(5, 3);

// С явными типами параметров
let multiply = |a: i32, b: i32| -> i32 { a * b };

// Захват переменных
let factor = 2;
let double = |x| x * factor;

// Move-замыкание
let s = String::from("hello");
let consume = move || println!("{}", s);

// Трейты замыканий
fn call_twice<F>(f: F)
where
    F: Fn(),
{
    f();
    f();
}
```

---

## B.4. Управление потоком

```rust
// if-else
if x > 0 {
    println!("positive");
} else if x < 0 {
    println!("negative");
} else {
    println!("zero");
}

// if как выражение
let status = if x > 0 { "positive" } else { "non-positive" };

// match
match x {
    0 => println!("zero"),
    1 => println!("one"),
    n if n < 0 => println!("negative"),
    _ => println!("other"),
}

// match как выражение
let text = match x {
    0 => "zero",
    1 => "one",
    _ => "other",
};

// if let
if let Some(value) = optional {
    println!("{}", value);
}

// let-else (Rust 2021+)
let Some(value) = optional else {
    return;
};

// while let
while let Some(item) = iterator.next() {
    println!("{}", item);
}

// for
for i in 0..10 {
    println!("{}", i);
}
for (i, item) in items.iter().enumerate() {
    println!("{}: {}", i, item);
}

// loop
loop {
    if condition {
        break;
    }
}
let result = loop {
    break 42;
};

// break и continue
for i in 0..10 {
    if i % 2 == 0 {
        continue;
    }
    if i > 5 {
        break;
    }
    println!("{}", i);
}

// Метки циклов
'outer: for i in 0..3 {
    for j in 0..3 {
        if i == 1 && j == 1 {
            break 'outer;
        }
    }
}
```

---

## B.5. Структуры (Structs)

```rust
// Структура
struct User {
    name: String,
    age: u32,
}

// Создание
let user = User {
    name: String::from("Alice"),
    age: 30,
};

// Обновление через ..
let user2 = User {
    name: String::from("Bob"),
    ..user
};

// Кортежная структура
struct Color(u8, u8, u8);
let black = Color(0, 0, 0);

// Unit-структура
struct Empty;
let e = Empty;

// Методы
impl User {
    fn is_adult(&self) -> bool {
        self.age >= 18
    }
}
```

---

## B.6. Перечисления (Enums)

```rust
// Базовое перечисление
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

// С данными
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}

// Использование
let msg = Message::Write(String::from("hello"));

match msg {
    Message::Quit => println!("quit"),
    Message::Move { x, y } => println!("move {} {}", x, y),
    Message::Write(text) => println!("{}", text),
    Message::ChangeColor(r, g, b) => println!("{},{},{}", r, g, b),
}

// Option
let opt: Option<i32> = Some(42);
match opt {
    Some(v) => println!("{}", v),
    None => println!("none"),
}

// Result
let res: Result<i32, String> = Ok(42);
match res {
    Ok(v) => println!("{}", v),
    Err(e) => println!("{}", e),
}
```

---

## B.7. Трейты (Traits)

```rust
// Определение трейта
trait Summary {
    fn summarize(&self) -> String;
}

// С реализацией по умолчанию
trait DefaultSummary {
    fn summarize(&self) -> String {
        String::from("Summary")
    }
}

// Реализация трейта
struct User {
    name: String,
}

impl Summary for User {
    fn summarize(&self) -> String {
        format!("User: {}", self.name)
    }
}

// Trait bound
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// impl Trait
fn print_summary_impl(item: &impl Summary) {
    println!("{}", item.summarize());
}

// where clause
fn print_summary_where<T, U>(item: &T, other: &U)
where
    T: Summary,
    U: std::fmt::Display,
{
    // ...
}

// dyn Trait
fn print_dynamic(item: &dyn Summary) {
    println!("{}", item.summarize());
}
```

---

## B.8. Generic

```rust
// Generic-функция
fn identity<T>(x: T) -> T {
    x
}

// Generic-структура
struct Point<T> {
    x: T,
    y: T,
}

// Generic enum
enum MyOption<T> {
    Some(T),
    None,
}

// Несколько параметров
struct Pair<T, U> {
    first: T,
    second: U,
}

// Trait bounds
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b {
        a
    } else {
        b
    }
}

// where clause
fn compare<T, U>(a: T, b: U)
where
    T: std::fmt::Display + Clone,
    U: std::fmt::Display,
{
    // ...
}
```

---

## B.9. Модули и импорты

```rust
// Определение модуля
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }
    fn private() {}
}

// Импорт
use math::add;
use std::collections::HashMap;
use std::fmt::*;
use std::io::{self, Write};

// Переименование
use std::collections::HashMap as Map;

// Публичный реэкспорт
pub use math::add;

// Модуль в отдельном файле
mod utils;
```

---

## B.10. Владение и заимствование

```rust
// Владение
let s = String::from("hello");
let t = s; // s перемещён

// Копирование (Copy)
let x = 5;
let y = x; // x копируется

// Ссылки
let s = String::from("hello");
let r = &s; // неизменяемая ссылка

let mut s = String::from("hello");
let r_mut = &mut s; // изменяемая ссылка

// Заимствование в функциях
fn take_ref(s: &String) {
    println!("{}", s);
}
fn take_mut_ref(s: &mut String) {
    s.push_str(" world");
}

// Времена жизни
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() {
        a
    } else {
        b
    }
}

// 'static
let s: &'static str = "hello";
```

---

## B.11. Обработка ошибок

```rust
// Result
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// match
match divide(10, 2) {
    Ok(v) => println!("{}", v),
    Err(e) => println!("{}", e),
}

// ? оператор
fn process() -> Result<i32, String> {
    let a = divide(10, 2)?;
    let b = divide(a, 1)?;
    Ok(b)
}

// unwrap / expect
let v = divide(10, 2).unwrap();
let v = divide(10, 2).expect("division failed");

// panic
panic!("something went wrong");

// Option
fn get_user(id: u32) -> Option<String> {
    if id == 1 {
        Some("Alice".to_string())
    } else {
        None
    }
}
```

---

## B.12. Итераторы

```rust
let v = vec![1, 2, 3, 4, 5];

// Адаптеры
let doubled: Vec<_> = v.iter().map(|x| x * 2).collect();
let evens: Vec<_> = v.iter().filter(|x| *x % 2 == 0).collect();
let sum: i32 = v.iter().sum();

// fold
let product = v.iter().fold(1, |acc, x| acc * x);

// enumerate
for (i, x) in v.iter().enumerate() {
    println!("{}: {}", i, x);
}

// zip
let names = vec!["Alice", "Bob"];
let ages = vec![30, 25];
for (name, age) in names.iter().zip(ages.iter()) {
    println!("{}: {}", name, age);
}

// take / skip
let first_three: Vec<_> = v.iter().take(3).collect();
let skip_two: Vec<_> = v.iter().skip(2).collect();
```

---

## B.13. Макросы

```rust
// Вывод
println!("Hello");
println!("x = {}", x);
println!("{:?}", debug_value);
println!("{:#?}", pretty_debug);

// Форматирование
format!("Hello {}", name);
format!("x = {x}, y = {y}");

// Vec
let v = vec![1, 2, 3];

// Assertions
assert!(condition);
assert_eq!(a, b);
assert_ne!(a, b);
assert!(condition, "message {}", x);

// TODO / unimplemented
todo!("implement this");
unimplemented!();

// Прочие
stringify!(x + y);              // "x + y"
concat!("Hello", " ", "World"); // "Hello World"
include_str!("file.txt");
cfg!(target_os = "linux");
```

---

## B.14. Асинхронность (Async)

```rust
// Async-функция
async fn fetch_data() -> String {
    "data".to_string()
}

// Использование
#[tokio::main]
async fn main() {
    let data = fetch_data().await;
    println!("{}", data);
}

// Async-блок
let future = async {
    println!("async block");
    42
};
let result = future.await;

// Конкурентность
use tokio::join;
let (a, b) = join!(task1(), task2());

// Таймауты
use tokio::time::{timeout, Duration};
let result = timeout(Duration::from_secs(1), operation()).await;

// Spawn
let handle = tokio::spawn(async {
    // задача
});
let result = handle.await.unwrap();
```

---

## B.15. Unsafe Rust

```rust
// Unsafe-блок
unsafe {
    // код
}

// Сырые указатели
let x = 5;
let r = &x as *const i32;
unsafe {
    println!("{}", *r);
}

// Unsafe-функция
unsafe fn dangerous() {
    // ...
}
unsafe {
    dangerous();
}

// Unsafe-трейт
unsafe trait UnsafeTrait {}
unsafe impl UnsafeTrait for i32 {}

// static mut
static mut COUNTER: i32 = 0;
unsafe {
    COUNTER += 1;
}
```

---

## B.16. Тесты

```rust
// Unit-тест
#[test]
fn test_add() {
    assert_eq!(add(2, 3), 5);
}

// Test-модуль
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_multiply() {
        assert_eq!(multiply(2, 3), 6);
    }
}

// Should panic
#[test]
#[should_panic(expected = "division by zero")]
fn test_panic() {
    divide(10, 0);
}

// Игнорирование
#[test]
#[ignore]
fn test_slow() {
    // ...
}

// Тест с Result
#[test]
fn test_result() -> Result<(), String> {
    Ok(())
}

// Документационные тесты
/// # Examples
/// ```
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
```

---

## B.17. Cargo команды

```bash
cargo new                 # создать проект
cargo init                # инициализировать проект
cargo build               # сборка (debug)
cargo build --release     # сборка (release)
cargo run                 # сборка и запуск
cargo check               # проверка без сборки
cargo test                # запуск тестов
cargo doc                 # генерация документации
cargo doc --open          # документация в браузере
cargo fmt                 # форматирование
cargo clippy              # линтинг
cargo add serde           # добавить зависимость
cargo remove serde        # удалить зависимость
cargo update              # обновить зависимости
cargo publish             # публикация на crates.io
cargo tree                # дерево зависимостей
cargo clean               # очистка артефактов
```

---

### Главное из этого приложения

После этого приложения мы имеем:

- **Полный справочник** синтаксиса Rust.
- **Быстрый доступ** к любым конструкциям.
- **Минимальные примеры** для каждого элемента.
- **Удобная навигация** по разделам.

**Самая важная идея:**

> Этот справочник — не учебник, а инструмент для быстрого поиска. Используйте его, когда нужно вспомнить синтаксис конкретной конструкции. Для глубокого понимания обращайтесь к основным главам книги.