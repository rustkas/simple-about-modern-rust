# Приложение G. Rust Standard Library Map

Карта стандартной библиотеки Rust — `std`, `core`, `alloc` и их основные модули. Показывает, что где находится и для чего используется.

---

## G.1. Три уровня: `std` / `core` / `alloc`

```text
┌───────────────────────────────────────────────────────────────┐
│ std (стандартная библиотека)                                  │
│ Зависит от ОС. Включает всё: файлы, сеть, потоки, I/O.        │
├───────────────────────────────────────────────────────────────┤
│ alloc (выделение памяти)                                      │
│ Не зависит от ОС. Требует глобального аллокатора.             │
│ Vec, String, Box, Rc, Arc                                     │
├───────────────────────────────────────────────────────────────┤
│ core (ядро)                                                   │
│ Не зависит от ОС. Не требует аллокатора.                      │
│ Базовые типы, Option, Result, итераторы, срезы, паника.       │
└───────────────────────────────────────────────────────────────┘
```

| Слой        | Зависит от ОС | Требует аллокатор | Используется                    |
| ----------- | ------------- | ----------------- | ------------------------------- |
| **`core`**  | Нет           | Нет               | Всегда                          |
| **`alloc`** | Нет           | Да                | Когда нужна динамическая память |
| **`std`**   | Да            | Да                | В приложениях с ОС              |

---

## G.2. Модули `std`

### Базовые модули

| Модуль             | Описание                                                         |
| ------------------ | ---------------------------------------------------------------- |
| `std::cmp`         | Сравнение: `PartialEq`, `Eq`, `PartialOrd`, `Ord`                |
| `std::collections` | Коллекции: `HashMap`, `HashSet`, `VecDeque`, `BTreeMap` и др.    |
| `std::convert`     | Преобразование: `From`, `Into`, `TryFrom`, `TryInto`             |
| `std::default`     | Значения по умолчанию: `Default`                                 |
| `std::fmt`         | Форматирование: `Display`, `Debug`, `format!`, `println!`        |
| `std::iter`        | Итераторы: `Iterator` и адаптеры                                 |
| `std::mem`         | Работа с памятью: `size_of`, `replace`, `take`, `drop`           |
| `std::ops`         | Операторы: `Add`, `Sub`, `Mul`, `Index`, `Deref`                 |
| `std::option`      | `Option<T>`                                                      |
| `std::result`      | `Result<T, E>`                                                   |
| `std::slice`       | Срезы: `[T]`                                                     |
| `std::str`         | Строковые срезы: `str`                                           |
| `std::string`      | `String`                                                         |
| `std::vec`         | `Vec<T>`                                                         |
| `std::marker`      | Маркеры: `Send`, `Sync`, `Copy`, `Sized`, `Unpin`, `PhantomData` |
| `std::pin`         | `Pin<T>`                                                         |

### Ввод-вывод

| Модуль         | Описание                                              |
| -------------- | ----------------------------------------------------- |
| `std::io`      | Ввод-вывод: `Read`, `Write`, `BufReader`, `BufWriter` |
| `std::fs`      | Файловая система: `File`, `read_dir`, `create_dir`    |
| `std::net`     | Сеть: `TcpStream`, `TcpListener`, `UdpSocket`         |
| `std::path`    | Пути: `Path`, `PathBuf`                               |
| `std::process` | Процессы: `Command`, `exit`, `spawn`                  |

### Параллелизм

| Модуль              | Описание                                                |
| ------------------- | ------------------------------------------------------- |
| `std::thread`       | Потоки: `spawn`, `sleep`, `join`                        |
| `std::sync`         | Синхронизация: `Mutex`, `RwLock`, `Arc`                 |
| `std::sync::atomic` | Атомарные типы: `AtomicUsize`, `AtomicBool`, `Ordering` |
| `std::sync::mpsc`   | Каналы: `channel`, `Sender`, `Receiver`                 |

### Время

| Модуль      | Описание                                   |
| ----------- | ------------------------------------------ |
| `std::time` | Время: `Duration`, `Instant`, `SystemTime` |

### Асинхронность

| Модуль        | Описание                             |
| ------------- | ------------------------------------ |
| `std::future` | `Future`, `Poll`                     |
| `std::task`   | `Waker`, `Context`, `RawWaker`       |

