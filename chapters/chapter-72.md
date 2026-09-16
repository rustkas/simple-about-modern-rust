# Часть XVII. Как думает современный Rust-программист

Здесь книга начинает переходить от **«знать Rust»** к **«уметь проектировать на Rust»**.

# Глава 72. Zero-Cost Abstractions

Одна из фундаментальных идей Rust — **zero-cost abstractions**.

Идея сформулирована так:

> Вы не должны платить за абстракцию во время выполнения, если эта стоимость не требуется её семантикой.

Иными словами, высокоуровневый код не обязан быть медленнее низкоуровневого.

Rust позволяет использовать:

- generics;
- traits;
- iterators;
- closures;
- `Option`;
- `Result`;
- `async`/`await`;
- const generics;
- другие типовые абстракции,

а затем компилятор и LLVM могут преобразовать этот код в эффективный машинный код.

Но здесь важно избежать распространённого заблуждения:

> **Zero-cost abstraction не означает, что любая абстракция буквально ничего не стоит.**

Например:

- `dyn Trait` требует dynamic dispatch;
- `Box<T>` обычно означает выделение памяти в куче;
- `Arc<T>` содержит атомарный счётчик ссылок;
- `Mutex<T>` требует синхронизации;
- `Vec<T>` хранит данные в динамически выделенной памяти;
- `async` создаёт state machine и требует runtime/executor для выполнения future.

Zero-cost означает прежде всего:

> **абстракция не должна иметь дополнительной стоимости по сравнению с ручным кодом, реализующим ту же семантику.**

Все примеры этой главы используют **Rust Edition 2024**.

Для примеров, где мы хотим увидеть результат оптимизации, используется **release mode**.

---

## 72.1. Что такое Zero-Cost Abstraction?

Рассмотрим два варианта одной операции.

Высокоуровневый:

```rust
fn sum(data: &[i32]) -> i32 {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * 2)
        .sum()
}
```

И ручной:

```rust
fn sum_manual(data: &[i32]) -> i32 {
    let mut sum = 0;

    for &x in data {
        if x > 0 {
            sum += x * 2;
        }
    }

    sum
}
```

Первый вариант использует несколько абстракций:

- `Iterator`;
- `filter`;
- `map`;
- closure;
- `sum`.

На уровне исходного текста он выглядит значительно сложнее.

Но это не означает, что программа будет выполнять:

```text
создать Iterator
        ↓
вызвать filter
        ↓
создать новый Iterator
        ↓
вызвать map
        ↓
создать новый Iterator
        ↓
вызвать sum
```

В оптимизированной программе значительная часть этих абстракций может исчезнуть.

Упрощённо процесс выглядит так:

```text
┌──────────────────────────────────────────────┐
│           Высокоуровневый Rust               │
│                                              │
│  iter().filter(...).map(...).sum()           │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│             rustc + LLVM                     │
│                                              │
│  monomorphization                            │
│  inlining                                    │
│  constant propagation                        │
│  dead-code elimination                       │
│  другие оптимизации                          │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│             Машинный код                     │
│                                              │
│     эффективная последовательность           │
│     операций над данными                     │
└──────────────────────────────────────────────┘
```

Поэтому правильный вопрос не:

> «Сколько абстракций написано в исходном коде?»

а:

> **«Какая стоимость действительно остаётся после компиляции?»**

Это принципиальное изменение мышления.

---

## 72.2. Итераторы против ручных циклов

Итераторы — один из лучших примеров zero-cost abstractions в Rust.

Рассмотрим:

