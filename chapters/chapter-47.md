# Глава 47. FFI

В предыдущих главах мы научились работать с `unsafe`-кодом, сырыми указателями и низкоуровневыми типами. Но одна из главных причин использовать `unsafe` в Rust — это **взаимодействие с кодом, написанным на других языках**, прежде всего с C.

**FFI (Foreign Function Interface)** позволяет:

- вызывать функции, реализованные на другом языке;
- экспортировать функции Rust для вызова из других языков;
- передавать структуры и массивы между языками;
- работать с системными API;
- использовать существующие C-библиотеки;
- создавать безопасные Rust-обёртки над небезопасными интерфейсами.

FFI — это граница между двумя мирами.

На одной стороне находится система типов Rust, ownership, borrowing и гарантии безопасности. На другой — ABI, указатели, ручное управление памятью и соглашения, которые компилятор Rust не может проверить.

Поэтому FFI — одна из областей, где особенно важно понимать, **что именно гарантирует Rust, а что становится обязанностью программиста**.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 47.1. Что такое FFI?

**FFI (Foreign Function Interface)** — механизм взаимодействия между программами или библиотеками, написанными на разных языках.

Например, Rust-программа может вызвать функцию из C-библиотеки:

```text
┌──────────────────────────────────────────────────────────┐
│                      Rust application                    │
│                                                          │
│   safe Rust                                              │
│       │                                                  │
│       ▼                                                  │
│   safe wrapper                                           │
│       │                                                  │
│       ▼                                                  │
│   unsafe FFI boundary                                    │
└───────┼──────────────────────────────────────────────────┘
        │
        │ C ABI
        ▼
┌──────────────────────────────────────────────────────────┐
│                    C library / OS API                    │
│                                                          │
│   raw pointers                                           │
│   manual memory management                               │
│   platform-specific contracts                            │
└──────────────────────────────────────────────────────────┘
```

Наиболее распространённый вариант — **C ABI**.

Причина проста: C ABI поддерживается огромным количеством системных библиотек и языков.

Например:

- Rust ↔ C;
- Rust ↔ C++;
- Rust ↔ Python через C ABI;
- Rust ↔ Swift;
- Rust ↔ Go;
- Rust ↔ системные API Windows, Linux и macOS.

При этом важно различать два понятия:

- **язык C**;
- **C ABI**.

Rust не обязан взаимодействовать именно с исходным кодом на C. Ему нужен бинарный интерфейс, совместимый с выбранным ABI.

---

## 47.2. `extern "C"` — объявление внешних функций

Для объявления функции, реализованной за пределами Rust, используется `extern`.

В Edition 2024 внешний блок должен быть явно помечен как `unsafe extern`. Это подчёркивает, что корректность объявления является ответственностью программиста. ([Rust Documentation][1])

Например:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn abs(value: c_int) -> c_int;
}

fn main() {
    let value = -42;

    let result = unsafe {
        abs(value)
    };

    println!("{result}");
}
```

Здесь Rust сообщает компилятору:

> «Функция `abs` существует где-то за пределами Rust и имеет именно такую сигнатуру и ABI».

Компилятор не может проверить, действительно ли внешняя библиотека предоставляет такую функцию.

Поэтому `unsafe` относится не только к самому вызову. Уже **объявление внешнего интерфейса является unsafe-контрактом**. ([Rust Documentation][1])

### Почему пример выше нельзя просто запустить в Rust Playground?

Потому что Playground должен найти внешнюю C-функцию и соответствующую системную библиотеку при линковке.

Для реальной программы это решается через линкер и `#[link]`, build script или системные библиотеки.

Поэтому для Playground лучше использовать самодостаточный пример, где C ABI демонстрируется без внешней библиотеки:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

unsafe extern "C" {
    fn add(a: i32, b: i32) -> i32;
}

