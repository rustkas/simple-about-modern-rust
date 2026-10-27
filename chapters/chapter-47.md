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

Все примеры этой главы используют **Rust Edition 2024**. Примеры, которые можно запустить одним файлом, снабжены ссылками на Rust Playground. Примеры, для которых нужны отдельный C-код, `build.rs` или динамическая библиотека, оформлены как небольшие Cargo-проекты: их результаты мы получили локальной сборкой.

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

    // SAFETY: abs — функция из libc, которая принимает int и не имеет побочных эффектов.
    let result = unsafe { abs(value) };

    println!("{result}");
}
```

Результат:

```text
42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3Ac_int%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20abs%28value%3A%20c_int%29%20-%3E%20c_int%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%20-42%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20abs%20%E2%80%94%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D0%B8%D0%B7%20libc%2C%20%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D0%B0%D1%8F%20%D0%BF%D1%80%D0%B8%D0%BD%D0%B8%D0%BC%D0%B0%D0%B5%D1%82%20int%20%D0%B8%20%D0%BD%D0%B5%20%D0%B8%D0%BC%D0%B5%D0%B5%D1%82%20%D0%BF%D0%BE%D0%B1%D0%BE%D1%87%D0%BD%D1%8B%D1%85%20%D1%8D%D1%84%D1%84%D0%B5%D0%BA%D1%82%D0%BE%D0%B2.%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20abs%28value%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

Здесь Rust сообщает компилятору:

> «Функция `abs` существует где-то за пределами Rust и имеет именно такую сигнатуру и ABI».

Компилятор не может проверить, действительно ли внешняя библиотека предоставляет такую функцию и с такой ли сигнатурой.

Поэтому `unsafe` относится не только к самому вызову. Уже **объявление внешнего интерфейса является unsafe-контрактом**. ([Rust Documentation][1])

### Что нужно линковщику?

Этот пример запускается без всяких дополнительных настроек, в том числе в Rust Playground. Причина в том, что `abs` — функция стандартной библиотеки C (libc), а программа на Rust на Linux и macOS и так линкуется с libc: от неё зависит сам `std`. Поэтому любую функцию libc можно объявить и вызвать (`abs`, `strlen`, `malloc`, `qsort` и т. д.). Мы будем использовать это в главе для запускаемых примеров.

Для **своей** C-библиотеки это уже не так: линковщик должен знать, где её искать. Об этом — в разделах 47.14 и 47.15.

### `safe fn` внутри `unsafe extern`

Внутри блока `unsafe extern` каждую функцию можно пометить как `safe` или `unsafe`. Если автор объявления уверен, что функцию безопасно вызывать при любых аргументах, он помечает её `safe`, и вызывающему коду `unsafe` уже не нужен:

```rust
use std::ffi::c_int;

unsafe extern "C" {
    // `safe fn` — автор объявления утверждает, что функцию безопасно
    // вызывать при любых аргументах, поэтому unsafe при вызове не нужен.
    pub safe fn abs(value: c_int) -> c_int;
}

fn main() {
    println!("{}", abs(-7));
}
```

Результат:

```text
7
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3Ac_int%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20%2F%2F%20%60safe%20fn%60%20%E2%80%94%20%D0%B0%D0%B2%D1%82%D0%BE%D1%80%20%D0%BE%D0%B1%D1%8A%D1%8F%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F%20%D1%83%D1%82%D0%B2%D0%B5%D1%80%D0%B6%D0%B4%D0%B0%D0%B5%D1%82%2C%20%D1%87%D1%82%D0%BE%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8E%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%0A%20%20%20%20%2F%2F%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%82%D1%8C%20%D0%BF%D1%80%D0%B8%20%D0%BB%D1%8E%D0%B1%D1%8B%D1%85%20%D0%B0%D1%80%D0%B3%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%D1%85%2C%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20unsafe%20%D0%BF%D1%80%D0%B8%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B5%20%D0%BD%D0%B5%20%D0%BD%D1%83%D0%B6%D0%B5%D0%BD.%0A%20%20%20%20pub%20safe%20fn%20abs%28value%3A%20c_int%29%20-%3E%20c_int%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20abs%28-7%29%29%3B%0A%7D%0A)

Это обещание даёт автор объявления, а не компилятор: `safe fn` оправдано только если функция действительно не имеет предусловий. Для функций, принимающих указатели, оставляйте `unsafe` (или пишите безопасную обёртку, раздел 47.10).

### Экспорт и импорт в одной программе

Чтобы показать C ABI на самодостаточном примере, можно экспортировать функцию Rust под C-именем и затем «импортировать» её через внешний блок. Но есть ловушка. Попытка использовать одно и то же имя в обоих местах:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

unsafe extern "C" {
    fn add(a: i32, b: i32) -> i32;
}

fn main() {
    let result = unsafe { add(20, 22) };

    println!("{result}");
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20add%2820%2C%2022%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

не компилируется:

```text
error[E0428]: the name `add` is defined multiple times
```

Функция `add` и объявление `add` в `unsafe extern`-блоке живут в одном пространстве имён Rust-модуля, а не только на уровне символов линковки. Чтобы развести два имени, внешняя функция объявляется под другим Rust-именем, а нужный символ указывается в `#[link_name]`:

```rust
// Экспортируем функцию Rust под C-именем `c_add`.
#[unsafe(no_mangle)]
pub extern "C" fn c_add(a: i32, b: i32) -> i32 {
    a + b
}

// Объявляем её как внешнюю под другим именем в Rust (link_name указывает C-символ).
unsafe extern "C" {
    #[link_name = "c_add"]
    fn imported_add(a: i32, b: i32) -> i32;
}

fn main() {
    // SAFETY: символ c_add определён выше с точно такой же сигнатурой.
    let result = unsafe { imported_add(20, 22) };

    println!("{result}");
}
```

Результат:

```text
42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%20%D0%AD%D0%BA%D1%81%D0%BF%D0%BE%D1%80%D1%82%D0%B8%D1%80%D1%83%D0%B5%D0%BC%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8E%20Rust%20%D0%BF%D0%BE%D0%B4%20C-%D0%B8%D0%BC%D0%B5%D0%BD%D0%B5%D0%BC%20%60c_add%60.%0A%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20c_add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0A%2F%2F%20%D0%9E%D0%B1%D1%8A%D1%8F%D0%B2%D0%BB%D1%8F%D0%B5%D0%BC%20%D0%B5%D1%91%20%D0%BA%D0%B0%D0%BA%20%D0%B2%D0%BD%D0%B5%D1%88%D0%BD%D1%8E%D1%8E%20%D0%BF%D0%BE%D0%B4%20%D0%B4%D1%80%D1%83%D0%B3%D0%B8%D0%BC%20%D0%B8%D0%BC%D0%B5%D0%BD%D0%B5%D0%BC%20%D0%B2%20Rust%20%28link_name%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20C-%D1%81%D0%B8%D0%BC%D0%B2%D0%BE%D0%BB%29.%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20%23%5Blink_name%20%3D%20%22c_add%22%5D%0A%20%20%20%20fn%20imported_add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%81%D0%B8%D0%BC%D0%B2%D0%BE%D0%BB%20c_add%20%D0%BE%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D1%91%D0%BD%20%D0%B2%D1%8B%D1%88%D0%B5%20%D1%81%20%D1%82%D0%BE%D1%87%D0%BD%D0%BE%20%D1%82%D0%B0%D0%BA%D0%BE%D0%B9%20%D0%B6%D0%B5%20%D1%81%D0%B8%D0%B3%D0%BD%D0%B0%D1%82%D1%83%D1%80%D0%BE%D0%B9.%0A%20%20%20%20let%20result%20%3D%20unsafe%20%7B%20imported_add%2820%2C%2022%29%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A)

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

```rust
#[link_name = "c_add"]
```

указывает, под каким именем символ ищется в бинарнике, если оно отличается от Rust-имени функции в блоке.

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

Различие между `"C"` и `"C-unwind"` — в том, что происходит при панике. Обычная `extern "C"` функция не допускает, чтобы паника вышла за её границу; подробнее об этом в разделе 47.12.

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

Результат:

```text
value = 42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20extern%20%22C%22%20fn%28i32%29%3B%0A%0Afn%20call_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20callback%2842%29%3B%0A%7D%0A%0Aextern%20%22C%22%20fn%20print_value%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22value%20%3D%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20call_callback%28print_value%29%3B%0A%7D%0A)

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

Строка Rust хранит длину и не обязана заканчиваться `\0`. Она обязана быть корректным UTF-8.

### C

```text
char*
```

обычно указывает на последовательность символов, заканчивающуюся нулевым байтом:

```text
'H' 'e' 'l' 'l' 'o' '\0'
```

Кодировка C-строки не определена: это просто байты.

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
    let c_string = CString::new("Hello from C").unwrap();
    let ptr = c_string.as_ptr();

    // SAFETY: ptr указывает на валидную C-строку, которой владеет c_string,
    // а c_string живёт дольше, чем c_str.
    let c_str = unsafe { CStr::from_ptr(ptr) };

    let text = c_str.to_str().unwrap();
    println!("{text}");

    // Внутренний NUL недопустим:
    println!("{:?}", CString::new("bad\0string").is_err());

    // Литерал C-строки (c"...") — &CStr, проверенный на этапе компиляции.
    let literal: &CStr = c"literal";
    println!("{}", literal.to_str().unwrap());
}
```

Результат:

```text
Hello from C
true
literal
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7BCStr%2C%20CString%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20c_string%20%3D%20CString%3A%3Anew%28%22Hello%20from%20C%22%29.unwrap%28%29%3B%0A%20%20%20%20let%20ptr%20%3D%20c_string.as_ptr%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%83%D1%8E%20C-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%2C%20%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D0%BE%D0%B9%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%B5%D1%82%20c_string%2C%0A%20%20%20%20%2F%2F%20%D0%B0%20c_string%20%D0%B6%D0%B8%D0%B2%D1%91%D1%82%20%D0%B4%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%2C%20%D1%87%D0%B5%D0%BC%20c_str.%0A%20%20%20%20let%20c_str%20%3D%20unsafe%20%7B%20CStr%3A%3Afrom_ptr%28ptr%29%20%7D%3B%0A%0A%20%20%20%20let%20text%20%3D%20c_str.to_str%28%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22%7Btext%7D%22%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%92%D0%BD%D1%83%D1%82%D1%80%D0%B5%D0%BD%D0%BD%D0%B8%D0%B9%20NUL%20%D0%BD%D0%B5%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%3A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20CString%3A%3Anew%28%22bad%5C0string%22%29.is_err%28%29%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9B%D0%B8%D1%82%D0%B5%D1%80%D0%B0%D0%BB%20C-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B8%20%28c%22...%22%29%20%E2%80%94%20%26CStr%2C%20%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9%20%D0%BD%D0%B0%20%D1%8D%D1%82%D0%B0%D0%BF%D0%B5%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D1%8F%D1%86%D0%B8%D0%B8.%0A%20%20%20%20let%20literal%3A%20%26CStr%20%3D%20c%22literal%22%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20literal.to_str%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Три момента из этого примера.

1. `CStr::from_ptr` создаёт `&CStr` из указателя. Это `unsafe`, потому что Rust не может проверить указатель.
2. `CString::new("bad\0string")` возвращает ошибку, потому что внутри есть `\0`.
3. Литерал `c"literal"` — это `&CStr`, проверенный на этапе компиляции: в нём гарантированно нет внутренних `\0`, а терминатор добавлен компилятором. Для C-строк, известных заранее, это удобнее и дешевле, чем `CString::new`.

Метод `to_str()` возвращает `Result`: C-строка может содержать байты, которые не являются UTF-8. Если важнее получить хоть какой-то текст, чем строгую проверку, используется `to_string_lossy()`.

### Почему `CStr::from_ptr` является `unsafe`?

Потому что Rust не может проверить указатель.

Программист обязан гарантировать, что:

1. указатель не `null`;
2. указатель указывает на действительную память;
3. последовательность заканчивается `\0`;
4. память доступна для чтения;
5. строка находится в памяти достаточно долго и не изменяется, пока жив полученный `&CStr`.

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

Тип `usize` в Rust соответствует `size_t` в C на всех распространённых платформах.

### Рабочий пример: `qsort`

Функции `process_data` в libc нет, поэтому вот реальный пример с тем же устройством — указатель, длина и callback. Это `qsort` из libc: она сортирует массив и вызывает наш callback для сравнения элементов:

```rust
use std::ffi::{c_int, c_void};

unsafe extern "C" {
    // void qsort(void *base, size_t nmemb, size_t size,
    //            int (*compar)(const void *, const void *));
    fn qsort(
        base: *mut c_void,
        nmemb: usize,
        size: usize,
        compar: Option<unsafe extern "C" fn(*const c_void, *const c_void) -> c_int>,
    );
}

extern "C" fn compare(a: *const c_void, b: *const c_void) -> c_int {
    // SAFETY: qsort передаёт указатели на элементы массива i32.
    let (a, b) = unsafe { (*(a as *const i32), *(b as *const i32)) };
    a.cmp(&b) as c_int
}

fn main() {
    let mut numbers = vec![5, 2, 9, 1, 7];

    // SAFETY: передаём указатель на первый элемент, число элементов и размер элемента;
    // compare корректно интерпретирует указатели как *const i32.
    unsafe {
        qsort(
            numbers.as_mut_ptr().cast(),
            numbers.len(),
            std::mem::size_of::<i32>(),
            Some(compare),
        );
    }

    println!("{numbers:?}");
}
```

Результат:

```text
[1, 2, 5, 7, 9]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_int%2C%20c_void%7D%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20%2F%2F%20void%20qsort%28void%20%2Abase%2C%20size_t%20nmemb%2C%20size_t%20size%2C%0A%20%20%20%20%2F%2F%20%20%20%20%20%20%20%20%20%20%20%20int%20%28%2Acompar%29%28const%20void%20%2A%2C%20const%20void%20%2A%29%29%3B%0A%20%20%20%20fn%20qsort%28%0A%20%20%20%20%20%20%20%20base%3A%20%2Amut%20c_void%2C%0A%20%20%20%20%20%20%20%20nmemb%3A%20usize%2C%0A%20%20%20%20%20%20%20%20size%3A%20usize%2C%0A%20%20%20%20%20%20%20%20compar%3A%20Option%3Cunsafe%20extern%20%22C%22%20fn%28%2Aconst%20c_void%2C%20%2Aconst%20c_void%29%20-%3E%20c_int%3E%2C%0A%20%20%20%20%29%3B%0A%7D%0A%0Aextern%20%22C%22%20fn%20compare%28a%3A%20%2Aconst%20c_void%2C%20b%3A%20%2Aconst%20c_void%29%20-%3E%20c_int%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20qsort%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%91%D1%82%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D0%B8%20%D0%BD%D0%B0%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D1%8B%20%D0%BC%D0%B0%D1%81%D1%81%D0%B8%D0%B2%D0%B0%20i32.%0A%20%20%20%20let%20%28a%2C%20b%29%20%3D%20unsafe%20%7B%20%28%2A%28a%20as%20%2Aconst%20i32%29%2C%20%2A%28b%20as%20%2Aconst%20i32%29%29%20%7D%3B%0A%20%20%20%20a.cmp%28%26b%29%20as%20c_int%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20numbers%20%3D%20vec%21%5B5%2C%202%2C%209%2C%201%2C%207%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%91%D0%BC%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BD%D0%B0%20%D0%BF%D0%B5%D1%80%D0%B2%D1%8B%D0%B9%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%2C%20%D1%87%D0%B8%D1%81%D0%BB%D0%BE%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%BE%D0%B2%20%D0%B8%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%20%D1%8D%D0%BB%D0%B5%D0%BC%D0%B5%D0%BD%D1%82%D0%B0%3B%0A%20%20%20%20%2F%2F%20compare%20%D0%BA%D0%BE%D1%80%D1%80%D0%B5%D0%BA%D1%82%D0%BD%D0%BE%20%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D0%BF%D1%80%D0%B5%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D0%B8%20%D0%BA%D0%B0%D0%BA%20%2Aconst%20i32.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20qsort%28%0A%20%20%20%20%20%20%20%20%20%20%20%20numbers.as_mut_ptr%28%29.cast%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20numbers.len%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20std%3A%3Amem%3A%3Asize_of%3A%3A%3Ci32%3E%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20Some%28compare%29%2C%0A%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22%7Bnumbers%3A%3F%7D%22%29%3B%0A%7D%0A)

Обратите внимание на три вещи.

1. `qsort` принимает `void *`, поэтому указатель на `i32` приводится к `*mut c_void` через `.cast()`; внутри callback — обратно.
2. Передаются указатель на первый элемент, число элементов и размер одного элемента. Чтобы такой вызов был корректным, все три значения должны быть согласованы: если бы вместо `size_of::<i32>()` мы передали другое число, `qsort` вышла бы за границы массива.
3. Для пустого вектора `as_mut_ptr()` возвращает ненулевой «висячий» указатель, и это допустимо, пока длина равна нулю: C-функция, которая корректно обрабатывает `len == 0`, ничего по нему не прочтёт.

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

### Рабочий пример: `malloc` и `free`

Ту же схему можно проверить на настоящих функциях libc. `malloc` играет роль `create_data`, `free` — роль `free_data`:

```rust
use std::ffi::c_void;
use std::ptr::NonNull;

unsafe extern "C" {
    fn malloc(size: usize) -> *mut c_void;
    fn free(ptr: *mut c_void);
}

/// Владеющая обёртка над памятью, выделенной C.
struct CBuffer {
    ptr: NonNull<u8>,
    len: usize,
}

impl CBuffer {
    fn new(len: usize) -> Option<Self> {
        // SAFETY: malloc можно вызывать с любым размером; результат проверяется ниже.
        let raw = unsafe { malloc(len) };

        // malloc может вернуть null — превращаем это в None.
        let ptr = NonNull::new(raw.cast::<u8>())?;

        // Память от malloc не инициализирована: заполняем нулями, чтобы
        // безопасно отдавать срез.
        // SAFETY: ptr валиден для записи len байт.
        unsafe { ptr.as_ptr().write_bytes(0, len) };

        Some(Self { ptr, len })
    }

    fn as_slice(&self) -> &[u8] {
        // SAFETY: ptr валиден для чтения len инициализированных байт, пока жив self.
        unsafe { std::slice::from_raw_parts(self.ptr.as_ptr(), self.len) }
    }

    fn as_mut_slice(&mut self) -> &mut [u8] {
        // SAFETY: &mut self гарантирует уникальный доступ.
        unsafe { std::slice::from_raw_parts_mut(self.ptr.as_ptr(), self.len) }
    }
}

impl Drop for CBuffer {
    fn drop(&mut self) {
        // SAFETY: ptr получен из malloc и освобождается ровно один раз.
        unsafe { free(self.ptr.as_ptr().cast()) };
    }
}

fn main() {
    let mut buffer = CBuffer::new(8).expect("malloc вернул null");

    buffer.as_mut_slice()[..3].copy_from_slice(b"abc");

    println!("{:?}", buffer.as_slice());
    // конец main: Drop вызовет free
}
```

Результат:

```text
[97, 98, 99, 0, 0, 0, 0, 0]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3Ac_void%3B%0Ause%20std%3A%3Aptr%3A%3ANonNull%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20malloc%28size%3A%20usize%29%20-%3E%20%2Amut%20c_void%3B%0A%20%20%20%20fn%20free%28ptr%3A%20%2Amut%20c_void%29%3B%0A%7D%0A%0A%2F%2F%2F%20%D0%92%D0%BB%D0%B0%D0%B4%D0%B5%D1%8E%D1%89%D0%B0%D1%8F%20%D0%BE%D0%B1%D1%91%D1%80%D1%82%D0%BA%D0%B0%20%D0%BD%D0%B0%D0%B4%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%D1%8E%2C%20%D0%B2%D1%8B%D0%B4%D0%B5%D0%BB%D0%B5%D0%BD%D0%BD%D0%BE%D0%B9%20C.%0Astruct%20CBuffer%20%7B%0A%20%20%20%20ptr%3A%20NonNull%3Cu8%3E%2C%0A%20%20%20%20len%3A%20usize%2C%0A%7D%0A%0Aimpl%20CBuffer%20%7B%0A%20%20%20%20fn%20new%28len%3A%20usize%29%20-%3E%20Option%3CSelf%3E%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20malloc%20%D0%BC%D0%BE%D0%B6%D0%BD%D0%BE%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%82%D1%8C%20%D1%81%20%D0%BB%D1%8E%D0%B1%D1%8B%D0%BC%20%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%D0%BE%D0%BC%3B%20%D1%80%D0%B5%D0%B7%D1%83%D0%BB%D1%8C%D1%82%D0%B0%D1%82%20%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D1%8F%D0%B5%D1%82%D1%81%D1%8F%20%D0%BD%D0%B8%D0%B6%D0%B5.%0A%20%20%20%20%20%20%20%20let%20raw%20%3D%20unsafe%20%7B%20malloc%28len%29%20%7D%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20malloc%20%D0%BC%D0%BE%D0%B6%D0%B5%D1%82%20%D0%B2%D0%B5%D1%80%D0%BD%D1%83%D1%82%D1%8C%20null%20%E2%80%94%20%D0%BF%D1%80%D0%B5%D0%B2%D1%80%D0%B0%D1%89%D0%B0%D0%B5%D0%BC%20%D1%8D%D1%82%D0%BE%20%D0%B2%20None.%0A%20%20%20%20%20%20%20%20let%20ptr%20%3D%20NonNull%3A%3Anew%28raw.cast%3A%3A%3Cu8%3E%28%29%29%3F%3B%0A%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%9F%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D0%BE%D1%82%20malloc%20%D0%BD%D0%B5%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B0%3A%20%D0%B7%D0%B0%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D0%B5%D0%BC%20%D0%BD%D1%83%D0%BB%D1%8F%D0%BC%D0%B8%2C%20%D1%87%D1%82%D0%BE%D0%B1%D1%8B%0A%20%20%20%20%20%20%20%20%2F%2F%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%20%D0%BE%D1%82%D0%B4%D0%B0%D0%B2%D0%B0%D1%82%D1%8C%20%D1%81%D1%80%D0%B5%D0%B7.%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B8%20len%20%D0%B1%D0%B0%D0%B9%D1%82.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20ptr.as_ptr%28%29.write_bytes%280%2C%20len%29%20%7D%3B%0A%0A%20%20%20%20%20%20%20%20Some%28Self%20%7B%20ptr%2C%20len%20%7D%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20as_slice%28%26self%29%20-%3E%20%26%5Bu8%5D%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%B5%D0%BD%20%D0%B4%D0%BB%D1%8F%20%D1%87%D1%82%D0%B5%D0%BD%D0%B8%D1%8F%20len%20%D0%B8%D0%BD%D0%B8%D1%86%D0%B8%D0%B0%D0%BB%D0%B8%D0%B7%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%8B%D1%85%20%D0%B1%D0%B0%D0%B9%D1%82%2C%20%D0%BF%D0%BE%D0%BA%D0%B0%20%D0%B6%D0%B8%D0%B2%20self.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20std%3A%3Aslice%3A%3Afrom_raw_parts%28self.ptr.as_ptr%28%29%2C%20self.len%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20as_mut_slice%28%26mut%20self%29%20-%3E%20%26mut%20%5Bu8%5D%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20%26mut%20self%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D1%83%D0%BD%D0%B8%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9%20%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20std%3A%3Aslice%3A%3Afrom_raw_parts_mut%28self.ptr.as_ptr%28%29%2C%20self.len%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Drop%20for%20CBuffer%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20ptr%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20malloc%20%D0%B8%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D1%80%D0%BE%D0%B2%D0%BD%D0%BE%20%D0%BE%D0%B4%D0%B8%D0%BD%20%D1%80%D0%B0%D0%B7.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20free%28self.ptr.as_ptr%28%29.cast%28%29%29%20%7D%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20buffer%20%3D%20CBuffer%3A%3Anew%288%29.expect%28%22malloc%20%D0%B2%D0%B5%D1%80%D0%BD%D1%83%D0%BB%20null%22%29%3B%0A%0A%20%20%20%20buffer.as_mut_slice%28%29%5B..3%5D.copy_from_slice%28b%22abc%22%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20buffer.as_slice%28%29%29%3B%0A%20%20%20%20%2F%2F%20%D0%BA%D0%BE%D0%BD%D0%B5%D1%86%20main%3A%20Drop%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B5%D1%82%20free%0A%7D%0A)

Что здесь важно:

- `malloc` может вернуть `null` — это превращается в `None` через `NonNull::new` и оператор `?`;
- память от `malloc` **не инициализирована**, поэтому перед выдачей среза мы заполняем её нулями (`write_bytes`); отдавать `&[u8]` над неинициализированной памятью было бы UB;
- `Drop` освобождает память ровно один раз;
- структура содержит `NonNull<u8>`, то есть сырой указатель, поэтому `CBuffer` **автоматически не `Send` и не `Sync`**. Это консервативный и безопасный выбор по умолчанию: если известно, что C-библиотека допускает передачу handle между потоками, это нужно оформить явно через `unsafe impl Send` (см. главу 44) и описать, почему это верно.

### Владение строкой: `CString::into_raw` и `from_raw`

Если C-функция должна **сохранить** переданную строку и вернуть её позже, строку нужно передать с владением. Для этого есть пара `CString::into_raw` / `CString::from_raw`:

```rust
use std::ffi::{c_char, CString};

fn main() {
    let owned = CString::new("hello").unwrap();

    // into_raw передаёт владение: Rust больше не освободит эту память сам.
    let raw: *mut c_char = owned.into_raw();

    // ... здесь raw можно передать в C, который сохранит строку ...

    // ПРАВИЛЬНО: вернуть владение Rust и дать ему освободить память.
    // SAFETY: raw получен из CString::into_raw и больше нигде не освобождался.
    let back = unsafe { CString::from_raw(raw) };
    println!("{:?}", back);

    // НЕПРАВИЛЬНО было бы вызвать C-функцию free(raw): память выделена
    // аллокатором Rust, а не malloc.
}
```

Результат:

```text
"hello"
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CString%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20owned%20%3D%20CString%3A%3Anew%28%22hello%22%29.unwrap%28%29%3B%0A%0A%20%20%20%20%2F%2F%20into_raw%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%91%D1%82%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%3A%20Rust%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B5%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B4%D0%B8%D1%82%20%D1%8D%D1%82%D1%83%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D1%81%D0%B0%D0%BC.%0A%20%20%20%20let%20raw%3A%20%2Amut%20c_char%20%3D%20owned.into_raw%28%29%3B%0A%0A%20%20%20%20%2F%2F%20...%20%D0%B7%D0%B4%D0%B5%D1%81%D1%8C%20raw%20%D0%BC%D0%BE%D0%B6%D0%BD%D0%BE%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%82%D1%8C%20%D0%B2%20C%2C%20%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D1%8B%D0%B9%20%D1%81%D0%BE%D1%85%D1%80%D0%B0%D0%BD%D0%B8%D1%82%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%20...%0A%0A%20%20%20%20%2F%2F%20%D0%9F%D0%A0%D0%90%D0%92%D0%98%D0%9B%D0%AC%D0%9D%D0%9E%3A%20%D0%B2%D0%B5%D1%80%D0%BD%D1%83%D1%82%D1%8C%20%D0%B2%D0%BB%D0%B0%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20Rust%20%D0%B8%20%D0%B4%D0%B0%D1%82%D1%8C%20%D0%B5%D0%BC%D1%83%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B4%D0%B8%D1%82%D1%8C%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C.%0A%20%20%20%20%2F%2F%20SAFETY%3A%20raw%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20CString%3A%3Ainto_raw%20%D0%B8%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%BD%D0%B8%D0%B3%D0%B4%D0%B5%20%D0%BD%D0%B5%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B0%D0%BB%D1%81%D1%8F.%0A%20%20%20%20let%20back%20%3D%20unsafe%20%7B%20CString%3A%3Afrom_raw%28raw%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20back%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%9D%D0%95%D0%9F%D0%A0%D0%90%D0%92%D0%98%D0%9B%D0%AC%D0%9D%D0%9E%20%D0%B1%D1%8B%D0%BB%D0%BE%20%D0%B1%D1%8B%20%D0%B2%D1%8B%D0%B7%D0%B2%D0%B0%D1%82%D1%8C%20C-%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8E%20free%28raw%29%3A%20%D0%BF%D0%B0%D0%BC%D1%8F%D1%82%D1%8C%20%D0%B2%D1%8B%D0%B4%D0%B5%D0%BB%D0%B5%D0%BD%D0%B0%0A%20%20%20%20%2F%2F%20%D0%B0%D0%BB%D0%BB%D0%BE%D0%BA%D0%B0%D1%82%D0%BE%D1%80%D0%BE%D0%BC%20Rust%2C%20%D0%B0%20%D0%BD%D0%B5%20malloc.%0A%7D%0A)

Главное правило: указатель, полученный из `CString::into_raw`, нужно освобождать **только** через `CString::from_raw`. Вызвать для него C-шный `free` нельзя: память выделена аллокатором Rust, а не `malloc`, и смешивание аллокаторов — UB. И наоборот: память, выделенную C через `malloc`, нельзя передавать в `Box::from_raw` или `CString::from_raw`.

---

## 47.7. Opaque handles — практический паттерн FFI

Очень распространённый C API выглядит так:

```c
typedef struct Connection Connection;

Connection *connection_create(void);
void connection_destroy(Connection *);
```

C-клиент не знает внутреннее устройство `Connection`.

В Rust можно представить такой объект как **opaque handle**. Самый простой вариант — `*mut c_void`:

```rust
use std::ffi::c_void;

struct Connection {
    ptr: *mut c_void,
}
```

Но надёжнее объявить отдельный пустой тип для каждого вида handle, чтобы компилятор не позволил перепутать указатели на разные объекты:

```rust
#[repr(C)]
struct RawConnection {
    _private: [u8; 0],
}
```

Тогда `*mut RawConnection` и, скажем, `*mut RawStream` — разные типы, и случайно передать одно вместо другого не получится. Если бы мы использовали `*mut c_void` везде, ошибка осталась бы незамеченной.

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

### Законченный пример

Чтобы пример можно было запустить одним файлом, «C-библиотеку» ниже эмулирует модуль на Rust: его функции экспортируются с C ABI и C-именами, а остальной код видит их только через `unsafe extern "C"`-объявления, как настоящий C API. (Пара `connection_create` / `connection_destroy` и безопасная обёртка с `Drop` — именно то, что нужно в задании 7.)

```rust
use std::ffi::{c_int, c_void};

// ---- "C-библиотека": ради самодостаточности эмулируем её модулем на Rust ----
mod c_side {
    use std::ffi::{c_int, c_void};

