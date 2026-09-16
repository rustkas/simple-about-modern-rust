# Глава 73. Compile Time vs Runtime

Одной из ключевых особенностей Rust является перенос значительной части работы с **времени выполнения** на **время компиляции**.

Во время компиляции Rust может:

- проверять типы;
- проверять владение и заимствования;
- проверять времена жизни;
- разрешать generics и traits;
- выполнять macro expansion;
- вычислять значения в `const`-контексте;
- генерировать специализированный код;
- оптимизировать машинный код.

Во время выполнения программа уже работает с конкретными данными и состоянием среды:

- читает файлы;
- выполняет сетевые операции;
- выделяет динамическую память;
- обрабатывает ввод пользователя;
- выполняет динамическую диспетчеризацию;
- выполняет асинхронные операции.

Но важно не превращать это в слишком простое правило:

**«Compile time — хорошо, runtime — плохо».**

Это не так.

Compile time и runtime решают разные задачи. Более того, некоторые конструкции Rust могут использоваться **и во время компиляции, и во время выполнения**. `const fn` — один из лучших примеров такого поведения.

Поэтому задача современного Rust-программиста — не просто «перенести всё на compile time», а понимать:

**что именно известно компилятору, когда это известно и какую работу действительно имеет смысл выполнять до запуска программы.**

Все примеры этой главы используют **Rust Edition 2024**.

---

## 73.1. Компиляция vs выполнение

Упрощённо жизненный цикл Rust-программы можно представить так:

```text
┌──────────────────────────────────────────────────────────────┐
│                      COMPILE TIME                            │
│                                                              │
│  Исходный код                                                │
│      │                                                       │
│      ├── macro expansion                                     │
│      ├── name resolution                                     │
│      ├── проверка типов                                      │
│      ├── borrow checking                                     │
│      ├── проверка lifetimes                                  │
│      ├── monomorphization                                    │
│      ├── const evaluation                                    │
│      └── оптимизация                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Машинный код программы
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                       RUNTIME                                │
│                                                              │
│  Конкретные данные и состояние системы                       │
│                                                              │
│  ├── вычисления                                              │
│  ├── вызовы функций                                          │
│  ├── heap allocation                                         │
│  ├── I/O                                                     │
│  ├── network                                                 │
│  ├── dynamic dispatch                                        │
│  └── async execution                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Это упрощённая схема. Например, **инлайнинг** и другие оптимизации являются решениями оптимизирующего компилятора, а не языковыми гарантиями.

И наоборот, некоторые вещи, которые концептуально относятся к runtime, могут быть полностью устранены оптимизатором.

Поэтому правильнее говорить:

**Compile time определяет и проверяет программу; runtime выполняет её с конкретными данными. Оптимизатор может преобразовать runtime-код так, что часть вычислений вообще исчезнет из итоговой программы.**

---

## 73.2. Проверки во время компиляции

Одна из самых важных особенностей Rust заключается в том, что большое количество ошибок обнаруживается **до запуска программы**.

### Типы

```rust
fn add(a: i32, b: i32) - i32 {
    a + b
}

fn main() {
    let result = add(10, 20);

    println!("{result}");
}
```

Компилятор знает, что `add` принимает два `i32`.

Поэтому такой код не скомпилируется:

```rust
fn add(a: i32, b: i32) - i32 {
    a + b
}

fn main() {
    let result = add("hello", 5);

    println!("{result}");
}
```

Ошибка обнаруживается **до запуска программы**.

---

### Владение и заимствование

```rust
fn use_string(s: &String) {
    println!("{s}");
}

fn main() {
    let s = String::from("hello");

    use_string(&s);

    println!("{s}");
}
```

После передачи `&s` функция получает только заимствование. Владение `s` не изменилось.

Поэтому `s` продолжает существовать после вызова.

---

### Времена жизни

Rust также обнаруживает потенциально висячие ссылки:

```rust
fn dangle() - &String {
    let s = String::from("hello");

    &s
}
```

Такой код не компилируется.

Локальная переменная `s` уничтожается при выходе из функции, поэтому ссылка на неё не может быть возвращена вызывающему коду.

---

### Compile-time проверка размеров

Особенно хорошо идея compile time проявляется в типах.

```rust
fn copy_array<const N: usize(input: [i32; N]) - [i32; N] {
    input
}

