# Приложение D. Trait System Quick Reference

Краткий справочник по системе трейтов в Rust. Все ключевые концепции в одном месте — для быстрого поиска и освежения памяти.

---

## D.1. Определение трейта

```rust
// Базовый трейт
trait Summary {
    fn summarize(&self) -> String;
}

// Трейт с реализацией по умолчанию
trait DefaultSummary {
    fn summarize(&self) -> String {
        String::from("(no summary)")
    }
}

// Трейт с ассоциированным типом
trait Container {
    type Item;
    fn get(&self) -> Option<&Self::Item>;
}

// Трейт с ассоциированной константой
trait Versioned {
    const VERSION: u32;
}

```

---

## D.2. Реализация трейта

```rust
// Структура
struct User {
    name: String,
    age: u32,
}

// Реализация трейта
impl Summary for User {
    fn summarize(&self) -> String {
        format!("User: {} ({} years old)", self.name, self.age)
    }
}

// Реализация с ассоциированным типом
impl Container for Vec<i32> {
    type Item = i32;
    fn get(&self) -> Option<&i32> {
        self.first()
    }
}

// Реализация с ассоциированной константой
impl Versioned for User {
    const VERSION: u32 = 1;
}

```

---

## D.3. Trait Bounds (ограничения)

```rust
// В сигнатуре функции
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// Несколько bounds
fn process<T: Summary + Clone>(item: &T) {
    let _copy = item.clone();
    println!("{}", item.summarize());
}

// where clause
fn process_where<T, U>(a: &T, b: &U)
where
    T: Summary + Clone,
    U: std::fmt::Display + std::fmt::Debug,
{
    println!("a: {}, b: {:?}", a.summarize(), b);
}

// В структуре
struct Wrapper<T: Summary> {
    item: T,
}

// В impl
impl<T: Summary> Wrapper<T> {
    fn summarize(&self) -> String {
        self.item.summarize()
    }
}

```

---

## D.4. `impl Trait`

```rust
// В параметрах (анонимный тип)
fn print_summary_impl(item: &impl Summary) {
    println!("{}", item.summarize());
}

// В возвращаемом значении
struct User {
    name: String,
    age: u32,
}

impl Summary for User {
    fn summarize(&self) -> String {
        self.name.clone()
    }
}

fn make_summarizable() -> impl Summary {
    User {
        name: String::from("Alice"),
        age: 30,
    }
}

```

**Ограничения:**

* Только один конкретный тип.
* Нельзя использовать в трейтах (без RPITIT).
* Нельзя использовать с `dyn`.

---

## D.5. `dyn Trait` (динамическая диспетчеризация)

```rust
// Трейт-объект
trait Draw {
    fn draw(&self);
}

struct Circle;
impl Draw for Circle {
    fn draw(&self) { println!("Отрисовка круга"); }
}

struct Square;
impl Draw for Square {
    fn draw(&self) { println!("Отрисовка квадрата"); }
}

fn main() {
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle),
        Box::new(Square),
    ];
    for shape in shapes {
        shape.draw();
    }
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dtrait%2520Draw%2520%257B%250A%2520%2520%2520%2520fn%2520draw(%2526self)%253B%250A%257D%250A%250Astruct%2520Circle%253B%250Aimpl%2520Draw%2520for%2520Circle%2520%257B%250A%2520%2520%2520%2520fn%2520draw(%2526self)%2520%257B%2520println!(%2522%25D0%259E%25D1%2582%25D1%2580%25D0%25B8%25D1%2581%25D0%25BE%25D0%25B2%25D0%25BA%25D0%25B0%2520%25D0%25BA%25D1%2580%25D1%2583%25D0%25B3%25D0%25B0%2522)%253B%2520%257D%250A%257D%250A%250Astruct%2520Square%253B%250Aimpl%2520Draw%2520for%2520Square%2520%257B%250A%2520%2520%2520%2520fn%2520draw(%2526self)%2520%257B%2520println!(%2522%25D0%259E%25D1%2582%25D1%2580%25D0%25B8%25D1%2581%25D0%25BE%25D0%25B2%25D0%25BA%25D0%25B0%2520%25D0%25BA%25D0%25B2%25D0%25B0%25D0%25B4%25D1%2580%25D0%25B0%25D1%2582%25D0%25B0%2522)%253B%2520%257D%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520shapes%253A%2520Vec%253CBox%253Cdyn%2520Draw%253E%253E%2520%253D%2520vec!%255B%250A%2520%2520%2520%2520%2520%2520%2520%2520Box%253A%253Anew(Circle)%252C%250A%2520%2520%2520%2520%2520%2520%2520%2520Box%253A%253Anew(Square)%252C%250A%2520%2520%2520%2520%255D%253B%250A%2520%2520%2520%2520for%2520shape%2520in%2520shapes%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520shape.draw()%253B%250A%2520%2520%2520%2520%257D%250A%257D)

**Object Safety (что нужно для `dyn`):**

* ❌ Нет обобщённых методов.
* ❌ Нет методов с `Self: Sized`.
* ❌ Нет статических методов.
* ❌ Нет методов, возвращающих `Self`.

---

## D.6. Associated Types (ассоциированные типы)

```rust
// Определение
trait ContainerType {
    type Item;
    fn get_first(&self) -> Option<&Self::Item>;
}