    pub struct Connection {
        pub sent: u32,
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn connection_create() -> *mut Connection {
        Box::into_raw(Box::new(Connection { sent: 0 }))
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn connection_send(conn: *mut Connection, value: c_int) -> c_int {
        if conn.is_null() {
            return -1;
        }
        // SAFETY: conn создан connection_create и ещё не уничтожен.
        unsafe { (*conn).sent += 1 };
        value * 2
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn connection_destroy(conn: *mut Connection) {
        if !conn.is_null() {
            // SAFETY: conn получен из Box::into_raw в connection_create.
            drop(unsafe { Box::from_raw(conn) });
        }
    }

    // Функция с callback и пользовательским контекстом.
    #[unsafe(no_mangle)]
    pub extern "C" fn for_each(
        count: c_int,
        callback: Option<extern "C" fn(c_int, *mut c_void)>,
        user_data: *mut c_void,
    ) {
        if let Some(callback) = callback {
            for index in 0..count {
                callback(index, user_data);
            }
        }
    }
}

// ---- Rust-сторона: объявление "внешнего" API ----
// Непрозрачный тип: Rust не знает и не должен знать его внутреннее устройство.
#[repr(C)]
struct RawConnection {
    _private: [u8; 0],
}

unsafe extern "C" {
    fn connection_create() -> *mut RawConnection;
    fn connection_send(conn: *mut RawConnection, value: c_int) -> c_int;
    fn connection_destroy(conn: *mut RawConnection);
    fn for_each(
        count: c_int,
        callback: Option<extern "C" fn(c_int, *mut c_void)>,
        user_data: *mut c_void,
    );
}

// ---- Безопасная обёртка с автоматическим освобождением ----
struct Connection {
    raw: std::ptr::NonNull<RawConnection>,
}

impl Connection {
    fn new() -> Option<Self> {
        // SAFETY: connection_create не имеет предусловий.
        let raw = unsafe { connection_create() };
        std::ptr::NonNull::new(raw).map(|raw| Self { raw })
    }

    fn send(&mut self, value: i32) -> i32 {
        // SAFETY: raw валиден, пока жив self, и &mut self гарантирует уникальный доступ.
        unsafe { connection_send(self.raw.as_ptr(), value) }
    }
}

impl Drop for Connection {
    fn drop(&mut self) {
        // SAFETY: raw получен из connection_create и освобождается ровно один раз.
        unsafe { connection_destroy(self.raw.as_ptr()) };
    }
}

// ---- Callback с состоянием: trampoline + context pointer ----
extern "C" fn trampoline(value: c_int, user_data: *mut c_void) {
    // SAFETY: user_data — это &mut Vec<i32>, переданный в main ниже;
    // for_each вызывает callback только на время своего выполнения.
    let collected = unsafe { &mut *(user_data as *mut Vec<i32>) };
    collected.push(value * 10);
}

fn main() {
    let mut conn = Connection::new().expect("не удалось создать соединение");
    println!("send(21) = {}", conn.send(21));

    let mut collected: Vec<i32> = Vec::new();

    // SAFETY: collected живёт дольше вызова, trampoline приводит указатель обратно к Vec<i32>.
    unsafe {
        for_each(4, Some(trampoline), &mut collected as *mut Vec<i32> as *mut c_void);
    }
    println!("{collected:?}");
}
```

Результат:

```text
send(21) = 42
[0, 10, 20, 30]
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_int%2C%20c_void%7D%3B%0A%0A%2F%2F%20----%20%22C-%D0%B1%D0%B8%D0%B1%D0%BB%D0%B8%D0%BE%D1%82%D0%B5%D0%BA%D0%B0%22%3A%20%D1%80%D0%B0%D0%B4%D0%B8%20%D1%81%D0%B0%D0%BC%D0%BE%D0%B4%D0%BE%D1%81%D1%82%D0%B0%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D1%81%D1%82%D0%B8%20%D1%8D%D0%BC%D1%83%D0%BB%D0%B8%D1%80%D1%83%D0%B5%D0%BC%20%D0%B5%D1%91%20%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B5%D0%BC%20%D0%BD%D0%B0%20Rust%20----%0Amod%20c_side%20%7B%0A%20%20%20%20use%20std%3A%3Affi%3A%3A%7Bc_int%2C%20c_void%7D%3B%0A%0A%20%20%20%20pub%20struct%20Connection%20%7B%0A%20%20%20%20%20%20%20%20pub%20sent%3A%20u32%2C%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Bunsafe%28no_mangle%29%5D%0A%20%20%20%20pub%20extern%20%22C%22%20fn%20connection_create%28%29%20-%3E%20%2Amut%20Connection%20%7B%0A%20%20%20%20%20%20%20%20Box%3A%3Ainto_raw%28Box%3A%3Anew%28Connection%20%7B%20sent%3A%200%20%7D%29%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Bunsafe%28no_mangle%29%5D%0A%20%20%20%20pub%20extern%20%22C%22%20fn%20connection_send%28conn%3A%20%2Amut%20Connection%2C%20value%3A%20c_int%29%20-%3E%20c_int%20%7B%0A%20%20%20%20%20%20%20%20if%20conn.is_null%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20return%20-1%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20conn%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%BD%20connection_create%20%D0%B8%20%D0%B5%D1%89%D1%91%20%D0%BD%D0%B5%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B5%D0%BD.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20%28%2Aconn%29.sent%20%2B%3D%201%20%7D%3B%0A%20%20%20%20%20%20%20%20value%20%2A%202%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Bunsafe%28no_mangle%29%5D%0A%20%20%20%20pub%20extern%20%22C%22%20fn%20connection_destroy%28conn%3A%20%2Amut%20Connection%29%20%7B%0A%20%20%20%20%20%20%20%20if%20%21conn.is_null%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20conn%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20Box%3A%3Ainto_raw%20%D0%B2%20connection_create.%0A%20%20%20%20%20%20%20%20%20%20%20%20drop%28unsafe%20%7B%20Box%3A%3Afrom_raw%28conn%29%20%7D%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20%D0%A4%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D1%81%20callback%20%D0%B8%20%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%D1%81%D0%BA%D0%B8%D0%BC%20%D0%BA%D0%BE%D0%BD%D1%82%D0%B5%D0%BA%D1%81%D1%82%D0%BE%D0%BC.%0A%20%20%20%20%23%5Bunsafe%28no_mangle%29%5D%0A%20%20%20%20pub%20extern%20%22C%22%20fn%20for_each%28%0A%20%20%20%20%20%20%20%20count%3A%20c_int%2C%0A%20%20%20%20%20%20%20%20callback%3A%20Option%3Cextern%20%22C%22%20fn%28c_int%2C%20%2Amut%20c_void%29%3E%2C%0A%20%20%20%20%20%20%20%20user_data%3A%20%2Amut%20c_void%2C%0A%20%20%20%20%29%20%7B%0A%20%20%20%20%20%20%20%20if%20let%20Some%28callback%29%20%3D%20callback%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20for%20index%20in%200..count%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20callback%28index%2C%20user_data%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0A%2F%2F%20----%20Rust-%D1%81%D1%82%D0%BE%D1%80%D0%BE%D0%BD%D0%B0%3A%20%D0%BE%D0%B1%D1%8A%D1%8F%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5%20%22%D0%B2%D0%BD%D0%B5%D1%88%D0%BD%D0%B5%D0%B3%D0%BE%22%20API%20----%0A%2F%2F%20%D0%9D%D0%B5%D0%BF%D1%80%D0%BE%D0%B7%D1%80%D0%B0%D1%87%D0%BD%D1%8B%D0%B9%20%D1%82%D0%B8%D0%BF%3A%20Rust%20%D0%BD%D0%B5%20%D0%B7%D0%BD%D0%B0%D0%B5%D1%82%20%D0%B8%20%D0%BD%D0%B5%20%D0%B4%D0%BE%D0%BB%D0%B6%D0%B5%D0%BD%20%D0%B7%D0%BD%D0%B0%D1%82%D1%8C%20%D0%B5%D0%B3%D0%BE%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B5%D0%BD%D0%BD%D0%B5%D0%B5%20%D1%83%D1%81%D1%82%D1%80%D0%BE%D0%B9%D1%81%D1%82%D0%B2%D0%BE.%0A%23%5Brepr%28C%29%5D%0Astruct%20RawConnection%20%7B%0A%20%20%20%20_private%3A%20%5Bu8%3B%200%5D%2C%0A%7D%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20connection_create%28%29%20-%3E%20%2Amut%20RawConnection%3B%0A%20%20%20%20fn%20connection_send%28conn%3A%20%2Amut%20RawConnection%2C%20value%3A%20c_int%29%20-%3E%20c_int%3B%0A%20%20%20%20fn%20connection_destroy%28conn%3A%20%2Amut%20RawConnection%29%3B%0A%20%20%20%20fn%20for_each%28%0A%20%20%20%20%20%20%20%20count%3A%20c_int%2C%0A%20%20%20%20%20%20%20%20callback%3A%20Option%3Cextern%20%22C%22%20fn%28c_int%2C%20%2Amut%20c_void%29%3E%2C%0A%20%20%20%20%20%20%20%20user_data%3A%20%2Amut%20c_void%2C%0A%20%20%20%20%29%3B%0A%7D%0A%0A%2F%2F%20----%20%D0%91%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%B0%D1%8F%20%D0%BE%D0%B1%D1%91%D1%80%D1%82%D0%BA%D0%B0%20%D1%81%20%D0%B0%D0%B2%D1%82%D0%BE%D0%BC%D0%B0%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%BC%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%D0%BC%20----%0Astruct%20Connection%20%7B%0A%20%20%20%20raw%3A%20std%3A%3Aptr%3A%3ANonNull%3CRawConnection%3E%2C%0A%7D%0A%0Aimpl%20Connection%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Option%3CSelf%3E%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20connection_create%20%D0%BD%D0%B5%20%D0%B8%D0%BC%D0%B5%D0%B5%D1%82%20%D0%BF%D1%80%D0%B5%D0%B4%D1%83%D1%81%D0%BB%D0%BE%D0%B2%D0%B8%D0%B9.%0A%20%20%20%20%20%20%20%20let%20raw%20%3D%20unsafe%20%7B%20connection_create%28%29%20%7D%3B%0A%20%20%20%20%20%20%20%20std%3A%3Aptr%3A%3ANonNull%3A%3Anew%28raw%29.map%28%7Craw%7C%20Self%20%7B%20raw%20%7D%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20send%28%26mut%20self%2C%20value%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20raw%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%B5%D0%BD%2C%20%D0%BF%D0%BE%D0%BA%D0%B0%20%D0%B6%D0%B8%D0%B2%20self%2C%20%D0%B8%20%26mut%20self%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D1%83%D0%BD%D0%B8%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9%20%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20connection_send%28self.raw.as_ptr%28%29%2C%20value%29%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Drop%20for%20Connection%20%7B%0A%20%20%20%20fn%20drop%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20raw%20%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%20%D0%B8%D0%B7%20connection_create%20%D0%B8%20%D0%BE%D1%81%D0%B2%D0%BE%D0%B1%D0%BE%D0%B6%D0%B4%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D1%80%D0%BE%D0%B2%D0%BD%D0%BE%20%D0%BE%D0%B4%D0%B8%D0%BD%20%D1%80%D0%B0%D0%B7.%0A%20%20%20%20%20%20%20%20unsafe%20%7B%20connection_destroy%28self.raw.as_ptr%28%29%29%20%7D%3B%0A%20%20%20%20%7D%0A%7D%0A%0A%2F%2F%20----%20Callback%20%D1%81%20%D1%81%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D0%B5%D0%BC%3A%20trampoline%20%2B%20context%20pointer%20----%0Aextern%20%22C%22%20fn%20trampoline%28value%3A%20c_int%2C%20user_data%3A%20%2Amut%20c_void%29%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20user_data%20%E2%80%94%20%D1%8D%D1%82%D0%BE%20%26mut%20Vec%3Ci32%3E%2C%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D0%B9%20%D0%B2%20main%20%D0%BD%D0%B8%D0%B6%D0%B5%3B%0A%20%20%20%20%2F%2F%20for_each%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20callback%20%D1%82%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE%20%D0%BD%D0%B0%20%D0%B2%D1%80%D0%B5%D0%BC%D1%8F%20%D1%81%D0%B2%D0%BE%D0%B5%D0%B3%D0%BE%20%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D0%B5%D0%BD%D0%B8%D1%8F.%0A%20%20%20%20let%20collected%20%3D%20unsafe%20%7B%20%26mut%20%2A%28user_data%20as%20%2Amut%20Vec%3Ci32%3E%29%20%7D%3B%0A%20%20%20%20collected.push%28value%20%2A%2010%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20conn%20%3D%20Connection%3A%3Anew%28%29.expect%28%22%D0%BD%D0%B5%20%D1%83%D0%B4%D0%B0%D0%BB%D0%BE%D1%81%D1%8C%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D1%82%D1%8C%20%D1%81%D0%BE%D0%B5%D0%B4%D0%B8%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5%22%29%3B%0A%20%20%20%20println%21%28%22send%2821%29%20%3D%20%7B%7D%22%2C%20conn.send%2821%29%29%3B%0A%0A%20%20%20%20let%20mut%20collected%3A%20Vec%3Ci32%3E%20%3D%20Vec%3A%3Anew%28%29%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20collected%20%D0%B6%D0%B8%D0%B2%D1%91%D1%82%20%D0%B4%D0%BE%D0%BB%D1%8C%D1%88%D0%B5%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B0%2C%20trampoline%20%D0%BF%D1%80%D0%B8%D0%B2%D0%BE%D0%B4%D0%B8%D1%82%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20%D0%BE%D0%B1%D1%80%D0%B0%D1%82%D0%BD%D0%BE%20%D0%BA%20Vec%3Ci32%3E.%0A%20%20%20%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20for_each%284%2C%20Some%28trampoline%29%2C%20%26mut%20collected%20as%20%2Amut%20Vec%3Ci32%3E%20as%20%2Amut%20c_void%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20println%21%28%22%7Bcollected%3A%3F%7D%22%29%3B%0A%7D%0A)

Посмотрите, как распределены обязанности:

- `Connection::new` обрабатывает `null`;
- `Connection::send` требует `&mut self`, поэтому одновременный доступ из безопасного кода невозможен;
- `Drop` вызывает `connection_destroy` ровно один раз;
- снаружи нет ни одного `unsafe`.

Вторая половина примера — callback с контекстом — разобрана в следующем разделе.

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

`None` соответствует отсутствующему callback. Это работает потому, что указатель на функцию в Rust не может быть нулевым, а значит `Option<extern "C" fn(...)>` имеет тот же размер и представление, что и C-шный указатель на функцию, где `None` — это `NULL`.

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

Результат:

```text
callback: 42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20Option%3Cextern%20%22C%22%20fn%28i32%29%3E%3B%0A%0Aextern%20%22C%22%20fn%20print_value%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22callback%3A%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20run_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20if%20let%20Some%28callback%29%20%3D%20callback%20%7B%0A%20%20%20%20%20%20%20%20callback%2842%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20run_callback%28Some%28print_value%29%29%3B%0A%20%20%20%20run_callback%28None%29%3B%0A%7D%0A)

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

Это фундаментальный паттерн для callback-based FFI. В примере из раздела 47.7 он реализован целиком:

```rust
extern "C" fn trampoline(value: c_int, user_data: *mut c_void) {
    // SAFETY: user_data — это &mut Vec<i32>, переданный в main;
    // for_each вызывает callback только на время своего выполнения.
    let collected = unsafe { &mut *(user_data as *mut Vec<i32>) };
    collected.push(value * 10);
}
```

а вызов выглядит так:

```rust
unsafe {
    for_each(4, Some(trampoline), &mut collected as *mut Vec<i32> as *mut c_void);
}
```

Функция `trampoline` — обычная `extern "C" fn`, которая превращает `*mut c_void` обратно в `&mut Vec<i32>`. Корректность держится на двух условиях: C-код передаёт обратно **тот же** указатель, который получил, и не вызывает callback после того, как Rust-состояние было уничтожено.

Паника внутри такого callback не должна выходить наружу: об этом в разделе 47.12.

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
    let point = Point { x: 10.0, y: 20.0 };

    println!("{point:?}");
}
```

Результат:

```text
Point { x: 10.0, y: 20.0 }
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Brepr%28C%29%5D%0A%23%5Bderive%28Debug%2C%20Copy%2C%20Clone%29%5D%0Astruct%20Point%20%7B%0A%20%20%20%20x%3A%20f64%2C%0A%20%20%20%20y%3A%20f64%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20point%20%3D%20Point%20%7B%20x%3A%2010.0%2C%20y%3A%2020.0%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7Bpoint%3A%3F%7D%22%29%3B%0A%7D%0A)

В C соответствующая структура:

```c
struct Point {
    double x;
    double y;
};
```

### Как увидеть разницу

Структура с полями разного размера показывает, зачем нужен `repr(C)`:

```rust
use std::mem::{align_of, offset_of, size_of};

#[allow(dead_code)]
struct RustLayout {
    a: u8,
    b: u32,
    c: u8,
}

#[repr(C)]
#[allow(dead_code)]
struct CLayout {
    a: u8,
    b: u32,
    c: u8,
}

fn main() {
    println!("CLayout:    size = {}, align = {}", size_of::<CLayout>(), align_of::<CLayout>());
    println!(
        "  offsets: a = {}, b = {}, c = {}",
        offset_of!(CLayout, a),
        offset_of!(CLayout, b),
        offset_of!(CLayout, c)
    );

    // Для обычной структуры Rust вправе переставлять поля;
    // конкретные числа ниже — не гарантия, а наблюдение.
    println!("RustLayout: size = {}, align = {}", size_of::<RustLayout>(), align_of::<RustLayout>());
}
```

Результат (x86-64 Linux):

```text
CLayout:    size = 12, align = 4
  offsets: a = 0, b = 4, c = 8
RustLayout: size = 8, align = 4
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Amem%3A%3A%7Balign_of%2C%20offset_of%2C%20size_of%7D%3B%0A%0A%23%5Ballow%28dead_code%29%5D%0Astruct%20RustLayout%20%7B%0A%20%20%20%20a%3A%20u8%2C%0A%20%20%20%20b%3A%20u32%2C%0A%20%20%20%20c%3A%20u8%2C%0A%7D%0A%0A%23%5Brepr%28C%29%5D%0A%23%5Ballow%28dead_code%29%5D%0Astruct%20CLayout%20%7B%0A%20%20%20%20a%3A%20u8%2C%0A%20%20%20%20b%3A%20u32%2C%0A%20%20%20%20c%3A%20u8%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22CLayout%3A%20%20%20%20size%20%3D%20%7B%7D%2C%20align%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3CCLayout%3E%28%29%2C%20align_of%3A%3A%3CCLayout%3E%28%29%29%3B%0A%20%20%20%20println%21%28%0A%20%20%20%20%20%20%20%20%22%20%20offsets%3A%20a%20%3D%20%7B%7D%2C%20b%20%3D%20%7B%7D%2C%20c%20%3D%20%7B%7D%22%2C%0A%20%20%20%20%20%20%20%20offset_of%21%28CLayout%2C%20a%29%2C%0A%20%20%20%20%20%20%20%20offset_of%21%28CLayout%2C%20b%29%2C%0A%20%20%20%20%20%20%20%20offset_of%21%28CLayout%2C%20c%29%0A%20%20%20%20%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%94%D0%BB%D1%8F%20%D0%BE%D0%B1%D1%8B%D1%87%D0%BD%D0%BE%D0%B9%20%D1%81%D1%82%D1%80%D1%83%D0%BA%D1%82%D1%83%D1%80%D1%8B%20Rust%20%D0%B2%D0%BF%D1%80%D0%B0%D0%B2%D0%B5%20%D0%BF%D0%B5%D1%80%D0%B5%D1%81%D1%82%D0%B0%D0%B2%D0%BB%D1%8F%D1%82%D1%8C%20%D0%BF%D0%BE%D0%BB%D1%8F%3B%0A%20%20%20%20%2F%2F%20%D0%BA%D0%BE%D0%BD%D0%BA%D1%80%D0%B5%D1%82%D0%BD%D1%8B%D0%B5%20%D1%87%D0%B8%D1%81%D0%BB%D0%B0%20%D0%BD%D0%B8%D0%B6%D0%B5%20%E2%80%94%20%D0%BD%D0%B5%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%8F%2C%20%D0%B0%20%D0%BD%D0%B0%D0%B1%D0%BB%D1%8E%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5.%0A%20%20%20%20println%21%28%22RustLayout%3A%20size%20%3D%20%7B%7D%2C%20align%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3CRustLayout%3E%28%29%2C%20align_of%3A%3A%3CRustLayout%3E%28%29%29%3B%0A%7D%0A)

`#[repr(C)]` раскладывает поля строго в порядке объявления, добавляя padding по правилам C: `a` в начале, три байта прокладки, `b` по смещению 4, `c` по смещению 8 и ещё три байта хвостового padding до кратности 4. Размер 12 совпадает с тем, что даст C-компилятор для такой структуры. Обычная структура Rust вправе переставлять поля, чтобы уменьшить размер (здесь получилось 8), и **никакой гарантии конкретного layout не даёт**: числа для `RustLayout` — наблюдение, а не контракт.

### Типичные `repr`

| Представление          | Назначение                                            |
| ---------------------- | ----------------------------------------------------- |
| `#[repr(C)]`           | C-compatible layout                                   |
| `#[repr(transparent)]` | тот же ABI/layout, что у единственного значимого поля |
| `#[repr(u8)]`          | фиксированное представление discriminant              |
| `#[repr(packed)]`      | уменьшение/устранение padding, требует осторожности   |
| `#[repr(align(N))]`    | увеличение alignment                                  |

`#[repr(packed)]` особенно опасен при работе с ссылками на поля, поскольку обычные требования выравнивания могут быть нарушены.

### Enum и `bool` из C

`#[repr(u8)]`, `#[repr(C)]` и подобные для `enum` задают размер и значения discriminant, но не избавляют от главной ловушки: если C передаст число, которого нет среди вариантов Rust-`enum`, получится недопустимое значение типа, то есть UB. Поэтому значения из C принимают как целое число и преобразуют в `enum` явной проверкой (`TryFrom`), а не объявляют параметр типом `enum`.

Типы `bool` в Rust и `_Bool` в C совместимы по ABI (значение 0 или 1). Но Rust `char` — это 4-байтовое значение Unicode, а не C `char`: для байтов C используют `c_char`.

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

(между `port` и `timeout` в такой структуре будет два байта padding: именно так её разместит и C-компилятор).

Или простые C-compatible типы:

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

Компилятор поможет заметить такие ошибки: lint `improper_ctypes` (для импортируемых функций) и `improper_ctypes_definitions` (для экспортируемых) по умолчанию выдают предупреждения, как мы увидим в эксперименте 4.

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

Допустим, внешняя библиотека предоставляет чистую функцию:

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

Теперь остальные части программы не должны знать о FFI. Такая обёртка корректна, потому что у `calculate` нет предусловий: любое значение `f64` допустимо.

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

Безопасная функция должна гарантировать, что **любой** допустимый вход удовлетворяет требованиям unsafe-операции.

Попробуем «улучшить» этот пример проверкой `null`:

```rust
// НЕПРАВИЛЬНО: безопасная функция, которая разыменовывает произвольный указатель.
fn read_value(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        return None;
    }

    // Из безопасного кода можно передать висячий указатель -> UB.
    Some(unsafe { *ptr })
}

fn main() {
    let dangling = {
        let x = 5;
        &x as *const i32
    };

    // Это компилируется без единого unsafe в месте вызова!
    println!("{:?}", read_value(dangling));
}
```

[▶ Открыть пример в Rust Playground (компилируется, но содержит проблему корректности)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%20%D0%9D%D0%95%D0%9F%D0%A0%D0%90%D0%92%D0%98%D0%9B%D0%AC%D0%9D%D0%9E%3A%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%B0%D1%8F%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%2C%20%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D0%B0%D1%8F%20%D1%80%D0%B0%D0%B7%D1%8B%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BF%D1%80%D0%BE%D0%B8%D0%B7%D0%B2%D0%BE%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C.%0Afn%20read_value%28ptr%3A%20%2Aconst%20i32%29%20-%3E%20Option%3Ci32%3E%20%7B%0A%20%20%20%20if%20ptr.is_null%28%29%20%7B%0A%20%20%20%20%20%20%20%20return%20None%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%2F%2F%20%D0%98%D0%B7%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%D0%B3%D0%BE%20%D0%BA%D0%BE%D0%B4%D0%B0%20%D0%BC%D0%BE%D0%B6%D0%BD%D0%BE%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D1%82%D1%8C%20%D0%B2%D0%B8%D1%81%D1%8F%D1%87%D0%B8%D0%B9%20%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%20-%3E%20UB.%0A%20%20%20%20Some%28unsafe%20%7B%20%2Aptr%20%7D%29%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20dangling%20%3D%20%7B%0A%20%20%20%20%20%20%20%20let%20x%20%3D%205%3B%0A%20%20%20%20%20%20%20%20%26x%20as%20%2Aconst%20i32%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20%2F%2F%20%D0%AD%D1%82%D0%BE%20%D0%BA%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D0%B8%D1%80%D1%83%D0%B5%D1%82%D1%81%D1%8F%20%D0%B1%D0%B5%D0%B7%20%D0%B5%D0%B4%D0%B8%D0%BD%D0%BE%D0%B3%D0%BE%20unsafe%20%D0%B2%20%D0%BC%D0%B5%D1%81%D1%82%D0%B5%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B0%21%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20read_value%28dangling%29%29%3B%0A%7D%0A)

Этот код компилируется без единого `unsafe` в месте вызова, и в нашем запуске даже напечатал `Some(5)`. Но он **некорректен**: `read_value` — безопасная функция, а значит, любой безопасный код вправе её вызвать с любым указателем, в том числе с висячим, как в `main`. Проверка `is_null` ловит только один из многих некорректных входов. Такая функция называется **unsound**: безопасный API, из которого можно получить UB.

Есть два правильных исхода.

**Вариант 1.** Оставить функцию `unsafe fn` и описать контракт в `# Safety`.

**Вариант 2.** Сделать функцию по-настоящему безопасной, приняв **безопасный тип**, который сам гарантирует нужные условия: ссылку или срез.

```rust
/// ПРАВИЛЬНО (вариант 1): функция помечена unsafe, контракт описан в # Safety.
///
/// # Safety
///
/// `ptr` must be non-null, aligned, and point to a valid initialized `i32`.
unsafe fn read_value(ptr: *const i32) -> i32 {
    // SAFETY: условия переданы вызывающему через контракт.
    unsafe { *ptr }
}

/// ПРАВИЛЬНО (вариант 2): безопасная функция принимает безопасные типы.
/// Ссылка и срез уже гарантируют валидность, выравнивание и длину.
fn sum(values: &[i32]) -> i32 {
    let mut total = 0;
    for index in 0..values.len() {
        // SAFETY: index < values.len().
        total += unsafe { *values.as_ptr().add(index) };
    }
    total
}

fn main() {
    let value = 7;
    // SAFETY: &value — валидная выровненная ссылка.
    println!("{}", unsafe { read_value(&value) });

    println!("{}", sum(&[1, 2, 3, 4]));
}
```

Результат:

```text
7
10
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%2F%20%D0%9F%D0%A0%D0%90%D0%92%D0%98%D0%9B%D0%AC%D0%9D%D0%9E%20%28%D0%B2%D0%B0%D1%80%D0%B8%D0%B0%D0%BD%D1%82%201%29%3A%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D0%BF%D0%BE%D0%BC%D0%B5%D1%87%D0%B5%D0%BD%D0%B0%20unsafe%2C%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%20%D0%BE%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%20%D0%B2%20%23%20Safety.%0A%2F%2F%2F%0A%2F%2F%2F%20%23%20Safety%0A%2F%2F%2F%0A%2F%2F%2F%20%60ptr%60%20must%20be%20non-null%2C%20aligned%2C%20and%20point%20to%20a%20valid%20initialized%20%60i32%60.%0Aunsafe%20fn%20read_value%28ptr%3A%20%2Aconst%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%D1%83%D1%81%D0%BB%D0%BE%D0%B2%D0%B8%D1%8F%20%D0%BF%D0%B5%D1%80%D0%B5%D0%B4%D0%B0%D0%BD%D1%8B%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%8E%D1%89%D0%B5%D0%BC%D1%83%20%D1%87%D0%B5%D1%80%D0%B5%D0%B7%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82.%0A%20%20%20%20unsafe%20%7B%20%2Aptr%20%7D%0A%7D%0A%0A%2F%2F%2F%20%D0%9F%D0%A0%D0%90%D0%92%D0%98%D0%9B%D0%AC%D0%9D%D0%9E%20%28%D0%B2%D0%B0%D1%80%D0%B8%D0%B0%D0%BD%D1%82%202%29%3A%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%B0%D1%8F%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D1%8F%20%D0%BF%D1%80%D0%B8%D0%BD%D0%B8%D0%BC%D0%B0%D0%B5%D1%82%20%D0%B1%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D1%8B%D0%B5%20%D1%82%D0%B8%D0%BF%D1%8B.%0A%2F%2F%2F%20%D0%A1%D1%81%D1%8B%D0%BB%D0%BA%D0%B0%20%D0%B8%20%D1%81%D1%80%D0%B5%D0%B7%20%D1%83%D0%B6%D0%B5%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D1%8E%D1%82%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%BE%D1%81%D1%82%D1%8C%2C%20%D0%B2%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20%D0%B8%20%D0%B4%D0%BB%D0%B8%D0%BD%D1%83.%0Afn%20sum%28values%3A%20%26%5Bi32%5D%29%20-%3E%20i32%20%7B%0A%20%20%20%20let%20mut%20total%20%3D%200%3B%0A%20%20%20%20for%20index%20in%200..values.len%28%29%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20SAFETY%3A%20index%20%3C%20values.len%28%29.%0A%20%20%20%20%20%20%20%20total%20%2B%3D%20unsafe%20%7B%20%2Avalues.as_ptr%28%29.add%28index%29%20%7D%3B%0A%20%20%20%20%7D%0A%20%20%20%20total%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20value%20%3D%207%3B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20%26value%20%E2%80%94%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%B0%D1%8F%20%D0%B2%D1%8B%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%BD%D0%BD%D0%B0%D1%8F%20%D1%81%D1%81%D1%8B%D0%BB%D0%BA%D0%B0.%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20unsafe%20%7B%20read_value%28%26value%29%20%7D%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20sum%28%26%5B1%2C%202%2C%203%2C%204%5D%29%29%3B%0A%7D%0A)

### Пример идеальной безопасной обёртки: `strlen`

C-функция `strlen` требует, чтобы указатель вёл на NUL-терминированную строку. Это в точности то, что гарантирует тип `&CStr`:

```rust
use std::ffi::{c_char, CStr};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

/// Безопасная обёртка: &CStr гарантирует валидную C-строку с нулевым терминатором,
/// поэтому любое значение допустимого типа удовлетворяет контракту strlen.
fn c_strlen(s: &CStr) -> usize {
    // SAFETY: s.as_ptr() указывает на NUL-терминированную строку, живущую не меньше вызова.
    unsafe { strlen(s.as_ptr()) }
}

fn main() {
    println!("{}", c_strlen(c"hello"));
}
```

Результат:

```text
5
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CStr%7D%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20strlen%28s%3A%20%2Aconst%20c_char%29%20-%3E%20usize%3B%0A%7D%0A%0A%2F%2F%2F%20%D0%91%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%B0%D1%8F%20%D0%BE%D0%B1%D1%91%D1%80%D1%82%D0%BA%D0%B0%3A%20%26CStr%20%D0%B3%D0%B0%D1%80%D0%B0%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B5%D1%82%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D1%83%D1%8E%20C-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%20%D1%81%20%D0%BD%D1%83%D0%BB%D0%B5%D0%B2%D1%8B%D0%BC%20%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D1%82%D0%BE%D1%80%D0%BE%D0%BC%2C%0A%2F%2F%2F%20%D0%BF%D0%BE%D1%8D%D1%82%D0%BE%D0%BC%D1%83%20%D0%BB%D1%8E%D0%B1%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%B4%D0%BE%D0%BF%D1%83%D1%81%D1%82%D0%B8%D0%BC%D0%BE%D0%B3%D0%BE%20%D1%82%D0%B8%D0%BF%D0%B0%20%D1%83%D0%B4%D0%BE%D0%B2%D0%BB%D0%B5%D1%82%D0%B2%D0%BE%D1%80%D1%8F%D0%B5%D1%82%20%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82%D1%83%20strlen.%0Afn%20c_strlen%28s%3A%20%26CStr%29%20-%3E%20usize%20%7B%0A%20%20%20%20%2F%2F%20SAFETY%3A%20s.as_ptr%28%29%20%D1%83%D0%BA%D0%B0%D0%B7%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%BD%D0%B0%20NUL-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%83%D1%8E%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%2C%20%D0%B6%D0%B8%D0%B2%D1%83%D1%89%D1%83%D1%8E%20%D0%BD%D0%B5%20%D0%BC%D0%B5%D0%BD%D1%8C%D1%88%D0%B5%20%D0%B2%D1%8B%D0%B7%D0%BE%D0%B2%D0%B0.%0A%20%20%20%20unsafe%20%7B%20strlen%28s.as_ptr%28%29%29%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20c_strlen%28c%22hello%22%29%29%3B%0A%7D%0A)