fn main() {
    let values = [1, 2, 3, 4];

    let result = copy_array(values);

    println!("{result:?}");
}
```

Здесь `N` является **const generic parameter**.

Тип функции содержит информацию о размере массива:

```text
[i32; 4]
      │
      └── часть типа
```

Размер известен компилятору.

Если API требует массив из четырёх элементов, передать массив другого размера невозможно:

```rust
fn only_four(values: [i32; 4]) {
    println!("{values:?}");
}

fn main() {
    let values = [1, 2, 3];

    only_four(values); // ошибка компиляции
}
```

Ошибка обнаруживается ещё до запуска программы.

`const generics` позволяют параметризовать типы и функции значениями, известными как часть конкретной инстанциации. ([Rust Documentation][5])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+only_four%28values%3A+%5Bi32%3B+4%5D%29+%7B%0A++++println%21%28%22%7Bvalues%3A%3F%7D%22%2C+values%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+values+%3D+%5B1%2C+2%2C+3%5D%3B%0A++++only_four%28values%29%3B%0A%7D)

---

## 73.3. Generics и мономорфизация

Generics позволяют писать код, не привязываясь к конкретному типу.

```rust
fn identity<T(value: T) - T {
    value
}

fn main() {
    let x = identity(42);
    let y = identity(3.14);
    let z = identity("hello");

    println!("{x}, {y}, {z}");
}
```

Здесь используются разные конкретные типы:

```text
identity::<i32
identity::<f64
identity::<&str
```

Для обычного generic-кода Rust использует **мономорфизацию**: компилятор создаёт специализированные версии кода для конкретных типов, которые используются программой.

Упрощённо можно представить результат как:

```rust
fn identity_i32(value: i32) - i32 {
    value
}

fn identity_f64(value: f64) - f64 {
    value
}

fn identity_str(value: &str) - &str {
    value
}
```

Это не буквальный Rust-код, который компилятор обязан генерировать именно в таком виде. Это **модель для понимания** процесса.

Преимущество:

- нет необходимости хранить информацию о типе для каждого generic-вызова во время выполнения;
- конкретный тип известен компилятору;
- специализированный код может дополнительно оптимизироваться.

Но есть и цена:

**Мономорфизация может увеличивать размер генерируемого кода и время компиляции.**

Поэтому нельзя говорить, что generics «совсем бесплатны». Они часто дают отсутствие runtime-стоимости абстракции, но могут иметь **compile-time и code-size cost**.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=fn+identity%3CT%3E%28value%3A+T%29+-%3E+T+%7B%0A++++value%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+x+%3D+identity%2842i32%29%3B%0A++++let+y+%3D+identity%283.14f64%29%3B%0A++++let+z+%3D+identity%28%22hello%22%29%3B%0A%0A++++println%21%28%22%7Bx%7D+%7By%7D+%7Bz%7D%22%29%3B%0A%7D)

---

## 73.4. Traits и статическая диспетчеризация

Рассмотрим trait:

```rust
trait Draw {
    fn draw(&self);
}

struct Circle;

struct Square;

impl Draw for Circle {
    fn draw(&self) {
        println!("Circle");
    }
}

impl Draw for Square {
    fn draw(&self) {
        println!("Square");
    }
}
```

Теперь можем использовать generic:

```rust
fn draw_static<T: Draw(item: &T) {
    item.draw();
}
```

При конкретном вызове:

```rust
let circle = Circle;

draw_static(&circle);
```

компилятор знает:

```text
T = Circle
```

и поэтому конкретная реализация `Draw::draw` известна уже при компиляции.

Это называется **static dispatch**.

---

### Dynamic dispatch

Теперь:

```rust
fn draw_dynamic(item: &dyn Draw) {
    item.draw();
}
```

Здесь конкретный тип объекта может быть неизвестен вызывающему коду:

```rust
let circle = Circle;
let square = Square;

let shapes: Vec<&dyn Draw = vec![&circle, &square];