```rust
fn sum_with_iterators(data: &[i32]) -> i32 {
    data.iter()
        .map(|x| x * 2)
        .filter(|x| x % 2 == 0)
        .sum()
}

fn sum_with_loop(data: &[i32]) -> i32 {
    let mut sum = 0;

    for &x in data {
        let doubled = x * 2;

        if doubled % 2 == 0 {
            sum += doubled;
        }
    }

    sum
}

fn main() {
    let data = [1, 2, 3, 4, 5, 6];

    assert_eq!(
        sum_with_iterators(&data),
        sum_with_loop(&data)
    );

    println!("{}", sum_with_iterators(&data));
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=fn+sum_with_iterators%28data%3A+%26%5Bi32%5D%29+-%3E+i32+%7B%0A++++data.iter%28%29.map%28%7Cx%7C+x+%2A+2%29.filter%28%7Cx%7C+x+%25+2+%3D%3D+0%29.sum%28%29%0A%7D%0A%0Afn+sum_with_loop%28data%3A+%26%5Bi32%5D%29+-%3E+i32+%7B%0A++++let+mut+sum+%3D+0%3B%0A++++for+%26x+in+data+%7B%0A++++++++let+doubled+%3D+x+%2A+2%3B%0A++++++++if+doubled+%25+2+%3D%3D+0+%7B+sum+%2B%3D+doubled%3B+%7D%0A++++%7D%0A++++sum%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+%5B1%2C+2%2C+3%2C+4%2C+5%2C+6%5D%3B%0A++++assert_eq%21%28sum_with_iterators%28%26data%29%2C+sum_with_loop%28%26data%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+sum_with_iterators%28%26data%29%29%3B%0A%7D)

Важно не утверждать, что Rust **гарантирует идентичный машинный код**.

Это зависит от:

- версии компилятора;
- LLVM;
- target architecture;
- optimization settings;
- конкретного кода;
- информации, доступной оптимизатору.

Правильнее сказать:

> В release-сборке компилятор часто способен устранить стоимость iterator abstraction и получить машинный код, сопоставимый с ручным циклом.

Именно это и является практическим смыслом zero-cost abstraction.

---

## 72.3. Мономорфизация

**Мономорфизация** — процесс создания специализированного кода для конкретных типов, используемых с generic-кодом.

Например:

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let x = identity(42_i32);
    let y = identity(3.14_f64);

    println!("{x}, {y}");
}
```

Здесь `identity` используется с двумя типами:

```text
T = i32
T = f64
```

Концептуально компилятор получает специализированные варианты:

```rust
fn identity_i32(value: i32) -> i32 {
    value
}

fn identity_f64(value: f64) -> f64 {
    value
}
```

Это не означает, что в итоговом бинарном файле обязательно будут функции именно с такими именами. Это **модель для понимания процесса**.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=fn+identity%3CT%3E%28value%3A+T%29+-%3E+T+%7B+value+%7D%0A%0Afn+main%28%29+%7B%0A++++let+x+%3D+identity%2842_i32%29%3B%0A++++let+y+%3D+identity%283.14_f64%29%3B%0A++++println%21%28%22%7B%7D%2C+%7B%7D%22%2C+x%2C+y%29%3B%0A%7D)

Главное преимущество:

```text
generic source code
       ↓
monomorphization
       ↓
concrete code
       ↓
optimization
```

Поэтому generic-код может быть очень эффективным.

Но у этого есть и обратная сторона.

Если generic-функция используется с большим количеством разных типов, компилятор потенциально создаёт много специализированного кода.

Это может привести к:

- увеличению размера бинарного файла;
- увеличению времени компиляции;
- дополнительной работе оптимизатора.

Поэтому утверждение:

> «Generics бесплатны»

слишком грубое.

Точнее:

> **Generics обычно не требуют dynamic dispatch во время выполнения, но могут иметь стоимость на этапе компиляции и влиять на размер кода.**

---

## 72.4. Инлайнинг

**Inlining** — это оптимизация, при которой тело функции помещается непосредственно в место вызова.

Например:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let result = add(5, 3);

    println!("{result}");
}
```

Оптимизатор может преобразовать это концептуально в:

```rust
let result = 5 + 3;
```

Но это **модель**, а не обещание компилятора.

Rust-компилятор самостоятельно принимает решения об инлайнинге.

Можно дать ему подсказку:

```rust
#[inline]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Есть и:

```rust
#[inline(always)]
```