fn main() {
    let result = unsafe {
        add(20, 22)
    };

    println!("{result}");
}
```

Здесь функция реализована самим Rust, но экспортируется с C ABI, а затем объявляется через FFI-интерфейс.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20add%2820%2C%2022%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D)

### Что означают эти элементы?

```rust
extern "C" fn
```

означает:

> функция использует C ABI.

```rust
unsafe extern "C" {
    fn add(...);
}
```

означает:

> эта функция находится за пределами обычной Rust-реализации, и корректность её объявления должна быть обеспечена программистом.

```rust
#[unsafe(no_mangle)]
```

говорит компилятору не применять обычное Rust name mangling к символу.

Это необходимо, например, когда внешняя программа должна найти функцию по конкретному имени. В Edition 2024 `no_mangle` является unsafe attribute и должен записываться как `#[unsafe(no_mangle)]`. ([Rust Documentation][2])

---

## 47.3. ABI — Application Binary Interface

**ABI** определяет правила взаимодействия функций на бинарном уровне.

В частности, ABI определяет такие вещи, как:

- соглашение о вызове;
- расположение аргументов;
- расположение возвращаемого значения;
- использование регистров;
- особенности stack;
- правила взаимодействия с бинарными символами.

Rust поддерживает несколько ABI.

Наиболее важные:

| ABI          | Назначение                              |
| ------------ | --------------------------------------- |
| `"Rust"`     | обычный Rust ABI                        |
| `"C"`        | C ABI                                   |
| `"system"`   | системный ABI платформы                 |
| `"C-unwind"` | C ABI с поддержкой unwind через границу |

Для обычного FFI чаще всего используется:

```rust
extern "C" fn
```

Для системных API:

```rust
extern "system" fn
```

Например:

```rust
type Callback = extern "C" fn(i32);

fn call_callback(callback: Callback) {
    callback(42);
}

extern "C" fn print_value(value: i32) {
    println!("value = {value}");
}

fn main() {
    call_callback(print_value);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20extern%20%22C%22%20fn%28i32%29%3B%0A%0Afn%20call_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20callback%2842%29%3B%0A%7D%0A%0Aextern%20%22C%22%20fn%20print_value%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22value%20%3D%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20call_callback%28print_value%29%3B%0A%7D)

### Важное замечание

Не следует воспринимать ABI как гарантию совместимости всего типа.

Например, `String` — Rust-тип. То, что функция имеет `"C"` ABI, **не превращает `String` в C-совместимый тип**.

C ABI отвечает за вызов функции. Представление данных необходимо обеспечивать отдельно.

---

## 47.4. C-строки: `CString` и `CStr`

Rust и C по-разному представляют строки.

### Rust

```rust
String
&str
```

Строка Rust хранит длину и не обязана заканчиваться `\0`.

### C

```text
char*
```

обычно указывает на последовательность символов, заканчивающуюся нулевым байтом:

```text
'H' 'e' 'l' 'l' 'o' '\0'
```

Для взаимодействия используются:

- `CString` — владеющая C-строка;
- `CStr` — заимствованное представление C-строки.

### Rust → C

```rust
use std::ffi::CString;

fn main() {
    let text = "Hello, C!";

    let c_string = CString::new(text)
        .expect("string contains an interior NUL");

    let ptr = c_string.as_ptr();

    println!("{ptr:p}");
}
```

`CString::new` проверяет, что внутри строки нет `\0`.

Это важно, потому что `\0` используется как терминатор C-строки.

### C → Rust

```rust
use std::ffi::{CStr, CString};