for shape in shapes {
    draw_dynamic(shape);
}
```

Trait object содержит указатель на объект и информацию, необходимую для динамической диспетчеризации, включая vtable. Вызов метода происходит через эту таблицу во время выполнения. ([Rust Documentation][4])

### Сравнение

| Характеристика                     | `T: Trait`              | `dyn Trait`                            |
| ---------------------------------- | ----------------------- | -------------------------------------- |
| Диспетчеризация                    | Статическая             | Динамическая                           |
| Конкретный тип                     | Известен при компиляции | Может различаться во время выполнения  |
| Мономорфизация                     | Да                      | Нет для самого trait-object интерфейса |
| Косвенный вызов                    | Обычно нет              | Да                                     |
| Возможность инлайнинга через вызов | Хорошая                 | Ограничена                             |
| Гетерогенная коллекция             | Нет напрямую            | Да                                     |
| Гибкость                           | Ниже                    | Выше                                   |

Но утверждение:

«`dyn Trait` всегда медленнее»

слишком упрощённое.

У динамической диспетчеризации есть дополнительная стоимость, но её реальное влияние зависит от конкретной программы. Иногда гибкость `dyn Trait` гораздо важнее небольшой стоимости косвенного вызова.

Поэтому правильный вопрос:

**Нужна ли здесь динамическая диспетчеризация?**

а не:

**Как полностью избавиться от `dyn`?**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=trait+Draw+%7B%0A++++fn+draw%28%26self%29%3B%0A%7D%0A%0Astruct+Circle%3B%0Astruct+Square%3B%0A%0Aimpl+Draw+for+Circle+%7B%0A++++fn+draw%28%26self%29+%7B%0A++++++++println%21%28%22Circle%22%29%3B%0A++++%7D%0A%7D%0A%0Aimpl+Draw+for+Square+%7B%0A++++fn+draw%28%26self%29+%7B%0A++++++++println%21%28%22Square%22%29%3B%0A++++%7D%0A%7D%0A%0Afn+draw_static%3CT%3A+Draw%3E%28item%3A+%26T%29+%7B%0A++++item.draw%28%29%3B%0A%7D%0A%0Afn+draw_dynamic%28item%3A+%26dyn+Draw%29+%7B%0A++++item.draw%28%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+circle+%3D+Circle%3B%0A++++let+square+%3D+Square%3B%0A%0A++++draw_static%28%26circle%29%3B%0A++++draw_dynamic%28%26square%29%3B%0A%7D)

---

## 73.5. Макросы — преобразование кода на этапе компиляции

Макросы позволяют расширять синтаксис Rust.

Например:

```rust
macro_rules! square {
    ($x:expr) = {
        $x * $x
    };
}

fn main() {
    let result = square!(5);

    println!("{result}");
}
```

Вызов:

```rust
square!(5)
```

раскрывается в Rust-код, который затем проходит обычную компиляцию.

Упрощённо:

```text
square!(5)
     │
     ▼
5 * 5
     │
     ▼
обычная компиляция Rust-кода
```

Rust Reference определяет macro invocation как конструкцию, которая раскрывается во время компиляции и заменяется результатом расширения. ([Rust Documentation][3])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=macro_rules%21+square+%7B%0A++++%28%24x%3Aexpr%29+%3D%3E+%7B%0A++++++++%7B+let+x+%3D+%24x%3B+x+%2A+x+%7D%0A++++%7D%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+result+%3D+square%21%285%29%3B%0A++++println%21%28%22%7Bresult%7D%22%29%3B%0A%7D)

### Важная деталь: аргумент может быть выражением

Наивная версия:

```rust
macro_rules! square {
    ($x:expr) = {
        $x * $x
    };
}
```

может вычислить аргумент дважды:

```rust
square!(next_value());
```

Получится концептуально:

```rust
next_value() * next_value()
```

Поэтому для более общего макроса можно сохранить значение:

```rust
macro_rules! square {
    ($x:expr) = {{
        let x = $x;
        x * x
    }};
}
```

Теперь выражение вычисляется один раз.

Это хороший пример того, почему макросы нельзя рассматривать просто как «текстовую замену».

---

### Procedural macros

Другой механизм — procedural macros:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}
```

`derive(Debug)` генерирует реализацию `Debug` во время компиляции.

Procedural macros существуют в трёх основных формах:

- function-like macros;
- derive macros;
- attribute macros.

Они работают с Rust tokens и создают другой Rust-код. ([Rust Documentation][6])

Главная идея:

**Макрос генерирует код; сгенерированный код затем становится частью обычной программы.**

---

## 73.6. `const` и `const fn`

`const` позволяет определить значение, которое может использоваться в compile-time контекстах:

```rust
const MAX_USERS: usize = 1000;

const BUFFER_SIZE: usize = MAX_USERS * 2;

fn main() {
    println!("{BUFFER_SIZE}");
}
```

Более интересный вариант — `const fn`:

```rust
const fn add(a: i32, b: i32) - i32 {
    a + b
}

const RESULT: i32 = add(5, 3);

fn main() {
    println!("{RESULT}");
}
```

Здесь вызов:

```rust
add(5, 3)
```

происходит в `const`-контексте и поэтому вычисляется во время компиляции. ([Rust Documentation][2])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=const+fn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B%0A++++a+%2B+b%0A%7D%0A%0Aconst+RESULT%3A+i32+%3D+add%285%2C+3%29%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22RESULT%3A+%7BRESULT%7D%22%29%3B%0A%7D)

### Но `const fn` не означает «функция работает только на compile time»

Это важный момент.

```rust
const fn square(x: i32) - i32 {
    x * x
}

const VALUE: i32 = square(10);

fn main() {
    let x = 7;

    let result = square(x);

    println!("{result}");
}
```

Здесь:

```rust
const VALUE: i32 = square(10);
```

может быть вычислено на этапе компиляции, потому что это `const`-контекст.

А:

```rust
let result = square(x);
```

работает с runtime-значением `x`.

Поэтому:

**`const fn` означает, что функцию разрешено использовать в const-контексте. Это не означает, что каждый её вызов обязательно выполняется во время компиляции.**

Rust Reference отдельно подчёркивает: выражения в `const`-контекстах вычисляются во время компиляции, тогда как за пределами таких контекстов даже константное выражение не обязано вычисляться compile time. ([Rust Documentation][2])

---

## 73.7. `const` vs `static`

`const` и `static` похожи внешне, но концептуально это разные механизмы.

| Характеристика             | `const`                                        | `static`                                   |
| -------------------------- | ---------------------------------------------- | ------------------------------------------ |
| Значение                   | Константное                                    | Единственный объект                        |
| Собственное место в памяти | Не обязательно                                 | Да                                         |
| Экземпляров                | Может рассматриваться как подстановка значения | Один                                       |
| Можно получить ссылку      | Да                                             | Да                                         |
| Может изменяться           | Нет                                            | `static mut` — специальный unsafe-механизм |
| Время жизни                | Значение доступно в рамках программы           | `'static`                                  |
| Основное назначение        | Константное значение                           | Глобальное состояние/объект                |

Пример:

```rust
const CONST_VALUE: i32 = 42;

static STATIC_VALUE: i32 = 42;

fn main() {
    let a = &CONST_VALUE;
    let b = &STATIC_VALUE;

    println!("{a}");
    println!("{b}");
}
```

Главное различие:

```text
const
  │
  └── значение

static
  │
  └── конкретное место в памяти
```

Rust Reference описывает `static` как значение, существующее на протяжении всей программы и представляющее конкретное место в памяти. ([Rust Documentation][1])

А `const` концептуально ближе к подстановке значения в места использования. ([Rust Documentation][7])

### Практическое правило

Используйте:

```rust
const
```

для именованных констант.

Используйте:

```rust
static
```

когда вам действительно нужен **один глобальный объект с конкретным адресом**.

Не следует выбирать `static` просто потому, что значение «вычисляется на этапе компиляции».

---

## 73.8. Compile-Time Function Evaluation — CTFE

**CTFE (Compile-Time Function Evaluation)** — это выполнение допустимых выражений во время компиляции.

Например:

```rust
const fn factorial(n: u64) - u64 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}

const FACTORIAL_5: u64 = factorial(5);

fn main() {
    println!("{FACTORIAL_5}");
}
```

Компилятору не нужно ждать запуска программы, чтобы получить значение:

```text
factorial(5)
     │
     ▼
120
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=const+fn+factorial%28n%3A+u64%29+-%3E+u64+%7B%0A++++if+n+%3C%3D+1+%7B%0A++++++++1%0A++++%7D+else+%7B%0A++++++++n+%2A+factorial%28n+-+1%29%0A++++%7D%0A%7D%0A%0Aconst+FACTORIAL_5%3A+u64+%3D+factorial%285%29%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7BFACTORIAL_5%7D%22%29%3B%0A%7D)

В `const`-контексте компилятор обязан вычислить выражение во время компиляции. Если вычисление невозможно или нарушает правила constant evaluation, это становится ошибкой компиляции. ([Rust Documentation][2])

Например:

```rust
const VALUE: i32 = 10 / 0;
```

не может быть корректной константой.

---

## 73.9. Const generics — перенос значений в типовую систему

Rust позволяет параметризовать типы и функции константными значениями:

```rust
struct Buffer<T, const N: usize {
    data: [T; N],
}
```

Теперь размер массива является частью конкретного типа:

```rust
type Buffer8 = Buffer<u8, 8;
type Buffer16 = Buffer<u8, 16;
```

Это разные типы:

```text
Buffer<u8, 8
      ≠