Но название может вводить в заблуждение.

`#[inline(always)]` не следует понимать как абсолютную гарантию того, что функция будет встроена во всех возможных ситуациях. Атрибут относится к оптимизации и является подсказкой компилятору. Более того, чрезмерный inlining способен увеличить размер кода и иногда ухудшить производительность. ([Rust Documentation][1])

Поэтому нормальная стратегия:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

и только если есть конкретная причина:

```rust
#[inline]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

`#[inline(always)]` следует использовать осторожно.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=%23%5Binline%5D%0Afn+add%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B+a+%2B+b+%7D%0A%0A%23%5Binline%28never%29%5D%0Afn+add_never%28a%3A+i32%2C+b%3A+i32%29+-%3E+i32+%7B+a+%2B+b+%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+add%285%2C+3%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+add_never%285%2C+3%29%29%3B%0A%7D)

---

## 72.5. Static Dispatch и Dynamic Dispatch

Rust позволяет выбирать между двумя принципиально разными способами вызова реализации trait.

### Static dispatch

```rust
trait Draw {
    fn draw(&self) -> i32;
}

struct Circle;
struct Square;

impl Draw for Circle {
    fn draw(&self) -> i32 {
        1
    }
}

impl Draw for Square {
    fn draw(&self) -> i32 {
        2
    }
}

fn draw_static<T: Draw>(item: &T) -> i32 {
    item.draw()
}
```

Здесь конкретный `T` известен при компиляции.

Это позволяет:

- выполнить monomorphization;
- оптимизировать вызов;
- потенциально выполнить inlining.

### Dynamic dispatch

Другой вариант:

```rust
fn draw_dynamic(item: &dyn Draw) -> i32 {
    item.draw()
}
```

Теперь конкретный тип за интерфейсом `dyn Draw` неизвестен вызывающему коду.

Trait object содержит:

```text
data pointer
+
vtable pointer
```

При вызове метода используется vtable. ([Rust Documentation][2])

Полный пример:

```rust
trait Draw {
    fn draw(&self) -> i32;
}

struct Circle;
struct Square;

impl Draw for Circle {
    fn draw(&self) -> i32 {
        1
    }
}

impl Draw for Square {
    fn draw(&self) -> i32 {
        2
    }
}

fn draw_static<T: Draw>(item: &T) -> i32 {
    item.draw()
}

fn draw_dynamic(item: &dyn Draw) -> i32 {
    item.draw()
}

fn main() {
    let circle = Circle;
    let square = Square;

    println!("{}", draw_static(&circle));
    println!("{}", draw_dynamic(&square));
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=trait+Draw+%7B%0A++++fn+draw%28%26self%29+-%3E+i32%3B%0A%7D%0A%0Astruct+Circle%3B%0Astruct+Square%3B%0A%0Aimpl+Draw+for+Circle+%7B%0A++++fn+draw%28%26self%29+-%3E+i32+%7B+1+%7D%0A%7D%0A%0Aimpl+Draw+for+Square+%7B%0A++++fn+draw%28%26self%29+-%3E+i32+%7B+2+%7D%0A%7D%0A%0Afn+draw_static%3CT%3A+Draw%3E%28item%3A+%26T%29+-%3E+i32+%7B+item.draw%28%29+%7D%0Afn+draw_dynamic%28item%3A+%26dyn+Draw%29+-%3E+i32+%7B+item.draw%28%29+%7D%0A%0Afn+main%28%29+%7B%0A++++let+circle+%3D+Circle%3B%0A++++let+square+%3D+Square%3B%0A++++println%21%28%22%7B%7D%22%2C+draw_static%28%26circle%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+draw_dynamic%28%26square%29%29%3B%0A%7D)

Сравним:

| Характеристика              | Static dispatch         | Dynamic dispatch |
| --------------------------- | ----------------------- | ---------------- |
| Когда выбирается реализация | Compile time            | Runtime          |
| Основной механизм           | Generics / `impl Trait` | `dyn Trait`      |
| Vtable                      | Нет                     | Да               |
| Возможность inlining        | Обычно выше             | Обычно ниже      |
| Размер кода                 | Может увеличиваться     | Часто меньше     |
| Гибкость                    | Ниже                    | Выше             |

Но очень важно не превращать это в правило:

> `dyn Trait` всегда медленнее.

Правильнее:

> **Dynamic dispatch имеет дополнительную косвенность и обычно ограничивает возможности оптимизатора, но уменьшение размера кода и архитектурная гибкость могут быть более важны.**

Например, plugin architecture часто естественно строится именно на `dyn Trait`.

---

## 72.6. Compile-Time Selection без Specialization

В исходном варианте этой главы использовался marker-trait пример как будто он реализует **specialization**.

Это некорректно.

Marker traits могут использоваться для ограничения generic-кода, но они не превращают обычный Rust-код в механизм specialization.

В стабильном Rust нет общего механизма specialization, при котором можно было бы написать:

```rust
impl<T> Trait for T {
    // generic implementation
}