impl ContainerType for Vec<String> {
    type Item = String;
    fn get_first(&self) -> Option<&Self::Item> {
        self.first()
    }
}

```

**Сравнение с generics:**

| Ассоциированные типы | Generic параметры |
| --- | --- |
| `type Item;` | `T` |
| Один тип на реализацию | Много типов на реализацию |
| `Iterator<Item i32>` | `Iterator<T>` |

---

## D.7. GAT — Generic Associated Types

```rust
// Определение
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// Реализация
struct Window<'a, T> {
    data: &'a [T],
    index: usize,
}

impl<'a, T> LendingIterator for Window<'a, T> {
    type Item<'b> = &'b T
    where
        Self: 'b;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.index < self.data.len() {
            let item = &self.data[self.index];
            self.index += 1;
            Some(item)
        } else {
            None
        }
    }
}

```

**Когда использовать:** когда ассоциированный тип зависит от времени жизни.

---

## D.8. RPIT — Return Position Impl Trait

```rust
// В функции
fn get_iter() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}

```

---

## D.9. RPITIT — RPIT In Traits

```rust
// Rust 2024+
trait Factory {
    fn create(&self) -> impl Iterator<Item = String>;
}

struct MyFactory;

impl Factory for MyFactory {
    fn create(&self) -> impl Iterator<Item = String> {
        vec!["a".to_string(), "b".to_string()].into_iter()
    }
}

```

**Ограничения:**

* Не object-safe.
* Нельзя использовать с `dyn`.

---

## D.10. Blanket Implementations

```rust
// Все типы, реализующие Display, автоматически реализуют ToString
// (пример из стандартной библиотеки Rust)

```

---

## D.11. Extension Traits

```rust
// Определение расширения
trait StringExt {
    fn reverse(&self) -> String;
}

// Реализация для существующего типа
impl StringExt for String {
    fn reverse(&self) -> String {
        self.chars().rev().collect()
    }
}

fn main() {
    let s = String::from("hello");
    println!("Перевернутая строка: {}", s.reverse());
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dtrait%2520StringExt%2520%257B%250A%2520%2520%2520%2520fn%2520reverse(%2526self)%2520-%253E%2520String%253B%250A%257D%250A%250Aimpl%2520StringExt%2520for%2520String%2520%257B%250A%2520%2520%2520%2520fn%2520reverse(%2526self)%2520-%253E%2520String%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520self.chars().rev().collect()%250A%2520%2520%2520%2520%257D%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520s%2520%253D%2520String%253A%253Afrom(%2522hello%2522)%253B%250A%2520%2520%2520%2520println!(%2522%25D0%259F%25D0%25B5%25D1%2580%25D0%25B5%25D0%25B2%25D0%25B5%25D1%2580%25D0%25BD%25D1%2583%25D1%2582%25D0%25B0%25D1%258F%2520%25D1%2581%25D1%2582%25D1%2580%25D0%25BE%25D0%25BA%25D0%25B0%253A%2520%257B%257D%2522%252C%2520s.reverse())%253B%250A%257D)