Buffer<u8, 16
```

Пример:

```rust
#![allow(dead_code)]
struct Buffer<T, const N: usize {
    data: [T; N],
}

impl<T: Copy, const N: usize Buffer<T, N {
    fn new(value: T) - Self {
        Self {
            data: [value; N],
        }
    }

    fn len(&self) - usize {
        N
    }
}

fn main() {
    let buffer = Buffer::<u8, 8::new(0);

    println!("length = {}", buffer.len());
}
```

Здесь `N` известно компилятору как параметр конкретного типа.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+Buffer%3CT%2C+const+N%3A+usize%3E+%7B%0A++++data%3A+%5BT%3B+N%5D%2C%0A%7D%0A%0Aimpl%3CT%3A+Copy%2C+const+N%3A+usize%3E+Buffer%3CT%2C+N%3E+%7B%0A++++fn+new%28value%3A+T%29+-%3E+Self+%7B%0A++++++++Self++%7B+data%3A+%5Bvalue%3B+N%5D+%7D%0A++++%7D%0A%0A++++fn+len%28%26self%29+-%3E+usize+%7B%0A++++++++N%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+buffer+%3D+Buffer%3A%3A%3C+u8%2C+8+%3E%3A%3Anew%280%29%3B%0A++++println%21%28%22length+%3D+%7B%7D%22%2C+buffer.len%28%29%29%3B%0A%7D)

Это один из важнейших инструментов современного Rust:

**значение, известное во время компиляции, может стать частью типа и использоваться для статических гарантий.**

---

## 73.10. Генерация кода и `build.rs`

В Rust существует несколько разных механизмов генерации кода.

### `macro_rules!`

Макрос генерирует Rust-код во время компиляции.

```rust
macro_rules! answer {
    () = {
        42
    };
}

fn main() {
    let value = answer!();

    println!("{value}");
}
```

---

### Procedural macros

Например:

```rust
#[derive(Debug, Clone)]
struct User {
    name: String,
    age: u32,
}
```

`derive` генерирует необходимую реализацию.

---

### `build.rs`

Можно также использовать Cargo build script:

```rust
// build.rs

fn main() {
    println!("cargo:rerun-if-changed=data/schema.txt");

    // Здесь можно:
    // - прочитать файл;
    // - проверить окружение;
    // - сгенерировать исходный код;
    // - сообщить Cargo параметры сборки.
}
```

Важно различать:

```text
const evaluation
        │
        └── механизм языка Rust / rustc

build.rs
        │
        └── отдельная программа, запускаемая Cargo во время сборки
```

`build.rs` может выполнять произвольную работу, доступную обычной программе на стороне сборки: читать файлы, запускать генераторы, анализировать схему и создавать код.

Поэтому **`build.rs` — это не `const fn` и не CTFE**.

---

## 73.11. Что остаётся на этапе выполнения

Compile time не может знать данные, которые появятся только во время работы программы.

Например:

```rust
use std::io;

fn main() {
    let mut input = String::new();

    io::stdin()
        .read_line(&mut input)
        .unwrap();

    println!("You entered: {input}");
}
```

Строка, которую введёт пользователь, неизвестна компилятору.

Поэтому её обработка происходит во время выполнения.

---

### Heap allocation

Например:

```rust
fn main() {
    let values = vec![1, 2, 3, 4, 5];

    println!("{values:?}");
}
```

Создание `Vec` обычно связано с динамическим выделением памяти.

Компилятор может оптимизировать конкретные случаи, но семантически `Vec` представляет динамически управляемый буфер.

---

### I/O

```rust
use std::fs;