Обёртка `c_strlen` безопасна: принять `&CStr` можно только, если строка валидна и терминирована, а значит все условия `strlen` выполнены **для любого значения допустимого типа**. Именно по этому признаку надо проверять любую FFI-обёртку: можно ли, имея только безопасный код, вызвать её так, чтобы нарушить контракт C-функции?

Именно поэтому FFI-обёртка должна документировать и проверять **контракт безопасности**.

---

## 47.11. Строки и владение: типичная FFI-ошибка

Рассмотрим опасный код:

```rust
use std::ffi::{c_char, CString};

struct Config {
    host: *const c_char,
}

fn to_c(config: &str) -> Config {
    let host = CString::new(config).unwrap();

    Config {
        host: host.as_ptr(), // host уничтожается в конце функции
    }
}

fn main() {
    let config = to_c("localhost");
    println!("{}", config.host.is_null());
}
```

[▶ Открыть пример в Rust Playground (компилируется, но содержит проблему корректности)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CString%7D%3B%0A%0Astruct%20Config%20%7B%0A%20%20%20%20host%3A%20%2Aconst%20c_char%2C%0A%7D%0A%0Afn%20to_c%28config%3A%20%26str%29%20-%3E%20Config%20%7B%0A%20%20%20%20let%20host%20%3D%20CString%3A%3Anew%28config%29.unwrap%28%29%3B%0A%0A%20%20%20%20Config%20%7B%0A%20%20%20%20%20%20%20%20host%3A%20host.as_ptr%28%29%2C%20%2F%2F%20host%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%B2%20%D0%BA%D0%BE%D0%BD%D1%86%D0%B5%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20config%20%3D%20to_c%28%22localhost%22%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20config.host.is_null%28%29%29%3B%0A%7D%0A)

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