impl Trait for i32 {
    // более специализированная implementation
}
```

и ожидать, что компилятор автоматически выберет вторую реализацию для `i32`.

Обычная модель Rust основана на правилах coherence и отсутствии конфликтующих implementations.

Но compile-time выбор можно получать другими способами.

### Generics

```rust
fn process<T: AsRef<str>>(value: T) -> usize {
    value.as_ref().len()
}
```

Конкретный тип известен при инстанцировании generic-функции.

### Associated types

```rust
trait Parser {
    type Output;

    fn parse(&self) -> Self::Output;
}
```

Associated type позволяет связать реализацию trait с конкретным типом результата.

### Const generics

Ещё один важный механизм compile-time параметризации:

```rust
struct Buffer<const N: usize> {
    data: [u8; N],
}

impl<const N: usize> Buffer<N> {
    fn len(&self) -> usize {
        N
    }
}

fn main() {
    let buffer = Buffer::<1024> {
        data: [0; 1024],
    };

    println!("{}", buffer.len());
}
```

Здесь `N` является частью типа.

```text
Buffer<1024>
Buffer<2048>
Buffer<4096>
```

— это разные конкретные типы.

Это позволяет переносить часть решений с runtime на compile time.

Главная идея:

> **Современный Rust часто решает задачу specialization не через runtime branching, а через систему типов, generics и const generics.**

---

## 72.7. Zero-cost в действии: `Option` и `Result`

`Option` и `Result` — прекрасный пример того, как высокоуровневое представление может эффективно компилироваться.

Рассмотрим:

```rust
fn value_or_zero(value: Option<i32>) -> i32 {
    value.unwrap_or(0)
}

fn main() {
    assert_eq!(value_or_zero(Some(42)), 42);
    assert_eq!(value_or_zero(None), 0);
}
```

Здесь логически существует проверка:

```text
Option
 ├── Some(value) → value
 └── None        → 0
```

Компилятор не обязан хранить эту проверку в каком-то буквально таком виде.

При конкретных условиях оптимизатор может:

- удалить невозможные ветви;
- распространить известные значения;
- упростить control flow;
- встроить функцию;
- преобразовать представление данных.

Но нельзя писать:

> `unwrap_or` всегда превращается в простое присваивание.

Если значение `Option` неизвестно во время компиляции, проверка может остаться в машинном коде.

Поэтому правильный принцип:

> **Высокоуровневая конструкция может быть оптимизирована до очень дешёвого кода, но это определяется конкретной программой и возможностями оптимизатора.**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=fn+value_or_zero%28value%3A+Option%3Ci32%3E%29+-%3E+i32+%7B%0A++++value.unwrap_or%280%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++assert_eq%21%28value_or_zero%28Some%2842%29%29%2C+42%29%3B%0A++++assert_eq%21%28value_or_zero%28None%29%2C+0%29%3B%0A%7D)

---

## 72.8. Итераторы, closures и композиция

Zero-cost подход особенно хорошо виден при композиции iterator operations.

```rust
fn process(data: &[i32]) -> i32 {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * 2)
        .sum()
}
```

Это выражает **что** мы хотим сделать:

```text
взять элементы
    ↓