fn main() - std::io::Result<() {
    let content = fs::read_to_string("file.txt")?;

    println!("{content}");

    Ok(())
}
```

Содержимое файла неизвестно во время компиляции.

Оно появляется только при запуске программы.

---

### Dynamic dispatch

```rust
trait Draw {
    fn draw(&self);
}

struct Circle;

impl Draw for Circle {
    fn draw(&self) {
        println!("Circle");
    }
}

fn draw(item: &dyn Draw) {
    item.draw();
}

fn main() {
    let circle = Circle;

    draw(&circle);
}
```

Вызов `item.draw()` через trait object использует динамическую диспетчеризацию. Конкретная реализация метода выбирается во время выполнения. ([Rust Documentation][4])

---

### Асинхронность

```rust
async fn compute() - u32 {
    42
}
```

`async fn` преобразуется компилятором в future/state machine, но **само выполнение этой state machine происходит во время выполнения**.

Например:

```rust
async fn fetch_data() - Result<String, reqwest::Error {
    reqwest::get("https://example.com")
        .await?
        .text()
        .await
}
```

Компилятор создаёт структуру, представляющую состояние future.

Но сетевой запрос:

```text
HTTP request
```

происходит только во время выполнения программы.

Это хороший пример различия:

**Компилятор может генерировать runtime-механику на compile time, но сама runtime-механика выполняется позже.**

---

## 73.12. Runtime reflection vs Compile time

Rust не предоставляет полноценную runtime reflection-модель в стиле Java или C#, где программа может произвольно исследовать тип и его поля во время выполнения.

Например, нет универсального механизма:

```rust
// Концептуальный псевдокод — такого API нет.

let type_name = type_of(value);
let fields = get_fields(value);
```

Вместо этого Rust активно использует compile-time механизмы:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}
```

Компилятор совместно с derive-механизмом генерирует реализацию `Debug`.

После этого:

```rust
println!("{user:?}");
```

уже является обычным runtime-кодом.

То есть схема выглядит так:

```text
Source code
     │
     ▼
#[derive(Debug)]
     │
     ▼
генерация реализации
     │
     ▼
обычная компиляция
     │
     ▼
runtime-код
```

Это принципиально отличается от модели:

```text
runtime
   │
   ▼
исследовать тип
   │
   ▼
найти поля
   │
   ▼
выполнить операцию
```

Compile-time подход часто позволяет получить:

- статическую проверку;
- отсутствие необходимости в общем reflection runtime;
- хорошие возможности оптимизации;
- более предсказуемый runtime.

Но это не означает, что Rust вообще не имеет средств получения информации о типах во время выполнения. Например, существуют механизмы вроде `std::any::TypeId` и `type_name`. Просто это **не полноценная рефлексия структуры типов**.

---

## 73.13. Важное различие: compile time ≠ оптимизация

Очень легко сделать неправильный вывод:

«Если что-то известно во время компиляции, значит runtime-кода для этого точно не будет».

Это не всегда так.

Например:

```rust
const fn square(x: i32) - i32 {
    x * x
}

fn main() {
    let x = 10;

    println!("{}", square(x));
}
```

`x` — runtime-значение.

Поэтому вызов концептуально относится к runtime.

С другой стороны:

```rust
const VALUE: i32 = square(10);
```

имеет compile-time evaluation.

Ещё одна важная мысль:

**Оптимизатор может удалить runtime-код даже тогда, когда язык не обещает, что вычисление происходило на compile time.**

Например:

```rust
fn main() {
    let x = 10 * 20;

    println!("{x}");
}
```

В release-сборке компилятор вполне может превратить это в код, который просто использует `200`.

Но это уже **оптимизация**, а не семантическая гарантия `const evaluation`.

Поэтому нужно различать три понятия:

```text
1. Compile-time semantics
   Что язык обязан вычислить/проверить до запуска.

2. Runtime semantics
   Что программа должна делать во время выполнения.

3. Optimization
   Что компилятор может изменить или устранить,
   сохранив наблюдаемое поведение программы.
```

Это различие чрезвычайно важно для понимания современного Rust.

---

## 73.14. Практические выводы

### 1. Используйте `const` и `const fn`, когда значение действительно известно заранее

Например:

```rust
const MAX_RETRIES: usize = 5;

const fn buffer_size(items: usize, item_size: usize) - usize {
    items * item_size
}

const BUFFER_SIZE: usize = buffer_size(1024, 8);
```

Это делает намерение программы явным.

---