### Другое

| Модуль       | Описание                                         |
| ------------ | ------------------------------------------------ |
| `std::env`   | Окружение: `args`, `vars`, `current_dir`         |
| `std::error` | Ошибки: `Error`                                  |
| `std::panic` | Паника: `panic!`, `set_hook`, `catch_unwind`     |
| `std::hint`  | Подсказки компилятору: `spin_loop`, `black_box`  |
| `std::num`   | Числовые типы и их методы                        |
| `std::ptr`   | Сырые указатели: `read`, `write`, `null`, `copy` |
| `std::ffi`   | FFI-строки: `OsString`, `CString` и др.          |

---

## G.3. Модули `core`

| Модуль          | Описание                                        |
| --------------- | ----------------------------------------------- |
| `core::option`  | `Option<T>`                                     |
| `core::result`  | `Result<T, E>`                                  |
| `core::iter`    | Итераторы                                       |
| `core::slice`   | Срезы                                           |
| `core::str`     | Строковые срезы                                 |
| `core::fmt`     | Форматирование                                  |
| `core::cmp`     | Сравнение                                       |
| `core::convert` | Преобразование                                  |
| `core::ops`     | Операторы                                       |
| `core::marker`  | Маркеры (`Send`, `Sync`, `Copy`, `PhantomData`) |
| `core::pin`     | `Pin<T>`                                        |
| `core::mem`     | Работа с памятью                                |
| `core::ptr`     | Сырые указатели                                 |
| `core::cell`    | `Cell`, `RefCell`, `UnsafeCell`                 |
| `core::num`     | Числовые типы                                   |
| `core::panic`   | `PanicInfo`, panic-handling                     |
| `core::time`    | `Duration`                                      |
| `core::future`  | `Future`, `Poll`                                |
| `core::task`    | `Waker`, `Context`                              |

---

## G.4. Модули `alloc`

| Модуль               | Описание                               |
| -------------------- | -------------------------------------- |
| `alloc::vec`         | `Vec<T>`                               |
| `alloc::string`      | `String`                               |
| `alloc::collections` | `VecDeque`, `LinkedList`, `BinaryHeap` |
| `alloc::boxed`       | `Box<T>`                               |
| `alloc::rc`          | `Rc<T>`, `Weak`                        |
| `alloc::sync`        | `Arc<T>`, `Weak`                       |
| `alloc::borrow`      | `Cow`, `ToOwned`                       |
| `alloc::fmt`         | Форматирование (с аллокацией)          |
| `alloc::slice`       | Операции над срезами (с аллокацией)    |

---

## G.5. Важные трейты

| Трейт                 | Описание                                 | Где определён  |
| --------------------- | ---------------------------------------- | -------------- |
| `Copy`                | Тип копируется бит-в-бит                 | `std::marker`  |
| `Clone`               | Тип можно клонировать                    | `std::clone`   |
| `PartialEq`           | Частичное равенство                      | `std::cmp`     |
| `Eq`                  | Полное равенство                         | `std::cmp`     |
| `PartialOrd`          | Частичное упорядочивание                 | `std::cmp`     |
| `Ord`                 | Полное упорядочивание                    | `std::cmp`     |
| `Display`             | Форматирование для пользователей         | `std::fmt`     |
| `Debug`               | Отладочное форматирование                | `std::fmt`     |
| `Default`             | Значение по умолчанию                    | `std::default` |
| `From` / `Into`       | Преобразование типов                     | `std::convert` |
| `TryFrom` / `TryInto` | Преобразование с ошибкой                 | `std::convert` |
| `Iterator`            | Итератор                                 | `std::iter`    |
| `Future`              | Асинхронная операция                     | `std::future`  |
| `Error`               | Ошибка                                   | `std::error`   |
| `Send`                | Можно передавать между потоками          | `std::marker`  |
| `Sync`                | Можно разделять между потоками           | `std::marker`  |
| `Sized`               | Известный размер                         | `std::marker`  |
| `Unpin`               | Можно перемещать из `Pin`                | `std::marker`  |

---

## G.6. Коллекции