оставить положительные
    ↓
удвоить
    ↓
сложить
```

А не **как именно** вручную организовать каждый шаг.

Эквивалентный ручной вариант:

```rust
fn process_manual(data: &[i32]) -> i32 {
    let mut sum = 0;

    for &x in data {
        if x > 0 {
            sum += x * 2;
        }
    }

    sum
}
```

При хорошей оптимизации iterator pipeline может быть преобразован в машинный код, не содержащий отдельных объектов `Filter`, `Map` и других runtime-абстракций.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=release&edition=2024&code=fn+process%28data%3A+%26%5Bi32%5D%29+-%3E+i32+%7B%0A++++data.iter%28%29.filter%28%7C%26%26x%7C+x+%3E+0%29.map%28%7C%26x%7C+x+%2A+2%29.sum%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+%5B-1%2C+2%2C+3%2C+-4%2C+5%5D%3B%0A++++println%21%28%22%7B%7D%22%2C+process%28%26data%29%29%3B%0A%7D)

То же относится к closures.

Например:

```rust
fn apply_twice<T, F>(value: T, f: F) -> T
where
    F: Fn(T) -> T + Copy,
{
    f(f(value))
}

fn main() {
    let result = apply_twice(3, |x| x * 2);

    assert_eq!(result, 12);
}
```

Closure здесь не обязательно означает выделение памяти в куче или вызов через function pointer.

Если тип closure известен статически, он является конкретным типом и может быть обработан компилятором как обычная часть generic-кода.

---

## 72.9. Где абстракция действительно имеет стоимость?

Теперь мы можем сформулировать более точную таблицу.

| Абстракция            | Возможная стоимость                                                          |
| --------------------- | ---------------------------------------------------------------------------- |
| Generics              | Обычно runtime dispatch отсутствует; возможны code size и compile-time costs |
| Static trait dispatch | Обычно нет отдельного runtime dispatch                                       |
| `dyn Trait`           | Vtable/dynamic dispatch                                                      |
| Iterators             | Часто устраняются оптимизатором                                              |
| Closures              | Часто устраняются/инлайнятся                                                 |
| `Box<T>`              | Heap allocation, если объект действительно выделяется динамически            |
| `Vec<T>`              | Heap allocation / reallocation                                               |
| `Rc<T>`               | Reference counting                                                           |
| `Arc<T>`              | Atomic reference counting                                                    |
| `Mutex<T>`            | Synchronization                                                              |
| `async`/`await`       | State machine + стоимость executor/runtime; зависит от программы             |

Особенно важно понимать `async`.

Нельзя говорить:

> `async` — почти бесплатная абстракция.

`async fn` преобразуется в future, представляющий состояние асинхронной операции. Это позволяет эффективно реализовывать большое количество concurrent операций, но у самой модели есть стоимость.

В зависимости от программы это может включать:

- размер future;
- хранение состояния между `await`;
- polling;
- executor;
- wakeups;
- allocation, если future помещается в `Box` или другую динамическую структуру.

Поэтому правильный вопрос:

> **Какая стоимость нужна данной семантике и какая дополнительная стоимость была устранена компилятором?**

---

## 72.10. Zero-cost не означает "без стоимости"

Это одна из самых важных оговорок всей главы.

Рассмотрим:

```rust
let value = Box::new(42);
```

Невозможно просто сказать:

> «Box — zero-cost abstraction».

`Box<T>` представляет ownership значения, находящегося в heap.

Само размещение значения в куче имеет стоимость.

Но `Box<T>` всё равно может считаться частью zero-cost philosophy Rust в более широком смысле:

- язык не добавляет скрытый garbage collector;
- стоимость heap allocation возникает потому, что **программа действительно запросила heap allocation**;
- владение и освобождение памяти выполняются предсказуемо.

Другой пример:

```rust
let value = Arc::new(42);
```

Здесь есть reference counting.

Это не "лишняя стоимость Rust".

Это стоимость выбранной семантики shared ownership.

Поэтому современное понимание zero-cost abstractions можно сформулировать так:

> **Rust не обещает, что высокоуровневые конструкции ничего не стоят. Rust стремится сделать так, чтобы вы не платили за абстракции, которые не требуют runtime-механизма для своей реализации.**

---

## 72.11. Как убедиться, что абстракция действительно дешёвая?

Нельзя определить стоимость программы только по исходному коду.

Нужно измерять.

### 1. Benchmark

Если вы считаете, что два варианта эквивалентны, измерьте их.

Например:

```rust
fn iterator_version(data: &[i32]) -> i32 {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * 2)
        .sum()
}