### 2. Используйте generics, когда тип известен на этапе компиляции

```rust
fn process<T: AsRef<str(value: T) {
    println!("{}", value.as_ref());
}
```

Это позволяет получить статическую диспетчеризацию и сохраняет типовую информацию.

---

### 3. Используйте `dyn Trait`, когда нужна runtime-гибкость

Например:

```rust
Vec<Box<dyn Draw
```

может содержать разные конкретные типы:

```text
Circle
Square
Triangle
...
```

Это не «ошибка архитектуры».

Это осознанный выбор в пользу динамической диспетчеризации.

---

### 4. Используйте макросы для генерации кода и расширения синтаксиса

Макросы особенно полезны, когда обычные generics или функции не позволяют выразить нужный API.

---

### 5. Не переносите работу на compile time только ради самого факта переноса

Compile-time computation тоже имеет стоимость:

- увеличивается время компиляции;
- может увеличиваться размер сгенерированного кода;
- сложная compile-time логика может ухудшать читаемость;
- генерация большого количества специализированного кода может приводить к code bloat.

Поэтому правильная цель:

**не максимизировать compile time, а использовать compile time там, где он даёт реальные преимущества.**

---

### 6. Не бойтесь runtime

Runtime — это не недостаток языка.

Именно во время выполнения программа:

- получает данные;
- взаимодействует с пользователем;
- работает с файлами;
- общается по сети;
- обрабатывает запросы;
- выполняет бизнес-логику.

Хороший Rust-код не пытается устранить runtime.

Он делает так, чтобы **то, что должно быть известно заранее, проверялось заранее**, а runtime выполнял только необходимую работу.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. `const fn` в compile time и runtime

```rust
const fn square(x: u32) - u32 {
    x * x
}

const COMPILE_TIME: u32 = square(10);

fn main() {
    let value = 7;

    let runtime = square(value);

    println!("compile time value = {COMPILE_TIME}");
    println!("runtime value = {runtime}");
}
```

Обратите внимание: одна и та же функция используется в двух разных контекстах.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=const+fn+square%28x%3A+u32%29+-%3E+u32+%7B%0A++++x+%2A+x%0A%7D%0A%0Aconst+COMPILE_TIME%3A+u32+%3D+square%2810%29%3B%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+7%3B%0A++++let+runtime+%3D+square%28value%29%3B%0A%0A++++println%21%28%22compile+time+value+%3D+%7BCOMPILE_TIME%7D%22%29%3B%0A++++println%21%28%22runtime+value+%3D+%7Bruntime%7D%22%29%3B%0A%7D)

---

### Эксперимент 2. Ошибка в `const`-вычислении

Попробуйте:

```rust
const VALUE: u32 = 10 / 0;

fn main() {
    println!("{VALUE}");
}
```

Компилятор должен обнаружить проблему до запуска программы.

Это демонстрирует важный принцип:

Ошибка в обязательном `const`-вычислении становится ошибкой компиляции.

---

### Эксперимент 3. Const generics

```rust
fn first<const N: usize(array: [i32; N]) - i32 {
    array[0]
}

fn main() {
    let values = [10, 20, 30];

    println!("{}", first(values));
}
```

Попробуйте изменить тип параметра:

```rust
fn first(array: [i32; 4]) - i32
```

и передать массив из трёх элементов.

Наблюдайте, как информация о размере массива превращается в часть типа.

---

### Эксперимент 4. `const` vs `static`

```rust
const VALUE: i32 = 42;

static GLOBAL: i32 = 42;

fn main() {
    println!("const = {VALUE}");
    println!("static = {GLOBAL}");

    println!("static address = {:p}", &GLOBAL);
}
```

Обратите внимание: `static` представляет конкретный объект, имеющий собственное место в памяти.

---

### Эксперимент 5. Static vs dynamic dispatch

Создайте две функции:

```rust
fn draw_static<T: Draw(item: &T) {
    item.draw();
}

fn draw_dynamic(item: &dyn Draw) {
    item.draw();
}
```

Посмотрите на generated assembly в release-сборке.

Не пытайтесь заранее угадать, что именно произойдёт с каждой функцией.

Смысл эксперимента — научиться **проверять гипотезы о компиляторе**, а не доверять упрощённым правилам.

---

## Практика

### Задание 1

Напишите:

```rust
const fn
```