---

## D.12. Sealed Traits (закрытые трейты)

```rust
// Внутри модуля
mod sealed {
    pub trait Sealed {}
}

// Публичный трейт, реализуемый только в этом крейте
pub trait MyTrait: sealed::Sealed {
    fn method(&self);
}

```

---

## D.13. SuperTraits

```rust
trait Printable {
    fn print(&self);
}

// Trait, который требует Printable
trait Debuggable: Printable {
    fn debug(&self);
}

struct MyType;

impl Printable for MyType {
    fn print(&self) { println!("Печать..."); }
}

impl Debuggable for MyType {
    fn debug(&self) { println!("Отладка..."); }
}

fn main() {
    let item = MyType;
    item.print();
    item.debug();
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dtrait%2520Printable%2520%257B%250A%2520%2520%2520%2520fn%2520print(%2526self)%253B%250A%257D%250A%250Atrait%2520Debuggable%253A%2520Printable%2520%257B%250A%2520%2520%2520%2520fn%2520debug(%2526self)%253B%250A%257D%250A%250Astruct%2520MyType%253B%250A%250Aimpl%2520Printable%2520for%2520MyType%2520%257B%250A%2520%2520%2520%2520fn%2520print(%2526self)%2520%257B%2520println!(%2522%25D0%259F%25D0%25B5%25D1%2587%25D0%25B0%25D1%2582%25D1%258C...%2522)%253B%2520%257D%250A%257D%250A%250Aimpl%2520Debuggable%2520for%2520MyType%2520%257B%250A%2520%2520%2520%2520fn%2520debug(%2526self)%2520%257B%2520println!(%2522%25D0%259E%25D1%2582%25D0%25BB%25D0%25B0%25D0%25B4%25D0%25BA%25D0%25B0...%2522)%253B%2520%257D%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520item%2520%253D%2520MyType%253B%250A%2520%2520%2520%2520item.print()%253B%250A%2520%2520%2520%2520item.debug()%253B%250A%257D)

---

## D.14. Trait Coherence

**Правила:**

| Правило | Описание |
| --- | --- |
| **Orphan rule** | Трейт или тип должен быть локальным для реализации |
| **Одна реализация** | Для каждого типа может быть только одна реализация трейта |

---

## D.15. Сводная таблица

| Концепция | Синтаксис | Назначение |
| --- | --- | --- |
| **Trait** | `trait T { ... }` | Определение поведения |
| **Impl** | `impl T for Type { ... }` | Реализация |
| **Trait bound** | `T: Trait` | Ограничение типа |
| **`impl Trait`** | `fn foo(x: impl Trait)` | Анонимный тип |
| **`dyn Trait`** | `fn foo(x: &dyn Trait)` | Динамическая диспетчеризация |
| **Associated Type** | `type Item;` | Тип, связанный с трейтом |
| **GAT** | `type Item<'a>;` | Generic Associated Type |
| **RPIT** | `-> impl Trait` | Return Position Impl Trait |
| **RPITIT** | `fn method() -> impl Trait` | RPIT in Traits |
| **Blanket impl** | `impl<T: Trait> ...` | Автоматическая реализация |
| **Extension trait** | `trait Ext { ... }` | Добавление методов |
| **Sealed trait** | `mod sealed { pub trait Sealed }` | Контроль реализаций |
| **SuperTrait** | `trait T: Super` | Наследование трейтов |

---

### Главное из этого приложения

После этого приложения мы:

* **Знаем** все основные концепции системы трейтов.
* **Умеем** определять и реализовывать трейты.
* **Понимаем** `impl Trait` и `dyn Trait`.
* **Используем** ассоциированные типы и GAT.
* **Понимаем** RPIT и RPITIT.
* **Знаем** когда использовать каждую концепцию.

**Самая важная идея:**

> Трейты — это сердце системы типов Rust. Они определяют поведение, позволяют создавать абстракции и писать универсальный код. Этот справочник охватывает все ключевые аспекты трейтов — от простых до самых продвинутых. Используйте его как шпаргалку при работе с трейтами.