fn loop_version(data: &[i32]) -> i32 {
    let mut sum = 0;

    for &x in data {
        if x > 0 {
            sum += x * 2;
        }
    }

    sum
}
```

Важно использовать корректный benchmark framework и не позволять компилятору удалить вычисления, которые benchmark должен измерять.

Для серьёзных измерений обычно используют `criterion`.

### 2. Смотреть assembly

Если вопрос:

> «Во что превратился этот Rust-код?»

то assembly — один из лучших инструментов.

Можно использовать, например, `cargo-show-asm`:

```bash
cargo install cargo-show-asm
```

После установки команда обычно используется через Cargo:

```bash
cargo asm
```

Конкретная форма команды зависит от версии инструмента и структуры crate.

### 3. Использовать Compiler Explorer

Для изучения code generation очень полезен Compiler Explorer:

```text
Rust source
    ↓
rustc
    ↓
LLVM IR / assembly
```

Это особенно удобно, когда нужно сравнить:

```text
iterator
vs
manual loop
```

или:

```text
generic
vs
dyn Trait
```

### 4. Profiling

Если проблема находится в реальном приложении, assembly одной функции может быть недостаточно.

Нужно определить:

- где программа проводит время;
- где происходят allocations;
- где возникают lock contention;
- где появляются cache misses;
- какие функции являются hot spots.

И только после этого оптимизировать.

---

## 72.12. Release mode имеет значение

Многие оптимизации, о которых мы говорим в этой главе, особенно важны для release-сборки.

Например:

```bash
cargo run
```

обычно запускает debug build.

А:

```bash
cargo run --release
```

использует release profile.

Поэтому эксперимент:

```text
debug binary
```

не следует автоматически использовать для выводов о производительности production-кода.

Однако и `--release` не означает:

> «Теперь результат гарантированно оптимален».

Конкретный результат зависит от:

- target;
- CPU;
- Rust compiler;
- LLVM;
- profile settings;
- LTO;
- codegen units;
- panic strategy;
- конкретного исходного кода.

Поэтому benchmark должен быть частью процесса.

---

## 72.13. Практические выводы

### 1. Не бойтесь высокоуровневого Rust-кода

Если iterator pipeline выражает задачу лучше:

```rust
data.iter()
    .filter(...)
    .map(...)
    .sum()
```

не нужно автоматически заменять его ручным циклом только из страха перед производительностью.

### 2. Не оптимизируйте исходный код вместо результата

Плохая привычка:

> «Здесь три iterator adapters, значит это медленно».

Правильная:

> «Есть ли здесь измеримая проблема производительности?»

### 3. Предпочитайте static dispatch, когда он действительно нужен

Generics часто дают оптимизатору больше информации:

```rust
fn process<T: Processor>(processor: &T) {
    processor.process();
}
```

Но это не означает, что `dyn Trait` плох.

Если архитектуре требуется runtime polymorphism:

```rust
fn process(processor: &dyn Processor) {
    processor.process();
}
```

это вполне нормальное решение.

### 4. Используйте dynamic dispatch осознанно

`dyn Trait` полезен, когда нужны:

- plugins;
- heterogeneous collections;
- runtime-selected implementations;
- уменьшение code duplication;
- более стабильные границы архитектуры.

### 5. Не злоупотребляйте `#[inline(always)]`

