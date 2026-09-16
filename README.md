# Simple About Modern Rust

> **Сначала изучаем язык. Потом создаём проекты.**

Пошаговые уроки по современному Rust для начинающих.

## О книге

**Simple About Modern Rust** — это практическое учебное пособие по языку программирования Rust.

Книга создаётся как продолжение и развитие проекта
[Simple About Rust](https://github.com/rustkas/simple-about-rust).

Её основная цель — не просто познакомить читателя с синтаксисом Rust,
а помочь сформировать правильную модель мышления о языке:

- типах;
- выражениях;
- владении;
- заимствовании;
- времени жизни;
- обобщениях;
- трейтах;
- замыканиях;
- итераторах;
- системе типов;
- безопасности памяти;
- и других фундаментальных механизмах Rust.

Книга ориентирована на **современный Rust** и не является простым
обновлением старого учебника.

---

## Главный принцип

> **Сначала изучаем язык. Потом создаём проекты.**

Первые главы посвящены непосредственно языку Rust.

Читателю не требуется начинать с создания полноценного проекта,
изучения Cargo, структуры каталогов или настройки сложной среды разработки.

Вместо этого мы сначала изучаем сам язык.

После того как фундаментальные концепции Rust будут освоены,
книга переходит к разработке настоящих проектов.

---

## Rust Playground

В первых главах основным инструментом является официальный
[Rust Playground](https://play.rust-lang.org/).

Это позволяет изучать Rust непосредственно в браузере.

Для экспериментов не требуется устанавливать Rust или создавать
проект на локальном компьютере.

Типичный цикл обучения выглядит так:

```text
Предположить
     ↓
Написать код
     ↓
Запустить в Rust Playground
     ↓
Посмотреть результат
     ↓
Изменить код
     ↓
Сломать код
     ↓
Изучить сообщение компилятора
     ↓
Понять причину
````

Особое внимание уделяется **экспериментам с компилятором**.

Мы не только читаем объяснение, но и проверяем его на практике.

---

## Почему «сломать код» — это часть обучения

Rust имеет очень сильный компилятор.

Он не только сообщает, что программа не может быть скомпилирована,
но часто объясняет, почему это произошло и как можно исправить код.

Поэтому ошибки компиляции используются в книге как учебный материал.

Для многих примеров будет применяться следующий подход:

1. Предскажите результат.
2. Запустите программу.
3. Сравните результат с предположением.
4. Измените программу.
5. Попробуйте намеренно получить ошибку.
6. Изучите сообщение компилятора.
7. Объясните, почему Rust отверг программу.

Цель — не научиться избегать ошибок компилятора.

Цель — **научиться понимать компилятор**.

---

## Структура обучения

Книга состоит из двух больших этапов.

### Часть I — язык Rust

Первые главы посвящены изучению Rust непосредственно через
небольшие самостоятельные эксперименты.

Основные темы:

```text
Values
    ↓
Types
    ↓
Expressions
    ↓
Variables
    ↓
Functions
    ↓
Collections
    ↓
Structs
    ↓
Enums
    ↓
Ownership
    ↓
Borrowing
    ↓
Lifetimes
    ↓
Generics
    ↓
Traits
    ↓
Closures
    ↓
Slices
    ↓
Iterators
```

Все примеры этой части рассчитаны прежде всего на запуск в
Rust Playground.

### Часть II — разработка проектов

После изучения основных механизмов языка мы переходим к реальной
разработке.

Здесь появляются:

```text
Cargo
    ↓
Packages
    ↓
Crates
    ↓
Modules
    ↓
Dependencies
    ↓
Testing
    ↓
Documentation
    ↓
Features
    ↓
Workspaces
    ↓
Concurrency
    ↓
Async
    ↓
Macros
    ↓
Unsafe Rust
    ↓
WASM
    ↓
Embedded Rust
```

Таким образом, читатель сначала получает понимание языка,
а затем учится применять его в проектах.

---

## Для кого эта книга

Книга предназначена прежде всего для:

* начинающих изучать Rust;
* программистов, знакомых с другими языками;
* разработчиков, изучавших старые версии Rust;
* программистов, которые хотят понять, **почему Rust работает именно так**.

Предполагается, что читатель уже имеет базовое представление
о программировании.

Предыдущий опыт с C, C++, Java, Go, JavaScript/TypeScript или
другими языками будет полезен, но не является обязательным.

---

## Что отличает эту книгу

Это не справочник по синтаксису Rust.

И не попытка просто переписать документацию Rust.

Основной акцент сделан на понимании.

Например, вместо простого утверждения:

> Rust запрещает перемещать значение после его использования.

мы будем задавать вопросы:

```text
Что именно перемещается?

Почему Rust считает значение перемещённым?

Что знает компилятор?

Можно ли изменить программу так,
чтобы она стала корректной?

Что изменится, если значение реализует Copy?

Что изменится, если использовать ссылку?

Почему это безопасно?
```

Таким образом, каждая новая конструкция рассматривается не
изолированно, а как часть модели языка.

---

## Современный Rust

Книга ориентирована на современный стабильный Rust и
**Rust 2024 Edition**.

Rust продолжает развиваться, поэтому содержание книги может
обновляться вместе с языком и его экосистемой.

Если материал зависит от конкретной версии Rust, это будет
отдельно указано в соответствующей главе.

---

## Связь с Simple About Rust

Первоначальная книга:

**[Simple About Rust](https://github.com/rustkas/simple-about-rust)**

была написана для знакомства с Rust и содержала последовательные
практические уроки по основам языка.

**Simple About Modern Rust** сохраняет этот подход:

> небольшая теория → эксперимент → изменение кода → ошибка →
> наблюдение → вывод

но использует современную модель Rust и существенно расширяет
темы, связанные с системой типов, владением, заимствованием,
трейтовой системой, итераторами, асинхронностью и другими
современными возможностями языка.

---

# Оглавление

## PART I. RUST AS A LANGUAGE

### Глава 01. Знакомство с Rust

- Что такое Rust
- Rust Playground
- Первая программа
- `main`
- `println!`
- Компиляция программы
- Compiler-driven development
- Первый эксперимент
- Как читать сообщения компилятора

[Читать главу →](chapters/chapter-01.md)

---

### Глава 02. Значения и типы

- Значение и тип
- Целые числа
- Числа с плавающей точкой
- Арифметические операции
- Литералы
- Переполнение
- Вывод типа
- Явное указание типа
- `usize` и `isize`

[Читать главу →](chapters/chapter-02.md)

---

### Глава 03. Переменные

- `let`
- Неизменяемость
- `mut`
- Изменение значения
- Shadowing
- Область видимости
- Константы
- `const`
- `static`
- Переменная как связывание имени со значением

[Читать главу →](chapters/chapter-03.md)

---

### Глава 04. Выражения и инструкции

- Выражения
- Инструкции
- Значение выражения
- `;`
- Блоки
- Возвращаемое значение блока
- `if` как выражение
- `match` как выражение
- Последнее выражение блока

[Читать главу →](chapters/chapter-04.md)

---

### Глава 05. Управление потоком

- `if`
- `else`
- `else if`
- `match`
- Pattern matching
- `if let`
- `let else`
- `while`
- `while let`
- `loop`
- `for`
- `break`
- `continue`
- `return`

[Читать главу →](chapters/chapter-05.md)

---

### Глава 06. Коллекции

- Массивы
- `Vec<T>`
- Создание `Vec`
- Добавление и удаление элементов
- Доступ к элементам
- Проверка границ
- `HashMap`
- `HashSet`
- Вложенные коллекции
- Коллекции и типы

[Читать главу →](chapters/chapter-06.md)

---

### Глава 07. Перечисления

- `enum`
- Варианты перечисления
- Данные внутри вариантов
- `match`
- `Option<T>`
- `Some`
- `None`
- `Result<T, E>`
- Состояния как тип
- Представление состояний с помощью `enum`
- Type-driven design

[Читать главу →](chapters/chapter-07.md)

---

### Глава 08. Структуры

- `struct`
- Поля структуры
- Создание экземпляров
- Изменение полей
- Tuple structs
- Unit structs
- Методы
- `impl`
- Associated functions
- `self`
- `&self`
- `&mut self`
- Производные реализации (`derive`)

[Читать главу →](chapters/chapter-08.md)

---

### Глава 09. Функции

- Объявление функций
- Параметры
- Возвращаемое значение
- Типы параметров
- Выражения и `return`
- Область видимости
- Передача значений
- Передача ссылок
- Методы и функции
- Associated functions
- Функции как значения
- Указатели на функции

[Читать главу →](chapters/chapter-09.md)

---

### Глава 10. Ссылки и заимствование

- Ownership как модель языка
- Ссылка
- `&T`
- `&mut T`
- Заимствование
- Изменяемое заимствование
- Правила заимствования
- Несколько неизменяемых ссылок
- Одна изменяемая ссылка
- Конфликтующие заимствования
- Время жизни ссылки
- Почему Rust проверяет заимствования

[Читать главу →](chapters/chapter-10.md)

---

### Глава 11. Владение

- Ownership
- Перемещение (`move`)
- Копирование
- `Copy`
- `Clone`
- Владение строками
- Владение коллекциями
- Передача владения функции
- Возврат владения
- `drop`
- `Drop`
- Stack и Heap
- Владение и память

[Читать главу →](chapters/chapter-11.md)

---

### Глава 12. Время жизни

- Lifetime
- Почему существуют lifetime
- Lifetime annotations
- `'a`
- Lifetime параметров
- Lifetime возвращаемого значения
- Lifetime в структурах
- Lifetime и ссылки
- Lifetime elision
- `'static`
- Lifetime как часть системы типов

[Читать главу →](chapters/chapter-12.md)

---

### Глава 13. Срезы и динамические типы

- Slice
- `&[T]`
- `&str`
- Срез массива
- Срез `Vec`
- String slices
- Fat pointers
- `Sized`
- Dynamically Sized Types
- `str` как DST
- Срез как заимствованное представление данных

[Читать главу →](chapters/chapter-13.md)

---

### Глава 14. Строки и текст

- `String`
- `str`
- `&str`
- UTF-8
- Байты
- `bytes()`
- Unicode
- `chars()`
- Индексация строк
- Срезы строк
- `String` и ownership
- `String` и `&str`
- `Cow<str>`
- `OsString`
- `Path` и `PathBuf`

[Читать главу →](chapters/chapter-14.md)

---

### Глава 15. Обобщения

- Generic functions
- Generic structs
- Generic enums
- Generic methods
- Параметры типов
- Вывод типов
- Несколько параметров типов
- Trait bounds
- `where`
- Несколько ограничений
- Generic code
- Zero-cost abstractions

[Читать главу →](chapters/chapter-15.md)

---

### Глава 16. Traits

- Что такое trait
- Реализация trait
- Trait bounds
- Default implementations
- Trait как контракт типа
- Traits и generics
- `impl Trait`
- `dyn Trait`
- Static dispatch
- Dynamic dispatch
- Trait objects
- Object safety

[Читать главу →](chapters/chapter-16.md)

---

### Глава 17. Замыкания

- Closures
- Захват переменных
- Ownership и closures
- `Fn`
- `FnMut`
- `FnOnce`
- `move`
- Closures как аргументы
- Closures как значения
- Возвращение closure
- Closures и iterator API

[Читать главу →](chapters/chapter-17.md)

---

### Глава 18. Итераторы

- `Iterator`
- `next`
- `for`
- `IntoIterator`
- `iter`
- `iter_mut`
- `into_iter`
- Lazy evaluation
- Iterator adapters
- `map`
- `filter`
- `filter_map`
- `flat_map`
- `flatten`
- `enumerate`
- `zip`
- `take`
- `skip`
- `chain`
- `fold`
- `collect`
- `FromIterator`
- Итераторы и ownership
- Zero-cost abstractions

[Читать главу →](chapters/chapter-18.md)

---

### Глава 19. Система типов в действии

- Тип как средство выражения инвариантов
- `Option`
- `Result`
- Generics
- Traits
- Associated types
- `impl Trait`
- `dyn Trait`
- Type-driven design
- Типы состояний
- Компилятор как проверяющий инварианты
- Как мыслить на Rust

[Читать главу →](chapters/chapter-19.md)

---

# PART II. RUST AS A DEVELOPMENT TOOL

## Часть VI. Память и управление ресурсами

### Глава 20. Где находятся данные

- Stack и Heap
- Static memory
- Значение и представление в памяти
- Размер типа
- `size_of`
- `align_of`
- `Sized`
- Allocation
- Deallocation
- Ownership и память
- Move как изменение владельца
- `Copy`
- Ссылки
- Указатели
- Memory layout
- Почему Rust не использует garbage collector
- Что Rust гарантирует на уровне памяти
- Что Rust не гарантирует

[Читать главу →](chapters/chapter-20.md)

---

### Глава 21. Smart Pointers

- Smart pointers
- `Box<T>`
- Heap allocation
- `Deref`
- `Drop`
- `Rc<T>`
- Reference counting
- `Arc<T>`
- `Weak<T>`
- Циклические ссылки
- `Rc` vs `Arc`
- Когда нужен smart pointer
- Выбор smart pointer

[Читать главу →](chapters/chapter-21.md)

---

### Глава 22. Interior Mutability

- Почему нужна interior mutability
- `Cell<T>`
- `RefCell<T>`
- Runtime borrow checking
- `Mutex<T>`
- `RwLock<T>`
- `UnsafeCell<T>`
- Compile-time vs runtime borrow checking
- Interior mutability и архитектура

[Читать главу →](chapters/chapter-22.md)

---

## Часть VII. Concurrent Rust

### Глава 23. Потоки

- `std::thread`
- `thread::spawn`
- `JoinHandle`
- `join`
- `move` в поток
- Ownership между потоками
- Lifetime потока
- `'static`
- Scoped threads
- Concurrency vs parallelism

[Читать главу →](chapters/chapter-23.md)

---

### Глава 24. `Send` и `Sync`

- Auto traits
- `Send`
- `Sync`
- `T: Send`
- `T: Sync`
- Почему `Rc` не `Send`
- Почему `Arc` может использоваться между потоками
- `RefCell`
- `Mutex`
- Thread safety как свойство типа
- `Send`/`Sync` как часть API

[Читать главу →](chapters/chapter-24.md)

---

### Глава 25. Shared State

- `Arc`
- `Mutex`
- `RwLock`
- Atomics
- `AtomicBool`
- `AtomicUsize`
- Lock contention
- Deadlock
- Poisoning
- Lock granularity
- Mutex vs atomic

[Читать главу →](chapters/chapter-25.md)

---

### Глава 26. Channels

- Message passing
- `std::sync::mpsc`
- Producer/consumer
- Ownership через channels
- Bounded vs unbounded communication
- Channel как архитектурная граница
- Worker pool
- Channels vs shared state

[Читать главу →](chapters/chapter-26.md)

---

## Часть VIII. Async Rust

### Глава 27. Что такое `Future`

- Synchronous execution
- Asynchronous execution
- `Future`
- `Future::poll`
- `Poll`
- `Pending`
- `Ready`
- `Waker`
- Executor
- Reactor
- Runtime
- Почему `Future` сам по себе ничего не выполняет
- Lazy futures

[Читать главу →](chapters/chapter-27.md)

---

### Глава 28. `async` и `.await`

- `async fn`
- Async block
- `.await`
- State machine
- Suspension points
- Lazy execution
- Sequential async operations
- Concurrent async operations
- Async vs threads

[Читать главу →](chapters/chapter-28.md)

---

### Глава 29. Async Runtime

- Runtime
- Executor
- Reactor
- Timers
- I/O
- Tasks
- Tokio
- `tokio::spawn`
- Task vs OS thread

[Читать главу →](chapters/chapter-29.md)

---

### Глава 30. Async Concurrency

- `join!`
- `try_join!`
- `select!`
- Spawning
- Cancellation
- Timeout
- Cancellation safety
- Bounded concurrency
- Backpressure
- Graceful shutdown

[Читать главу →](chapters/chapter-30.md)

---

### Глава 31. Async Closures

- Почему обычной closure иногда недостаточно
- Async block внутри closure
- `async move`
- Async closures
- `async |x|`
- Borrowing внутри async closure
- Ownership async closure
- Async callbacks
- Async closures как параметры API
- Async closure traits
- Async iterator patterns

[Читать главу →](chapters/chapter-31.md)

---

### Глава 32. `Pin` и `Unpin`

- Почему существует `Pin`
- Self-referential structures
- `Future` и pinning
- `Pin<&mut T>`
- `Pin<Box<T>>`
- `Unpin`
- Moving vs pinning
- Когда `Pin` действительно нужен

[Читать главу →](chapters/chapter-32.md)

---

### Глава 33. Async Traits

- Async methods в traits
- `async fn` в traits
- Async trait API
- Associated futures
- Object safety
- `dyn Trait`
- Async trait design
- Generic async APIs

[Читать главу →](chapters/chapter-33.md)

---

### Глава 34. Streams

- `Stream`
- Async iteration
- Stream adapters
- Stream processing
- Buffering
- Backpressure
- Cancellation
- Async producer/consumer
- Stream composition

[Читать главу →](chapters/chapter-34.md)

---

# Часть IX. Modules, Crates и архитектура

### Глава 35. Modules

- `mod`
- Module tree
- `pub`
- Visibility
- `pub(crate)`
- `pub(super)`
- Nested modules
- `use`
- `self`
- `super`
- `crate`
- Module boundaries

[Читать главу →](chapters/chapter-35.md)

---

### Глава 36. Crates

- Crate как единица компиляции
- Binary crate
- Library crate
- `lib.rs`
- `main.rs`
- Dependencies
- `use`
- Re-export
- Public API
- Crate boundary
- Library design

[Читать главу →](chapters/chapter-36.md)

---

### Глава 37. Cargo

- `Cargo.toml`
- Package
- Dependencies
- Versions
- Cargo commands
- Profiles
- Build artifacts
- `Cargo.lock`
- Cargo configuration

[Читать главу →](chapters/chapter-37.md)

---

### Глава 38. Cargo Workspaces

- Workspace
- Members
- Shared dependencies
- Workspace inheritance
- Package boundaries
- Dependency management
- Monorepo architecture
- Binary + libraries
- Domain/infrastructure separation

[Читать главу →](chapters/chapter-38.md)

---

### Глава 39. Features и Conditional Compilation

- Cargo features
- Feature flags
- `cfg`
- `cfg!`
- `#[cfg(...)]`
- Optional dependencies
- Feature unification
- Feature resolver
- Platform-specific code
- Feature design
- API compatibility

[Читать главу →](chapters/chapter-39.md)

---

# Часть X. Macros

### Глава 40. `macro_rules!`

- Declarative macros
- Macro invocation
- Patterns
- Fragments
- Repetition
- Token trees
- Hygiene
- Macro expansion
- Debugging macros
- Macro design

[Читать главу →](chapters/chapter-40.md)

---

### Глава 41. Procedural Macros

- Proc-macro crate
- `TokenStream`
- Derive macros
- Attribute macros
- Function-like macros
- Compile-time code generation
- Diagnostics

[Читать главу →](chapters/chapter-41.md)

---

### Глава 42. `syn` и `quote`

- Parsing Rust syntax
- AST
- `syn`
- `quote`
- Generating Rust code
- Derive macro
- Attributes
- Diagnostics
- Практический procedural macro

[Читать главу →](chapters/chapter-42.md)

---

### Глава 43. Macros как инструмент создания framework

- DSL
- Code generation
- API ergonomics
- Compile-time programming
- Macros vs runtime
- Macros vs generics
- Macros vs build scripts
- Когда macro оправдана
- Когда macro делает API хуже

[Читать главу →](chapters/chapter-43.md)

---

# Часть XI. Unsafe Rust

### Глава 44. Что действительно означает `unsafe`

- Пять unsafe superpowers
- Unsafe blocks
- Unsafe functions
- Unsafe traits
- Safety contracts
- Invariants
- Safe abstraction
- Ответственность автора `unsafe`
- Документирование safety requirements

[Читать главу →](chapters/chapter-44.md)

---

### Глава 45. Raw Pointers

- `*const T`
- `*mut T`
- Dereferencing
- Pointer arithmetic
- Null pointers
- Alignment
- Validity
- Provenance
- Raw pointers vs references

[Читать главу →](chapters/chapter-45.md)

---

### Глава 46. `MaybeUninit`, `NonNull` и `UnsafeCell`

- Uninitialized memory
- `MaybeUninit<T>`
- Partial initialization
- `NonNull<T>`
- `UnsafeCell<T>`
- Manual memory management
- Safe abstraction over unsafe code
- Типичные ошибки

[Читать главу →](chapters/chapter-46.md)

---

### Глава 47. FFI

- C ABI
- `extern "C"`
- `unsafe extern`
- C strings
- Raw pointers
- Ownership через FFI
- Callbacks
- Layout compatibility
- Безопасная Rust-обёртка
- FFI boundary

[Читать главу →](chapters/chapter-47.md)

---

# Часть XII. Testing и качество

### Глава 48. Unit Tests

- `#[test]`
- Assertions
- `assert!`
- `assert_eq!`
- `assert_ne!`
- Test modules
- Testing private code
- Test organization

[Читать главу →](chapters/chapter-48.md)

---

### Глава 49. Integration Tests

- `tests/`
- Testing public API
- Test fixtures
- Test helpers
- Integration boundary

[Читать главу →](chapters/chapter-49.md)

---

### Глава 50. Documentation Tests

- Documentation comments
- `///`
- Examples
- Executable documentation
- `cargo test`
- Documentation как часть API

[Читать главу →](chapters/chapter-50.md)

---

### Глава 51. Clippy и Rustfmt

- `cargo fmt`
- `cargo clippy`
- Lints
- Warnings
- Idiomatic Rust
- Automatic formatting
- CI quality gates

[Читать главу →](chapters/chapter-51.md)

---

### Глава 52. Benchmarks и производительность

- Debug vs Release
- Benchmarking
- Allocations
- Profiling
- Compiler optimizations
- Measuring before optimizing
- Zero-cost abstractions
- Performance regressions

[Читать главу →](chapters/chapter-52.md)

---

# Часть XIII. Rust для реальных приложений

### Глава 53. Работа с файлами

- `std::fs`
- `Path`
- `PathBuf`
- Reading
- Writing
- Buffered I/O
- Directories
- Filesystem errors
- Permissions

[Читать главу →](chapters/chapter-53.md)

---

### Глава 54. Command-Line Applications

- CLI architecture
- Arguments
- Environment
- Exit codes
- `clap`
- Configuration
- Subcommands
- User-facing errors

[Читать главу →](chapters/chapter-54.md)

---

### Глава 55. Serialization

- `serde`
- `Serialize`
- `Deserialize`
- JSON
- TOML
- YAML
- Serialization/deserialization
- Typed configuration
- Schema evolution

[Читать главу →](chapters/chapter-55.md)

---

### Глава 56. HTTP

- HTTP client
- HTTP server
- `reqwest`
- `axum`
- Routing
- Extractors
- Middleware
- State
- JSON API
- Async HTTP

[Читать главу →](chapters/chapter-56.md)

---

### Глава 57. Database

- SQL
- `sqlx`
- Connection pools
- Migrations
- Transactions
- Prepared queries
- Async database access
- Error handling
- Database boundaries

[Читать главу →](chapters/chapter-57.md)

---

# Часть XIV. Rust + WebAssembly

### Глава 58. Что такое WebAssembly

- WASM
- WASI
- WebAssembly module
- JavaScript
- Imports
- Exports
- Linear memory
- ABI
- Sandboxing

[Читать главу →](chapters/chapter-58.md)

---

### Глава 59. Rust → WebAssembly

- `wasm32`
- `wasm-bindgen`
- JavaScript interop
- `wasm-pack`
- Browser integration
- Data exchange
- Errors

[Читать главу →](chapters/chapter-59.md)

---

### Глава 60. WebAssembly как runtime

- Rust library → WASM
- Host application
- ABI
- Data exchange
- Performance
- Sandbox
- Plugin architecture
- Embedded WASM

[Читать главу →](chapters/chapter-60.md)

---

### Глава 61. WASM вне браузера

- WASI
- WASM runtimes
- Plugins
- Sandboxing
- Component Model
- Component-oriented architecture
- Portable execution

[Читать главу →](chapters/chapter-61.md)

---

# Часть XV. Rust для Embedded

### Глава 62. Почему Rust подходит для Embedded

- `no_std`
- Predictable performance
- Memory safety
- Zero-cost abstractions
- Ownership и hardware
- Deterministic resource management

[Читать главу →](chapters/chapter-62.md)

---

### Глава 63. `no_std`

- `#![no_std]`
- `core`
- `alloc`
- Panic handler
- Отсутствие `std`
- Custom runtime
- Target platforms

[Читать главу →](chapters/chapter-63.md)

---

### Глава 64. Hardware Abstraction

- HAL
- Peripherals
- Ownership hardware resources
- Registers
- Interrupts
- Типобезопасные состояния hardware
- Embedded crates

[Читать главу →](chapters/chapter-64.md)

---

### Глава 65. Embedded Concurrency

- Interrupts
- Critical sections
- Atomics
- RTOS
- Async embedded
- Shared peripherals
- Real-time constraints

[Читать главу →](chapters/chapter-65.md)

---

# Часть XVI. Современный дизайн Rust API

### Глава 66. Builder Pattern

- Builders
- Fluent API
- Validation
- Typestate builders
- Compile-time validation
- Runtime validation

[Читать главу →](chapters/chapter-66.md)

---

### Глава 67. Newtype Pattern

- Domain types
- Type safety
- Semantic types
- Preventing accidental mixing
- Zero-cost abstractions
- `From` / `Into`

[Читать главу →](chapters/chapter-67.md)

---

### Глава 68. Typestate

- State encoded in types
- Compile-time state machines
- Typestate APIs
- Transitions
- Invalid states
- Phantom types
- API ergonomics

[Читать главу →](chapters/chapter-68.md)

---

### Глава 69. Error API Design

- Error enums
- `Result`
- Error propagation
- `?`
- Opaque errors
- Application errors
- Library errors
- Error context
- Error conversion
- `thiserror`
- `anyhow`

[Читать главу →](chapters/chapter-69.md)

---

### Глава 70. Designing Traits

- Small traits
- Single responsibility
- Associated types
- Generic parameters
- Object safety
- Extension traits
- Blanket implementations
- Sealed traits
- Trait coherence

[Читать главу →](chapters/chapter-70.md)

---

### Глава 71. Designing Library APIs

- Public/private boundary
- API stability
- SemVer
- MSRV
- Compatibility
- Documentation
- Examples
- Deprecation
- API evolution

[Читать главу →](chapters/chapter-71.md)

---

# Часть XVII. Как думает современный Rust-программист

### Глава 72. Zero-Cost Abstractions

- Abstraction without runtime penalty
- Monomorphization
- Inlining
- Generics
- Iterators
- Static dispatch
- Стоимость абстракции

[Читать главу →](chapters/chapter-72.md)

---

### Глава 73. Compile Time vs Runtime

- Что можно сделать во время компиляции
- Generics
- Traits
- Macros
- `const`
- Const evaluation
- Code generation
- Compile-time mechanisms

[Читать главу →](chapters/chapter-73.md)

---

### Глава 74. Type-Driven Design

- Тип как контракт
- Illegal states unrepresentable
- Enums
- Newtypes
- Typestate
- Phantom types
- State machines
- Domain modeling

[Читать главу →](chapters/chapter-74.md)

---

### Глава 75. Ownership как инструмент архитектуры

- Ownership boundaries
- API design
- Resource management
- Lifetime architecture
- Concurrency
- Shared ownership
- Ownership как средство моделирования системы

[Читать главу →](chapters/chapter-75.md)

---

# Часть XVIII. Практический Rust-проект

### Глава 76. Создаём CLI-приложение

- Cargo project
- Project structure
- Configuration
- Arguments
- Errors
- Filesystem
- Tests
- Documentation

[Читать главу →](chapters/chapter-76.md)

---

### Глава 77. Превращаем приложение в library

- Library crate
- Public API
- Modules
- Domain types
- Documentation
- Integration tests
- Separation of concerns

[Читать главу →](chapters/chapter-77.md)

---

### Глава 78. Добавляем async

- Tokio
- Tasks
- Concurrent operations
- Cancellation
- Timeout
- Error propagation

[Читать главу →](chapters/chapter-78.md)

---

### Глава 79. Добавляем HTTP API

- Axum
- Routing
- State
- Extractors
- Middleware
- JSON
- API errors

[Читать главу →](chapters/chapter-79.md)

---

### Глава 80. Добавляем database

- SQLx
- PostgreSQL
- Migrations
- Connection pool
- Transactions
- Repository boundary

[Читать главу →](chapters/chapter-80.md)

---

### Глава 81. Создаём workspace

- Application crate
- Domain crate
- Infrastructure crate
- Shared types
- Dependency boundaries
- Workspace architecture

[Читать главу →](chapters/chapter-81.md)

---

### Глава 82. Создаём собственный trait API

- Traits
- Associated types
- Generic implementations
- Async methods
- Abstraction boundary
- Testing implementations

[Читать главу →](chapters/chapter-82.md)

---

### Глава 83. Создаём procedural macro

- Derive
- Parsing
- Code generation
- Generated implementation
- Integration with application

[Читать главу →](chapters/chapter-83.md)

---

# Часть XIX. Rust 2018 → Rust 2026

### Глава 84. Что изменилось после Rust 2018

- Edition 2021
- Edition 2024
- Major language improvements
- Standard library evolution
- Cargo evolution
- Tooling evolution
- Что программист Rust 2018 должен знать сегодня

[Читать главу →](chapters/chapter-84.md)

---

### Глава 85. Async Rust: от эксперимента к основной модели

- Async/await
- Futures
- Runtimes
- Async traits
- Async closures
- Streams
- Cancellation
- Structured concurrency
- Оставшиеся ограничения

[Читать главу →](chapters/chapter-85.md)

---

### Глава 86. Эволюция системы типов

- GAT
- RPIT
- RPITIT
- Associated types
- Const generics
- Improved trait ergonomics
- Type system evolution

[Читать главу →](chapters/chapter-86.md)

---

### Глава 87. Что пока не стало stable

- Nightly
- Feature gates
- Unstable APIs
- Experimental features
- Tracking issues
- Почему nightly не следует использовать без причины
- Как оценивать unstable feature

[Читать главу →](chapters/chapter-87.md)

---

### Глава 88. Как следить за развитием Rust

- Rust Blog
- Inside Rust
- Rust Reference
- Edition Guide
- RFC
- Tracking issues
- Release notes
- Rust Users Forum
- crates.io
- docs.rs

[Читать главу →](chapters/chapter-88.md)

---

# Часть XX. Переход от изучения Rust к профессиональному Rust

### Глава 89. Как читать чужой Rust-код

- Cargo workspace
- Crate graph
- Modules
- Traits
- Generics
- Macros
- Lifetimes
- Async
- Поиск архитектуры проекта

[Читать главу →](chapters/chapter-89.md)

---

### Глава 90. Как читать документацию crate

- docs.rs
- API docs
- Examples
- Feature flags
- MSRV
- Dependencies
- Changelog
- Source code
- Поиск intended usage

[Читать главу →](chapters/chapter-90.md)

---

### Глава 91. Как проектировать Rust-проект

- Architecture
- Modules
- Crates
- Traits
- Errors
- Tests
- CI
- Dependencies
- Observability
- Configuration

[Читать главу →](chapters/chapter-91.md)

---

### Глава 92. Как публиковать crate

- crates.io
- Package metadata
- Documentation
- README
- Examples
- SemVer
- Changelog
- MSRV
- Licensing
- Release process

[Читать главу →](chapters/chapter-92.md)

---

### Глава 93. Как поддерживать Rust-проект годами

- Dependency updates
- Security
- Rust editions
- MSRV
- Breaking changes
- Deprecations
- Migration guides
- Reproducible builds
- Supply-chain security

[Читать главу →](chapters/chapter-93.md)

---

# Приложения

## Приложение A. Rust 2018 → Rust 2024

Таблица изменений языка.

[Читать приложение →](appendix/appendix-01.md)

---

## Приложение B. Rust Syntax Quick Reference

Краткий справочник синтаксиса Rust.

[Читать приложение →](appendix/appendix-02.md)

---

## Приложение C. Ownership Quick Reference

Основные правила ownership, borrowing и lifetimes.

[Читать приложение →](appendix/appendix-03.md)

---

## Приложение D. Trait System Quick Reference

- Generics
- Bounds
- Associated types
- `impl Trait`
- `dyn Trait`
- GAT
- RPIT
- RPITIT

[Читать приложение →](appendix/appendix-04.md)

---

## Приложение E. Async Rust Quick Reference

- `Future`
- `async`
- `.await`
- `Pin`
- `Waker`
- Runtime
- Tasks
- Channels
- Streams

[Читать приложение →](appendix/appendix-05.md)

---

## Приложение F. Cargo Quick Reference

- Cargo commands
- `Cargo.toml`
- Workspace
- Features
- Profiles

[Читать приложение →](appendix/appendix-06.md)

---

## Приложение G. Rust Standard Library Map

- `std`
- `core`
- `alloc`
- Основные модули стандартной библиотеки

[Читать приложение →](appendix/appendix-07.md)

---

## Приложение H. Rust Error Messages

Как читать и понимать сообщения компилятора Rust.

[Читать приложение →](appendix/appendix-08.md)

---

## Приложение I. Common Rust Patterns

- Builder
- Newtype
- Typestate
- RAII
- Extension Trait
- Strategy
- State Machine
- Repository

[Читать приложение →](appendix/appendix-09.md)

---

## Приложение J. Unsafe Rust Checklist

Что проверить перед использованием `unsafe`.

[Читать приложение →](appendix/appendix-10.md)

---

## Приложение K. Rust + WebAssembly Checklist

Практический checklist для проектов Rust + WebAssembly.

[Читать приложение →](appendix/appendix-11.md)

---

## Приложение L. Rust + Embedded Checklist

Практический checklist для Rust Embedded проектов.

[Читать приложение →](appendix/appendix-12.md)

---

## Приложение M. Rust Developer Toolchain

- `rustup`
- `rustc`
- `cargo`
- `rustfmt`
- `clippy`
- `rust-analyzer`
- `cargo-audit`
- `cargo-deny`
- `cargo-nextest`

[Читать приложение →](appendix/appendix-13.md)

---

## Приложение N. Rust Glossary

Основные термины Rust.

[Читать приложение →](appendix/appendix-14.md)

---

## Приложение O. Rust 2018 → Rust 2026 Migration Map

**Что знал программист Rust 2018 → что необходимо знать программисту Rust 2026.**

[Читать приложение →](appendix/appendix-15.md)

---

## Статус

Книга находится в разработке.

Структура глав, порядок тем и отдельные объяснения могут
изменяться по мере развития проекта.

---

## Лицензия

Текст книги и другие оригинальные материалы этого репозитория
распространяются под лицензией
**Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**.

Полный текст лицензии находится в файле [`LICENSE`](LICENSE).

---

## Автор

**Anatoly Kosorukov**

GitHub: [@rustkas](https://github.com/rustkas)