для вычисления площади прямоугольника.

Затем используйте её для инициализации `const`.

---

### Задание 2

Напишите `const fn factorial`.

Используйте её:

1. в `const`;
2. с runtime-значением.

Объясните разницу между двумя вызовами.

---

### Задание 3

Создайте:

```rust
struct Matrix<T, const ROWS: usize, const COLS: usize {
    data: [[T; COLS]; ROWS],
}
```

Создайте матрицы:

```rust
Matrix<i32, 2, 3
Matrix<i32, 3, 3
```

Объясните, почему это разные типы.

---

### Задание 4

Напишите `macro_rules!`-макрос:

```rust
square!(value)
```

который вычисляет квадрат выражения.

Убедитесь, что аргумент вычисляется только один раз.

---

### Задание 5

Создайте trait:

```rust
trait Processor {
    fn process(&self) - i32;
}
```

Реализуйте его для двух типов.

Создайте две функции:

```rust
fn process_static<T: Processor(value: &T) - i32
```

и

```rust
fn process_dynamic(value: &dyn Processor) - i32
```

Сравните их.

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте generic-функцию, которая используется с большим количеством различных типов.

Посмотрите, как изменение архитектуры с generics на `dyn Trait` влияет на:

- размер кода;
- возможность инлайнинга;
- структуру вызовов.

Не делайте вывод заранее — сначала измерьте.

---

### Задание 7

🔨 **Эксперимент с compile-time evaluation.**

Попробуйте использовать в `const fn` операции, которые недоступны в текущем `const`-контексте.

Изучите сообщение компилятора.

Затем проверьте, поддерживается ли нужная операция в вашей версии Rust.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Compile time** — этап, на котором Rust анализирует программу, проверяет типы и владение, раскрывает макросы, специализирует generic-код и выполняет допустимые compile-time вычисления.
- **Runtime** — выполнение с конкретными данными и состоянием внешней среды.
- **Generics** — позволяют использовать статическую типизацию и, как правило, мономорфизацию.
- **`dyn Trait`** — механизм динамической диспетчеризации во время выполнения.
- **`macro_rules!`** — механизм генерации Rust-кода во время компиляции. ([Rust Documentation][3])
- **Procedural macros** — compile-time расширения, работающие с Rust tokens и генерирующие Rust-код. ([Rust Documentation][6])
- **`const`** — способ выразить значение, пригодное для compile-time использования.
- **`const fn`** — функция, которую можно вызывать в `const`-контексте; это не означает, что все её вызовы выполняются на этапе компиляции.
- **Const generics** — способ передавать compile-time значения как параметры типов и функций. ([Rust Documentation][5])
- **`static`** — глобальный объект с конкретным местом в памяти и временем жизни всей программы. ([Rust Documentation][1])
- **CTFE** — механизм вычисления допустимых выражений во время компиляции. ([Rust Documentation][2])
- **Оптимизация** — отдельное понятие: компилятор может устранить или преобразовать runtime-код, но это не следует путать с языковой гарантией compile-time evaluation.

### Самая важная идея

**Современный Rust позволяет переносить на этап компиляции не только проверку типов, владения и времён жизни, но и часть вычислений, генерацию кода и информацию о структуре программы.**

**Но цель не в том, чтобы сделать как можно больше работы на compile time. Цель — использовать compile time там, где информация уже известна, а runtime оставить для работы с реальными данными и внешним миром.**

Именно это разделение — **что известно компилятору заранее и что становится известно только во время выполнения** — является одной из фундаментальных идей проектирования программ на современном Rust.

[1]: https://doc.rust-lang.org/stable/core/keyword.static.html 'static - Rust'
[2]: https://doc.rust-lang.org/reference/const_eval.html 'Constant evaluation - The Rust Reference'
[3]: https://doc.rust-lang.org/reference/macros.html 'Macros - The Rust Reference'
[4]: https://doc.rust-lang.org/book/ch18-02-trait-objects.html 'Using Trait Objects to Abstract over Shared Behavior - The Rust Programming Language'
[5]: https://doc.rust-lang.org/reference/items/generics.html 'Generic parameters - The Rust Reference'
[6]: https://doc.rust-lang.org/beta/reference/procedural-macros.html 'Procedural macros - The Rust Reference'
[7]: https://doc.rust-lang.org/stable/core/keyword.const.html 'const - Rust'