Пусть компилятор сначала сам принимает решение.

### 6. Измеряйте

Основной цикл оптимизации:

```text
Hypothesis
    ↓
Benchmark / Profile
    ↓
Optimization
    ↓
Benchmark / Profile
    ↓
Accept or revert
```

а не:

```text
Looks slow
    ↓
Rewrite everything
```

---

## 72.14. Zero-Cost Abstractions и проектирование API

Теперь становится понятна связь этой главы с предыдущей — **Designing Library APIs**.

Хороший API Rust-библиотеки должен одновременно решать две задачи:

```text
удобство пользователя
        +
возможность эффективной компиляции
```

Например, generic API:

```rust
pub fn process<T: AsRef<[u8]>>(data: T) -> usize {
    data.as_ref().len()
}
```

позволяет использовать разные типы, сохраняя статическую информацию о конкретном `T`.

Но иногда API лучше построить через trait object:

```rust
pub trait Storage {
    fn read(&self) -> &[u8];
}

pub fn process(storage: &dyn Storage) -> usize {
    storage.read().len()
}
```

Теперь библиотека получает больше runtime-гибкости.

Нельзя сказать, что один вариант **всегда правильный**.

Современный Rust-программист должен уметь задавать вопросы:

- Нужен ли runtime polymorphism?
- Нужно ли уменьшить размер кода?
- Важен ли inlining?
- Будет ли API использоваться с большим количеством типов?
- Нужна ли heterogeneous collection?
- Где находится hot path?
- Есть ли вообще измеримая проблема?

И только после этого выбирать между:

```text
generics
impl Trait
dyn Trait
associated types
const generics
concrete types
```

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: Iterator vs loop

Создайте две функции:

```rust
fn iterator_version(data: &[i32]) -> i32 {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * 2)
        .sum()
}

fn loop_version(data: &[i32]) -> i32 {
    let mut sum = 0;

    for &x in data {
        if x > 0 {
            sum += x * 2;
        }
    }

    sum
}
```

Скомпилируйте программу в release mode и попробуйте посмотреть assembly.

**Вопрос:**

> Удалось ли компилятору устранить промежуточные iterator abstractions?

---

## Эксперимент 2: Generic function

Используйте:

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let a = identity(10_i32);
    let b = identity(20_i64);

    println!("{a} {b}");
}
```

**Вопрос:**

> Какие конкретные специализации generic-функции потребовались?

---

## Эксперимент 3: Static vs dynamic dispatch

Сравните:

```rust
fn use_static<T: Trait>(value: &T) {
    value.method();
}

fn use_dynamic(value: &dyn Trait) {
    value.method();
}
```

Посмотрите assembly.

Обратите внимание не только на количество инструкций, но и на:

- indirect call;
- inlining;
- размер функции;
- количество специализированного кода.

**Важно:** не делайте вывод «dynamic dispatch всегда медленнее» только по одному маленькому примеру.

---

## Эксперимент 4: `#[inline]`

Сравните:

```rust
#[inline]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

с:

```rust
#[inline(never)]
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Посмотрите на generated assembly.

**Вопрос:**

> Видите ли вы разницу?

---

## Эксперимент 5: `Option`

Сравните:

```rust
fn get(value: Option<i32>) -> i32 {
    value.unwrap_or(0)
}
```

с:

```rust
fn get_manual(value: Option<i32>) -> i32 {
    match value {
        Some(value) => value,
        None => 0,
    }
}
```

Посмотрите, насколько близким оказывается generated code.

Здесь цель эксперимента не доказать:

> "`unwrap_or` всегда бесплатен".

Цель:

> **увидеть, что высокоуровневая конструкция может компилироваться в очень простой control flow.**

---

# Практика

## Задание 1

Сравните iterator implementation и ручной цикл.