Это классическая ошибка FFI. Обратите внимание, что компилятор здесь **ничего не сказал**: сохранить сырой указатель — безопасная операция, а UB возникнет только при разыменовании внешним кодом.

Есть и более коварный вариант той же ошибки: временное значение.

```rust
use std::ffi::{c_char, CString};

fn main() {
    // Временный CString уничтожается в конце этого оператора.
    let ptr: *const c_char = CString::new("hello").unwrap().as_ptr();

    println!("{}", ptr.is_null());
}
```

[▶ Открыть пример с предупреждением в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CString%7D%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20%D0%92%D1%80%D0%B5%D0%BC%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9%20CString%20%D1%83%D0%BD%D0%B8%D1%87%D1%82%D0%BE%D0%B6%D0%B0%D0%B5%D1%82%D1%81%D1%8F%20%D0%B2%20%D0%BA%D0%BE%D0%BD%D1%86%D0%B5%20%D1%8D%D1%82%D0%BE%D0%B3%D0%BE%20%D0%BE%D0%BF%D0%B5%D1%80%D0%B0%D1%82%D0%BE%D1%80%D0%B0.%0A%20%20%20%20let%20ptr%3A%20%2Aconst%20c_char%20%3D%20CString%3A%3Anew%28%22hello%22%29.unwrap%28%29.as_ptr%28%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20ptr.is_null%28%29%29%3B%0A%7D%0A)

Здесь компилятор уже умеет предупредить:

```text
warning: this creates a dangling pointer because temporary `CString` is dropped at end of statement
  |
5 |     let ptr: *const c_char = CString::new("hello").unwrap().as_ptr();
  |                              ------------------------------ ^^^^^^ pointer created here
  |                              |
  |                              this `CString` is dropped at end of statement
  |
  = help: bind the `CString` to a variable such that it outlives the pointer returned by `as_ptr`
  = note: `#[warn(dangling_pointers_from_temporaries)]` on by default
```

Но этот lint ловит только очевидные случаи, а не любую потерю владельца.

### Правильный подход

Нужно хранить `CString` столько же, сколько требуется указателю:

```rust
use std::ffi::{c_char, CString};

struct CConfig {
    host: CString,
}

impl CConfig {
    fn new(host: &str) -> Self {
        Self {
            host: CString::new(host).unwrap(),
        }
    }

    fn as_ptr(&self) -> *const c_char {
        self.host.as_ptr()
    }
}

fn main() {
    let config = CConfig::new("localhost");

    println!("{:p}", config.as_ptr());
}
```

(Значение указателя в выводе отличается от запуска к запуску.)

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CString%7D%3B%0A%0Astruct%20CConfig%20%7B%0A%20%20%20%20host%3A%20CString%2C%0A%7D%0A%0Aimpl%20CConfig%20%7B%0A%20%20%20%20fn%20new%28host%3A%20%26str%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20host%3A%20CString%3A%3Anew%28host%29.unwrap%28%29%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20as_ptr%28%26self%29%20-%3E%20%2Aconst%20c_char%20%7B%0A%20%20%20%20%20%20%20%20self.host.as_ptr%28%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20config%20%3D%20CConfig%3A%3Anew%28%22localhost%22%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%3Ap%7D%22%2C%20config.as_ptr%28%29%29%3B%0A%7D%0A)

Это общий принцип:

> **Владеющий Rust-объект должен жить не меньше, чем указатель, который он предоставляет внешнему коду.**

---

## 47.12. Экспорт Rust-функций для C

FFI работает и в обратную сторону.

Rust может предоставить API, которое вызывается из C.

Например:

```rust
// SAFETY: символ `multiply` не должен совпадать ни с каким другим в программе.
#[unsafe(no_mangle)]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}

fn main() {
    println!("{}", multiply(6, 7));
}
```

Результат:

```text
42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%20SAFETY%3A%20%D1%81%D0%B8%D0%BC%D0%B2%D0%BE%D0%BB%20%60multiply%60%20%D0%BD%D0%B5%20%D0%B4%D0%BE%D0%BB%D0%B6%D0%B5%D0%BD%20%D1%81%D0%BE%D0%B2%D0%BF%D0%B0%D0%B4%D0%B0%D1%82%D1%8C%20%D0%BD%D0%B8%20%D1%81%20%D0%BA%D0%B0%D0%BA%D0%B8%D0%BC%20%D0%B4%D1%80%D1%83%D0%B3%D0%B8%D0%BC%20%D0%B2%20%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B5.%0A%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20multiply%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2A%20b%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20multiply%286%2C%207%29%29%3B%0A%7D%0A)

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

Атрибут «unsafe», потому что экспортируемое имя попадает в глобальное пространство символов: если другая библиотека или другой модуль экспортирует функцию с тем же именем, программа либо не соберётся, либо будет вызвана не та функция. Ответственность за уникальность имени лежит на программисте, и полезно написать об этом комментарий (как в примере выше).

### Как собрать библиотеку для C

Сама функция ещё не делает Rust-код библиотекой, которую можно подключить из C. Для этого указывают тип крейта в `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib", "staticlib"]
```

- `cdylib` — динамическая библиотека (`.so`, `.dylib`, `.dll`) для загрузки из других языков;
- `staticlib` — статическая библиотека (`.a`, `.lib`) для линковки внутрь C-программы.

Проверим на полном примере. `src/lib.rs`:

```rust
use std::ffi::{c_char, CStr};

#[unsafe(no_mangle)]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}

/// Возвращает число символов в C-строке или -1, если указатель нулевой
/// или строка не является корректным UTF-8.
///
/// # Safety
/// `s` должен быть либо null, либо указывать на NUL-терминированную строку.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn utf8_len(s: *const c_char) -> isize {
    if s.is_null() {
        return -1;
    }
    // SAFETY: контракт функции требует валидную NUL-терминированную строку.
    let c_str = unsafe { CStr::from_ptr(s) };
    match c_str.to_str() {
        Ok(text) => text.chars().count() as isize,
        Err(_) => -1,
    }
}
```

C-заголовок `mathlib.h` (для больших API его генерируют инструментом `cbindgen`):

```c
#include <stdint.h>
#include <stddef.h>
int32_t multiply(int32_t a, int32_t b);
ptrdiff_t utf8_len(const char *s);
```

C-программа `main.c`:

```c
#include <stdio.h>
#include "mathlib.h"

int main(void) {
    printf("%d\n", multiply(6, 7));
    printf("%td\n", utf8_len("Привет"));
    printf("%td\n", utf8_len(NULL));
    return 0;
}
```

Сборка и запуск (Linux):

```bash
cargo build
gcc main.c -L target/debug -lmathlib -o cmain
LD_LIBRARY_PATH=target/debug ./cmain
```

Результат:

```text
42
6
-1
```

Строка «Привет» содержит 6 символов, хотя в UTF-8 занимает 12 байт: `utf8_len` считает именно символы. Для статической линковки вместо `-lmathlib` указывают файл `target/debug/libmathlib.a` и системные библиотеки (`-lpthread -ldl -lm`); результат тот же.

### Паники и FFI

Одно из важнейших правил экспорта: **паника не должна пересекать границу FFI**. C не знает о размотке стека (unwinding) Rust, и если бы паника вышла из `extern "C"` функции, это было бы UB. Поэтому Rust останавливает её: программа аварийно завершается.

```rust
#[unsafe(no_mangle)]
pub extern "C" fn risky(value: i32) -> i32 {
    if value < 0 {
        panic!("отрицательное значение");
    }
    value * 2
}

fn main() {
    println!("{}", risky(21));
    // Паника внутри extern "C" fn не может «пройти» через границу FFI:
    println!("{}", risky(-1));
    println!("эта строка не напечатается");
}
```