fn main() {
    let c_string = CString::new("Hello from C")
        .unwrap();

    let ptr = c_string.as_ptr();

    let c_str = unsafe {
        CStr::from_ptr(ptr)
    };

    let text = c_str.to_str().unwrap();

    println!("{text}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%7BCStr%2C%20CString%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20c_string%20%3D%20CString%3A%3Anew%28%22Hello%20from%20C%22%29.unwrap%28%29%3B%0A%20%20%20%20let%20ptr%20%3D%20c_string.as_ptr%28%29%3B%0A%0A%20%20%20%20let%20c_str%20%3D%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20CStr%3A%3Afrom_ptr%28ptr%29%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20let%20text%20%3D%20c_str.to_str%28%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22%7Btext%7D%22%29%3B%0A%7D)

### Почему `CStr::from_ptr` является `unsafe`?

Потому что Rust не может проверить указатель.

Программист обязан гарантировать, что:

1. указатель не `null`;
2. указатель указывает на действительную память;
3. последовательность заканчивается `\0`;
4. память доступна для чтения;
5. строка находится в памяти достаточно долго.

Например, следующий код опасен:

```rust
let ptr = std::ptr::null();

let text = unsafe {
    CStr::from_ptr(ptr)
};
```

Здесь нарушается контракт функции.

---

## 47.5. Raw pointers в FFI

При передаче массивов между Rust и C обычно передаются:

```text
pointer + length
```

Например, C-функция может иметь интерфейс:

```c
void process_data(int *data, size_t len);
```

В Rust это представляется примерно так:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    fn process_data(
        data: *mut c_int,
        len: usize,
    );
}
```

Вызов:

```rust
let mut numbers = vec![1, 2, 3, 4, 5];

unsafe {
    process_data(
        numbers.as_mut_ptr(),
        numbers.len(),
    );
}
```

Здесь `Vec<T>` **не передаётся целиком**.

Передаются только:

```text
pointer → first element
length  → number of elements
```

C ничего не знает о `Vec<T>` и его внутреннем устройстве.

### `*const T` и `*mut T`

```rust
*const T
```

означает указатель, через который данные предполагается читать.

```rust
*mut T
```

означает указатель, через который данные может изменять вызываемая функция.

### Важное правило

Нельзя передавать в C произвольный Rust-тип только потому, что он содержит простые поля.

Например, это **не является C ABI-структурой**:

```rust
struct Bad {
    name: String,
    values: Vec<i32>,
}
```

`String` и `Vec` имеют Rust-specific representation.

Для FFI нужно использовать явно определённое представление.

---

## 47.6. Ownership через FFI

Самая важная проблема FFI — не вызов функции, а **владение памятью**.

Перед использованием FFI необходимо ответить на вопрос:

> Кто выделяет память и кто её освобождает?

Рассмотрим типичный C API:

```c
void *create_data(void);
void free_data(void *ptr);
```

В Rust:

```rust
use std::ffi::c_void;

unsafe extern "C" {
    fn create_data() -> *mut c_void;
    fn free_data(ptr: *mut c_void);
}
```

Нельзя просто получить указатель и забыть о нём:

```rust
let ptr = unsafe {
    create_data()
};
```

Иначе возникает риск утечки памяти.

Лучше скрыть владение внутри Rust-типа:

```rust
use std::ffi::c_void;

unsafe extern "C" {
    fn create_data() -> *mut c_void;
    fn free_data(ptr: *mut c_void);
}

struct Data {
    ptr: *mut c_void,
}

impl Drop for Data {
    fn drop(&mut self) {
        if !self.ptr.is_null() {
            unsafe {
                free_data(self.ptr);
            }
        }
    }
}
```

Теперь освобождение происходит автоматически.

### Но этого всё ещё недостаточно

Настоящая безопасная обёртка должна также определить:

- может ли `create_data()` вернуть `null`;
- кто владеет объектом;
- можно ли копировать handle;
- можно ли передавать его между потоками;
- можно ли вызвать `free_data()` несколько раз;
- какая функция должна освобождать объект.

Например:

```rust
impl Data {
    fn new() -> Option<Self> {
        let ptr = unsafe {
            create_data()
        };

        if ptr.is_null() {
            None
        } else {
            Some(Self { ptr })
        }
    }
}
```

Это уже гораздо ближе к production-подходу.

---

## 47.7. Opaque handles — практический паттерн FFI

Очень распространённый C API выглядит так:

```c
typedef struct Connection Connection;

Connection *connection_create(void);
void connection_destroy(Connection *);
```

C-клиент не знает внутреннее устройство `Connection`.

В Rust можно представить такой объект как **opaque handle**:

```rust
use std::ffi::c_void;

struct Connection {
    ptr: *mut c_void,
}
```

Rust-код не должен пытаться интерпретировать внутреннюю память объекта.

Он только передаёт handle обратно в C:

```text
Rust
  │
  │ Connection { ptr }
  ▼
┌───────────────┐
│ opaque handle │
└───────┬───────┘
        │
        ▼
     C library
        │
        ▼
 actual Connection
```

Это один из наиболее надёжных способов проектирования FFI API.

---

## 47.8. Callbacks — передача Rust-функций в C

FFI работает в обе стороны.

Rust может вызвать C:

```text
Rust → C
```

но C может вызвать функцию Rust:

```text
C → Rust
```

Для этого используется callback.

Например, C API может выглядеть так:

```c
typedef void (*callback_t)(int);

void register_callback(callback_t callback);
```

В Rust:

```rust
unsafe extern "C" {
    fn register_callback(
        callback: Option<extern "C" fn(i32)>
    );
}
```

Rust callback:

```rust
extern "C" fn rust_callback(value: i32) {
    println!("Callback: {value}");
}
```

И регистрация:

```rust
unsafe {
    register_callback(Some(rust_callback));
}
```

### Почему `Option<extern "C" fn(...)>`?

Потому что в C callback часто может быть:

```c
NULL
```

В Rust естественная модель такого значения:

```rust
Option<extern "C" fn(...)>
```

`None` соответствует отсутствующему callback.

### Самостоятельный пример

Чтобы пример можно было запускать без C-библиотеки, callback можно вызвать непосредственно:

```rust
type Callback = Option<extern "C" fn(i32)>;

extern "C" fn print_value(value: i32) {
    println!("callback: {value}");
}

fn run_callback(callback: Callback) {
    if let Some(callback) = callback {
        callback(42);
    }
}

fn main() {
    run_callback(Some(print_value));
    run_callback(None);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20Option%3Cextern%20%22C%22%20fn%28i32%29%3E%3B%0A%0Aextern%20%22C%22%20fn%20print_value%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22callback%3A%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20run_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20if%20let%20Some%28callback%29%20%3D%20callback%20%7B%0A%20%20%20%20%20%20%20%20callback%2842%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20run_callback%28Some%28print_value%29%29%3B%0A%20%20%20%20run_callback%28None%29%3B%0A%7D)

### А что делать, если callback должен иметь состояние?

Обычная C-функция не может захватить Rust closure:

```rust
let callback = |value| {
    // состояние
};
```

Такое closure не является обычным `extern "C" fn`.

Типичный C API поэтому использует паттерн:

```c
typedef void (*callback_t)(int, void *user_data);
```

Rust передаёт отдельный context pointer:

```text
callback(value, user_data)
                  │
                  ▼
              Rust state
```

Это фундаментальный паттерн для callback-based FFI.

---

## 47.9. Layout compatibility — совместимость памяти

FFI требует совместимости не только функций, но и представления данных.

По умолчанию Rust **не предоставляет C-compatible layout** для обычной структуры.

Поэтому:

```rust
struct Point {
    x: f64,
    y: f64,
}
```

не следует считать C ABI-структурой.

Для C используется:

```rust
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}
```

Теперь Rust гарантирует C-compatible layout для этой структуры. `#[repr(C)]` задаёт представление, предназначенное, в частности, для взаимодействия с C. ([Rust Documentation][3])

Например:

```rust
#[repr(C)]
#[derive(Debug, Copy, Clone)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let point = Point {
        x: 10.0,
        y: 20.0,
    };

    println!("{point:?}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Brepr%28C%29%5D%0A%23%5Bderive%28Debug%2C%20Copy%2C%20Clone%29%5D%0Astruct%20Point%20%7B%0A%20%20%20%20x%3A%20f64%2C%0A%20%20%20%20y%3A%20f64%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20point%20%3D%20Point%20%7B%0A%20%20%20%20%20%20%20%20x%3A%2010.0%2C%0A%20%20%20%20%20%20%20%20y%3A%2020.0%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bpoint%3F%7D%22%2C%20point%29%3B%0A%7D)

В C соответствующая структура:

```c
struct Point {
    double x;
    double y;
};
```

### Типичные `repr`

| Представление          | Назначение                                            |
| ---------------------- | ----------------------------------------------------- |
| `#[repr(C)]`           | C-compatible layout                                   |
| `#[repr(transparent)]` | тот же ABI/layout, что у единственного значимого поля |
| `#[repr(u8)]`          | фиксированное представление discriminant              |
| `#[repr(packed)]`      | уменьшение/устранение padding, требует осторожности   |
| `#[repr(align(N))]`    | увеличение alignment                                  |

`#[repr(packed)]` особенно опасен при работе с ссылками на поля, поскольку обычные требования выравнивания могут быть нарушены.

### Главное правило

Для FFI нельзя передавать «любой Rust-тип».

Хорошими кандидатами являются:

```rust
#[repr(C)]
struct Config {
    port: u16,
    timeout: u32,
}
```

или простые C-compatible типы:

```rust
i32
u32
f64
*const T
*mut T
```

Плохими кандидатами без специального протокола являются:

```rust
String
Vec<T>
&str
&[T]
Box<T>
dyn Trait
```

---

## 47.10. Безопасная обёртка над FFI

Главный принцип Rust FFI:

> **Небезопасность должна быть локализована на границе системы.**

Вместо того чтобы заставлять весь код работать с raw pointers, создают безопасную обёртку.

Например, внешний API:

```text
C function
    ↓
unsafe Rust declaration
    ↓
unsafe call
    ↓
validation
    ↓
safe Rust API
```

Рассмотрим простой пример.

Допустим, внешняя библиотека предоставляет:

```c
double calculate(double value);
```

Rust объявляет:

```rust
unsafe extern "C" {
    fn calculate(value: f64) -> f64;
}
```

Внутренний API:

```rust
fn calculate_value(value: f64) -> f64 {
    unsafe {
        calculate(value)
    }
}
```

Теперь остальные части программы не должны знать о FFI.

### Но безопасная обёртка действительно должна быть безопасной

Нельзя просто сделать:

```rust
fn wrapper(ptr: *const i32) -> i32 {
    unsafe {
        *ptr
    }
}
```

и назвать функцию безопасной.

Безопасная функция должна гарантировать, что любой допустимый вход удовлетворяет требованиям unsafe-операции.

Например:

```rust
fn read_value(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        return None;
    }

    Some(unsafe {
        *ptr
    })
}
```

Теперь `null` обработан.

Но этого всё ещё недостаточно для произвольного указателя: необходимо гарантировать, что указатель указывает на действительный объект `i32`, доступный для чтения.

Именно поэтому FFI-обёртка должна документировать и проверять **контракт безопасности**.

---

## 47.11. Строки и владение: типичная FFI-ошибка

Рассмотрим опасный код:

```rust
use std::ffi::CString;

struct Config {
    host: *const std::ffi::c_char,
}

fn to_c(config: &str) -> Config {
    let host = CString::new(config).unwrap();

    Config {
        host: host.as_ptr(),
    }
}
```

На первый взгляд всё выглядит нормально.

Но `host` уничтожается в конце `to_c()`.

Следовательно:

```text
to_c()
  │
  ├── CString создаётся
  │
  ├── pointer сохраняется
  │
  └── CString уничтожается
             │
             ▼
        pointer dangling
```

Это классическая ошибка FFI.

### Правильный подход

Нужно хранить `CString` столько же, сколько требуется указателю:

```rust
use std::ffi::CString;

struct CConfig {
    host: CString,
}

impl CConfig {
    fn new(host: &str) -> Self {
        Self {
            host: CString::new(host).unwrap(),
        }
    }

    fn as_ptr(&self) -> *const std::ffi::c_char {
        self.host.as_ptr()
    }
}

fn main() {
    let config = CConfig::new("localhost");

    println!("{:p}", config.as_ptr());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Affi%3A%3ACString%3B%0A%0Astruct+CConfig+%7B%0A++++host%3A+CString%2C%0A%7D%0A%0Aimpl+CConfig+%7B%0A++++fn+new%28host%3A+%26str%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++host%3A+CString%3A%3Anew%28host%29.unwrap%28%29%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+as_ptr%28%26self%29+-%3E+*const+std%3A%3Affi%3A%3Ac_char+%7B%0A++++++++self.host.as_ptr%28%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+config+%3D+CConfig%3A%3Anew%28%22localhost%22%29%3B%0A%0A++++println%21%28%22%7B%3Ap%7D%22%2C+config.as_ptr%28%29%29%3B%0A%7D)

Это общий принцип:

> **Владеющий Rust-объект должен жить не меньше, чем указатель, который он предоставляет внешнему коду.**

---

## 47.12. Экспорт Rust-функций для C

FFI работает и в обратную сторону.

Rust может предоставить API, которое вызывается из C.

Например:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn multiply(
    a: i32,
    b: i32,
) -> i32 {
    a * b
}
```

Здесь используются три важных элемента:

### `pub`

Функция доступна за пределами текущего модуля/крейта.

### `extern "C"`

Функция использует C ABI.

### `#[unsafe(no_mangle)]`

Функция получает предсказуемое имя символа:

```text
multiply
```

В Edition 2024 unsafe attributes должны явно маркироваться как `unsafe(...)`. ([Rust Documentation][2])

Полезно также написать safety-комментарий:

```rust
// SAFETY: `multiply` is exported under a unique symbol name.
#[unsafe(no_mangle)]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}
```

---

## 47.13. FFI-типы из `std::ffi`

Для FFI желательно использовать типы, явно предназначенные для взаимодействия с C.

Например:

```rust
use std::ffi::{
    c_char,
    c_int,
    c_uint,
    c_void,
};
```

Это лучше, чем предполагать, что конкретный Rust integer всегда соответствует конкретному C-типу.

Например:

```c
int
```

лучше представлять через:

```rust
c_int
```

а:

```c
void *
```

через:

```rust
c_void
```

Пример:

```rust
use std::ffi::{c_int, c_void};

#[repr(C)]
struct Data {
    value: c_int,
    context: *mut c_void,
}

fn main() {
    let data = Data {
        value: 42,
        context: std::ptr::null_mut(),
    };

    println!("{}", data.value);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Affi%3A%3A%7Bc_int%2C+c_void%7D%3B%0A%0A%23%5Brepr%28C%29%5D%0Astruct+Data+%7B%0A++++value%3A+c_int%2C%0A++++context%3A+*mut+c_void%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+Data+%7B%0A++++++++value%3A+42%2C%0A++++++++context%3A+std%3A%3Aptr%3A%3Anull_mut%28%29%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+data.value%29%3B%0A%7D)

---

## 47.14. FFI с библиотеками

Для настоящего FFI-проекта необходимо решить две разные задачи:

1. объявить внешний API в Rust;
2. сообщить системе сборки, где находится библиотека.

Например:

```rust
unsafe extern "C" {
    fn external_function(value: i32) -> i32;
}
```

само по себе не говорит линкеру, где искать реализацию.

Для библиотек используются механизмы вроде:

```rust
#[link(name = "some_library")]
unsafe extern "C" {
    fn external_function(value: i32) -> i32;
}
```

Однако конкретный способ линковки зависит от платформы и библиотеки.

В реальном проекте могут использоваться:

- `#[link]`;
- `build.rs`;
- `cc` crate;
- `pkg-config`;
- CMake;
- системный linker;
- заранее собранные `.a`;
- `.so`;
- `.dylib`;
- `.dll`.

Поэтому FFI — это не только Rust-код. Это ещё и **build system integration**.

---

## 47.15. Динамические библиотеки

Иногда библиотека загружается во время выполнения.

Типичный подход:

```text
Rust application
       │
       ▼
Load library
       │
       ▼
Find symbol
       │
       ▼
Convert symbol → function pointer
       │
       ▼
Call function
```

Для этого существуют сторонние библиотеки, например `libloading`.

Пример концептуально выглядит так:

```rust
use libloading::{Library, Symbol};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    unsafe {
        let library = Library::new("libexample.so")?;

        let function: Symbol<unsafe extern "C" fn(i32) -> i32> =
            library.get(b"process")?;

        let result = function(42);

        println!("{result}");
    }

    Ok(())
}
```

Этот пример **не предназначен для Rust Playground**, поскольку требует внешней динамической библиотеки.

При использовании динамической загрузки особенно важно, чтобы:

- библиотека оставалась загруженной, пока используются её символы;
- сигнатура функции точно соответствовала реальному символу;
- ABI совпадал;
- lifetime загруженной библиотеки учитывался в архитектуре программы.

---

## 47.16. FFI с C++

Rust не имеет общего стабильного C++ ABI, аналогичного C ABI.

Поэтому обычно нельзя просто написать:

```rust
extern "C++" {
    fn cpp_function();
}
```

и ожидать, что Rust сможет напрямую использовать произвольную C++ библиотеку.

Основные подходы:

### 1. C API поверх C++

C++ библиотека предоставляет тонкий C-compatible слой:

```text
Rust
  │
  ▼
C API
  │
  ▼
C++ implementation
```

Например:

```cpp
extern "C" int calculate(int value) {
    return cpp_object.calculate(value);
}
```

Это самый простой архитектурный вариант.

### 2. `cxx`

Проект [`cxx`](https://cxx.rs/) предоставляет более высокий уровень интеграции Rust/C++.

Он позволяет описывать совместимые интерфейсы и генерировать необходимый glue code.

### 3. `bindgen`

`bindgen` используется для автоматической генерации Rust bindings из C/C++ заголовков.

Это особенно полезно для больших существующих C API.

Но автоматически сгенерированный binding **не делает внешний API безопасным**.

Это лишь экономит ручной труд.

---

## 47.17. Что действительно находится внутри `unsafe`

При FFI `unsafe` — это не «разрешение компилятору делать что угодно».

Это утверждение:

> **Программист самостоятельно обеспечивает определённый контракт безопасности.**

Например:

```rust
unsafe {
    CStr::from_ptr(ptr)
}
```

означает, что программист гарантирует корректность `ptr`.

А:

```rust
unsafe {
    external_function(ptr, len)
}
```

может означать контракт:

```text
ptr != null
ptr points to len valid elements
memory is readable/writable as required
object lives long enough
ABI is correct
```

Поэтому хороший FFI-код должен локализовать `unsafe`.

Плохой:

```rust
// unsafe разбросан по всему приложению
unsafe { ... }
unsafe { ... }
unsafe { ... }
unsafe { ... }
```

Хороший:

```text
┌───────────────────────────────┐
│ Safe Rust application         │
├───────────────────────────────┤
│ Safe Rust wrapper             │
├───────────────────────────────┤
│ Small, audited unsafe module  │
├───────────────────────────────┤
│ FFI                           │
└───────────────────────────────┘
```

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: `extern` в Edition 2024

Попробуйте написать:

```rust
extern "C" {
    fn external_function();
}

fn main() {}
```

В Edition 2024 это ошибка.

Нужно:

```rust
unsafe extern "C" {
    fn external_function();
}

fn main() {}
```

Причина в том, что начиная с Edition 2024 `extern`-блоки должны быть явно `unsafe`. ([Rust Documentation][1])

---

## Эксперимент 2: `#[repr(C)]`

Создайте две структуры:

```rust
struct RustPoint {
    x: i32,
    y: i32,
}

#[repr(C)]
struct CPoint {
    x: i32,
    y: i32,
}
```

Первая использует обычное Rust representation.

Вторая имеет C-compatible representation.

Rust Reference прямо указывает, что layout обычных Rust-типов не имеет тех же гарантий, что `#[repr(C)]`. ([Rust Documentation][3])

---

## Эксперимент 3: `CString` lifetime

Рассмотрите:

```rust
use std::ffi::CString;

fn get_pointer() -> *const std::ffi::c_char {
    let string = CString::new("hello").unwrap();

    string.as_ptr()
}
```

Проблема не в компиляторе.

Проблема в том, что после завершения функции:

```text
string
   │
   ▼
Drop
   │
   ▼
memory released
   │
   ▼
returned pointer is dangling
```

Это один из важнейших классов ошибок при FFI.

---

## Эксперимент 4: неправильный Rust-тип для C

Попробуйте представить C API:

```c
void process(char *);
```

как:

```rust
unsafe extern "C" {
    fn process(value: &mut String);
}
```

Так делать нельзя.

C не знает о Rust `String`.

Правильный интерфейс должен использовать C-compatible представление:

```rust
unsafe extern "C" {
    fn process(value: *mut std::ffi::c_char);
}
```

или другой интерфейс, точно соответствующий реальному C API.

---

## Эксперимент 5: `no_mangle` в Edition 2024

Сравните:

```rust
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

и:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Для Edition 2024 второй вариант является правильным синтаксисом. `no_mangle` относится к unsafe attributes, поскольку неправильное использование может привести к проблемам на уровне символов и линковки. ([Rust Documentation][2])

---

# Практика

## Задание 1

Напишите функцию:

```rust
extern "C" fn
```

которая принимает два `i32` и возвращает их сумму.

Экспортируйте её с помощью:

```rust
#[unsafe(no_mangle)]
```

---

## Задание 2

Создайте:

```rust
#[repr(C)]
struct Point
```

с полями:

```rust
x: f64
y: f64
```

и функцию:

```rust
extern "C" fn distance(...)
```

которая вычисляет расстояние между двумя точками.

---

## Задание 3

Напишите функцию, которая принимает:

```rust
*const i32
```

и длину массива:

```rust
usize
```

и безопасно вычисляет сумму элементов.

Продумайте:

- что делать с `null`;
- что делать с нулевой длиной;
- как проверить границы;
- где должен находиться `unsafe`.

---

## Задание 4

Создайте Rust-обёртку над C-строкой:

```rust
CString
```

и функцию, которая получает:

```rust
*const c_char
```

и возвращает:

```rust
Option<&str>
```

Обратите внимание на две отдельные проблемы:

1. корректность указателя;
2. корректность UTF-8.

---

## Задание 5

Создайте callback:

```rust
type Callback = Option<extern "C" fn(i32)>;
```

и функцию:

```rust
fn run_callback(callback: Callback)
```

которая вызывает callback, если он установлен.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20Option%3Cextern%20%22C%22%20fn%28i32%29%3E%3B%0A%0Aextern%20%22C%22%20fn%20callback%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22value%20%3D%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20run_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20if%20let%20Some%28callback%29%20%3D%20callback%20%7B%0A%20%20%20%20%20%20%20%20callback%2842%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20run_callback%28Some%28callback%29%29%3B%0A%20%20%20%20run_callback%28None%29%3B%0A%7D)

---

## Задание 6

Исследуйте ошибочную конструкцию:

```rust
struct Config {
    host: *const std::ffi::c_char,
}
```

и создайте правильную версию, которая владеет `CString`.

Главный вопрос:

> Как гарантировать, что C-строка существует всё время, пока используется указатель?

---

## Задание 7

Спроектируйте небольшой opaque C API:

```text
connection_create()
connection_send()
connection_destroy()
```

Представьте его в Rust через raw pointer и создайте безопасную обёртку:

```rust
struct Connection
```

с автоматическим освобождением через `Drop`.

---

# Главное из этой главы

После этой главы мы понимаем:

- **FFI** — механизм взаимодействия Rust с кодом на других языках.
- **C ABI** — основной механизм совместимости Rust с большим количеством внешних библиотек.
- **`unsafe extern "C"`** — современный способ объявления внешнего API в Edition 2024. ([Rust Documentation][1])
- **`extern "C" fn`** — функция, использующая C ABI.
- **`CString` / `CStr`** — основные инструменты работы с C-строками.
- **Raw pointers** — базовый механизм передачи адресов через FFI.
- **`#[repr(C)]`** — способ задать C-compatible layout для структур, unions и enum-представлений. ([Rust Documentation][3])
- **`#[unsafe(no_mangle)]`** — современный способ экспортировать символ с определённым именем в Edition 2024. ([Rust Documentation][2])
- **Callbacks** — механизм вызова Rust-кода из внешней библиотеки.
- **Opaque handles** — практический способ скрывать внутреннее состояние библиотечного объекта.
- **Ownership** — одна из главных проблем FFI.
- **Safe wrappers** — основной способ сделать FFI удобным и безопасным для остального Rust-кода.
- **C++ FFI** требует дополнительных инструментов и не сводится к простому `extern "C"`.

**Самая важная идея:**

> **FFI — это не просто вызов функции на другом языке. Это граница между двумя системами типов, двумя моделями памяти и двумя наборами гарантий.**
>
> Rust не может проверить, правильно ли объявлена внешняя функция, действительно ли указатель валиден, кто владеет памятью или сколько времени живёт переданный объект. Эти обязанности ложатся на разработчика FFI.
>
> Поэтому хороший Rust FFI-код строится по принципу:
>
> **C ABI → минимальная unsafe-граница → проверка контракта → безопасная Rust-обёртка.**
>
> Чем меньше `unsafe` остаётся за пределами этой границы, тем надёжнее и понятнее становится вся система.

[1]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html?utm_source=chatgpt.com 'Unsafe extern blocks - The Rust Edition Guide'
[2]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html?utm_source=chatgpt.com 'Unsafe attributes - The Rust Edition Guide'
[3]: https://doc.rust-lang.org/stable/reference/type-layout.html?utm_source=chatgpt.com 'Type layout - The Rust Reference'