Не ограничивайтесь чтением исходного кода.

Проверьте:

1. benchmark;
2. release assembly;
3. размер бинарного файла.

---

## Задание 2

Напишите generic-функцию:

```rust
fn process<T>(value: T) -> T {
    value
}
```

Используйте её с несколькими типами.

Исследуйте, что происходит после monomorphization.

---

## Задание 3

Напишите две версии обработки коллекции:

```text
Iterator API
```

и:

```text
for loop
```

Сравните generated assembly.

---

## Задание 4

Создайте trait:

```rust
trait Processor {
    fn process(&self, value: i32) -> i32;
}
```

Реализуйте:

```text
generic/static version
dynamic/dyn version
```

Сравните их.

---

## Задание 5

Используйте `#[inline]` и `#[inline(never)]`.

Посмотрите, как это влияет на generated assembly.

Не пытайтесь сделать вывод о реальной производительности только по размеру функции.

---

## Задание 6

🔨 **Эксперимент с компилятором.**

Создайте функцию:

```rust
fn process(value: Option<i32>) -> i32 {
    value.unwrap_or(0)
}
```

Сравните её с `match`.

Определите:

> одинаков ли generated code для этих двух реализаций?

---

## Задание 7

🔨 **Эксперимент с компилятором.**

Создайте generic-функцию, которая вызывается с большим количеством различных типов.

Исследуйте:

- размер бинарного файла;
- время компиляции;
- количество специализированного кода.

Сделайте вывод:

> Почему zero-cost runtime abstraction всё равно может иметь compile-time cost?

---

# Главное из этой главы

После этой главы мы понимаем:

- **Zero-cost abstraction** — отсутствие дополнительной runtime-стоимости по сравнению с эквивалентной ручной реализацией, когда такая стоимость не требуется семантикой.
- **Мономорфизация** — создание специализированного generic-кода для конкретных типов.
- **Inlining** — оптимизация, позволяющая устранить часть стоимости вызова функции и открыть дополнительные возможности оптимизации.
- **Static dispatch** — выбор конкретной реализации на основе статически известного типа.
- **Dynamic dispatch** — выбор реализации во время выполнения через trait object и vtable. ([Rust Documentation][2])
- **Iterators и closures** часто могут компилироваться без существенной дополнительной стоимости.
- **Generics** не являются абсолютно бесплатными: возможны увеличение размера кода и стоимости компиляции.
- **`dyn Trait`** не является «плохим» вариантом: он предоставляет runtime polymorphism и может уменьшать размер кода. ([Rust Documentation][2])
- **`Option` и `Result`** не означают автоматически отсутствие runtime-проверок — оптимизатор удаляет их только тогда, когда это возможно.
- **`async`**, heap allocation и synchronization имеют реальную стоимость, связанную с их семантикой.
- **Assembly и benchmarks** позволяют проверить предположения о производительности.
- **Release build** необходим для осмысленного анализа оптимизированного кода.

## Самая важная идея

> **Zero-cost abstraction — это не обещание, что Rust-код ничего не стоит. Это принцип: абстракция не должна заставлять вас платить за механизм, который ей не нужен.**
>
> Rust позволяет писать код на высоком уровне — через generics, traits, iterators, closures, `Option`, `Result` и другие конструкции — и при этом сохранять возможность получить эффективный машинный код.
>
> Но современный Rust-программист не верит в это на слово. Он **измеряет**.
>
> Он не спрашивает:
>
> > «Этот код выглядит высокоуровневым — значит, он медленный?»
>
> Он спрашивает:
>
> > **«Какую стоимость имеет эта абстракция после компиляции и действительно ли эта стоимость является проблемой?»**
>
> Именно это и есть переход от **«писать код на Rust»** к **«проектировать эффективные системы на Rust»**.

[1]: https://doc.rust-lang.org/beta/core/attribute.inline.html 'inline - Rust'
[2]: https://doc.rust-lang.org/std/keyword.dyn.html 'dyn - Rust'