[▶ Открыть пример в Rust Playground (программа завершится аварийно)](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20risky%28value%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20if%20value%20%3C%200%20%7B%0A%20%20%20%20%20%20%20%20panic%21%28%22%D0%BE%D1%82%D1%80%D0%B8%D1%86%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%22%29%3B%0A%20%20%20%20%7D%0A%20%20%20%20value%20%2A%202%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20risky%2821%29%29%3B%0A%20%20%20%20%2F%2F%20%D0%9F%D0%B0%D0%BD%D0%B8%D0%BA%D0%B0%20%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B8%20extern%20%22C%22%20fn%20%D0%BD%D0%B5%20%D0%BC%D0%BE%D0%B6%D0%B5%D1%82%20%C2%AB%D0%BF%D1%80%D0%BE%D0%B9%D1%82%D0%B8%C2%BB%20%D1%87%D0%B5%D1%80%D0%B5%D0%B7%20%D0%B3%D1%80%D0%B0%D0%BD%D0%B8%D1%86%D1%83%20FFI%3A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20risky%28-1%29%29%3B%0A%20%20%20%20println%21%28%22%D1%8D%D1%82%D0%B0%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0%20%D0%BD%D0%B5%20%D0%BD%D0%B0%D0%BF%D0%B5%D1%87%D0%B0%D1%82%D0%B0%D0%B5%D1%82%D1%81%D1%8F%22%29%3B%0A%7D%0A)

Результат:

```text
42

thread 'main' panicked at src/main.rs:4:9:
отрицательное значение
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

thread 'main' panicked at library/core/src/panicking.rs:225:5:
panic in a function that cannot unwind
```

и затем процесс завершается аварийно (`Aborted`). Строка «эта строка не напечатается» действительно не печатается.

Правильный способ — перехватывать панику на границе и превращать её в код ошибки, как это принято в C:

```rust
use std::panic::{self, AssertUnwindSafe};

#[unsafe(no_mangle)]
pub extern "C" fn safe_risky(value: i32) -> i32 {
    // Ни одна паника не должна пересечь границу FFI.
    let result = panic::catch_unwind(AssertUnwindSafe(|| {
        if value < 0 {
            panic!("отрицательное значение");
        }
        value * 2
    }));

    match result {
        Ok(doubled) => doubled,
        Err(_) => -1, // C-стиль: код ошибки вместо паники
    }
}

fn main() {
    println!("{}", safe_risky(21));
    println!("{}", safe_risky(-1));
}
```

Результат (сообщение о панике печатается обработчиком в stderr, но программа продолжает работу):

```text
42
-1
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Apanic%3A%3A%7Bself%2C%20AssertUnwindSafe%7D%3B%0A%0A%23%5Bunsafe%28no_mangle%29%5D%0Apub%20extern%20%22C%22%20fn%20safe_risky%28value%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20%2F%2F%20%D0%9D%D0%B8%20%D0%BE%D0%B4%D0%BD%D0%B0%20%D0%BF%D0%B0%D0%BD%D0%B8%D0%BA%D0%B0%20%D0%BD%D0%B5%20%D0%B4%D0%BE%D0%BB%D0%B6%D0%BD%D0%B0%20%D0%BF%D0%B5%D1%80%D0%B5%D1%81%D0%B5%D1%87%D1%8C%20%D0%B3%D1%80%D0%B0%D0%BD%D0%B8%D1%86%D1%83%20FFI.%0A%20%20%20%20let%20result%20%3D%20panic%3A%3Acatch_unwind%28AssertUnwindSafe%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20if%20value%20%3C%200%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20panic%21%28%22%D0%BE%D1%82%D1%80%D0%B8%D1%86%D0%B0%D1%82%D0%B5%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D0%B5%22%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20value%20%2A%202%0A%20%20%20%20%7D%29%29%3B%0A%0A%20%20%20%20match%20result%20%7B%0A%20%20%20%20%20%20%20%20Ok%28doubled%29%20%3D%3E%20doubled%2C%0A%20%20%20%20%20%20%20%20Err%28_%29%20%3D%3E%20-1%2C%20%2F%2F%20C-%D1%81%D1%82%D0%B8%D0%BB%D1%8C%3A%20%D0%BA%D0%BE%D0%B4%20%D0%BE%D1%88%D0%B8%D0%B1%D0%BA%D0%B8%20%D0%B2%D0%BC%D0%B5%D1%81%D1%82%D0%BE%20%D0%BF%D0%B0%D0%BD%D0%B8%D0%BA%D0%B8%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20safe_risky%2821%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20safe_risky%28-1%29%29%3B%0A%7D%0A)

Это правило касается обоих направлений: callback, который C вызывает из Rust (раздел 47.8), тоже не должен паниковать наружу. Если же вы сознательно хотите пропустить unwinding через границу (например, между Rust и C++ с исключениями), используйте ABI `"C-unwind"` вместо `"C"` с обеих сторон, учитывая, что такое поведение нужно согласовать с вызываемым кодом.

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

Особенно важно это для типов, размер которых зависит от платформы:

```rust
use std::ffi::{c_char, c_int, c_long, c_void};
use std::mem::size_of;

fn main() {
    println!("c_char = {}", size_of::<c_char>());
    println!("c_int  = {}", size_of::<c_int>());
    // Размер c_long зависит от платформы: на Linux x86-64 это 8, на Windows — 4.
    println!("c_long = {}", size_of::<c_long>());
    println!("*mut c_void = {}", size_of::<*mut c_void>());
    println!("char (Rust) = {}", size_of::<char>());
}
```

Результат (Linux x86-64):

```text
c_char = 1
c_int  = 4
c_long = 8
*mut c_void = 8
char (Rust) = 4
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20c_int%2C%20c_long%2C%20c_void%7D%3B%0Ause%20std%3A%3Amem%3A%3Asize_of%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22c_char%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3Cc_char%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22c_int%20%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3Cc_int%3E%28%29%29%3B%0A%20%20%20%20%2F%2F%20%D0%A0%D0%B0%D0%B7%D0%BC%D0%B5%D1%80%20c_long%20%D0%B7%D0%B0%D0%B2%D0%B8%D1%81%D0%B8%D1%82%20%D0%BE%D1%82%20%D0%BF%D0%BB%D0%B0%D1%82%D1%84%D0%BE%D1%80%D0%BC%D1%8B%3A%20%D0%BD%D0%B0%20Linux%20x86-64%20%D1%8D%D1%82%D0%BE%208%2C%20%D0%BD%D0%B0%20Windows%20%E2%80%94%204.%0A%20%20%20%20println%21%28%22c_long%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3Cc_long%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22%2Amut%20c_void%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3C%2Amut%20c_void%3E%28%29%29%3B%0A%20%20%20%20println%21%28%22char%20%28Rust%29%20%3D%20%7B%7D%22%2C%20size_of%3A%3A%3Cchar%3E%28%29%29%3B%0A%7D%0A)

`c_long` на Linux x86-64 занимает 8 байт, а на Windows — 4. Если в объявлении написать `i64`, на Windows программа получит несоответствие размеров. Также обратите внимание на последнюю строку: Rust `char` — 4 байта, это не C `char`.

Пример структуры с такими типами:

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
    println!("{}", data.context.is_null());
}
```

Результат:

```text
42
true
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_int%2C%20c_void%7D%3B%0A%0A%23%5Brepr%28C%29%5D%0Astruct%20Data%20%7B%0A%20%20%20%20value%3A%20c_int%2C%0A%20%20%20%20context%3A%20%2Amut%20c_void%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20data%20%3D%20Data%20%7B%0A%20%20%20%20%20%20%20%20value%3A%2042%2C%0A%20%20%20%20%20%20%20%20context%3A%20std%3A%3Aptr%3A%3Anull_mut%28%29%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20data.value%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20data.context.is_null%28%29%29%3B%0A%7D%0A)

### Вариативные функции C

C-функции с переменным числом аргументов (`printf`, `snprintf`) можно **вызывать** из Rust, объявив их с `...`, но **определять** такие функции в stable Rust нельзя:

```rust
use std::ffi::{c_char, c_int};

unsafe extern "C" {
    // Вариативные функции C можно вызывать, но нельзя определять в stable Rust.
    fn snprintf(buf: *mut c_char, size: usize, format: *const c_char, ...) -> c_int;
}

fn main() {
    let mut buffer = [0 as c_char; 32];

    // SAFETY: buffer достаточно большой, формат — валидная C-строка,
    // аргумент типа c_int соответствует спецификатору %d.
    let written = unsafe {
        snprintf(buffer.as_mut_ptr(), buffer.len(), c"value = %d".as_ptr(), 42 as c_int)
    };

    // SAFETY: snprintf записал NUL-терминированную строку в buffer.
    let text = unsafe { std::ffi::CStr::from_ptr(buffer.as_ptr()) };
    println!("{written}: {}", text.to_str().unwrap());
}
```

Результат:

```text
10: value = 42
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20c_int%7D%3B%0A%0Aunsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20%2F%2F%20%D0%92%D0%B0%D1%80%D0%B8%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D1%8B%D0%B5%20%D1%84%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%B8%20C%20%D0%BC%D0%BE%D0%B6%D0%BD%D0%BE%20%D0%B2%D1%8B%D0%B7%D1%8B%D0%B2%D0%B0%D1%82%D1%8C%2C%20%D0%BD%D0%BE%20%D0%BD%D0%B5%D0%BB%D1%8C%D0%B7%D1%8F%20%D0%BE%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D1%8F%D1%82%D1%8C%20%D0%B2%20stable%20Rust.%0A%20%20%20%20fn%20snprintf%28buf%3A%20%2Amut%20c_char%2C%20size%3A%20usize%2C%20format%3A%20%2Aconst%20c_char%2C%20...%29%20-%3E%20c_int%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20buffer%20%3D%20%5B0%20as%20c_char%3B%2032%5D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20buffer%20%D0%B4%D0%BE%D1%81%D1%82%D0%B0%D1%82%D0%BE%D1%87%D0%BD%D0%BE%20%D0%B1%D0%BE%D0%BB%D1%8C%D1%88%D0%BE%D0%B9%2C%20%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%20%E2%80%94%20%D0%B2%D0%B0%D0%BB%D0%B8%D0%B4%D0%BD%D0%B0%D1%8F%20C-%D1%81%D1%82%D1%80%D0%BE%D0%BA%D0%B0%2C%0A%20%20%20%20%2F%2F%20%D0%B0%D1%80%D0%B3%D1%83%D0%BC%D0%B5%D0%BD%D1%82%20%D1%82%D0%B8%D0%BF%D0%B0%20c_int%20%D1%81%D0%BE%D0%BE%D1%82%D0%B2%D0%B5%D1%82%D1%81%D1%82%D0%B2%D1%83%D0%B5%D1%82%20%D1%81%D0%BF%D0%B5%D1%86%D0%B8%D1%84%D0%B8%D0%BA%D0%B0%D1%82%D0%BE%D1%80%D1%83%20%25d.%0A%20%20%20%20let%20written%20%3D%20unsafe%20%7B%0A%20%20%20%20%20%20%20%20snprintf%28buffer.as_mut_ptr%28%29%2C%20buffer.len%28%29%2C%20c%22value%20%3D%20%25d%22.as_ptr%28%29%2C%2042%20as%20c_int%29%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20%2F%2F%20SAFETY%3A%20snprintf%20%D0%B7%D0%B0%D0%BF%D0%B8%D1%81%D0%B0%D0%BB%20NUL-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D1%83%D1%8E%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%20%D0%B2%20buffer.%0A%20%20%20%20let%20text%20%3D%20unsafe%20%7B%20std%3A%3Affi%3A%3ACStr%3A%3Afrom_ptr%28buffer.as_ptr%28%29%29%20%7D%3B%0A%20%20%20%20println%21%28%22%7Bwritten%7D%3A%20%7B%7D%22%2C%20text.to_str%28%29.unwrap%28%29%29%3B%0A%7D%0A)

Тут компилятор не проверяет соответствие аргументов строке формата: если написать `%d`, а передать `f64`, получится UB. Поэтому каждый вызов вариативной функции требует особого внимания, а типы аргументов следует приводить явно (`42 as c_int`).

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

### Законченный пример: C-код в проекте через `cc`

Самый удобный способ добавить собственный C-код — крейт `cc`: он вызывает системный C-компилятор из `build.rs` и сам настраивает линковку.

Структура проекта:

```text
ccdemo/
├── Cargo.toml
├── build.rs
└── src/
    ├── helper.c
    └── main.rs
```

`Cargo.toml`:

```toml
[package]
name = "ccdemo"
version = "0.1.0"
edition = "2024"

[build-dependencies]
cc = "1"
```

`build.rs`:

```rust
fn main() {
    cc::Build::new()
        .file("src/helper.c")
        .compile("helper");

    println!("cargo::rerun-if-changed=src/helper.c");
}
```

`src/helper.c`:

```c
#include <stdint.h>
#include <stddef.h>

typedef struct { double x; double y; } Point;

double point_length_squared(Point p) {
    return p.x * p.x + p.y * p.y;
}