| Коллекция        | Где определена     | Описание                             |
| ---------------- | ------------------ | ------------------------------------ |
| `Vec<T>`         | `std::vec`         | Динамический массив                  |
| `VecDeque<T>`    | `std::collections` | Двусторонняя очередь                 |
| `LinkedList<T>`  | `std::collections` | Двусвязный список                    |
| `HashMap<K, V>`  | `std::collections` | Хеш-таблица                          |
| `HashSet<T>`     | `std::collections` | Хеш-множество                        |
| `BTreeMap<K, V>` | `std::collections` | Отсортированное дерево               |
| `BTreeSet<T>`    | `std::collections` | Отсортированное множество            |
| `BinaryHeap<T>`  | `std::collections` | Приоритетная очередь                 |

---

## G.7. Строки

| Тип        | Описание                                    |
| ---------- | ------------------------------------------- |
| `String`   | Владеющая строка в куче (`std::string`)     |
| `&str`     | Строковый срез                              |
| `OsString` | Строка ОС (`std::ffi`)                      |
| `OsStr`    | Срез строки ОС (`std::ffi`)                 |
| `CString`  | C-строка с нуль-терминатором (`std::ffi`)   |
| `CStr`     | Срез C-строки (`std::ffi`)                  |
| `Path`     | Путь (`std::path`)                          |
| `PathBuf`  | Владеющий путь (`std::path`)                |

---

## G.8. Умные указатели

| Умный указатель | Где определён           | Описание                                   |
| --------------- | ----------------------- | ------------------------------------------ |
| `Box<T>`        | `std::boxed`            | Один владелец, данные в куче               |
| `Rc<T>`         | `std::rc`               | Подсчёт ссылок (однопоточный)              |
| `Arc<T>`        | `std::sync`             | Атомарный подсчёт ссылок                   |
| `Weak<T>`       | `std::rc` / `std::sync` | Слабая ссылка                              |
| `Cell<T>`       | `std::cell`             | Внутренняя изменяемость (`Copy`)           |
| `RefCell<T>`    | `std::cell`             | Внутренняя изменяемость (runtime-проверка) |
| `UnsafeCell<T>` | `std::cell`             | Основа interior mutability                 |
| `Mutex<T>`      | `std::sync`             | Взаимное исключение                        |
| `RwLock<T>`     | `std::sync`             | Разделение чтение/запись                   |

---

## G.9. Пути импорта

```rust
// core (no_std)
use core::option::Option;
use core::result::Result;
use core::iter::Iterator;

// alloc (no_std + allocator)
// В Cargo.toml: [dependencies] или использование через std
use alloc::vec::Vec;
use alloc::string::String;
use alloc::boxed::Box;
use alloc::rc::Rc;

// std (полный)
use std::collections::HashMap;
use std::fs::File;
use std::io::{Read, Write};
use std::net::TcpStream;
use std::sync::Arc;
use std::thread;
```

---

## G.10. Шпаргалка: что использовать

| Задача                     | Что использовать   |
| -------------------------- | ------------------ |
| Динамический массив        | `Vec<T>`           |
| Хеш-таблица                | `HashMap<K, V>`    |
| Отсортированная коллекция  | `BTreeMap<K, V>`   |
| Приоритетная очередь       | `BinaryHeap<T>`    |
| Владеющая строка           | `String`           |
| Строковый срез             | `&str`             |
| Ошибки                     | `Result<T, E>`     |
| Отсутствующее значение     | `Option<T>`        |
| Данные в куче              | `Box<T>`           |
| Множество владельцев       | `Rc<T>` / `Arc<T>` |
| Изменяемость через `&self` | `RefCell<T>`       |
| Потоки                     | `std::thread`      |
| Асинхронность              | `std::future`      |
| I/O                        | `std::io`          |
| Файлы                      | `std::fs`          |
| Сеть                       | `std::net`         |
| Время                      | `std::time`        |

---

### Главное из этого приложения

После этого приложения мы:

- **Понимаем** структуру `std` / `core` / `alloc`.
- **Знаем** основные модули стандартной библиотеки.
- **Умеем** быстро находить нужный тип или функцию.
- **Понимаем** разницу между слоями библиотеки.

**Самая важная идея:**

> Стандартная библиотека Rust состоит из трёх слоёв: `core` (для всех), `alloc` (для динамической памяти) и `std` (для ОС). Знание этих слоёв помогает выбирать правильные инструменты — от `no_std` до полноценных приложений. Этот справочник помогает быстро найти нужный модуль.