int32_t sum_array(const int32_t *data, size_t len) {
    int32_t total = 0;
    for (size_t i = 0; i < len; i++) total += data[i];
    return total;
}
```

`src/main.rs`:

```rust
#[repr(C)]
#[derive(Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

unsafe extern "C" {
    fn point_length_squared(p: Point) -> f64;
    fn sum_array(data: *const i32, len: usize) -> i32;
}

/// Безопасная обёртка: срез гарантирует валидный указатель и длину.
fn sum(values: &[i32]) -> i32 {
    // SAFETY: values.as_ptr() валиден для чтения values.len() элементов i32.
    unsafe { sum_array(values.as_ptr(), values.len()) }
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    // SAFETY: Point имеет repr(C) и совпадает с C-структурой.
    let length_squared = unsafe { point_length_squared(p) };

    println!("{length_squared}");
    println!("{}", sum(&[1, 2, 3, 4]));
    println!("{}", sum(&[]));
}
```

Результат `cargo run`:

```text
25
10
0
```

Обратите внимание, что структура `Point` передаётся **по значению** — это допустимо, потому что она `#[repr(C)]` и совпадает с C-структурой. Для пустого среза `sum(&[])` указатель ненулевой «висячий», но C-функция при `len == 0` по нему ничего не читает.

Запись `cargo::rerun-if-changed=...` (с двойным двоеточием) поддерживается начиная с Rust 1.77; в проектах со старым компилятором используйте `cargo:rerun-if-changed=...`.

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

Проверим на реальном примере. Динамическая библиотека `example.c`:

```c
#include <stdint.h>
int32_t process(int32_t value) { return value * 2 + 1; }
```

Сборка (Linux):

```bash
gcc -shared -fPIC -o libexample.so example.c
```

`Cargo.toml` содержит зависимость `libloading = "0.8"`. `src/main.rs`:

```rust
use libloading::{Library, Symbol};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // SAFETY: загружаем нашу собственную библиотеку; её инициализаторы безопасны.
    let library = unsafe { Library::new("./libexample.so")? };

    // SAFETY: тип Symbol должен ТОЧНО соответствовать сигнатуре C-функции:
    // int32_t process(int32_t).
    let process: Symbol<unsafe extern "C" fn(i32) -> i32> =
        unsafe { library.get(b"process\0")? };

    // SAFETY: library жива, пока жив process (Symbol заимствует Library).
    let result = unsafe { process(42) };

    println!("{result}");

    // Несуществующий символ — это обычная ошибка, а не UB:
    let missing: Result<Symbol<unsafe extern "C" fn()>, _> =
        unsafe { library.get(b"nope\0") };
    println!("missing is_err = {}", missing.is_err());

    Ok(())
}
```

Результат:

```text
85
missing is_err = true
```

Этот пример **не предназначен для Rust Playground**, поскольку требует внешней динамической библиотеки.

Обратите внимание на три детали.

- Путь `"./libexample.so"` указывает на текущий каталог. Если написать просто `"libexample.so"`, система будет искать библиотеку в стандартных каталогах загрузчика, а не рядом с программой.
- Если мы передаём в `get` строку формата `b"process\0"`, то нулевой терминатор избавляет от лишнего выделения памяти; без него `libloading` добавит его сам.
- `Symbol` **заимствует** `Library`: так lifetime библиотеки учитывается компилятором. Попытка вернуть `Symbol` из функции, в которой `Library` уничтожается, не скомпилируется:

```text
error[E0515]: cannot return value referencing local variable `library`
```

Это защищает от самой опасной ошибки динамической загрузки: вызова функции из уже выгруженной библиотеки.

При использовании динамической загрузки особенно важно, чтобы:

- библиотека оставалась загруженной, пока используются её символы;
- сигнатура функции точно соответствовала реальному символу (компилятор не может это проверить: неверный тип `Symbol` — UB);
- ABI совпадал;
- lifetime загруженной библиотеки учитывался в архитектуре программы.

---

## 47.16. FFI с C++

Rust не имеет общего стабильного C++ ABI, аналогичного C ABI.

Более того, в Rust нет ABI-строки `"C++"`: запись `extern "C++" { ... }` не является допустимым синтаксисом. Поэтому нельзя просто написать нечто вроде:

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

Это самый простой архитектурный вариант. Нужно помнить, что C++ исключение не должно выйти из такой функции (оно пересечёт границу C ABI): в `extern "C"` слое их перехватывают и превращают в коды ошибок — зеркальное правило тому, что мы разобрали для паник Rust.

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

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=extern%20%22C%22%20%7B%0A%20%20%20%20fn%20external_function%28%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%7D%0A)

В Edition 2024 это ошибка:

```text
error: extern blocks must be unsafe
```

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

Возьмите пример `layout` из раздела 47.9 и выведите для обеих структур размеры и смещения полей. Затем добавьте к `RustLayout` атрибут `#[repr(C)]` и убедитесь, что числа стали такими же, как у `CLayout`.

Rust Reference прямо указывает, что layout обычных Rust-типов не имеет тех же гарантий, что `#[repr(C)]`. ([Rust Documentation][3])

---

## Эксперимент 3: `CString` lifetime

Рассмотрите:

```rust
use std::ffi::{c_char, CString};

fn get_pointer() -> *const c_char {
    let string = CString::new("hello").unwrap();

    string.as_ptr()
}

fn main() {
    println!("{}", get_pointer().is_null());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Affi%3A%3A%7Bc_char%2C%20CString%7D%3B%0A%0Afn%20get_pointer%28%29%20-%3E%20%2Aconst%20c_char%20%7B%0A%20%20%20%20let%20string%20%3D%20CString%3A%3Anew%28%22hello%22%29.unwrap%28%29%3B%0A%0A%20%20%20%20string.as_ptr%28%29%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20get_pointer%28%29.is_null%28%29%29%3B%0A%7D%0A)

Проблема не в компиляторе. В нашем запуске он вообще не выдал предупреждения: возвращать сырой указатель на локальную переменную — безопасная операция.

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

Это один из важнейших классов ошибок при FFI. Исправление — вернуть владеющий `CString` (или структуру, как `CConfig` в разделе 47.11), а указатель получать уже из него на стороне вызывающего.

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

fn main() {}
```

[▶ Открыть пример с предупреждением в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=unsafe%20extern%20%22C%22%20%7B%0A%20%20%20%20fn%20process%28value%3A%20%26mut%20String%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%7D%0A)

Так делать нельзя.

C не знает о Rust `String`. Компилятор предупредит об этом:

```text
warning: `extern` block uses type `String`, which is not FFI-safe
  |
2 |     fn process(value: &mut String);
  |                       ^^^^^^^^^^^ not FFI-safe
  |
  = help: consider adding a `#[repr(C)]` or `#[repr(transparent)]` attribute to this struct
  = note: this struct has unspecified layout
  = note: `#[warn(improper_ctypes)]` on by default
```

Это именно предупреждение, а не ошибка: lint можно отключить, и тогда ошибка останется незамеченной. Поэтому в crate с FFI полезно поднять его до `deny`.

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

fn main() {}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bno_mangle%5D%0Apub%20extern%20%22C%22%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0Afn%20main%28%29%20%7B%7D%0A)

Компилятор ответит:

```text
error: unsafe attribute used without unsafe
 --> src/main.rs:1:3
  |
1 | #[no_mangle]
  |   ^^^^^^^^^ usage of unsafe attribute
  |
help: wrap the attribute in `unsafe(...)`
  |
1 | #[unsafe(no_mangle)]
  |   +++++++         +
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

## Эксперимент 6: паника через границу FFI

Запустите примеры `panic_abort` и `catch_unwind` из раздела 47.12. Что изменится, если заменить `extern "C"` на `extern "C-unwind"` в первом из них? Подумайте, чьё поведение при этом меняется и что должна уметь сторона, вызывающая такую функцию.

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

и вызовите её через `unsafe extern "C"` с другим Rust-именем (`#[link_name]`), как в разделе 47.2.

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

которая вычисляет расстояние между двумя точками. Передайте точки по значению и проверьте результат на `(0, 0)` и `(3, 4)`.

---

## Задание 3

Напишите две функции.

Первая — `unsafe fn sum_raw(ptr: *const i32, len: usize) -> i32` — суммирует элементы по указателю. Опишите её `# Safety`.

Вторая — безопасная `fn sum(values: &[i32]) -> i32` — использует первую.

Продумайте:

- почему безопасная версия принимает срез, а не указатель с длиной;
- что делать с нулевой длиной (корректен ли ненулевой «висячий» указатель пустого среза);
- можно ли в `sum_raw` проверить границы и почему «проверка» `len` не заменяет контракт;
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

Подумайте, какой lifetime должен быть у возвращаемого `&str` и можно ли честно сделать такую функцию безопасной.

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

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=type%20Callback%20%3D%20Option%3Cextern%20%22C%22%20fn%28i32%29%3E%3B%0A%0Aextern%20%22C%22%20fn%20print_value%28value%3A%20i32%29%20%7B%0A%20%20%20%20println%21%28%22callback%3A%20%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn%20run_callback%28callback%3A%20Callback%29%20%7B%0A%20%20%20%20if%20let%20Some%28callback%29%20%3D%20callback%20%7B%0A%20%20%20%20%20%20%20%20callback%2842%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20run_callback%28Some%28print_value%29%29%3B%0A%20%20%20%20run_callback%28None%29%3B%0A%7D%0A)

Затем добавьте второй параметр — контекст `*mut c_void` — и сделайте callback, накапливающий значения в `Vec<i32>`.

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

с автоматическим освобождением через `Drop`. Реализуйте «C-сторону» модулем на Rust, как в разделе 47.7, и проверьте, что `connection_destroy` вызывается ровно один раз (добавьте счётчик).

---

## Задание 8

Экспортируйте из `cdylib` функцию `utf8_len` из раздела 47.12 и вызовите её из C-программы. Добавьте функцию, которая принимает указатель на буфер и его длину и записывает в него строку, и опишите контракт: кто выделяет буфер и что функция возвращает, если буфер мал.

---

# Главное из этой главы

После этой главы мы понимаем:

- **FFI** — механизм взаимодействия Rust с кодом на других языках.
- **C ABI** — основной механизм совместимости Rust с большим количеством внешних библиотек.
- **`unsafe extern "C"`** — современный способ объявления внешнего API в Edition 2024. ([Rust Documentation][1]) Внутри можно использовать `safe fn` и `unsafe fn`.
- **`extern "C" fn`** — функция, использующая C ABI.
- Функции libc можно объявлять и вызывать без дополнительной настройки линковки; для своей C-библиотеки нужны `#[link]`, `build.rs` или `cc`.
- **`CString` / `CStr`** — основные инструменты работы с C-строк; литерал `c"..."` — проверенная на этапе компиляции `&CStr`. Указатель из `CString::into_raw` освобождается только через `CString::from_raw`.
- **Raw pointers** — базовый механизм передачи адресов через FFI; указатель, длина и размер элемента должны быть согласованы.
- **`#[repr(C)]`** — способ задать C-compatible layout для структур, unions и enum-представлений. ([Rust Documentation][3]) Значения `enum` из C принимают как целые числа с проверкой.
- **`#[unsafe(no_mangle)]`** — современный способ экспортировать символ с определённым именем в Edition 2024. ([Rust Documentation][2])
- **Callbacks** — механизм вызова Rust-кода из внешней библиотеки; состояние передаётся через `*mut c_void` и функцию-trampoline.
- **Opaque handles** — практический способ скрывать внутреннее состояние библиотечного объекта; отдельный пустой тип на каждый handle защищает от путаницы.
- **Ownership** — одна из главных проблем FFI: кто выделил, тот и освобождает, тем же аллокатором.
- **Safe wrappers** — основной способ сделать FFI удобным и безопасным; обёртка безопасна только если нельзя нарушить контракт C-функции из безопасного кода (принимайте `&CStr`, `&[T]`, а не сырые указатели).
- **Паника не должна пересекать границу FFI**: `extern "C"` останавливает её аварийным завершением; перехватывайте её через `catch_unwind` или используйте `"C-unwind"`.
- **C++ FFI** требует дополнительных инструментов (`cxx`, `bindgen` или C-слой) и не сводится к простому `extern "C"`.

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

[1]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html 'Unsafe extern blocks - The Rust Edition Guide'
[2]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html 'Unsafe attributes - The Rust Edition Guide'
[3]: https://doc.rust-lang.org/stable/reference/type-layout.html 'Type layout - The Rust Reference'
